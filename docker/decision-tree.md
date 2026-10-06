# Docker: Decision Trees (ASCII & Visual)

This document provides structured decision logic trees and visual workflows for choosing **Base Images**, **Storage Mount Strategies**, **Network Drivers**, **Build Optimizations**, and **Container Security Modes**.

---

## 1. Base Image Selection (ASCII Decision Tree)

```
================================================================================
                    DOCKER BASE IMAGE SELECTION DECISION TREE
================================================================================

              What language/runtime is your application and what are its dependencies?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Statically Compiled ]       [ Interpreted / JIT ]          [ Complex Native Deps ]
  Go, Rust, C++ (Static)        Node.js, Python, Java, Ruby    C-extensions, GDAL, CUDA,
        │                              │                       glibc-dependent binaries
        │                              │                              │
        ▼                              ▼                              ▼
  Is a shell or certs needed?    Need debugging tools in prod?  Do you require alpine/musl?
        │                              │                              │
   ┌────┴────┐                    ┌────┴────┐                    ┌────┴────┐
   ▼         ▼                    ▼         ▼                    ▼         ▼
 [ No ]    [ Yes ]              [ No ]    [ Yes ]              [ No ]    [ Yes ]
   │         │                    │         │                    │         │
   ▼         ▼                    ▼         ▼                    ▼         ▼
 USE       USE                  USE       USE                  USE       USE
 SCRATCH   DISTROLESS-STATIC    DISTRO-   SLIM LINUX           DEBIAN-   ALPINE WITH
 - 0 MB    - ~2 MB              LESS      - e.g. python:slim   SLIM or   BUILD DEPS
 - Max     - CA certificates    - No root - Has /bin/sh        UBUNTU    - Smallest
   security  & timezone files     shell   - Full package mgr     SLIM      footprint
```

### Visual Workflow: Base Image Selection

```mermaid
flowchart TD
    Start([Start Base Image Selection]) --> Compile{Is binary statically compiled? <br> Go, Rust, C++}
    
    Compile -- Yes --> Shell{Need CA certs or tzdata?}
    Shell -- No --> Scratch["FROM scratch <br> • Size: ~0 MB <br> • Maximum attack surface reduction"]
    Shell -- Yes --> DistrolessStatic["FROM gcr.io/distroless/static-debian12 <br> • Includes ca-certificates & /etc/passwd"]
    
    Compile -- No --> Interpreted{Is it an interpreted runtime? <br> Node.js, Python, Java}
    Interpreted -- Yes --> DebugProd{Need shell/package manager <br> in final production image?}
    DebugProd -- No --> DistrolessLang["FROM gcr.io/distroless/nodejs20-debian12 <br> • Minimal CVEs, no package manager, non-root"]
    DebugProd -- Yes --> AlpineGlibc{Compatible with musl libc <br> or needs glibc?}
    
    AlpineGlibc -- "musl is fine" --> AlpineBase["FROM node:20-alpine / python:3.12-alpine <br> • Lightweight (~5-50 MB)"]
    AlpineGlibc -- "Needs glibc / C-extensions" --> SlimBase["FROM python:3.12-slim / node:20-bookworm-slim <br> • 100% glibc compatible, apt available"]
    
    Interpreted -- No --> Heavy["FROM debian:bookworm-slim / ubuntu:24.04 <br> • Native dependencies & heavy build toolchains"]
```

---

## 2. Storage Mount Strategy Selection (ASCII Decision Tree)

```
================================================================================
                 DOCKER STORAGE MOUNT STRATEGY DECISION TREE
================================================================================

                  What is the nature and destination of your container data?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Database / State ]          [ Source Code Sync ]           [ Transient / In-Memory ]
  PostgreSQL, MySQL, Redis,     Local dev hot-reload,          API Session tokens,
  Kafka, Persistent Logs        Host config injection          Secrets, /dev/shm, Build cache
        │                              │                              │
        ▼                              ▼                              ▼
  USE NAMED VOLUMES             USE BIND MOUNTS                USE TMPFS MOUNTS
  - Managed by Docker Engine    - Maps host path directly      - Written strictly to Host RAM
  - Stored in /var/lib/docker   - Fast developer iteration     - Zero disk I/O latency
  - High I/O performance        - File permission coupling     - Auto-cleared on container stop
  - Independent of container    - Environment-specific paths   - No risk of sensitive data
    lifecycle                                                    persisting on disk
```

### Visual Workflow: Storage Mount Architecture

```mermaid
flowchart TD
    StorageStart([Storage Requirement]) --> Persist{Does data need to survive <br> container restarts?}
    
    Persist -- No --> Sensitive{Is data sensitive or high-speed temporary? <br> e.g. RAM disk, secrets}
    Sensitive -- Yes --> TmpfsMount["Use tmpfs Mount (--tmpfs /path) <br> • Stored in host memory only <br> • Never written to disk layer"]
    Sensitive -- No --> Ephemeral["Use Container Writable Layer (OverlayFS) <br> • Fast for ephemeral scratch files <br> • CoW overhead for large mutations"]
    
    Persist -- Yes --> DevOrProd{Host filesystem access pattern?}
    DevOrProd -- "Local Dev / Host Config (/etc)" --> BindMount["Use Bind Mount (-v /host/path:/container/path) <br> • Live source code hot reload <br> • Host-dependent file paths"]
    DevOrProd -- "Production State / Database" --> NamedVolume["Use Named Docker Volume (-v my_vol:/var/lib/postgresql/data) <br> • Managed in /var/lib/docker/volumes/ <br> • Driver plugins for NFS, Ceph, Cloud CSI"]
```

---

## 3. Container Network Driver Selection (ASCII Decision Tree)

```
================================================================================
                 DOCKER NETWORK DRIVER SELECTION DECISION TREE
================================================================================

              Where do containers need to communicate and what are the performance specs?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Single Host Microservices ] [ Multi-Host Cluster Mesh ]    [ Direct L2/L3 Physical Net ]
  Standard Web App + DB on      Swarm cluster or multi-node    Legacy workloads requiring
  one machine                   service mesh                   physical IP from LAN subnet
        │                              │                              │
        ▼                              ▼                              ▼
  USE USER-DEFINED BRIDGE       USE OVERLAY NETWORK            USE MACVLAN / IPVLAN
  - Isolated bridge per app     - Encapsulated VXLAN (UDP 4789)- Direct MAC per container
  - Built-in DNS service name   - Cross-host routing           - Appears as physical device
    resolution (127.0.0.11)     - Encrypted control plane      - Bypasses Linux bridge
        │
        ├───────────────────────┐
        ▼                       ▼
  High-frequency / zero-NAT?    Complete isolation / Air-gapped?
        │                               │
        ▼                               ▼
  USE HOST NETWORK              USE NONE NETWORK
  - Bypasses NAT stack          - Only loopback (127.0.0.1)
  - Direct host port binding    - Air-gapped security
```

---

## 4. Multi-Stage Build & Optimization Strategy (ASCII Decision Tree)

```
================================================================================
               DOCKER MULTI-STAGE BUILD OPTIMIZATION DECISION TREE
================================================================================

                  What assets are needed in your runtime container?
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        ▼                                                             ▼
  [ Source code + Compilers ]                                   [ Compiled Artifacts Only ]
  Go build tools, npm node_modules (dev),                       Binary, dist/ bundle, production
  Maven/Gradle, gcc/make, test suites                           node_modules, minimal runtime
        │                                                             │
        ▼                                                             ▼
  STAGE 1: BUILD ENVIRONMENT                                    STAGE 2: RUNTIME ENVIRONMENT
  - Heavy base image (e.g. golang:1.23, node:20)                - Minimal image (distroless / alpine)
  - Use cache mounts:                                           - COPY --from=build-stage
    `--mount=type=cache,target=/root/.cache`                    - Zero compiler tools in prod
  - Build test & compile binaries                               - Radically smaller attack surface
```

### Visual Workflow: BuildKit Multi-Stage Pipeline

```mermaid
flowchart TD
    BuildTrigger([Start Build Pipeline]) --> MultiStage{Is application compiled or interpreted?}
    
    MultiStage -- "Compiled (Go / Rust / Java)" --> StageBuild["Stage 1: 'builder' <br> • Heavy SDK image <br> • Compiles binary via BuildKit cache"]
    StageBuild --> CopyArtifact["COPY --from=builder /app/bin /app/bin"]
    CopyArtifact --> RuntimeScratch["Stage 2: 'runtime' <br> • FROM scratch / distroless <br> • Result: <20MB image"]
    
    MultiStage -- "Interpreted (Node / Python)" --> StageDeps["Stage 1: 'deps' <br> • Install full devDependencies <br> • Run linting, TypeScript build, tests"]
    StageDeps --> StageProdDeps["Stage 2: 'prod-deps' <br> • npm prune --production"]
    StageProdDeps --> StageFinal["Stage 3: 'runner' <br> • FROM node:slim / distroless <br> • Copy built files + prod-deps <br> • Non-root user execution"]
```
