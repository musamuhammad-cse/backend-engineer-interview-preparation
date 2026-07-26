# Nginx — Question Bank

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Type:** 120+ rapid-fire Q&A, debugging scenarios, config puzzles

---

## Table of Contents

1. Rapid-Fire Q&A (120+ questions)
2. Debugging Scenarios
3. Config Puzzles

---

## 1. Rapid-Fire Q&A

### Architecture & Basics

**Q: What's the Nginx process model?**
A: Master process (config, management) + worker processes (connection handling via event loop).

**Q: How does Nginx handle concurrent connections?**
A: Event-driven (epoll/kqueue). Single-threaded workers handle thousands of connections.

**Q: What's the difference between Nginx and Apache?**
A: Nginx: event-driven, process-per-core, high concurrency. Apache: process/thread-per-connection, .htaccess, dynamic module loading.

**Q: What's the default Nginx config file location?**
A: `/etc/nginx/nginx.conf` on most Linux distributions.

**Q: What's the difference between `sites-available` and `sites-enabled`?**
A: `sites-available` = all configs. `sites-enabled` = symlinks to active configs. Nginx includes `sites-enabled/*`.

**Q: How do you reload Nginx without dropping connections?**
A: `nginx -s reload`. Master sends SIGHUP to workers, starts new workers with new config, old workers finish current connections then exit.

**Q: What's the graceful shutdown process?**
A: `nginx -s quit`. Workers finish current connections then exit. `nginx -s stop` kills immediately.

### Server & Location Blocks

**Q: How does Nginx match server blocks?**
A: Based on `listen` IP/port + `Host` header. First match wins.

**Q: What's the default server?**
A: First server block or the one with `default_server` flag. Catches requests that don't match any `server_name`.

**Q: How does Nginx evaluate location blocks?**
A: Exact match (=) → prefix with ^~ → regex (~/~*) → longest prefix match.

**Q: What does `try_files $uri $uri/ /index.php?$query_string` do?**
A: Try exact URI, try directory, fall back to index.php with query string. Standard Laravel pattern.

**Q: What's `alias` vs `root`?**
A: `root` prepends path to URI. `alias` replaces the matched location prefix. Use `alias` for different filesystem paths per location.

### Reverse Proxy

**Q: What does `proxy_pass http://backend;` do?**
A: Forwards requests to the backend server. Response is streamed back to client.

**Q: What headers should a reverse proxy set?**
A: Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto.

**Q: How do you proxy WebSocket connections?**
A: Set `proxy_http_version 1.1`, `proxy_set_header Upgrade $http_upgrade`, `proxy_set_header Connection "upgrade"`.

**Q: What's `proxy_redirect`?**
A: Rewrites Location/Refresh headers from backend response. Use when backend redirects to internal URLs.

**Q: Can Nginx proxy requests to multiple backends?**
A: Yes. Use `upstream` block with multiple servers. Nginx load-balances automatically.

### Load Balancing

**Q: What load balancing algorithms does Nginx support?**
A: Round-robin, least_conn, ip_hash, hash, random.

**Q: How does `ip_hash` work?**
A: Hash of client IP determines which server receives the request. Ensures same client hits same server (sticky sessions).

**Q: What's `least_conn`?**
A: Routes to server with fewest active connections. Best when requests have variable processing times.

**Q: What's `backup` in upstream?**
A: Server only used when all primary servers are down.

**Q: What's `max_fails` and `fail_timeout`?**
A: `max_fails` failed attempts within `fail_timeout` seconds marks server as down. Nginx retries after `fail_timeout`.

### Caching

**Q: How do you enable caching in Nginx?**
A: `proxy_cache_path` (define storage) + `proxy_cache` (enable in location).

**Q: What's `proxy_cache_key`?**
A: Defines what makes a cache entry unique. Default: `$scheme$proxy_host$request_uri`.

**Q: What's `proxy_cache_use_stale`?**
A: Serve stale cached content when backend is down or updating. Parameters: error, timeout, updating, http_500.

**Q: What's the `updating` parameter?**
A: Serves stale content while the cache is being updated. Prevents thundering herd.

**Q: What's `$upstream_cache_status`?**
A: Variable with cache status: HIT, MISS, EXPIRED, STALE, BYPASS, UPDATING.

**Q: What's microcaching?**
A: Very short cache TTL (1s). Absorbs traffic spikes. Responses are at most 1 second stale.

### Rate Limiting

**Q: How do you rate limit in Nginx?**
A: `limit_req_zone` defines rate and storage. `limit_req` applies to location.

**Q: What's `burst` in rate limiting?**
A: Allow up to N excess requests over the rate limit (queued temporarily).

**Q: What's `nodelay` in rate limiting?**
A: Process burst requests immediately (no queuing delay). Requests beyond rate + burst are rejected.

**Q: How do you rate limit by API key instead of IP?**
A: Use `map` to extract API key from header into a variable. Use that variable in `limit_req_zone`.

**Q: What's `limit_conn`?**
A: Limits concurrent connections per IP (or other key). Different from request rate.

### SSL

**Q: What SSL protocols should you use?**
A: TLSv1.2 and TLSv1.3 only. Disable SSLv3, TLSv1.0, TLSv1.1.

**Q: What's HSTS?**
A: Strict-Transport-Security header. Tells browsers to always use HTTPS for this domain.

**Q: What's OCSP stapling?**
A: Server fetches OCSP certificate status and sends it during TLS handshake. Browser doesn't need to contact CA separately. Improves performance.

**Q: How do you redirect HTTP to HTTPS?**
A: Separate server block listening on port 80, return 301 https://$host$request_uri.

### Rewrite & Redirect

**Q: What's the difference between `rewrite` and `return`?**
A: `rewrite` changes URI and continues processing (internal). `return` sends response immediately.

**Q: What's the difference between `last` and `break` in rewrite?**
A: `last` stops current location and searches for new location. `break` stops processing within current location.

**Q: How do you redirect www to non-www?**
A: `server_name ~^(www\.)?(.+)$; if ($host ~* ^www\.(.+)) { return 301 $scheme://$1$request_uri; }`

### Security

**Q: How do you hide Nginx version?**
A: `server_tokens off;`

**Q: How do you limit request body size?**
A: `client_max_body_size 10M;`

**Q: How do you block hidden files?**
A: `location ~ /\. { deny all; }`

**Q: How do you block access to `.env` files?**
A: `location ~ \.env { deny all; return 404; }`

**Q: What security headers should you set?**
A: X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Referrer-Policy, Permissions-Policy, Strict-Transport-Security.

### Logging

**Q: Where are Nginx logs stored?**
A: Default: `/var/log/nginx/access.log` and `/var/log/nginx/error.log`.

**Q: How do you create a custom log format?**
A: `log_format myformat '$remote_addr - $remote_user [$time_local] "$request" $status';`

**Q: How do you disable access logging for health checks?**
A: Use `map` to set a variable based on URI. `access_log /var/log/nginx/access.log main if=$loggable;`

### Performance

**Q: What's `sendfile`?**
A: Zero-copy file transfer. Serves files from disk to socket without copying through userspace.

**Q: What's `tcp_nopush`?**
A: Optimizes packet sending. Works with sendfile to send headers + file in one packet.

**Q: What's `tcp_nodelay`?**
A: Disables Nagle's algorithm. Sends data immediately. Critical for interactive apps.

**Q: What's `worker_rlimit_nofile`?**
A: Max open file descriptors per worker. Must match system ulimit.

**Q: How many connections can Nginx handle?**
A: `worker_processes × worker_connections`. 4 workers × 1024 connections = 4096. With keepalive, effectively more.

### PHP-FPM

**Q: How does Nginx communicate with PHP-FPM?**
A: Via `fastcgi_pass` (Unix socket or TCP). Nginx passes environment variables via `fastcgi_param`.

**Q: What's SCRIPT_FILENAME?**
A: FastCGI parameter telling PHP-FPM which file to execute. Usually `$document_root$fastcgi_script_name`.

**Q: What's the Laravel Nginx pattern?**
A: `try_files $uri $uri/ /index.php?$query_string` — serve static files directly, route everything else through index.php.

**Q: How do you increase the upload size for PHP?**
A: `client_max_body_size 10M` in Nginx AND `upload_max_filesize` and `post_max_size` in PHP.

### General

**Q: What's Nginx Plus?**
A: Commercial version with active health checks, session persistence, status dashboard, API gateway features.

**Q: What's OpenResty?**
A: Nginx with embedded LuaJIT. Run Lua code in Nginx for API gateway logic, dynamic config.

**Q: What's the difference between Nginx and Caddy?**
A: Caddy has auto HTTPS (Let's Encrypt), simpler config, but less flexible and lower performance for high traffic.

**Q: What's the difference between Nginx and Traefik?**
A: Traefik is K8s-native with service discovery, auto HTTPS, dynamic config. Nginx is more performant for traditional workloads.

**Q: How do you test Nginx config before reloading?**
A: `nginx -t` — validates syntax and file paths.

**Q: How do you see Nginx status?**
A: `nginx -s status` (if status module enabled) or `curl http://localhost/nginx_status`.

**Q: How do you debug a 502 Bad Gateway?**
A: Check error log (upstream didn't respond), check if backend is running, check firewall, check fastcgi/proxy config.

**Q: How do you debug a 504 Gateway Timeout?**
A: Backend took too long. Increase `proxy_read_timeout` or `fastcgi_read_timeout`. Or optimize the backend.

---

## 2. Debugging Scenarios

### Scenario 1: 502 Bad Gateway

**Symptom:** All requests return 502. Nginx error log: `connect() failed (111: Connection refused) while connecting to upstream`.

**Debug:**
1. Check if PHP-FPM is running: `systemctl status php8.2-fpm`
2. Check PHP-FPM socket: `ls -la /var/run/php/php8.2-fpm.sock`
3. Check listen directive in PHP-FPM pool config (socket vs TCP)
4. Check Nginx `fastcgi_pass` matches PHP-FPM config
5. Check if PHP-FPM socket permissions allow Nginx user

**Root cause:** PHP-FPM pool was configured to listen on TCP `127.0.0.1:9000` but Nginx was configured for Unix socket.

### Scenario 2: 504 Gateway Timeout

**Symptom:** Requests timeout after 60 seconds. Error log: `upstream timed out (110: Connection timed out) while reading response header`.

**Debug:**
1. Check `proxy_read_timeout` (default 60s)
2. Check `fastcgi_read_timeout` (for PHP)
3. Check backend response time (is it taking > 60s?)
4. Determine if timeout is appropriate or if backend needs optimization

**Fix:** Increase `fastcgi_read_timeout 300;` for long-running reports. Or optimize the slow query.

### Scenario 3: Static files not loading

**Symptom:** CSS/JS files return 404. HTML loads but styling is broken.

**Debug:**
1. Check `root` directive — does the file exist at the absolute path?
2. Check URL path vs filesystem path (is it `/css/app.css` or `/public/css/app.css`?)
3. Check `try_files` — if `.php` fallback catches static files, they route through Laravel
4. Check file permissions — Nginx user must have read access

**Root cause:** `root /var/www/app;` but files are at `/var/www/app/public/css/app.css`. URL `/css/app.css` doesn't match. Should be `root /var/www/app/public;` or `alias`.

### Scenario 4: Rate limiting too strict

**Symptom:** Legitimate users get 503 Service Unavailable.

**Debug:**
1. Check `limit_req_zone` rate — is it too low?
2. Check `burst` setting — does it handle spikes?
3. Check if users are behind a NAT (same IP) — rate limiting per IP punishes many users behind one IP
4. Check if API key rate limiting is more appropriate than IP-based

**Fix:** Increase burst, use header-based rate limiting, or use `$binary_remote_addr` only for untrusted traffic.

### Scenario 5: SSL handshake failure

**Symptom:** Browser reports SSL_ERROR or similar. `curl` works with `-k` flag.

**Debug:**
1. Check certificate chain — does Nginx send intermediate certs?
2. Check `ssl_protocols` — is TLSv1.2 enabled?
3. Check `ssl_ciphers` — are modern ciphers configured?
4. Check certificate file — is it in correct format (PEM)?
5. Check private key — does it match certificate?

**Fix:** Use fullchain certificate (server + intermediates). Test with `openssl s_client -connect example.com:443`.

---

## 3. Config Puzzles

### Puzzle 1: Laravel + API + static files

**Write a Nginx config that:**
- Serves Laravel app on port 80
- Routes `/api/*` to Node.js backend on port 3000
- Serves `/static/*` with 1-year cache
- Blocks `.env` access
- Limits uploads to 20MB

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/laravel/public;

    # Static files — 1 year cache
    location /static/ {
        expires 365d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # API — proxy to Node.js
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Laravel
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Security
    location ~ /\.(?!well-known).* {
        deny all;
    }
    location ~ \.env {
        deny all;
        return 404;
    }
    client_max_body_size 20M;
}
```

### Puzzle 2: Load balanced PHP app

**Write a config that:**
- Load balances across 3 PHP-FPM servers
- Uses least_conn algorithm
- Caches successful responses for 10 minutes
- Sets 5 req/s rate limit with 10 burst
- Enables keepalive to upstream

```nginx
upstream php_servers {
    least_conn;
    server 10.0.1.10:9000 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:9000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:9000 backup;
    keepalive 32;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=php_cache:10m max_size=1g;

limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

server {
    listen 80;
    server_name app.example.com;

    limit_req zone=api_limit burst=10 nodelay;
    proxy_cache php_cache;
    proxy_cache_valid 200 10m;
    proxy_cache_use_stale error timeout updating;
    add_header X-Cache-Status $upstream_cache_status;

    location / {
        proxy_pass http://php_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Puzzle 3: Debug the broken config

**Problem:** This config causes 502 errors. Find the issue.

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/app/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        include fastcgi_params;
    }
}
```

**Issues:**
1. Missing `fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;` — PHP-FPM doesn't know which file to execute.
2. Missing `fastcgi_index index.php;` — optional but recommended.
3. No access/error log paths (not a bug but bad practice).

### Puzzle 4: Optimize for high traffic

**Problem:** Config for a high-traffic API (10K req/s). Identify bottlenecks and optimize.

```nginx
worker_processes 1;

events {
    worker_connections 128;
}

http {
    access_log /var/log/nginx/access.log;

    server {
        listen 80;
        location /api/ {
            proxy_pass http://backend:8080;
        }
    }
}
```

**Fixes:**
1. `worker_processes auto;` (was 1 — only one core used)
2. `worker_connections 2048;` (was 128 — only 128 concurrent connections)
3. `sendfile on; tcp_nopush on;` — zero-copy + optimized sending
4. `gzip on;` — compress JSON responses
5. Cache `proxy_cache` for cacheable responses
6. Add `keepalive` upstream connections
7. Add access log buffering or conditional logging
