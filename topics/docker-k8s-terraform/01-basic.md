# Docker, Kubernetes & Terraform — Basic Tier (Docker)

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — ground-level container concepts  
> **Estimated time:** 4–6 hours

---

## Table of Contents

1. Containers vs Virtual Machines
2. Docker Architecture
3. Dockerfile Deep Dive
4. Building and Running Images
5. Docker Compose
6. Docker Networking
7. Docker Volumes
8. Container Registries
9. Q&A

---

## 1. Containers vs Virtual Machines

```
┌─────────────────────┐  ┌─────────────────────┐
│      VM             │  │    Container        │
│                     │  │                     │
│ ┌─────────────────┐ │  │ ┌─────────────────┐ │
│ │ App + Binaries  │ │  │ │ App + Binaries  │ │
│ │ + Guest OS      │ │  │ │ (shared host OS)│ │
│ └─────────────────┘ │  │ └─────────────────┘ │
│ ┌─────────────────┐ │  │ ┌─────────────────┐ │
│ │ Hypervisor      │ │  │ │ Container       │ │
│ └─────────────────┘ │  │ │ Runtime (Docker) │ │
│ ┌─────────────────┐ │  │ └─────────────────┘ │
│ │ Host OS         │ │  │ ┌─────────────────┐ │
│ └─────────────────┘ │  │ │ Host OS         │ │
│ ┌─────────────────┐ │  │ └─────────────────┘ │
│ │ Hardware        │ │  │ ┌─────────────────┐ │
│ └─────────────────┘ │  │ │ Hardware        │ │
└─────────────────────┘  │ └─────────────────┘ │
                         └─────────────────────┘
```

| Aspect | VM | Container |
|--------|----|-----------|
| Boot time | Minutes | Seconds |
| Size | GBs (guest OS per VM) | MBs (just app + runtime) |
| Isolation | Strong (separate kernel) | Weak (shared host kernel) |
| Resource overhead | High (hypervisor + guest OS) | Low (cgroups + namespaces) |
| OS support | Any guest OS | Linux (Windows/macOS via VM) |
| Migration | Heavy (disk image) | Light (OCI image) |

**Why containers for your stack:**
- Your Laravel app runs in a container on ECS Fargate
- Chronos is Go — compiled binary in a scratch container (~5 MB)
- Microservices for the trading platform (separate containers per service)

### Linux primitives that make containers work

**Namespaces** — isolate process view:
- `PID namespace` — processes inside see only their own PID tree
- `Network namespace` — own network stack (interfaces, iptables, routing)
- `Mount namespace` — own filesystem mount points
- `UTS namespace` — own hostname and domain
- `User namespace` — own UID/GID mapping (root inside ≠ root outside)
- `IPC namespace` — inter-process communication isolation

**Cgroups** — limit resource usage:
- CPU shares, quotas, cpusets
- Memory limits (soft + hard)
- Block I/O limits
- `pids.max` (limit number of processes)

> **Trap:** "Containers don't contain" — containers share the host kernel. A kernel exploit breaks ALL containers on the host. This is the primary security argument against containers vs VMs.

---

## 2. Docker Architecture

```
┌──────────────────────────────────────────┐
│              Docker Client               │
│  (docker build, run, exec, push, pull)  │
└─────────────────┬────────────────────────┘
                  │ REST API (Unix socket / TCP)
                  ▼
┌──────────────────────────────────────────┐
│           Docker Daemon (dockerd)        │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ Images   │  │ Containers│  │ Volumes│ │
│  │ (build/  │  │ (run/    │  │        │ │
│  │  pull/   │  │  exec/   │  │        │ │
│  │  push)   │  │  logs)   │  │        │ │
│  └──────────┘  └──────────┘  └────────┘ │
└─────────────────┬────────────────────────┘
                  │ containerd
                  ▼
┌──────────────────────────────────────────┐
│              containerd                  │
│  (manages container lifecycle: create,   │
│   start, stop, delete; pulls images)     │
└─────────────────┬────────────────────────┘
                  │ runc
                  ▼
┌──────────────────────────────────────────┐
│    runc (OCI runtime spec)              │
│  (creates the container using namespaces │
│   + cgroups, reads OCI bundle)          │
└──────────────────────────────────────────┘
```

> **Trap:** Docker is NOT a container runtime. Docker Daemon (`dockerd`) talks to `containerd` (the container supervisor), which talks to `runc` (the OCI runtime). Kubernetes also uses `containerd` directly without the Docker Daemon.

---

## 3. Dockerfile Deep Dive

### Laravel Dockerfile (multi-stage)

```dockerfile
# Stage 1: Build dependencies
FROM php:8.2-fpm-alpine AS build

RUN apk add --no-cache \
    postgresql-dev \
    libzip-dev \
    unzip \
    git \
    && docker-php-ext-install pdo_pgsql zip

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-interaction

COPY . .
RUN php artisan optimize

# Stage 2: Runtime
FROM php:8.2-fpm-alpine AS runtime

RUN apk add --no-cache \
    postgresql-dev \
    libzip-dev \
    && docker-php-ext-install pdo_pgsql zip \
    && docker-php-ext-enable opcache

COPY --from=build /app /app
WORKDIR /app

COPY --from=build /usr/bin/composer /usr/bin/composer
COPY --from=build /app/vendor /app/vendor

EXPOSE 9000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD php artisan health:check || exit 1

USER www-data
CMD ["php-fpm"]
```

### Go Dockerfile (scratch — minimal image)

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS build

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/chronos ./cmd/chronos

# Stage 2: Runtime (scratch — nothing but binary)
FROM scratch AS runtime

COPY --from=build /app/chronos /chronos
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080 9090

ENTRYPOINT ["/chronos"]
CMD ["server"]
```

> **Trap:** Using `scratch` base image means NO shell, NO package manager, NO tools. `docker exec` into a scratch container gives you nothing useful. For debugging, use `distroless` instead (busybox-like utilities without a full OS).

### Key Dockerfile instructions

| Instruction | Description | Gotcha |
|-------------|-------------|--------|
| `FROM` | Base image | Prefer specific tags (`php:8.2-fpm-alpine` not `php:latest`) |
| `RUN` | Execute command during build | Each `RUN` creates a layer — chain with `&&` |
| `CMD` | Default command (can be overridden) | `CMD ["php", "-a"]` vs `CMD php -a` (prefer exec form) |
| `ENTRYPOINT` | Command that runs (hard to override) | Combine with CMD for default args: `ENTRYPOINT ["/chronos"]` `CMD ["server"]` |
| `COPY` | Copy files from build context | Respect `.dockerignore` |
| `ADD` | Copy + auto-extract tar/URL | Prefer COPY unless needed |
| `WORKDIR` | Set working directory | Creates if not exists |
| `EXPOSE` | Documentation only (no effect) | Just metadata |
| `ENV` | Environment variable | Available at build-time AND runtime |
| `ARG` | Build-time variable | Not available at runtime |
| `USER` | Switch user | Default is root — use `USER www-data` |
| `HEALTHCHECK` | Container health check | Docker does NOT restart based on health; orchestration tools do |

### .dockerignore

```
vendor/
node_modules/
.git/
.env*
*.log
storage/framework/cache/data/*
```

> **Trap:** Every file in the build context (even ignored ones) is sent to the Docker daemon. Large build contexts slow down builds. Keep context small and use `.dockerignore`.

### Layer caching

Docker caches each layer. A layer only rebuilds if its instruction changes AND all previous layers are cached.

```dockerfile
# BAD: cache busted on every code change
COPY . .
RUN composer install

# GOOD: dependencies cached separately
COPY composer.json composer.lock ./
RUN composer install
COPY . .
```

> **Trap:** `COPY . .` busts the cache because file checksums change. Always copy dependency files first, install, then copy the rest.

---

## 4. Building and Running Images

### Build

```bash
# Build image with tag
docker build -t api:latest .

# Build with platform (M1 Mac → x86 for deployment)
docker build --platform linux/amd64 -t api:latest .

# Build with build args
docker build --build-arg APP_ENV=production -t api:latest .

# BuildKit features (faster builds, secrets, SSH mounts)
DOCKER_BUILDKIT=1 docker build .
```

### Run

```bash
# Run container in foreground
docker run -p 8080:80 api:latest

# Run in detached mode
docker run -d --name api -p 8080:80 api:latest

# Mount volume, env vars, restart policy
docker run -d \
  --name api \
  -p 8080:80 \
  -v "$PWD/storage:/app/storage" \
  -e DB_HOST=database.internal \
  --restart unless-stopped \
  api:latest

# Limit resources
docker run -d \
  --memory="512m" \
  --cpus="0.5" \
  --memory-reservation="256m" \
  api:latest
```

### Exec

```bash
# Shell into running container
docker exec -it api bash

# Run command and exit
docker exec api php artisan cache:clear

# Check logs
docker logs -f api

# Tail last 50 lines with timestamps
docker logs --tail 50 -t api
```

### Cleanup

```bash
# Stop and remove container
docker stop api && docker rm api

# Remove unused images (dangling)
docker image prune

# Remove everything unused
docker system prune -a

# Check disk usage
docker system df
```

---

## 5. Docker Compose

### Laravel + PostgreSQL + Redis stack

```yaml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    image: inventory-api:latest
    ports:
      - "8080:80"
    environment:
      DB_HOST: postgres
      DB_DATABASE: inventory
      DB_USERNAME: app
      DB_PASSWORD: "${DB_PASSWORD}"
      REDIS_HOST: redis
    env_file:
      - .env
    volumes:
      - storage-data:/app/storage
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "php", "artisan", "health:check"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  queue-worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    image: inventory-api:latest
    command: php artisan queue:work --sleep=3 --tries=3
    environment:
      DB_HOST: postgres
      DB_DATABASE: inventory
      DB_USERNAME: app
      DB_PASSWORD: "${DB_PASSWORD}"
      REDIS_HOST: redis
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: inventory
      POSTGRES_USER: app
      POSTGRES_PASSWORD: "${DB_PASSWORD}"
    ports:
      - "5432:5432"
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres-data:
  redis-data:
  storage-data:

networks:
  backend:
    driver: bridge
```

### Compose key features

| Feature | Description |
|---------|-------------|
| `depends_on` | Start order (without `condition`, only waits for container start) |
| `condition: service_healthy` | Wait for health check to pass |
| `healthcheck` | Define application health |
| `volumes` | Named volumes persist across restarts |
| `networks` | Isolated network per stack |
| `env_file` | Load environment from file |
| `profiles` | Optional services (e.g., `docker compose --profile dev up`) |

> **Trap:** `depends_on` without `condition: service_healthy` only waits for the container to START, not for the database to be READY (accepting connections). Your app still needs a retry loop.

### Common compose commands

```bash
# Start all services
docker compose up -d

# Build and start (rebuild images)
docker compose up -d --build

# View logs
docker compose logs -f app

# Scale a service
docker compose up -d --scale queue-worker=3

# Run one-off command
docker compose exec app php artisan migrate

# Stop and remove
docker compose down -v  # -v removes volumes!
```

---

## 6. Docker Networking

### Network drivers

| Driver | Scope | Use case |
|--------|-------|----------|
| `bridge` | Single host | Default — containers communicate via IP |
| `host` | Single host | No network isolation (web server on port 80) |
| `none` | — | No network (security sandbox) |
| `overlay` | Multi-host | Docker Swarm (rarely used) |

### Bridge network (default)

```yaml
# Default — created automatically. Containers linked via --link (deprecated).
# Better: create a custom bridge network for DNS resolution.
services:
  app:
    networks:
      - mynet

networks:
  mynet:
    driver: bridge
```

On a custom bridge, containers resolve each other by service name (`app`, `db`). On the default bridge, you must use IP or `--link`.

> **Trap:** The default `bridge` network does NOT provide DNS-based service discovery. Use custom bridge networks for DNS resolution by container name.

### Host network

```bash
docker run --network host nginx
# Container shares host's network stack (no port mapping needed)
# Performance benefit (no NAT) at cost of isolation
```

### Port publishing

```bash
# Random host port → container port 80
docker run -p 80 nginx

# Host port 8080 → container port 80
docker run -p 8080:80 nginx

# Host IP 127.0.0.1:8080 → container port 80
docker run -p 127.0.0.1:8080:80 nginx
```

> **Trap:** `-p 8080:80` publishes to `0.0.0.0:8080` (all interfaces). Use `-p 127.0.0.1:8080:80` to bind only localhost.

---

## 7. Docker Volumes

### Volume types

| Type | Managed by | Lifecycle | Use case |
|------|-----------|-----------|----------|
| **Named volume** | Docker (`/var/lib/docker/volumes/`) | Persists until removed | Database data, shared app storage |
| **Bind mount** | You (any host path) | Persists until removed | Development (hot reload), config files |
| **tmpfs mount** | Host memory | Ephemeral (deleted on stop) | Secrets, temporary files |

### Named volume

```bash
# Create volume
docker volume create postgres-data

# Mount in container
docker run -v postgres-data:/var/lib/postgresql/data postgres

# Inspect
docker volume inspect postgres-data
```

### Bind mount

```bash
# Mount host directory (development)
docker run -v "$PWD:/app" -w /app node npm run dev
```

> **Trap:** Bind mounts on macOS (Docker Desktop) have significant filesystem performance issues. Use Docker's delegated/g cached mount options: `:cached` or `:delegated`.

### Volume drivers

- `local` (default) — stores on host filesystem
- `rclone` — cloud storage (S3, GCS)
- `efs` — AWS EFS for shared storage across containers/hosts
- `nfs` — NFS volumes

---

## 8. Container Registries

### Docker Hub

```bash
# Login
docker login

# Push image
docker tag api:latest username/api:latest
docker push username/api:latest
```

### Amazon ECR

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t api:latest .
docker tag api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:latest

# ECR lifecycle policy — auto-delete old images
```

### Image tagging strategies

```bash
# Specific tag (immutable — preferred for deployments)
docker tag api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3

# Git SHA (traceable)
docker tag api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:abc123de

# Latest (mutable — avoid in production)
docker tag api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:latest
```

> **Trap:** `latest` is a convention, not a special tag. It's mutable and dangerous. Always use semantic version tags + Git SHA in CI/CD.

---

## 9. Q&A

### Basic

**Q: What's the difference between a container and an image?**
A: An image is a read-only template (layers). A container is a running instance of an image (writable layer added).

**Q: What's the difference between CMD and ENTRYPOINT?**
A: CMD provides defaults that can be overridden. ENTRYPOINT is the main command. Together: `ENTRYPOINT ["/chronos"]` + `CMD ["server"]` → runs `/chronos server`.

**Q: How do you minimize Docker image size?**
A: Multi-stage builds, alpine base images, chain RUN commands, use `.dockerignore`, remove build-time dependencies.

**Q: What's the difference between a bind mount and a named volume?**
A: Bind mount is any host path. Named volume is managed by Docker (in `/var/lib/docker/volumes/`). Prefer named volumes for production.

**Q: How do containers communicate with each other in Docker Compose?**
A: Via custom bridge network and service name (e.g., app resolves `postgres` to the PostgreSQL container). Port mapping not needed for inter-container communication.

**Q: What's the difference between `docker run` and `docker start`?**
A: `docker run` creates + starts a new container. `docker start` starts an existing stopped container.

**Q: What's the purpose of `.dockerignore`?**
A: Exclude files from the Docker build context (faster builds, smaller context, no secrets leaked).

### Trap questions

**Q: You build a Docker image on an M1 Mac. It fails to run on an EC2 instance. Why?**
A: Architecture mismatch. M1 builds ARM images. EC2 is x86_64. Use `--platform linux/amd64` when building.

**Q: A container exits immediately (`Exited (0)`). What do you check?**
A: The process ran and exited. For a web server, it should run in foreground. Check that CMD/ENTRYPOINT doesn't fork to background (e.g., `CMD apachectl -D FOREGROUND`).

**Q: You change code but Docker build uses cached layers — changes aren't applied. Why?**
A: Layer caching — `COPY . .` may come after dependency installation. If you didn't change `composer.json`, the install layer is cached and `COPY . .` runs — this should work. If changes still don't apply, check `.dockerignore` or use `--no-cache`.

**Q: `docker compose up` starts services in the wrong order. How do you fix?**
A: Use `depends_on` with `condition: service_healthy`. But this only controls start order. Your app must still handle DB unavailability with retries.

**Q: A container fills up disk space. Where do you check?**
A: `docker system df` shows disk usage by images, containers, volumes, build cache. Check container logs (often the culprit with no log rotation), or unused images/volumes.

### Follow-up questions

**Q: You mentioned multi-stage builds. What goes in the first stage vs the final stage?**
A: First stage: build tools, compilers, dependencies (Go compiler, npm install, composer install). Final stage: only the compiled binary, runtime dependencies, config. Nothing unnecessary.

**Q: What's the difference between Docker Compose and Kubernetes?**
A: Compose is for single-host container orchestration (dev, test). K8s is for multi-host production container orchestration (scheduling, scaling, service discovery, self-healing).

**Q: How do you handle secrets in Docker Compose?**
A: `env_file` from `.env` (local), or `secrets` (Compose v3.8+). Never hardcode in compose file. In production with K8s, use Secrets.
