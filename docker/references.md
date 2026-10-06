# Docker: Official Reference Links & Best Practices

This document provides official Docker documentation links, Open Container Initiative (OCI) specifications, CIS Security Benchmark standards, and enterprise hardening best practices.

---

## 1. Official Documentation & Specifications

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **Docker Engine Documentation** | [docs.docker.com/engine/](https://docs.docker.com/engine/) | Official landing page for Docker Engine, CLI commands, and daemon configuration. |
| **Docker Compose Specification** | [docs.docker.com/compose/](https://docs.docker.com/compose/) | Specification for declarative multi-container application orchestration. |
| **Dockerfile Reference** | [docs.docker.com/reference/dockerfile/](https://docs.docker.com/reference/dockerfile/) | Complete instruction set for `FROM`, `RUN`, `COPY`, `ENTRYPOINT`, and `HEALTHCHECK`. |
| **BuildKit Documentation** | [docs.docker.com/build/buildkit/](https://docs.docker.com/build/buildkit/) | Next-generation build engine, DAG cache mechanics, and secret mount syntax. |
| **OCI Image & Runtime Specs** | [opencontainers.org](https://opencontainers.org/) | Industry standards governing container image distribution (`image-spec`) and runtime execution (`runtime-spec`). |
| **CIS Docker Benchmark** | [cisecurity.org/benchmark/docker](https://www.cisecurity.org/benchmark/docker) | Center for Internet Security (CIS) compliance benchmarks for securing Docker host environments. |
| **Distroless Container Images** | [github.com/GoogleContainerTools/distroless](https://github.com/GoogleContainerTools/distroless) | Language-focused minimal runtime images maintained by Google without package managers or shells. |

---

## 2. Production Best Practices & Hardening Standards

### 2.1 Image Optimization & Layer Caching
1. **Order Dockerfile Instructions by Frequency of Change**: Place infrequently changing operations (`apt-get install`, `package.json`, `requirements.txt`) before application source code copies (`COPY . .`) to maximize layer cache hits.
2. **Always Use `.dockerignore`**: Exclude `.git`, `node_modules`, `venv`, `*.log`, `.env`, and secret files from the build context to speed up daemon transfers and prevent credential leakage.
3. **Pin Base Images with Immutable SHA-256 Digests**: Avoid floating tags like `:latest` or `:3.12`. Pin exact digests in production pipelines to prevent unexpected upstream drift:
   ```dockerfile
   FROM node:20-bookworm-slim@sha256:d6b63c6d88...
   ```
4. **Leverage BuildKit Cache Mounts**: Use `--mount=type=cache` in `RUN` instructions to keep package manager caches across builds without persisting them into final image layers:
   ```dockerfile
   RUN --mount=type=cache,target=/root/.cache/pip \
       pip install --no-cache-dir -r requirements.txt
   ```

---

### 2.2 Security & Least-Privilege Execution
5. **Enforce Non-Root Execution**: Never run application processes as `root` (UID 0). Create a dedicated unprivileged user and switch to it before the `ENTRYPOINT`:
   ```dockerfile
   RUN groupadd -g 10001 appgroup && \
       useradd -u 10001 -g appgroup -s /sbin/nologin appuser
   USER 10001:10001
   ```
6. **Immutable Root Filesystem**: Deploy containers with `--read-only` and mount temporary in-memory filesystems (`--tmpfs /tmp`) for necessary transient scratch space.
7. **Drop All Kernel Capabilities by Default**: Use `--cap-drop=ALL` and selectively grant only strictly required capabilities (such as `--cap-add=NET_BIND_SERVICE`).
8. **Disable Privilege Escalation**: Pass `--security-opt=no-new-privileges:true` to block setuid/setgid binaries from escalating permissions inside the container.

---

### 2.3 Production Multi-Stage Hardened Dockerfile Template (Node.js Example)

```dockerfile
# ── STAGE 1: DEPENDENCY RESOLUTION & BUILD ──────────────────────────────────
FROM node:20-bookworm-slim AS builder
WORKDIR /build

# Copy dependency manifests
COPY package.json package-lock.json ./

# Install full dependencies with BuildKit npm cache
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Copy application source and compile
COPY . .
RUN npm run build && npm prune --production

# ── STAGE 2: MINIMAL HARDENED PRODUCTION RUNTIME ────────────────────────────
FROM gcr.io/distroless/nodejs20-debian12:nonroot AS runner
WORKDIR /app

# Distroless default unprivileged user: nonroot (UID 65532)
USER nonroot:nonroot

# Copy only production runtime artifacts from builder
COPY --from=builder --chown=nonroot:nonroot /build/dist ./dist
COPY --from=builder --chown=nonroot:nonroot /build/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /build/package.json ./package.json

ENV NODE_ENV=production
ENV PORT=8080
EXPOSE 8080

ENTRYPOINT ["/nodejs/bin/node", "dist/main.js"]
```
