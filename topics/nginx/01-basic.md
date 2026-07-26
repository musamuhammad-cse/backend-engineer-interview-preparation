# Nginx — Basic Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — ground-level Nginx concepts  
> **Estimated time:** 4–6 hours

---

## Table of Contents

1. Nginx Architecture
2. Server Blocks (Virtual Hosts)
3. Location Blocks
4. Static File Serving
5. Reverse Proxy
6. PHP-FPM Integration
7. Basic Security Headers
8. Q&A

---

## 1. Nginx Architecture

### Master-Worker Process Model

```
┌─────────────────────────────────────┐
│         Master Process              │
│  (reads config, binds ports,        │
│   spawns workers, graceful reload)  │
└─────────────┬───────────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐
│Worker│ │Worker│ │Worker│
│  1   │ │  2   │ │  3   │
│(epoll│ │(epoll│ │(epoll│
│ loop)│ │ loop)│ │ loop)│
└──────┘ └──────┘ └──────┘
```

**Master process:**
- Reads and validates configuration
- Binds to privileged ports (80, 443)
- Spawns worker processes
- Handles signals (reload, upgrade, stop)

**Worker processes:**
- Handle all client connections
- Each worker is single-threaded (event-driven)
- Each worker handles thousands of connections via epoll/kqueue
- Workers are independent (no shared state)

> **Trap:** Nginx workers are single-threaded but handle many connections concurrently via event-based processing (epoll). This is NOT the same as Node.js event loop — Nginx workers handle I/O, not application logic.

### Event loop (epoll)

```
Worker starts → opens listening socket → epoll_wait() → new connection arrives
                                                              │
                                                    accept() → add to epoll
                                                              │
                                                    read event (data arrives)
                                                              │
                                                    process request
                                                              │
                                                    send response
                                                              │
                                                    close or keepalive
```

### Key config files

```
/etc/nginx/
├── nginx.conf             # Main configuration
├── sites-available/       # Server block configurations (enabled via symlink)
├── sites-enabled/         # Symlinks to sites-available
├── conf.d/                # Additional config includes
└── modules/               # Dynamic modules
```

---

## 2. Server Blocks (Virtual Hosts)

### Basic server block

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example/public;

    index index.php index.html;

    access_log /var/log/nginx/example_access.log;
    error_log  /var/log/nginx/example_error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### Listening directives

```nginx
# HTTP
listen 80;

# HTTPS
listen 443 ssl http2;

# Specific IP
listen 192.168.1.1:80;

# Unix socket
listen unix:/var/run/nginx.sock;

# Default server (catch-all)
listen 80 default_server;
```

### Multiple server blocks

```nginx
server {
    listen 80;
    server_name api.example.com;
    location / {
        proxy_pass http://api_backend;
    }
}

server {
    listen 80;
    server_name app.example.com;
    location / {
        proxy_pass http://app_backend;
    }
}
```

### Wildcard server names

```nginx
server_name *.example.com;       # matches subdomain.example.com
server_name example.*;            # matches example.com, example.org
server_name ~^(www\.)?(.+)$;      # regex: matches www.anything
```

---

## 3. Location Blocks

### Location matching order

Nginx processes locations in this order:

1. `=` exact match (highest priority)
2. `^~` prefix match (stops searching)
3. `~` or `~*` regex match (first match wins)
4. Prefix match (longest wins)

```nginx
location = /favicon.ico {                    # Exact match
    log_not_found off;
    access_log off;
    expires max;
}

location ^~ /static/ {                        # Prefix match (if matched, don't check regex)
    root /var/www/example;
    expires 30d;
}

location ~ \.php$ {                           # Regex match (case-sensitive)
    fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
}

location ~* \.(jpg|jpeg|png|gif|ico)$ {       # Regex match (case-insensitive)
    expires 30d;
    add_header Cache-Control "public, immutable";
}

location / {                                  # Prefix match (fallback)
    try_files $uri $uri/ /index.php?$query_string;
}
```

### Nested locations

```nginx
location /api/ {
    # /api/ prefix matched

    location ~ \.php$ {
        # Only PHP files under /api/
        fastcgi_pass api_backend;
    }

    location /api/v2/ {
        # More specific prefix
        proxy_pass http://v2_backend;
    }
}
```

### try_files

```nginx
# Laravel: try files on disk, then route through index.php
location / {
    try_files $uri $uri/ /index.php?$query_string;
}

# Try files, then fallback to another server
location / {
    try_files $uri @backend;
}

location @backend {
    proxy_pass http://backend;
}
```

---

## 4. Static File Serving

### Basic static config

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    location / {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
```

### File not found handling

```nginx
location / {
    try_files $uri $uri/ =404;
}

# Custom error page
error_page 404 /404.html;
location = /404.html {
    internal;
}
```

### Directory listing (off by default)

```nginx
location /downloads {
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
}
```

> **Trap:** Never enable `autoindex` in production unless the directory is specifically intended for listing. It exposes file structure.

---

## 5. Reverse Proxy

### Basic reverse proxy

```nginx
location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### Important proxy headers

| Header | Purpose |
|--------|---------|
| `Host` | Original hostname (backend needs this for virtual hosting) |
| `X-Real-IP` | Client's real IP (backend logs this instead of proxy IP) |
| `X-Forwarded-For` | Chain of IPs (proxy → proxy → client) |
| `X-Forwarded-Proto` | Original protocol (http or https) |
| `X-Forwarded-Host` | Original host |

### Proxy to upstream (load balancing)

```nginx
upstream backend {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
    }
}
```

### WebSocket proxy

```nginx
location /ws {
    proxy_pass http://websocket_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;  # 24 hours for persistent connections
}
```

---

## 6. PHP-FPM Integration

### Laravel config

```nginx
server {
    listen 80;
    server_name inventory.example.com;
    root /var/www/inventory/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### PHP-FPM upstream

```nginx
upstream php-fpm {
    server unix:/var/run/php/php8.2-fpm.sock;
    # or TCP
    # server 127.0.0.1:9000;
}
```

### Common directives for PHP

```nginx
fastcgi_pass             unix:/var/run/php/php8.2-fpm.sock;
fastcgi_index            index.php;
fastcgi_param            SCRIPT_FILENAME $document_root$fastcgi_script_name;
fastcgi_param            PHP_VALUE "error_log=/var/log/php_errors.log";
fastcgi_read_timeout     300;
fastcgi_buffer_size      128k;
fastcgi_buffers          4 256k;
fastcgi_busy_buffers_size 256k;
fastcgi_temp_file_write_size 256k;
```

---

## 7. Basic Security Headers

```nginx
# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

# Hide Nginx version
server_tokens off;

# Limit request body size (prevents large upload attacks)
client_max_body_size 10M;

# Disable methods (if not needed)
if ($request_method !~ ^(GET|HEAD|POST)$) {
    return 405;
}

# Block access to hidden files
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# Block access to sensitive files
location ~ (\.env|\.git|composer\.json|artisan) {
    deny all;
    return 404;
}
```

---

## 8. Q&A

**Q: What's the Nginx master-worker process model?**
A: Master reads config, manages workers. Workers handle connections via event loop (epoll). No shared state across workers.

**Q: What's the order of location matching in Nginx?**
A: Exact match (=) → prefix with ^~ → regex (~/~*) → prefix match (longest).

**Q: What's the difference between `proxy_pass http://backend` and `proxy_pass http://backend/`?**
A: Trailing slash affects URI handling. With trailing slash, the matched location prefix is replaced. Without, the full URI is passed.

**Q: How do you configure Nginx as a reverse proxy?**
A: `proxy_pass` directive. Set headers (Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto) for the backend to know the original client.

**Q: How does Nginx handle PHP requests?**
A: Nginx passes `.php` requests to PHP-FPM via `fastcgi_pass` (Unix socket or TCP). `SCRIPT_FILENAME` tells PHP-FPM which file to execute.

**Q: What's `try_files` used for?**
A: Tries files in order. Common pattern: `try_files $uri $uri/ /index.php?$query_string` — serves static files directly, routes everything else through Laravel's front controller.

**Q: How does Nginx handle concurrent connections?**
A: Each worker uses epoll (Linux) or kqueue (BSD) to handle thousands of connections in a single thread. No thread-per-connection overhead.
