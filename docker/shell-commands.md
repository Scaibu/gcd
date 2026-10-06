# Docker & Docker Compose (`docker`) Production Command Reference

A comprehensive, production-grade CLI reference manual for administering **Docker Engine** and **Docker Compose**. Every command is organized into logical architectural sections (`# ── SECTION ──`), with explicit breakdowns of valid values, data formats, enums, units, and enterprise trade-offs.

---

## 1. Container Lifecycle & Execution

### 1.1 Hardened Production Container Execution (`docker run`)

Runs a container with explicit memory caps, CPU pinning, process limits, security profiles, log rotation, and custom networking.

```bash
docker run -d \
  # ── CONTAINER IDENTITY & DAEMON MODE ─────────────────────────────────────
  --name api-gateway-prod \
  # String: Unique container identifier on the local host.
  --hostname api-gateway.internal \
  # String: Internal FQDN visible inside UTS namespace.

  # ── RESOURCE GOVERNANCE & CGROUPS V2 LIMITS ──────────────────────────────
  --memory=2g \
  # Memory string: Hard ceiling on RAM allocation (e.g., "512m", "2g", "4g").
  # Behavior: Linux OOM killer sends SIGKILL (Exit 137) if process exceeds limit.
  --memory-reservation=1g \
  # Memory string: Soft memory threshold triggering proactive page cache reclamation.
  --cpus=2.5 \
  # Float: Quota allocated in cgroups v2 (2.5 = 250% of a single CPU core).
  --cpuset-cpus="0,1,2" \
  # String: Pins execution strictly to physical CPU cores 0, 1, and 2.
  --pids-limit=256 \
  # Integer: Maximum concurrent processes/threads inside container (prevents fork bombs).

  # ── AUTOMATIC RECOVERY & RESTART POLICIES ────────────────────────────────
  --restart=unless-stopped \
  # Enum: "no" | "always" | "on-failure[:max-retries]" | "unless-stopped".
  # Default: "no". "unless-stopped" persists restarts across host reboots.

  # ── NETWORK & PORT MAPPING ───────────────────────────────────────────────
  --network=corp-backend-net \
  # String: Connects container to custom user-defined bridge network (enables embedded DNS).
  --ip=172.28.5.10 \
  # IPv4 String: Static IP assignment within the custom subnet.
  -p 127.0.0.1:8443:443/tcp \
  # Port mapping: [host_ip:][host_port:]container_port[/protocol].
  # Binding to 127.0.0.1 prevents accidental public exposure via host iptables NAT.

  # ── STORAGE & PERSISTENCE ────────────────────────────────────────────────
  --mount type=volume,source=api-data-vol,target=/app/data,readonly=false \
  # Mount spec: type="volume"|"bind"|"tmpfs", src=name/path, dst=target_path.
  --mount type=tmpfs,target=/tmp,tmpfs-size=128m,tmpfs-mode=1777 \
  # In-memory filesystem: Fast scratch storage that leaves zero traces on host disk.

  # ── ZERO-TRUST SECURITY & HARDENING ──────────────────────────────────────
  --user=10001:10001 \
  # UID:GID: Enforces non-root execution inside the container.
  --read-only \
  # Flag: Mounts container root filesystem as strictly read-only (OverlayFS upperdir blocked).
  --security-opt=no-new-privileges:true \
  # Security opt: Prevents binaries from acquiring higher privileges via setuid/setgid flags.
  --cap-drop=ALL \
  # Capability drop: Strips all default Linux kernel capabilities.
  --cap-add=NET_BIND_SERVICE \
  # Capability add: Grants only specific kernel capability (bind to privileged ports <1024).

  # ── PRODUCTION LOGGING & ROTATION DRIVER ─────────────────────────────────
  --log-driver=json-file \
  # Enum: "json-file" | "syslog" | "journald" | "fluentd" | "awslogs" | "gcplogs".
  --log-opt max-size=20m \
  # Byte string: Caps individual log file size before automatic rotation.
  --log-opt max-file=5 \
  # Integer: Maximum number of rotated log backup files retained.

  # ── ENVIRONMENT & METADATA ───────────────────────────────────────────────
  --env-file=/opt/config/api-gateway.env \
  # Filepath: Injects key-value pairs without exposing secrets in bash history.
  -e NODE_ENV=production \
  # Key=Value: Sets individual environment variable.
  --label="environment=production" \
  --label="team=core-infrastructure" \

  # ── IMAGE & RUNTIME ARGS ─────────────────────────────────────────────────
  my-registry.io/corp/api-gateway:v2.4.1 \
  --config=/app/data/config.yaml
```

---

### 1.2 Container State Control Commands

```bash
# Gracefully stop container (sends SIGTERM, waits 30s before SIGKILL)
docker stop --time 30 api-gateway-prod

# Immediately send uncatchable SIGKILL (Exit 137)
docker kill api-gateway-prod

# Freeze all processes in container cgroup using SIGSTOP
docker pause api-gateway-prod

# Resume frozen container processes using SIGCONT
docker unpause api-gateway-prod

# Restart container with a 15-second graceful shutdown timeout
docker restart --time 15 api-gateway-prod

# Remove stopped container and its anonymous attached volumes
docker rm -v api-gateway-prod

# Force remove running container immediately
docker rm -f api-gateway-prod
```

---

## 2. Container Inspection, Debugging & Observability

### 2.1 Process & State Inspection

```bash
# Format running containers into structured tabular columns
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Size}}"

# Filter containers by exit code or label
docker ps -a --filter "status=exited" --filter "label=environment=production"

# Extract precise JSON fields using Go templates
# Extract container IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' api-gateway-prod

# Extract OOMKilled boolean status and exit code
docker inspect -f 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}} Dead={{.State.Dead}}' api-gateway-prod

# Extract container restart count and health check status
docker inspect -f 'Restarts={{.RestartCount}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}None{{end}}' api-gateway-prod
```

---

### 2.2 Live Streaming Logs, Metrics & Process Trees

```bash
# Stream live logs with timestamps from the last 15 minutes
docker logs -f --timestamps --since=15m --tail=100 api-gateway-prod

# Real-time resource usage stream (CPU, Memory, Block I/O, Network I/O, PIDs)
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}\t{{.PIDs}}"

# Non-streaming snapshot of container resource consumption across all active containers
docker stats --no-stream

# Display processes running inside the container namespace from the host perspective
docker top api-gateway-prod -ef
```

---

### 2.3 Interactive Debugging & Filesystem Diffs

```bash
# Execute interactive shell inside running container with rootless user
docker exec -it -u 10001:10001 api-gateway-prod /bin/sh

# Execute non-interactive diagnostic command and capture output
docker exec api-gateway-prod netstat -tuln

# Inspect filesystem mutations on the container's writable OverlayFS upperdir
# Outputs: 'A' (Added), 'C' (Changed), 'D' (Deleted)
docker diff api-gateway-prod

# Copy files between host and container
docker cp api-gateway-prod:/app/logs/error.log /tmp/error.log
docker cp /opt/certs/tls.crt api-gateway-prod:/etc/ssl/certs/tls.crt
```

---

## 3. Image Management, BuildKit & Registry Operations

### 3.1 Next-Gen BuildKit Parallel Compilation (`docker build`)

```bash
# Enable BuildKit engine
export DOCKER_BUILDKIT=1

docker build \
  # ── BUILD CONTEXT & DOCKERFILE ───────────────────────────────────────────
  -f Dockerfile.production \
  -t my-registry.io/corp/api-gateway:v2.4.1 \
  -t my-registry.io/corp/api-gateway:latest \
  . \
  # ── TARGET STAGE & MULTI-STAGE SELECTION ─────────────────────────────────
  --target=runner \
  # String: Executes only up to specified stage in multi-stage Dockerfile.

  # ── IN-MEMORY SECRETS MOUNT ──────────────────────────────────────────────
  --secret id=npmrc,src=/home/dev/.npmrc \
  # Secret file: Exposed inside build via RUN --mount=type=secret,id=npmrc (never saved in layer).

  # ── HIGH-PERFORMANCE REMOTE LAYER CACHING ────────────────────────────────
  --cache-from type=registry,ref=my-registry.io/corp/api-gateway:buildcache \
  --cache-to type=registry,ref=my-registry.io/corp/api-gateway:buildcache,mode=max \
  # mode=max: Caches intermediate build layers across all stages in registry.

  # ── BUILD ARGUMENTS ──────────────────────────────────────────────────────
  --build-arg BUILD_VERSION=2.4.1 \
  --build-arg GIT_COMMIT=$(git rev-parse --short HEAD) \

  # ── MULTI-ARCH CROSS-COMPILATION (docker buildx) ─────────────────────────
  --platform linux/amd64,linux/arm64 \
  --push
```

---

### 3.2 Image Inspection, History & Pruning

```bash
# Display image layer history with commands and size per layer
docker image history --no-trunc --format "table {{.CreatedBy}}\t{{.Size}}" my-registry.io/corp/api-gateway:v2.4.1

# Authenticate against private OCI container registry
echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" --password-stdin my-registry.io

# Push image tags to remote registry
docker push my-registry.io/corp/api-gateway:v2.4.1

# Prune dangling (untagged <none>) images
docker image prune -f

# Prune all unused images older than 7 days (168h)
docker image prune -a --force --filter "until=168h"
```

---

## 4. Storage & Volume Administration

```bash
# Create managed Docker named volume with custom driver labels
docker volume create \
  --driver local \
  --label environment=production \
  --label project=api-gateway \
  api-data-vol

# List all Docker volumes with custom projection
docker volume ls --filter "label=environment=production"

# Inspect volume physical storage path on the Linux host filesystem
docker volume inspect -f '{{.Mountpoint}}' api-data-vol
# Output: /var/lib/docker/volumes/api-data-vol/_data

# Backup named volume into a compressed tar archive on host
docker run --rm \
  -v api-data-vol:/source:ro \
  -v $(pwd):/backup \
  alpine:latest \
  tar -czvf /backup/api-data-vol-backup-$(date +%F).tar.gz -C /source .

# Restore compressed tar archive back into named volume
docker run --rm \
  -v api-data-vol:/target \
  -v $(pwd):/backup:ro \
  alpine:latest \
  tar -xzvf /backup/api-data-vol-backup-2026-10-06.tar.gz -C /target

# Prune all unattached Docker volumes
docker volume prune -f
```

---

## 5. Network Administration & Custom Topologies

```bash
# Create isolated custom bridge network with defined CIDR and gateway
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  --ip-range 172.28.5.0/24 \
  --gateway 172.28.0.1 \
  --opt "com.docker.network.bridge.name"="br-corp-backend" \
  --opt "com.docker.network.bridge.enable_icc"="true" \
  corp-backend-net

# Connect running container to an additional network
docker network connect --ip 172.28.5.20 corp-backend-net redis-cache-prod

# Disconnect container from default bridge network
docker network disconnect bridge redis-cache-prod

# Inspect connected containers and assigned IP addresses in network
docker network inspect -f '{{json .Containers}}' corp-backend-net | jq .

# Remove network
docker network rm corp-backend-net
```

---

## 6. Multi-Container Orchestration (Docker Compose)

### 6.1 Declarative Stack Definition (`docker-compose.yml`)

```yaml
version: "3.9"

services:
  web-api:
    image: my-registry.io/corp/api-gateway:v2.4.1
    container_name: web-api
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      - DB_HOST=postgres-db
      - REDIS_HOST=redis-cache
    depends_on:
      postgres-db:
        condition: service_healthy
      redis-cache:
        condition: service_started
    networks:
      - app-tier
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2048M
        reservations:
          cpus: "0.5"
          memory: 512M
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s

  postgres-db:
    image: postgres:16-alpine
    container_name: postgres-db
    restart: always
    environment:
      POSTGRES_DB: app_production
      POSTGRES_USER_FILE: /run/secrets/db_user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_user
      - db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-tier
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d app_production"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis-cache:
    image: redis:7-alpine
    container_name: redis-cache
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "prod_redis_secret"]
    volumes:
      - redisdata:/data
    networks:
      - app-tier

volumes:
  pgdata:
    driver: local
  redisdata:
    driver: local

networks:
  app-tier:
    driver: bridge

secrets:
  db_user:
    file: ./secrets/db_user.txt
  db_password:
    file: ./secrets/db_password.txt
```

---

### 6.2 Docker Compose Production Commands

```bash
# Start all services in detached mode, rebuilding stale images and removing orphan containers
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build --remove-orphans

# View status of all stack services including healthcheck states
docker compose ps

# Tail aggregated logs across all services with timestamps
docker compose logs -f --tail=100 --timestamps web-api

# Execute database migration command inside running web-api container
docker compose exec web-api npm run db:migrate

# Stop and gracefully tear down stack, preserving persistent volumes
docker compose down

# Stop stack and completely remove named volumes
docker compose down -v --remove-orphans
```

---

## 7. System Maintenance & Pruning

```bash
# Display detailed Docker disk space utilization (Images, Containers, Local Volumes, Build Cache)
docker system df -v

# Real-time daemon event stream (container lifecycle, image pulls, volume mounts)
docker events --filter "type=container" --filter "event=die"

# Comprehensive system cleanup: removes stopped containers, dangling images, unused networks, and build cache
docker system prune -f

# Total aggressive cleanup: removes ALL unused containers, all unreferenced images, and volumes older than 7 days
docker system prune -a --volumes --force --filter "until=168h"

# Clear dangling BuildKit compiler cache
docker builder prune -a -f --filter "until=24h"
```

---

## 8. Exhaustive Error Resolution Matrix

| Error Message & Indicator | Exit Code / Type | Root Cause | Diagnosis Command | Immediate Resolution & Prevention |
| :--- | :--- | :--- | :--- | :--- |
| `OOMKilled: true` / Process terminated abruptly | `Exit Code 137` | Process exceeded hard memory limit (`--memory` or host RAM). Linux OOM killer issued `SIGKILL`. | `docker inspect -f '{{.State.OOMKilled}}' <container>` | Increase `--memory` limit; optimize application memory leaks; set JVM/Node.js heap ceilings (`--max-old-space-size`). |
| `exec /entrypoint.sh: no such file or directory` | `Exit Code 127` | Windows CRLF line endings in script or invalid shebang (e.g. `#!/bin/bash` on Alpine with only `/bin/sh`). | `head -n 1 entrypoint.sh && file entrypoint.sh` | Run `dos2unix entrypoint.sh`; change shebang to `#!/bin/sh` or install bash (`apk add --no-cache bash`). |
| `exec /app/server: permission denied` | `Exit Code 126` | Executable file is missing POSIX execute permissions (`+x`) inside container image. | `docker run --rm <image> ls -la /app/server` | Add `RUN chmod +x /app/server` in Dockerfile prior to executing `CMD`/`ENTRYPOINT`. |
| `Bind for 0.0.0.0:80 failed: port is already allocated` | Network Conflict | Host port is already claimed by another Docker container or native host service (e.g. Nginx/Apache). | `sudo lsof -i :80` or `sudo ss -tulpn \| grep :80` | Stop conflicting host process or bind container to an alternate host port (`-p 8080:80`). |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Daemon Socket Error | Docker daemon (`dockerd`) is stopped, crashed, or current user lacks permissions on `/var/run/docker.sock`. | `sudo systemctl status docker` | Start daemon: `sudo systemctl start docker`; grant user access: `sudo usermod -aG docker $USER && newgrp docker`. |
| `no space left on device` during `docker build` | Storage Exhaustion | Host root disk or `/var/lib/docker` partition is 100% full from stale layers and build cache. | `df -h /var/lib/docker` | Execute aggressive pruning: `docker system prune -a --volumes -f && docker builder prune -a -f`. |
| `dial tcp: lookup registry.example.com on 127.0.0.11:53: server misbehaving` | DNS Resolution Timeout | Embedded Docker DNS resolver unable to forward queries to upstream host nameservers. | `docker exec <container> cat /etc/resolv.conf` | Configure fallback nameservers in `/etc/docker/daemon.json`: `{"dns": ["8.8.8.8", "1.1.1.1"]}` and reload daemon. |
| `standard_init_linux.go: exec user process caused "exec format error"` | CPU Architecture Mismatch | Binary or base image was compiled for a different architecture (e.g., `linux/arm64` image run on `linux/amd64`). | `docker inspect -f '{{.Architecture}}' <image>` | Cross-compile using Docker Buildx with explicit platform flag: `docker buildx build --platform linux/amd64 ...`. |
