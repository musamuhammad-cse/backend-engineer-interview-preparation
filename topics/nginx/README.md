# Nginx — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Laravel apps behind Nginx (FPM), reverse proxy configuration, static asset serving, SSL termination  
> **Note:** Nginx is the most common web server for PHP apps. The senior signal is **performance tuning (worker connections, buffers, caching), security hardening, and advanced configs (rate limiting, load balancing, reverse proxy)** — not just basic server blocks.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud | 20 min/section |
| 2 | Write Nginx configs from scratch (reverse proxy, static cache, rate limit) | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff | 20 min/section |
| 4 | Debug a broken Nginx config scenario | 10 min |

**The senior signal is performance tuning and troubleshooting.** Knowing `location` blocks is table stakes; diagnosing 502 errors, tuning worker processes, and configuring caching layers is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Nginx architecture (master/worker, event loop), server blocks, location blocks, static file serving, reverse proxy, PHP-FPM integration, basic security headers | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Load balancing (upstream, algorithms), caching (proxy_cache, fastcgi_cache), rate limiting (limit_req, limit_conn), SSL/TLS configuration, gzip compression, access/error logs, rewrite rules | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Performance tuning (worker_processes, worker_connections, sendfile, tcp_nopush, keepalive), large-scale config patterns, buffer/packet tuning, HTTP/2 and HTTP/3, Nginx + Lua (OpenResty), microcaching, Nginx vs Caddy/Traefik | 8–10 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 120+ interview questions, debugging scenarios, config puzzles | Ongoing drill |

---

## Study order recommendation

Focus on reverse proxy (your daily need) → caching/performance → security/advanced.

```
Week 1:  01-basic.md          + Write a Laravel Nginx config from scratch
Week 2:  02-intermediate.md   + Load balancing, rate limiting, SSL
Week 3:  03-senior.md         + Performance tuning, microcaching
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** Linux/Git (per DevOps & Infra list).
