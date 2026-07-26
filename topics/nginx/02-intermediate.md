# Nginx — Intermediate Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Nginx basics (server blocks, locations, reverse proxy)  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Load Balancing
2. Caching
3. Rate Limiting
4. SSL/TLS Configuration
5. Compression
6. Logging
7. Rewrite Rules
8. Q&A

---

## 1. Load Balancing

### Upstream block

```nginx
upstream api_servers {
    # Load balancing algorithm (default: round-robin)
    least_conn;  # or ip_hash, random, hash $request_uri

    server 10.0.1.10:8080 weight=3;
    server 10.0.1.11:8080 weight=1;
    server 10.0.1.12:8080 backup;  # only used if others are down

    keepalive 32;  # max idle connections kept for reuse
}
```

### Algorithms

| Algorithm | Description | Use case |
|-----------|-------------|----------|
| **Round-robin** | Default. Each server in turn | Equal capacity servers |
| **least_conn** | Route to server with fewest connections | Unequal load, variable request times |
| **ip_hash** | Consistent hash of client IP | Sticky sessions (no ALB stickiness) |
| **hash $request_uri** | Hash of any variable | Cache affinity |
| **random** | Random with two choices | Large server pools |

### Health checks

```nginx
upstream backend {
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
}
```

Nginx marks a server as down after `max_fails` failures within `fail_timeout`. After `fail_timeout`, Nginx tries again.

### Passive vs Active health checks

- **Passive** (default): Nginx detects failures based on response codes/timeouts
- **Active** (Nginx Plus): Nginx periodically pings the server

### Slow start

```nginx
upstream backend {
    server 10.0.1.10:8080 slow_start=30s;  # gradually increase weight after recovery
}
```

Prevents thundering herd when a server comes back online.

---

## 2. Caching

### proxy_cache (for reverse proxy)

```nginx
# Define cache zone
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m
                 max_size=10g inactive=60m use_temp_path=off;

server {
    location / {
        proxy_cache api_cache;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        proxy_cache_valid 200 302 60m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503;

        # Bypass cache with cookie/header
        proxy_cache_bypass $http_cache_control;

        # Add header showing cache status
        add_header X-Cache-Status $upstream_cache_status;

        proxy_pass http://backend;
    }
}
```

### Cache status values

| Value | Meaning |
|-------|---------|
| `HIT` | Served from cache |
| `MISS` | Not in cache, fetched from backend |
| `EXPIRED` | Cache entry expired, backend fetched |
| `STALE` | Served stale content (upstream down) |
| `BYPASS` | Request bypassed cache |
| `UPDATING` | Cache entry is stale and being updated |

### fastcgi_cache (for PHP/Laravel)

```nginx
fastcgi_cache_path /var/cache/nginx/fastcgi levels=1:2 keys_zone=php_cache:10m
                   max_size=1g inactive=60m;

server {
    location ~ \.php$ {
        fastcgi_cache php_cache;
        fastcgi_cache_key "$scheme$request_method$host$request_uri";
        fastcgi_cache_valid 200 60m;

        # Don't cache based on cookie (bypass for logged-in users)
        fastcgi_cache_bypass $cookie_session;
        fastcgi_no_cache $cookie_session;

        add_header X-Cache-Status $upstream_cache_status;

        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }
}
```

### Microcaching (1 second)

```nginx
# Cache for 1 second — absorbs traffic spikes without stale content
fastcgi_cache_valid 200 1s;
fastcgi_cache_use_stale updating;
```

The `updating` parameter serves a stale cache entry while the cache is being refreshed. This prevents the "thundering herd" problem (many requests all hitting PHP-FPM when cache expires).

### Cache purging

Nginx doesn't have built-in purge. Use a module (nginx-cache-purge) or:

```bash
# Manual purge — delete cache files
rm -rf /var/cache/nginx/*

# Or use a location to purge via API
location ~ ^/purge(/.*) {
    allow 127.0.0.1;
    deny all;
    proxy_cache_purge api_cache $1;
}
```

---

## 3. Rate Limiting

### Request rate limit

```nginx
# Define limit zone
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=30r/s;

server {
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        # burst=20: allow up to 20 requests over the rate in a short burst
        # nodelay: process burst immediately (no delay)
        proxy_pass http://backend;
    }
}
```

Without `nodelay`: excess requests are delayed (queued). With `nodelay`: excess requests are processed immediately but within the burst limit. Requests exceeding `rate + burst` are returned `503`.

### Connection rate limit

```nginx
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

server {
    limit_conn conn_limit 10;  # max 10 concurrent connections per IP
}
```

### Two-stage rate limiting

```nginx
# First: 30 req/s burst 20
# Second: after first limit, 10 req/s
limit_req_zone $binary_remote_addr zone=first:10m rate=30r/s;
limit_req_zone $binary_remote_addr zone=second:10m rate=10r/s;

location /api/ {
    limit_req zone=first burst=20 nodelay;
    limit_req zone=second burst=10;
    proxy_pass http://backend;
}
```

### Rate limiting by header (API keys)

```nginx
# Rate limit per API key instead of IP
map $http_x_api_key $limit_key {
    default $binary_remote_addr;
    ~(.+) $http_x_api_key;
}

limit_req_zone $limit_key zone=api_key_limit:10m rate=100r/s;
```

---

## 4. SSL/TLS Configuration

### Basic SSL server block

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HTTP → HTTPS redirect
    error_page 497 =301 https://$host$request_uri;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### SSL best practices

```nginx
# Modern SSL config
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers off;  # Let client choose (preferred for modern)

# OCSP Stapling (improves TLS handshake performance)
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 1.1.1.1 valid=300s;
resolver_timeout 5s;

# HSTS (Strict Transport Security)
add_header Strict-Transport-Security "max-age=63072000" always;

# Session cache
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 4h;
ssl_session_tickets off;  # Disable session tickets (security)
```

### Redirect HTTP to HTTPS

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

---

## 5. Compression

```nginx
# Enable gzip
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;           # 1-9 (higher = more compression, more CPU)
gzip_min_length 256;         # Don't compress small responses
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    image/svg+xml;

# Brotli (if module is installed)
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/javascript;
```

> **Trap:** Gzip compression level 6 is the sweet spot. Level 9 saves ~2% more space but uses 3x more CPU. Don't compress images (already compressed) or very small responses (< 256 bytes).

---

## 6. Logging

### Custom log format

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" $request_time $upstream_response_time';

log_format json escape=json '{'
    '"time": "$time_local",'
    '"remote_addr": "$remote_addr",'
    '"request": "$request",'
    '"status": $status,'
    '"body_bytes": $body_bytes_sent,'
    '"request_time": $request_time,'
    '"upstream_time": "$upstream_response_time",'
    '"http_x_forwarded_for": "$http_x_forwarded_for"'
'}';

access_log /var/log/nginx/access.log main;
error_log  /var/log/nginx/error.log warn;
```

### Conditional logging

```nginx
# Don't log health checks
map $uri $loggable {
    /health 0;
    default 1;
}
access_log /var/log/nginx/access.log main if=$loggable;

# Don't log static assets
location ~* \.(jpg|png|css|js)$ {
    access_log off;
}
```

### Log rotation

```nginx
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

---

## 7. Rewrite Rules

### rewrite vs return

```nginx
# rewrite — internal URI rewrite (processing continues)
rewrite ^/old-path/(.*)$ /new-path/$1 permanent;   # 301 redirect
rewrite ^/users/(\d+)$ /index.php?user_id=$1 last;  # internal rewrite

# return — returns response immediately (faster)
return 301 https://$server_name$request_uri;
return 404;
```

**Use `return` instead of `rewrite` when possible** — `return` is faster (Nginx stops processing).

### Common rewrite patterns

```nginx
# Force trailing slash
rewrite ^([^.]*[^/])$ $1/ permanent;

# Remove index.php from URL
if ($request_uri ~* "index\.php") {
    rewrite ^(.*)index\.php$ $1 permanent;
}

# Canonical domain (www → non-www)
server_name ~^(www\.)?(.+)$;
if ($host ~* ^www\.(.+)) {
    return 301 $scheme://$1$request_uri;
}

# HTTPS redirect
if ($scheme != "https") {
    return 301 https://$host$request_uri;
}
```

---

## 8. Q&A

**Q: What load balancing algorithms does Nginx support?**
A: Round-robin (default), least_conn, ip_hash, hash, random.

**Q: How does Nginx cache work?**
A: `proxy_cache_path` defines cache storage. `proxy_cache_key` defines what makes a cache entry unique. `proxy_cache_valid` sets TTL per status code.

**Q: What's microcaching?**
A: Caching for very short durations (1s). Absorbs traffic spikes. `fastcgi_cache_use_stale updating` serves stale content while the cache refreshes.

**Q: How do you rate limit with Nginx?**
A: `limit_req_zone` defines rate + storage. `limit_req` applies the limit to a location. `burst` allows short spikes. `nodelay` processes bursts immediately.

**Q: What's the difference between `rewrite` and `return`?**
A: `rewrite` changes URI and continues processing. `return` sends a response immediately. Use `return` when possible — it's faster.

**Q: How do you configure SSL in Nginx?**
A: `ssl_certificate` and `ssl_certificate_key`. Set `ssl_protocols TLSv1.2 TLSv1.3`. Enable HSTS, OCSP stapling, and session cache.

**Q: What's `$upstream_cache_status`?**
A: Variable showing cache status: HIT, MISS, EXPIRED, STALE, BYPASS, UPDATING. Useful for debugging caching behavior.
