# Feature 11: Docker & Container Engine (CLI Manual, Runtime Architecture & Orchestration)

Welcome to the dedicated documentation index for **Docker & Container Engine**. This feature module provides comprehensive architectural blueprints, low-level Linux kernel containerization mechanics, ASCII & Mermaid decision trees, an exhaustive production CLI reference manual, security hardening standards, and a troubleshooting failure matrix.

---

## Module Documentation Index

| Section | Document File | Description |
| :--- | :--- | :--- |
| **Case Studies & Hands-On Labs** | [`casestudy/README.md`](./casestudy/README.md) | Case Study Index & Hands-On Engineering Labs. |
| **Case Study 1: Full-Stack Setup** | [`casestudy/fullstack-local-setup-docker-compose-build.md`](./casestudy/fullstack-local-setup-docker-compose-build.md) | Fullstack onboarding runbook (`docker compose` & `docker build`), MySQL DB setup, zero-secret-exposure `.env` audit, Prisma ORM migration & seeding, and microservice containerization. |
| **Architecture (HLD & LLD)** | [`hld-lld-design.md`](./hld-lld-design.md) | High-Level Architecture (Docker Client, Daemon, containerd, containerd-shim, runc) & Low-Level Mechanics (Namespaces, cgroups v2, OverlayFS 3-tier storage, Copy-on-Write, and Container Lifecycle State Machine). |
| **Decision Trees** | [`decision-tree.md`](./decision-tree.md) | ASCII & Visual Mermaid decision trees for Base Image selection (Alpine vs Distroless vs Slim vs Scratch), Storage Mount strategy (Volume vs Bind vs tmpfs), Network driver selection (Bridge vs Host vs Macvlan vs Overlay), and Multi-stage Build optimization. |
| **CLI Commands & Error Matrix** | [`shell-commands.md`](./shell-commands.md) | In-depth `docker` and `docker compose` command manual, sectioned parameter breakdowns, custom formatting/projections, resource throttling, BuildKit caching, security capabilities, system maintenance, and an exhaustive **Error Resolution Matrix**. |
| **Official References & Best Practices** | [`references.md`](./references.md) | Official Docker documentation links, OCI specifications, CIS Docker Benchmark standards, Dockerfile optimization rules, and security hardening best practices. |

---

## Key Technical & Operational Concepts Covered

1. **Declarative Database & Multi-Service Provisioning**: Isolating datastores (MySQL / PostgreSQL / Redis) using managed Docker named volumes (`mysql_data`), loopback port binding (`127.0.0.1:3306`), and health-aware container dependency chains.
2. **Zero-Trust Environment & Secret Hygiene**: Parsing `.env.example` templates without logging, printing, or leaking plain-text credentials or API tokens in logs or git history.
3. **Database Schema Lifecycles & Prisma ORM**: Synchronizing live MySQL containers with Prisma schema migrations (`prisma migrate dev`) and seeding mock datasets (`prisma db seed`).
4. **Client-Server Architecture & OCI Runtime Stack**: Separation of concerns between the Docker CLI, the `dockerd` REST API daemon, high-level container manager `containerd`, execution supervisor `containerd-shim`, and low-level OCI runtime `runc`.
5. **Linux Kernel Primitives**: Process isolation through 7 core namespaces (`pid`, `net`, `ipc`, `mnt`, `uts`, `user`, `cgroup`) and deterministic resource throttling via **cgroups v2** (`cpu.max`, `memory.max`, `io.weight`, `pids.max`).
6. **Layered Filesystem & OverlayFS**: Copy-on-Write (CoW) mechanics across read-only image layers (`lowerdir`), transient container read-write layer (`upperdir`), kernel working directory (`workdir`), and unified mount (`merged`).
7. **Deterministic Container Lifecycle**: State transitions across `Created`, `Running`, `Paused`, `Restarting`, `Exited`, and `Dead`, with exact Unix process exit codes (`0` clean, `1` application error, `137` OOM/SIGKILL, `143` SIGTERM, `126` non-executable, `127` not found).
8. **Storage Mount Architecture**: Performance and security trade-offs between managed Docker Named Volumes (`/var/lib/docker/volumes`), direct Host Bind Mounts, high-throughput in-memory `tmpfs` mounts, and named pipes.
9. **Container Networking Topologies**: Datapath packet routing across default and user-defined `bridge` networks (veth pairs, iptables NAT/MASQUERADE, embedded DNS server `127.0.0.11`), direct `host` binding, L2/L3 `macvlan`/`ipvlan`, and multi-host `overlay` (VXLAN UDP 4789).
10. **Next-Gen BuildKit Engine**: Directed Acyclic Graph (DAG) build planning, concurrent multi-stage target execution, cache mounts (`--mount=type=cache`), secret mounts (`--mount=type=secret`), and multi-platform compilation (`--platform linux/amd64,linux/arm64`).
11. **Multi-Container Orchestration (Docker Compose)**: Declarative multi-tier application stacks, healthcheck dependency chains (`depends_on: condition: service_healthy`), environment interpolation, resource constraints, and dynamic profiles.
12. **Zero-Trust Security & Hardening**: Defense-in-depth enforcement utilizing rootless container daemons, non-root user execution (`USER 10001:10001`), immutable root filesystems (`--read-only`), Linux capability dropping (`--cap-drop=ALL`), and Seccomp/AppArmor isolation profiles.

---

## Quick Navigation

1. [View Case Study Index (`casestudy/README.md`)](./casestudy/README.md)
2. [View Case Study 1: Full-Stack Setup (`casestudy/fullstack-local-setup-docker-compose-build.md`)](./casestudy/fullstack-local-setup-docker-compose-build.md)
3. [View High-Level & Low-Level Design (`hld-lld-design.md`)](./hld-lld-design.md)
4. [View Decision Trees (`decision-tree.md`)](./decision-tree.md)
5. [View Shell Command Reference & Error Matrix (`shell-commands.md`)](./shell-commands.md)
6. [View Official Reference Links & Best Practices (`references.md`)](./references.md)
