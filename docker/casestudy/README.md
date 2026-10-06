# Docker: Case Studies & Hands-On Engineering Labs

Welcome to the **Docker Case Studies & Hands-On Engineering Labs** directory. This section provides detailed implementation blueprints, lab runbooks, architectural analyses, and verification workflows for containerizing, orchestrating, and troubleshooting fullstack multi-service applications using `docker build` and `docker compose`.

---

## Lab & Case Study Index

| Lab / Case Study | Documentation File | Description & Architecture Focus |
| :--- | :--- | :--- |
| **Case Study 1: Full-Stack Local Environment Setup (`docker compose` & `docker build`)** | [`fullstack-local-setup-docker-compose-build.md`](./fullstack-local-setup-docker-compose-build.md) | In-depth engineering lab: local multi-tier orchestration, MySQL database configuration, zero-credential-exposure environment mapping, Prisma ORM schema migrations & seeding, multi-stage BuildKit container compilation, and end-to-end service verification. |

---

## Key Competencies & Engineering Highlights

1. **Declarative Database & Service Provisioning**:
   - Isolating stateful datastores (MySQL / PostgreSQL / Redis) using managed Docker volumes and deterministic healthchecks (`mysqladmin ping`).
   - Granular port mapping and host isolation (binding to `127.0.0.1` vs `0.0.0.0`).
2. **Strict Environment & Secret Hygiene**:
   - Auditing `.env.example` templates without logging, printing, or leaking plain-text credentials or API secrets.
   - Constructing secure connection URIs (`mysql://${DB_USER}:${DB_PASS}@${DB_HOST}:${DB_PORT}/${DB_NAME}`).
3. **Database Schema Lifecycles with Prisma ORM**:
   - Synchronizing containerized database instances with Prisma schema migrations (`prisma migrate dev` / `prisma db push`).
   - Seeding mock / demonstration records (`prisma db seed`) in isolated local container environments.
4. **Optimized Multi-Stage Container Compilation (`docker build`)**:
   - Leveraging BuildKit cache mounts (`--mount=type=cache`) and non-root execution (`USER nonroot:nonroot`) for enterprise backend and frontend container packaging.
5. **Full-Stack End-to-End Orchestration (`docker compose`)**:
   - Running complete microservice topologies with health-aware dependency graphs (`depends_on: condition: service_healthy`).
