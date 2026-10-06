# Case Study: Full-Stack Local Environment Setup (`docker compose` & `docker build`)

A complete, production-grade implementation guide and engineering runbook for onboarding, containerizing, configuring, and orchestrating a fullstack application (e.g., Apli.ai 3-tier architecture: Node/TypeScript backend, React/Vite frontend, MySQL datastore, and Prisma ORM) with zero-leak secret hygiene and multi-stage container builds.

---

## 1. Architectural Topology & Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          LOCAL HOST ENVIRONMENT                             │
│                                                                             │
│   ┌──────────────────────────┐             ┌────────────────────────────┐   │
│   │   Frontend (React/Vite)  │             │    Backend API (Node.js)   │   │
│   │   Port: 3000 / 5173      │             │    Port: 4000 / 8080       │   │
│   │   (docker build / local) │             │    (docker build / local)  │   │
│   └─────────────┬────────────┘             └─────────────┬──────────────┘   │
│                 │                                        │                  │
│                 │ HTTP REST / GraphQL                    │ Prisma Client    │
│                 └──────────────────────────────────────┐ │ TCP 3306         │
│                                                        ▼ ▼                  │
│                                            ┌────────────────────────────┐   │
│                                            │  MySQL Database Container  │   │
│                                            │  (docker compose up -d)    │   │
│                                            │  Port: 3306 (127.0.0.1)    │   │
│                                            │  Volume: mysql_data        │   │
│                                            └────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Step 1: Inspect `docker-compose.yml` (MySQL Database Architecture)

Before starting services, inspect the database service configuration in `docker-compose.yml` to understand volumes, exposed ports, environment variable names, and healthcheck strategies.

### 2.1 Standard MySQL Compose Service Specification

```yaml
version: "3.8"

services:
  mysql-db:
    image: mysql:8.0
    container_name: apli_mysql_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-rootpassword}
      MYSQL_DATABASE: ${DB_NAME:-apli_db}
      MYSQL_USER: ${DB_USER:-apli_user}
      MYSQL_PASSWORD: ${DB_PASSWORD:-apli_password}
    ports:
      # Expose MySQL port to host localhost for Prisma ORM access
      - "127.0.0.1:3306:3306"
    volumes:
      # Named persistent volume to retain database data across restarts
      - mysql_data:/var/lib/mysql
    networks:
      - apli_internal_network
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root --password=$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

volumes:
  mysql_data:
    driver: local

networks:
  apli_internal_network:
    driver: bridge
```

### 2.2 Key Architectural Configuration Highlights:
- **Image**: `mysql:8.0` (standard official OCI MySQL image).
- **Binding**: `127.0.0.1:3306:3306` — binds to loopback interface on host machine to prevent accidental external network exposure.
- **Persistence**: Managed named volume `mysql_data` attached to `/var/lib/mysql`.
- **Healthcheck**: Uses `mysqladmin ping` to report `healthy` only when the mysqld socket is ready to accept client TCP connections.

---

## 3. Step 2 & 3: Start MySQL Database & Verify Port / Health

### 3.1 Start Database Service via Docker Compose

```bash
# Start only the MySQL database container in background detached mode
docker compose up -d mysql-db

# Or if targeting a specific compose file
docker compose -f docker-compose.yml up -d mysql-db
```

### 3.2 Verify Container State, Port Mapping & Health

```bash
# 1. Check container runtime status and health state
docker compose ps mysql-db

# Expected Output:
# NAME             IMAGE       COMMAND                  SERVICE    CREATED          STATUS                    PORTS
# apli_mysql_db    mysql:8.0   "docker-entrypoint.s…"   mysql-db   30 seconds ago   Up 30 seconds (healthy)   127.0.0.1:3306->3306/tcp

# 2. Inspect exact port mapping via Docker CLI
docker port apli_mysql_db 3306
# Output: 127.0.0.1:3306

# 3. Verify TCP port listening on host localhost
nc -zv 127.0.0.1 3306
# Output: Connection to 127.0.0.1 3306 port [tcp/mysql] succeeded!

# 4. View live initialization logs
docker compose logs --tail=50 mysql-db
```

---

## 4. Step 4 & 5: Environment Variables Hygiene & Configuration (`.env`)

Audit the `server/.env.example` file to identify required runtime variables without exposing, logging, or printing any plain-text secrets.

### 4.1 Schema Breakdown of Required Environment Variables

| Variable Name | Purpose / Function | Recommended Local Format / Value (Non-Sensitive) |
| :--- | :--- | :--- |
| `NODE_ENV` | Runtime execution environment | `development` / `production` / `test` |
| `PORT` | HTTP port for the backend server | `4000` or `8080` |
| `DATABASE_URL` | Prisma ORM connection string to MySQL | `mysql://<USER>:<PASSWORD>@127.0.0.1:3306/<DATABASE_NAME>?schema=public` |
| `JWT_SECRET` | Secret key for signing authentication tokens | Random 256-bit cryptographic string |
| `JWT_EXPIRES_IN` | Session token expiration duration | `7d` or `24h` |
| `CORS_ORIGIN` | Allowed cross-origin frontend URL | `http://localhost:3000` or `http://localhost:5173` |
| `REDIS_URL` | In-memory cache endpoint (optional) | `redis://127.0.0.1:6379` |
| `LOG_LEVEL` | Logging verbosity | `debug` / `info` / `warn` |

### 4.2 Safe Environment Setup Command (Zero Plaintext Exposure)

```bash
# Copy template file to active .env without printing or committing credentials
cp server/.env.example server/.env

# Update DATABASE_URL inside server/.env using interactive or script variable expansion
# Format: mysql://apli_user:apli_password@127.0.0.1:3306/apli_db
```

> [!IMPORTANT]
> **Zero Credential Exposure Mandate**: Never print or echo active `.env` files, production passwords, or API tokens into terminal logs, git commit history, or chat transcripts. Ensure `.env` is listed in `.gitignore`.

---

## 5. Step 6: Prisma ORM Migrations & Database Seeding

Once the MySQL container is in a `healthy` state, execute Prisma migrations to create tables and seed mock data.

```bash
# Navigate to server/ directory
cd server

# 1. Install project dependencies (if not already installed)
npm install

# 2. Generate Prisma Client bindings from schema.prisma
npx prisma generate

# 3. Run Prisma database migrations to apply SQL DDL schema to MySQL container
npx prisma migrate dev --name init_local_schema

# Note: If working with an unmanaged prototyping schema:
# npx prisma db push

# 4. Seed database with initial demonstration/mock data
npx prisma db seed

# 5. (Optional) Open Prisma Studio GUI to visually inspect seeded tables in browser
npx prisma studio --port 5555
```

---

## 6. Step 7: Build & Run Backend & Frontend (Local & `docker build`)

### 6.1 Multi-Stage `docker build` Compilation

#### Hardened Multi-Stage Backend Dockerfile (`server/Dockerfile`)

```dockerfile
# ── STAGE 1: BUILD & PRISMA COMPILATION ─────────────────────────────────────
FROM node:20-bookworm-slim AS builder
WORKDIR /app

COPY package*.json ./
COPY prisma ./prisma/

# Install dependencies and generate Prisma Client with BuildKit cache
RUN --mount=type=cache,target=/root/.npm \
    npm ci && \
    npx prisma generate

COPY . .
RUN npm run build && npm prune --production

# ── STAGE 2: RUNTIME ────────────────────────────────────────────────────────
FROM node:20-bookworm-slim AS runner
WORKDIR /app
ENV NODE_ENV=production

# Non-root user security enforcement
USER node

COPY --from=builder --chown=node:node /app/package*.json ./
COPY --from=builder --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/dist ./dist
COPY --from=builder --chown=node:node /app/prisma ./prisma

EXPOSE 4000
CMD ["node", "dist/main.js"]
```

#### Multi-Stage Frontend Dockerfile (`client/Dockerfile`)

```dockerfile
# ── STAGE 1: BUILD ASSETS ───────────────────────────────────────────────────
FROM node:20-bookworm-slim AS builder
WORKDIR /app

COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

COPY . .
RUN npm run build

# ── STAGE 2: NGINX STATIC FILE SERVING ──────────────────────────────────────
FROM nginx:alpine-slim AS runner
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

### 6.2 Executing Docker Build Commands

```bash
# Build Backend Image with BuildKit
DOCKER_BUILDKIT=1 docker build \
  -t apli-server:local \
  -f server/Dockerfile \
  ./server

# Build Frontend Image with BuildKit
DOCKER_BUILDKIT=1 docker build \
  -t apli-client:local \
  -f client/Dockerfile \
  ./client
```

---

### 6.3 Starting Full Stack via Docker Compose or Local Dev Servers

#### Option A: Full Containerized Stack via Docker Compose
```bash
# Start all microservices (MySQL + Server + Client) with rebuild
docker compose up -d --build

# Check status of all stack services
docker compose ps

# Tail aggregated logs
docker compose logs -f
```

#### Option B: Local Hot-Reload Development (with Containerized MySQL)
```bash
# Terminal 1: Backend Server
cd server
npm run dev

# Terminal 2: Frontend Client
cd client
npm run dev
```

---

## 7. Step 8: Safe Engineering Protocols & Verification Checklist

To guarantee stability, follow strict change management protocols:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SAFETY & VERIFICATION PROTOCOL                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  [✓] 1. Inspect & verify without modifying application code without consent.│
│  [✓] 2. Verify container port allocations to prevent binding conflicts.     │
│  [✓] 3. Validate zero secret exposure across terminal outputs and logs.     │
│  [✓] 4. Ensure Prisma migration succeeds before firing backend server.      │
│  [✓] 5. Stop and request explicit user confirmation before applying diffs.  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Troubleshooting Common Local Setup Failures

| Issue / Symptom | Root Cause | Immediate Diagnostic Command | Fix Action |
| :--- | :--- | :--- | :--- |
| `Can't reach database server at 127.0.0.1:3306` | MySQL container is still initializing or crashed. | `docker compose ps mysql-db` and `docker compose logs mysql-db` | Wait for `healthy` status; ensure port `3306` is mapped to host `127.0.0.1`. |
| `P1001: Authentication failed for user 'apli_user'` | Password or username in `DATABASE_URL` does not match `MYSQL_USER`/`MYSQL_PASSWORD`. | Check `docker-compose.yml` environment block. | Realign `server/.env` `DATABASE_URL` with compose credentials. |
| `Port 3306 already allocated` | Native MySQL service running on host machine. | `sudo lsof -i :3306` or `sudo ss -tulpn \| grep :3306` | Stop host service (`sudo systemctl stop mysql`) or remap port in compose (`- "127.0.0.1:3307:3306"`). |
| `Prisma schema validation error: Environment variable not found` | `DATABASE_URL` missing from `server/.env`. | `test -f server/.env && echo "Exists"` | Create `server/.env` from `server/.env.example`. |
