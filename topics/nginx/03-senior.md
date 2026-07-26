# Nginx — Senior Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Nginx intermediate (load balancing, caching, rate limiting, SSL)  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Performance Tuning
2. Buffer and Timeout Tuning
3. HTTP/2 and HTTP/3
4. Nginx + Lua (OpenResty)
5. Nginx vs Caddy vs Traefik
6. Q&A

---

## 1. Performance Tuning

### worker_processes

```nginx
# Number of worker processes
worker_processes auto;  # 1 per CPU core (recommended)

# CPU affinity (pin workers to cores)
worker_cpu_affinity auto;
```

**Rule of thumb:** `worker_processes = number of CPU cores`. Setting it higher causes context switching overhead. Setting it lower underutilizes cores.

### worker_connections

```nginx
events {
    worker_connections 1024;  # max connections per worker
    # Total connections = worker_processes × worker_connections
    # Example: 4 workers × 1024 = 4096 concurrent connections
}
```

**Formula:** `max_clients = worker_processes × worker_connections / (2 for HTTP/1.1 keepalive)`

For high-traffic Laravel apps:

```nginx
worker_processes auto;
events {
    worker_connections 2048;
    multi_accept on;       # accept all new connections at once
    use epoll;             # Linux (default for modern nginx)
}
```

### sendfile and TCP options

```nginx
sendfile on;             # Zero-copy file transfer (kernel → socket, no userspace copy)
tcp_nopush on;           # Optimize for sending headers + file in one packet
tcp_nodelay on;          # Disable Nagle's algorithm (low-latency)
```

**sendfile**: When serving static files, sendfile copies data directly from disk to network socket — no userspace copy. Essential for static file performance.

**tcp_nopush**: Causes Nginx to send headers and file data in one TCP packet. Works with sendfile.

**tcp_nodelay**: Disables Nagle's algorithm (which waits for more data before sending). Critical for interactive/latency-sensitive apps.

### Keepalive settings

```nginx
keepalive_timeout 65;       # Time to keep idle connection open
keepalive_requests 100;     # Max requests per keepalive connection

# Upstream keepalive (backend connections)
upstream backend {
    server 10.0.1.10:8080;
    keepalive 32;            # Max idle connections to upstream
}

location / {
    proxy_http_version 1.1;  # Required for upstream keepalive
    proxy_set_header Connection "";
}
```

### Open file limit

```nginx
worker_rlimit_nofile 4096;  # Max open files per worker (must match system ulimit)
```

Must also set system limit: `ulimit -n 65536` or in `/etc/security/limits.conf`.

---

## 2. Buffer and Timeout Tuning

### Proxy buffers

```nginx
# Buffer size for reading response headers
proxy_buffer_size 4k;

# Number and size of buffers for response body
proxy_buffers 8 16k;

# Size of busy buffers (when sending to client)
proxy_busy_buffers_size 32k;

# Max temp file size (buffer overflow → write to disk)
proxy_max_temp_file_size 1024m;

# Write temp files in chunks
proxy_temp_file_write_size 32k;
```

**Tuning:**
- Small buffers = less memory per connection, but more disk I/O (temp files)
- Large buffers = more memory, less disk I/O
- For JSON APIs with small responses: `proxy_buffer_size 4k; proxy_buffers 4 8k;`
- For large responses: increase `proxy_buffers`

### FastCGI buffers

```nginx
fastcgi_buffer_size 128k;
fastcgi_buffers 4 256k;
fastcgi_busy_buffers_size 256k;
fastcgi_temp_file_write_size 256k;
```

For PHP (Laravel) with large responses or slow output:

```nginx
fastcgi_buffer_size 128k;
fastcgi_buffers 8 256k;       # 8 × 256k = 2MB total buffering
fastcgi_busy_buffers_size 512k;
fastcgi_temp_file_write_size 256k;
```

### Timeouts

```nginx
# Client side
client_body_timeout   30;     # Stop reading body after inactivity
client_header_timeout 30;     # Stop reading headers after inactivity
send_timeout          30;     # Stop sending response after inactivity

# Proxy upstream
proxy_connect_timeout   30;   # Time to connect to upstream
proxy_read_timeout      60;   # Time to wait for upstream response
proxy_send_timeout      60;   # Time to send request to upstream

# PHP-FPM
fastcgi_read_timeout   300;   # PHP scripts may take long (reports)
fastcgi_send_timeout   300;
```

> **Trap:** `proxy_read_timeout` is NOT the max request duration. It's the time between TCP packets. A slow PHP script sends data continuously and won't timeout. A stuck script that sends NOTHING for 60s will timeout.

### Client body buffer

```nginx
# Buffer client request body in memory before sending to upstream
client_body_buffer_size 128k;

# Max request body size (POST uploads)
client_max_body_size 10M;

# Temp file directory for overflow
client_body_temp_path /tmp/nginx_body;
```

---

## 3. HTTP/2 and HTTP/3

### HTTP/2

```nginx
server {
    listen 443 ssl http2;  # Enable HTTP/2

    # HTTP/2 server push (deprecated in Chrome)
    http2_push_preload on;
}
```

HTTP/2 benefits:
- Multiplexing (multiple requests over one connection)
- Header compression (HPACK)
- Server push (deprecated — avoid)
- Binary protocol

**Note:** HTTP/2 requires SSL (TLS). No HTTP/2 over cleartext in browsers.

### HTTP/3 (QUIC)

```nginx
server {
    # HTTP/3 (QUIC) — requires nginx 1.25+ with quiche or similar
    listen 443 quic reuseport;

    # Enable 0-RTT
    ssl_early_data on;

    # HTTP/3 Alt-Svc header
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

HTTP/3 benefits:
- Zero RTT connection establishment
- Better performance on lossy networks (no head-of-line blocking)
- Built on UDP (QUIC)

---

## 4. Nginx + Lua (OpenResty)

### What is OpenResty?

A web platform based on Nginx + LuaJIT. Runs Lua scripts directly in Nginx.

### Use cases

- API gateway logic (auth, routing, rate limiting)
- Dynamic configuration (no reload needed)
- Complex request/response transformation
- Custom caching logic
- Circuit breaking

### Lua in Nginx

```nginx
# Load Lua module
lua_package_path "/etc/nginx/lua/?.lua;;";

# Set up shared dict for rate limiting
lua_shared_dict my_limit 10m;

server {
    location /api {
        access_by_lua_block {
            local count = ngx.shared.my_limit:incr(ngx.var.remote_addr, 1)
            if count and count > 100 then
                ngx.exit(429)  # Too Many Requests
            end
        }
        proxy_pass http://backend;
    }
}
```

### OpenResty vs Nginx comparison

| Feature | Nginx | OpenResty (Nginx + Lua) |
|---------|-------|------------------------|
| Dynamic config | No (reload needed) | Yes (Lua reads from Redis/DB) |
| API gateway | Limited (basic) | Full (auth, routing, transformation) |
| Performance | Excellent | Slightly slower (LuaJIT) |
| Complexity | Simple | Higher |
| Use case | Web server, reverse proxy | API gateway, edge computing |

---

## 5. Nginx vs Caddy vs Traefik

| Feature | Nginx | Caddy | Traefik |
|---------|-------|-------|---------|
| **Configuration** | Files (nginx.conf) | Caddyfile (simple) | YAML/Toml + auto-discovery |
| **Auto HTTPS** | No (manual certbot) | Yes (automatic Let's Encrypt) | Yes (automatic) |
| **Dynamic config** | Reload required | Reload required | Hot-reload (no restart) |
| **Service discovery** | No (manual upstreams) | No | Native (K8s, Consul, Docker) |
| **Lua scripting** | Via OpenResty | Via plugins | Via plugins |
| **Performance** | Excellent | Good | Good |
| **K8s native** | Manual | Manual | Yes (Ingress Controller) |
| **Best for** | Traditional web serving | Simple static sites, auto HTTPS | K8s, microservices |

### When to use what

- **Nginx:** Your current stack. Best for traditional PHP/Laravel serving, reverse proxy, high-performance static serving.
- **Caddy:** Simple projects, need auto HTTPS, minimal config.
- **Traefik:** K8s-native environments, microservices, need service discovery.

---

## 6. Q&A

**Q: How do you tune Nginx worker processes and connections?**
A: `worker_processes auto` (1 per CPU core). `worker_connections 1024+` per worker. Total connections = processes × connections. Monitor with `nginx status` and adjust.

**Q: What's sendfile and why does it matter?**
A: Zero-copy file transfer. Data goes from disk cache → socket directly, bypassing userspace. Critical for static file serving performance.

**Q: What's the difference between `tcp_nopush` and `tcp_nodelay`?**
A: `tcp_nopush`: optimize sending (wait for full packet). `tcp_nodelay`: disable Nagle's algorithm (send immediately). They work at different stages — nopush for file sending, nodelay for interactive/low-latency.

**Q: How do you tune proxy buffers for an API with JSON responses?**
A: Small buffers: `proxy_buffer_size 4k; proxy_buffers 4 8k;`. For large JSON responses, increase `proxy_buffers`. Monitor `proxy_temp_path` writes (too much disk I/O = buffers too small).

**Q: What's the difference between HTTP/2 and HTTP/3?**
A: HTTP/2: multiplexed, binary, header compression, requires TLS. HTTP/3: based on QUIC (UDP), 0-RTT, no head-of-line blocking, better on lossy networks.

**Q: What's OpenResty?**
A: Nginx + LuaJIT. Run Lua code directly in Nginx. Used for API gateways, dynamic config, complex request processing.

**Q: How do you handle a DDoS with Nginx?**
A: (1) Rate limiting per IP (`limit_req`). (2) Limit connections (`limit_conn`). (3) Block bad user agents. (4) Use `geo` module to block known bad IPs. (5) Use `ngx_http_limit_req_module` with burst. (6) For large DDoS, use Cloudflare/AWS Shield in front.

**Q: What's the `updating` parameter in `proxy_cache_use_stale`?**
A: Serves stale cached content when the cache is being refreshed to a new version. Prevents multiple requests from all hitting the backend simultaneously when cache expires.

**Q: How do you debug Nginx performance issues?**
A: (1) Check `nginx status` module for active connections. (2) Check `worker_connections` (are you hitting the limit?). (3) Check `error.log` for timeouts/errors. (4) Check system resources (CPU, memory, open files). (5) Check upstream response times. (6) Enable $upstream_cache_status to see cache hit ratio.
