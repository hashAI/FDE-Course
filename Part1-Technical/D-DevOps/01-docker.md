# Docker: From Fundamentals to Production-Grade Containers

**FDE Role Context:** As a Forward Deployment Engineer, Docker is the packaging layer between your integration code and every customer environment. You will Dockerize your own services, debug customer container issues, and explain build decisions to DevOps-skeptical enterprise teams. The most common complaint you will hear: *"it works on my machine but crashes in Docker."* This file gives you the depth to solve that on-site, fast.

---

## Table of Contents

1. [Core Concepts: Images vs Containers](#1-core-concepts-images-vs-containers)
2. [Docker Layers and the Build Cache](#2-docker-layers-and-the-build-cache)
3. [Dockerfile Best Practices](#3-dockerfile-best-practices)
4. [Choosing a Base Image](#4-choosing-a-base-image)
5. [Multi-Stage Builds](#5-multi-stage-builds)
6. [Non-Root User Security](#6-non-root-user-security)
7. [HEALTHCHECK Instruction](#7-healthcheck-instruction)
8. [.dockerignore](#8-dockerignore)
9. [Running FastAPI in Docker](#9-running-fastapi-in-docker)
10. [docker-compose for Local Dev](#10-docker-compose-for-local-dev)
11. [Environment Variables in Docker](#11-environment-variables-in-docker)
12. [Volumes: Named vs Bind Mounts](#12-volumes-named-vs-bind-mounts)
13. [docker-compose.override.yml](#13-docker-composeoverrideyml)
14. [Debugging Containers](#14-debugging-containers)
15. [Building Lean Images](#15-building-lean-images)
16. [Complete Production Dockerfile: FastAPI + uv](#16-complete-production-dockerfile-fastapi--uv)
17. [Complete docker-compose: App + Postgres + Redis](#17-complete-docker-compose-app--postgres--redis)
18. [Common Failure Modes](#18-common-failure-modes)
19. [Interview Angles](#19-interview-angles)
20. [Practice Exercise](#20-practice-exercise)

---

## 1. Core Concepts: Images vs Containers

### The Mental Model

Think of it this way:
- **Image** = a read-only blueprint (like a class in OOP, or a VM snapshot)
- **Container** = a running instance of that image (like an object instantiated from a class)

You can run 10 containers from the same image simultaneously. Each container gets its own writable layer on top of the shared read-only image layers.

```
┌──────────────────────────────────────────────┐
│  Container A (writable layer)                │
├──────────────────────────────────────────────┤
│  Container B (writable layer)                │
├──────────────────────────────────────────────┤
│  Image Layer 3: COPY . /app                  │  ← read-only, shared
│  Image Layer 2: RUN pip install -r req.txt   │  ← read-only, shared
│  Image Layer 1: FROM python:3.12-slim        │  ← read-only, shared
└──────────────────────────────────────────────┘
```

### Registry

A **registry** is a storage and distribution system for Docker images.

- **Docker Hub**: public registry, `docker pull nginx` pulls from here by default
- **AWS ECR** (Elastic Container Registry): private registry you'll use in production
- **GitHub Container Registry (GHCR)**: often used in CI/CD

Image naming convention:
```
registry/namespace/repository:tag

# Docker Hub
python:3.12-slim                              # official image, tag = 3.12-slim
myuser/myapp:v1.2.3                           # user/repo:tag

# ECR
123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

**Tags are mutable.** `latest` today might point to a different image tomorrow. In production, always pin to a specific digest or semantic version tag.

```bash
# Pull by digest (immutable — the SHA never changes)
docker pull python:3.12-slim@sha256:abc123...
```

### Key Commands

```bash
# Build
docker build -t myapp:latest .
docker build -t myapp:latest -f docker/Dockerfile .  # custom Dockerfile path

# Run
docker run myapp:latest                              # run and attach
docker run -d myapp:latest                           # detached (background)
docker run -d -p 8080:8000 myapp:latest             # publish port host:container
docker run -d --name myservice myapp:latest          # named container
docker run --rm myapp:latest python -c "print('ok')" # run and delete on exit

# List
docker ps                    # running containers
docker ps -a                 # all containers including stopped
docker images                # list images

# Stop/Remove
docker stop myservice
docker rm myservice
docker rmi myapp:latest

# Push/Pull
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3
docker pull nginx:1.25.3
```

---

## 2. Docker Layers and the Build Cache

Every `RUN`, `COPY`, and `ADD` instruction in a Dockerfile creates a new layer. Understanding this is the single most important Docker optimization concept.

### How the Build Cache Works

Docker computes a cache key for each instruction. If the cache key matches a previous build, Docker reuses that layer — skipping the re-execution entirely.

Cache invalidation rules:
1. If an instruction changes → layer is rebuilt, **and all subsequent layers are invalidated**
2. For `COPY`/`ADD` → if any file content changes → cache miss
3. For `RUN` → if the instruction string changes → cache miss

### The Critical Ordering Rule

**Always copy dependency files before application code.**

```dockerfile
# BAD: application code changes on every commit → reinstalls dependencies every build
COPY . /app
RUN pip install -r /app/requirements.txt   # ← this runs every single build

# GOOD: requirements.txt rarely changes → cache hit on pip install most builds
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt   # ← cached unless requirements.txt changes
COPY . /app                                 # ← only this layer misses cache on code changes
```

### Visualizing Layer Cache Behavior

```
Build 1 (first time):
  FROM python:3.12-slim          → MISS (download)
  COPY requirements.txt .        → MISS
  RUN pip install ...            → MISS (3 minutes: network + compile)
  COPY . /app                    → MISS
  Total: ~4 minutes

Build 2 (only app code changed):
  FROM python:3.12-slim          → HIT (cached)
  COPY requirements.txt .        → HIT (file unchanged)
  RUN pip install ...            → HIT (cached!) ← saves 3 minutes
  COPY . /app                    → MISS (code changed)
  Total: ~10 seconds
```

### Squashing Layers (When It Helps)

Chaining `RUN` commands reduces layer count:

```dockerfile
# Creates 4 layers, but each apt-get clean in separate RUN is useless
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*

# Creates 1 layer, apt cleanup actually reduces size
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

**Rule:** Chain related setup commands. Split at natural cache boundaries (dependencies vs code).

---

## 3. Dockerfile Best Practices

Full annotated example before we go deep:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

WORKDIR /app

# 1. Copy dependency spec first (cache optimization)
COPY pyproject.toml uv.lock ./

# 2. Install uv and dependencies into a virtual env
RUN pip install uv \
    && uv sync --frozen --no-dev

# Final stage: minimal runtime image
FROM python:3.12-slim AS runtime

WORKDIR /app

# 3. System dependencies (if any) — installed before app files
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libpq5 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 4. Non-root user
RUN groupadd --gid 1001 appgroup \
    && useradd --uid 1001 --gid appgroup --no-create-home appuser

# 5. Copy virtual env from builder
COPY --from=builder /app/.venv /app/.venv

# 6. Copy application code
COPY --chown=appuser:appgroup src/ /app/src/

# 7. Switch to non-root user
USER appuser

# 8. Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# 9. Use virtual env Python
ENV PATH="/app/.venv/bin:$PATH"

# 10. Entrypoint + CMD split
ENTRYPOINT ["python", "-m", "uvicorn"]
CMD ["src.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

### Key Practices Explained

**Use explicit versions everywhere:**
```dockerfile
# Bad — "latest" is non-deterministic
FROM python:latest

# Good — reproducible builds
FROM python:3.12.4-slim
```

**Minimize what runs as root:**
```dockerfile
# Everything before USER runs as root (unavoidable for system setup)
RUN apt-get install ...
RUN useradd appuser

# Everything after runs as that user
USER appuser
COPY --chown=appuser:appgroup . /app
```

**Use `--no-install-recommends` for apt:**
```bash
# Recommended packages balloon image size with unneeded tools
apt-get install -y --no-install-recommends libpq-dev
```

**Expose is documentation, not firewall:**
```dockerfile
EXPOSE 8000   # tells readers (and docker-compose) the expected port; doesn't publish it
```

---

## 4. Choosing a Base Image

This decision has major consequences for image size, security surface, and build complexity.

### Comparison Table

| Base Image | Compressed Size | Python Included | Package Manager | Use Case |
|---|---|---|---|---|
| `python:3.12` | ~330 MB | Yes (full) | apt (Debian full) | Dev/debugging |
| `python:3.12-slim` | ~45 MB | Yes (minimal Debian) | apt (limited) | **Production standard** |
| `python:3.12-alpine` | ~18 MB | Yes (musl libc) | apk | When size is critical |
| `gcr.io/distroless/python3` | ~25 MB | Yes (no shell) | None | Security-hardened prod |

### `python:3.12-slim` — The Right Default

```dockerfile
FROM python:3.12-slim
```

**Pros:**
- Based on Debian Bookworm slim — glibc (compatible with virtually all C extensions)
- Small enough (~45MB base) for fast ECS/ECR pull times
- Has `bash`, `apt-get` for debugging
- Most Python packages (numpy, psycopg2, etc.) have wheels for glibc — zero compilation

**Cons:**
- Slightly larger than Alpine
- Still has more than a distroless image

**Use when:** Building production services. This is the 90% answer.

### `python:3.12-alpine` — Proceed With Caution

```dockerfile
FROM python:3.12-alpine
```

**Pros:**
- Smallest size (~18MB)
- apk package manager

**Cons:**
- Uses **musl libc** instead of glibc — many Python C extensions don't have Alpine wheels
- Packages that compile from source (cryptography, numpy, psycopg2) need `-dev` packages, which often ends up making the image *larger* than slim
- Slower builds because of compilation
- Stack traces can differ (musl vs glibc memory allocation behavior)

**Use when:** You control all dependencies and they're pure Python. Avoid for most data/ML/DB workloads.

### Distroless — For Security-Hardened Deployments

```dockerfile
FROM gcr.io/distroless/python3-debian12
```

**Pros:**
- No shell, no package manager, no `curl`, no `wget` — minimal attack surface
- Smallest runtime image

**Cons:**
- Cannot `docker exec -it container bash` — no shell for debugging
- Must use multi-stage build
- Debugging requires a debug variant: `gcr.io/distroless/python3-debian12:debug`

**FDE Context:** Some enterprise security teams *require* distroless. Know what it is and how to use it.

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM gcr.io/distroless/python3-debian12
COPY --from=builder /install /usr/local
COPY src/ /app/src/
WORKDIR /app
ENTRYPOINT ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 5. Multi-Stage Builds

Multi-stage builds solve the problem of build dependencies leaking into production images.

### The Problem Without Multi-Stage

```dockerfile
# Single stage — everything in one image
FROM python:3.12-slim
RUN pip install build-tool-A compiler-B dev-dependency-C
# Final image contains all build tools — waste of space + security risk
```

### Multi-Stage Pattern

```dockerfile
# Stage 1: "builder" — has everything needed to build the app
FROM python:3.12-slim AS builder
WORKDIR /build

# Install build dependencies (won't be in final image)
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev  # creates .venv with production deps only

# Stage 2: "runtime" — minimal, only what's needed to run
FROM python:3.12-slim AS runtime

# Copy ONLY the virtual environment from builder
COPY --from=builder /build/.venv /app/.venv

# Copy application code
COPY src/ /app/src/

ENV PATH="/app/.venv/bin:$PATH"
WORKDIR /app
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### What Gets Left Behind in Builder

The final image (`runtime`) does not contain:
- `uv` itself
- pip build cache
- gcc/g++ (if any compilation happened)
- Test dependencies
- Documentation
- Source distribution files

### Three-Stage Pattern for Complex Builds

```dockerfile
# Stage 1: Base — shared between builder and runtime
FROM python:3.12-slim AS base
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Stage 2: Builder — compile and install deps
FROM base AS builder
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# Stage 3: Runtime — clean final image
FROM base AS runtime
COPY --from=builder /install /usr/local
COPY src/ /app/src/
WORKDIR /app
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0"]
```

**Note:** The `base` stage ensures runtime has `libpq5` (the PostgreSQL runtime lib) without needing `libpq-dev` (the header/compiler files, which are only for building psycopg2).

---

## 6. Non-Root User Security

By default, Docker containers run as root (UID 0). This is a significant security risk.

**If an attacker exploits your containerized app:**
- Root in container = potentially root on host (especially if mounted volumes or privileged mode)
- Non-root user limits blast radius

### Creating and Using a Non-Root User

```dockerfile
FROM python:3.12-slim

# Create a system group and user with specific UID/GID
# --system: creates a system account (no home dir, no login shell by default)
# Explicit UID/GID (1001) avoids collisions and is auditable
RUN groupadd --gid 1001 appgroup \
    && useradd \
        --uid 1001 \
        --gid appgroup \
        --no-create-home \
        --shell /sbin/nologin \
        appuser

WORKDIR /app

# Copy files with correct ownership BEFORE switching user
# (cheaper than chown after the fact — avoids a full copy)
COPY --chown=appuser:appgroup . .

# Switch to non-root for all subsequent instructions and at runtime
USER appuser

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Verifying Non-Root at Runtime

```bash
# Check who you're running as
docker run --rm myapp:latest whoami    # should print: appuser
docker run --rm myapp:latest id        # uid=1001(appuser) gid=1001(appgroup)
```

### Volume Permission Issues with Non-Root

This is a common pain point:

```bash
# If you bind-mount a directory created as root:
docker run -v $(pwd)/data:/app/data myapp:latest
# Container process (uid=1001) may not have write access to data/ if host created it as root

# Solution 1: pre-create with correct permissions
mkdir -p data && chmod 777 data   # permissive but simple for dev

# Solution 2: use named volumes instead (Docker manages permissions)
docker run -v myapp_data:/app/data myapp:latest

# Solution 3: entrypoint script that fixes permissions (requires brief root then drop)
```

---

## 7. HEALTHCHECK Instruction

A health check tells Docker (and ECS/K8s) whether the container is actually healthy, not just running.

### Without HEALTHCHECK

```
Container status: Up 2 hours
# But your FastAPI app might be in a deadlock, returning 500s to every request
# Docker reports "healthy" because the process is running
# ECS won't replace the task — it thinks it's fine
```

### With HEALTHCHECK

```dockerfile
# HTTP-based health check
HEALTHCHECK \
    --interval=30s \      # check every 30 seconds
    --timeout=10s \       # fail if no response in 10 seconds
    --start-period=20s \  # grace period after container start (app startup time)
    --retries=3 \         # mark unhealthy after 3 consecutive failures
    CMD curl -f http://localhost:8000/health || exit 1
```

**Problem with curl:** `curl` might not be in your slim/distroless image.

**Better: use Python:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD python -c "\
import urllib.request, sys; \
try: urllib.request.urlopen('http://localhost:8000/health', timeout=5); sys.exit(0) \
except: sys.exit(1)"
```

**Or use a dedicated health check script:**
```dockerfile
COPY --chown=appuser:appgroup scripts/healthcheck.sh /healthcheck.sh
RUN chmod +x /healthcheck.sh
HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD ["/healthcheck.sh"]
```

```bash
# scripts/healthcheck.sh
#!/bin/sh
set -e
response=$(wget --quiet --spider --server-response http://localhost:8000/health 2>&1 | awk 'NR==1{print $2}')
if [ "$response" = "200" ]; then
    exit 0
else
    exit 1
fi
```

### The FastAPI Health Endpoint

```python
# src/routers/health.py
from fastapi import APIRouter
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends

router = APIRouter()

@router.get("/health", tags=["ops"])
async def health_check():
    """Liveness check — is the process alive?"""
    return {"status": "ok"}

@router.get("/health/ready", tags=["ops"])
async def readiness_check(db: AsyncSession = Depends(get_db)):
    """Readiness check — is the app ready to serve traffic?"""
    try:
        await db.execute(text("SELECT 1"))
        return {"status": "ready", "db": "connected"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"DB unavailable: {e}")
```

---

## 8. .dockerignore

`.dockerignore` prevents files from being sent to the Docker build context. This matters for:
1. **Build speed**: Docker sends entire build context to daemon before starting — large contexts are slow
2. **Cache invalidation**: files in context affect `COPY . /app` cache key — don't let `.git` invalidate your cache
3. **Security**: don't accidentally copy `.env` files into your image

```
# .dockerignore

# Version control
.git/
.gitignore

# Python artifacts
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
dist/
build/
.eggs/

# Virtual environments (never copy these — use deps from requirements.txt)
.venv/
venv/
env/

# Environment files (CRITICAL: never include in image)
.env
.env.*
*.env

# Development tools
.pytest_cache/
.coverage
htmlcov/
.mypy_cache/
.ruff_cache/

# IDE files
.idea/
.vscode/
*.swp
*.swo

# Docker files themselves (not needed inside container)
Dockerfile*
docker-compose*
.dockerignore

# Documentation (not needed at runtime)
docs/
*.md
README*

# Test files (not in production image)
tests/
test_*

# CI/CD configs
.github/
.gitlab-ci.yml
.circleci/

# Local dev scripts
scripts/dev_*
Makefile

# OS artifacts
.DS_Store
Thumbs.db

# Terraform/IaC (not needed in app container)
terraform/
*.tf
*.tfstate
```

### Verify What Goes Into Build Context

```bash
# Before building, check what would be sent
docker build --no-cache --progress=plain . 2>&1 | head -20
# Line like: "=> transferring context: 1.23MB" — should be small

# Or list what would be included
docker run --rm -v $(pwd):/src alpine sh -c \
    "cd /src && find . -not -path './.git/*' -not -path './.venv/*'" | head -50
```

---

## 9. Running FastAPI in Docker

### CMD vs ENTRYPOINT

```dockerfile
# ENTRYPOINT: the executable — hard to override
# CMD: default arguments — easy to override at run time

# Pattern 1: CMD only (common for simple cases)
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
# Override: docker run myapp uvicorn src.main:app --reload

# Pattern 2: ENTRYPOINT + CMD split (recommended for FDE services)
ENTRYPOINT ["uvicorn"]
CMD ["src.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
# Override CMD: docker run myapp src.main:app --reload
# Override ENTRYPOINT: docker run --entrypoint python myapp -m pytest
```

**Best practice for FastAPI:**
```dockerfile
# Use exec form (JSON array), not shell form
# Shell form: CMD uvicorn ... → process runs as /bin/sh -c "uvicorn ..." → shell is PID 1, app is not
# Exec form: CMD ["uvicorn", ...] → uvicorn is PID 1 → receives signals directly

# Shell form (BAD — app doesn't receive SIGTERM, causes unclean shutdown)
CMD uvicorn src.main:app --host 0.0.0.0 --port 8000

# Exec form (GOOD — uvicorn receives SIGTERM for graceful shutdown)
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Development vs Production uvicorn Flags

```dockerfile
# Development (hot reload, single worker)
CMD ["uvicorn", "src.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--reload", \
     "--log-level", "debug"]

# Production (multiple workers, no reload, structured logs)
CMD ["uvicorn", "src.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "2", \
     "--log-level", "info", \
     "--access-log"]
```

**Workers in containers:** For ECS Fargate, typically use 1-2 workers. Don't over-provision workers — you scale by running more tasks. Rule of thumb: `workers = (2 × vCPU) + 1` for CPU-bound, but in ECS you're usually better with `--workers 1` and horizontal scaling.

### Gunicorn + Uvicorn Workers (for larger production use)

```dockerfile
# Install gunicorn as process manager
RUN pip install gunicorn uvicorn[standard]

# gunicorn manages worker lifecycle, uvicorn handles async I/O
CMD ["gunicorn", "src.main:app", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--workers", "2", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--graceful-timeout", "30", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

---

## 10. docker-compose for Local Dev

### Basic Structure

```yaml
# docker-compose.yml
version: "3.9"    # compose file format version

services:          # each service = one container (or group of replicas)
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@postgres:5432/mydb
    depends_on:
      postgres:
        condition: service_healthy   # wait for health check, not just started

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Networks in docker-compose

Services in the same docker-compose file share a default network. They reach each other by **service name** as hostname:

```yaml
# In docker-compose.yml:
# app can reach postgres at hostname "postgres" port 5432
# app can reach redis at hostname "redis" port 6379
# These are NOT localhost — they're container-to-container DNS

services:
  app:
    environment:
      DATABASE_URL: "postgresql+asyncpg://user:pass@postgres:5432/mydb"
      #                                              ^^^^^^^^ service name
      REDIS_URL: "redis://redis:6379/0"
      #                    ^^^^^ service name
```

**Custom networks** (for isolation between service groups):
```yaml
networks:
  backend:          # private network for app + db
    driver: bridge
  frontend:         # network for app + nginx

services:
  app:
    networks:
      - backend
      - frontend
  postgres:
    networks:
      - backend    # postgres only accessible from backend network
  nginx:
    networks:
      - frontend
```

---

## 11. Environment Variables in Docker

Three ways to inject environment variables, in order of preference for production:

### Method 1: `environment` key (direct, visible in compose file)

```yaml
services:
  app:
    environment:
      APP_ENV: production
      LOG_LEVEL: info
      # NEVER put secrets here — they end up in git
```

### Method 2: `env_file` (loaded from a file, file not committed)

```yaml
services:
  app:
    env_file:
      - .env             # loaded first (base config)
      - .env.local       # loaded second (local overrides, gitignored)
```

```bash
# .env (committed — non-secret config)
APP_ENV=development
LOG_LEVEL=debug
APP_PORT=8000
POSTGRES_DB=mydb
POSTGRES_USER=myuser

# .env.local (gitignored — secrets and local overrides)
POSTGRES_PASSWORD=mysecretpassword
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

### Method 3: `environment` with variable substitution from shell

```yaml
services:
  app:
    environment:
      DATABASE_URL: "${DATABASE_URL}"     # reads from shell environment or .env
      SECRET_KEY: "${SECRET_KEY:?SECRET_KEY must be set}"  # fails if not set
```

### Variable Substitution Syntax

```yaml
# Required (fail if missing)
SECRET_KEY: "${SECRET_KEY:?error message if missing}"

# Optional with default
LOG_LEVEL: "${LOG_LEVEL:-info}"

# Just pass through from host environment
AWS_REGION: "${AWS_REGION}"
```

---

## 12. Volumes: Named vs Bind Mounts

### Named Volumes (Managed by Docker)

```yaml
volumes:
  postgres_data:      # Docker manages storage location (~/.local/share/docker/volumes/)
  redis_data:

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    volumes:
      - redis_data:/data
```

**Pros:** Portable, Docker manages location, proper permissions, survive container deletion  
**Use for:** Database data, caches, anything that needs to persist between container restarts

### Bind Mounts (Host Path → Container Path)

```yaml
services:
  app:
    volumes:
      - ./src:/app/src          # host ./src/ → container /app/src
      - ./tests:/app/tests
      - ./.env:/app/.env:ro    # :ro = read-only inside container
```

**Pros:** Files on host are immediately reflected in container (hot reload without rebuilding)  
**Use for:** Source code in development (enables `--reload`)  
**Cons:** Host OS file permission differences can cause issues; path is host-specific

### Choosing

| Scenario | Use |
|---|---|
| Database data | Named volume |
| Redis/cache data | Named volume |
| App code in dev | Bind mount (enables --reload) |
| Config files | Bind mount (`:ro`) |
| Secrets (dev only) | Bind mount (`:ro`) — use Secrets Manager in prod |

---

## 13. docker-compose.override.yml

The override file is automatically merged with `docker-compose.yml` when you run `docker-compose up`. This pattern lets you:
- Keep `docker-compose.yml` as a production-compatible base
- Override in `docker-compose.override.yml` for local dev (add reload, bind mounts, debug ports)

```yaml
# docker-compose.yml (base — production compatible, committed to git)
version: "3.9"

services:
  app:
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
    ports:
      - "8000:8000"
    environment:
      APP_ENV: production
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

```yaml
# docker-compose.override.yml (dev overrides — committed to git, no secrets)
version: "3.9"

services:
  app:
    # Override: build from local Dockerfile instead of pulling from ECR
    build:
      context: .
      target: runtime      # stop at the runtime stage for dev

    # Override: use local code with hot reload
    volumes:
      - ./src:/app/src

    # Override: development command with reload
    command: ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

    # Override: development environment additions
    environment:
      APP_ENV: development
      LOG_LEVEL: debug

    # Override: load local secrets (never committed)
    env_file:
      - .env.local

  postgres:
    # Override: expose postgres port for local tooling (DataGrip, etc.)
    ports:
      - "5432:5432"
```

```bash
# Commands:
docker-compose up           # uses docker-compose.yml + docker-compose.override.yml
docker-compose -f docker-compose.yml up   # ignores override (production mode test)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up  # explicit files
```

---

## 14. Debugging Containers

### `docker logs` — See Container Output

```bash
# All logs
docker logs mycontainer

# Follow (tail -f equivalent)
docker logs -f mycontainer

# Last N lines
docker logs --tail 100 mycontainer

# Since a time
docker logs --since 10m mycontainer          # last 10 minutes
docker logs --since "2024-01-15T10:00:00" mycontainer

# Timestamps
docker logs -t mycontainer

# Combined: follow last 50 lines with timestamps
docker logs -f --tail 50 -t mycontainer
```

### `docker exec` — Run Commands Inside a Running Container

```bash
# Open an interactive shell
docker exec -it mycontainer bash
docker exec -it mycontainer sh     # for alpine/distroless-lite (no bash)

# Run a single command
docker exec mycontainer env                    # list environment variables
docker exec mycontainer cat /etc/os-release    # check OS
docker exec mycontainer python -c "import sys; print(sys.path)"
docker exec mycontainer ps aux                 # check running processes

# Check connectivity from inside the container
docker exec mycontainer curl -f http://postgres:5432  # can app reach db?
docker exec mycontainer python -c "import psycopg2; print('ok')"

# Check file permissions
docker exec mycontainer ls -la /app/
docker exec mycontainer whoami
```

### `docker inspect` — Low-Level Container/Image Info

```bash
# Full JSON inspection
docker inspect mycontainer

# Extract specific fields with Go template
docker inspect --format '{{.State.Status}}' mycontainer   # running/exited/etc.
docker inspect --format '{{.State.ExitCode}}' mycontainer  # exit code
docker inspect --format '{{.NetworkSettings.IPAddress}}' mycontainer
docker inspect --format '{{json .Config.Env}}' mycontainer | python -m json.tool
docker inspect --format '{{json .Mounts}}' mycontainer | python -m json.tool

# Inspect an image
docker inspect python:3.12-slim
docker history python:3.12-slim   # see all layers and their sizes
```

### Debugging a Container That Crashes on Start

This is the most common debugging scenario. The container exits immediately and you can't `exec` into it.

```bash
# Step 1: Check exit code and last logs
docker ps -a                              # find the container ID
docker logs <container_id>               # see what it printed before dying
docker inspect --format '{{.State.ExitCode}}' <container_id>

# Common exit codes:
# 0  = clean exit (CMD finished normally)
# 1  = application error (exception, assertion)
# 127 = command not found (bad CMD/ENTRYPOINT)
# 137 = killed by OOM killer (out of memory)
# 139 = segfault (rare in Python)
# 143 = SIGTERM received (normal graceful shutdown)

# Step 2: Override the entrypoint to get a shell
docker run --rm -it --entrypoint bash myapp:latest
# Now you can manually run your app and see the error:
# $ python -m uvicorn src.main:app
# ImportError: No module named 'src'   ← ah, working directory issue

# Step 3: Run with shell for alpine (no bash)
docker run --rm -it --entrypoint sh myapp:latest

# Step 4: Inspect image layers for what's there
docker run --rm -it --entrypoint sh myapp:latest -c "ls -la /app/"
docker run --rm -it --entrypoint sh myapp:latest -c "pip list"
docker run --rm -it --entrypoint sh myapp:latest -c "python -c 'import src.main'"
```

### Debugging Network Issues

```bash
# Test if two services can reach each other
docker-compose exec app ping postgres         # basic connectivity
docker-compose exec app curl http://postgres:5432  # http check
docker-compose exec app python -c "\
import socket; \
print(socket.getaddrinfo('postgres', 5432))"  # DNS resolution

# Check what network a container is on
docker inspect --format '{{json .NetworkSettings.Networks}}' myapp_app_1 | python -m json.tool

# Check if postgres is actually listening
docker-compose exec postgres pg_isready -U myuser -d mydb
```

---

## 15. Building Lean Images

Why image size matters in FDE context:
- **ECS task startup time**: pulling a 2GB image from ECR → 45 seconds cold start. 100MB image → 5 seconds.
- **ECR storage costs**: $0.10/GB/month × 50 image versions × 2GB = $10/month vs $0.50 for 100MB images
- **Security surface**: every binary in the image is a potential vulnerability
- **CI/CD feedback loop**: 10-minute Docker push vs 2-minute push on every PR

### Techniques

**1. Multi-stage builds** (already covered — the highest-impact technique)

**2. Remove apt cache after install**
```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
# One RUN instruction — the cleanup happens in the same layer
```

**3. Use `uv` instead of pip (3-10x faster, cleaner venv)**
```dockerfile
RUN pip install uv \
    && uv sync --frozen --no-dev \
    && pip uninstall uv -y    # remove uv itself from final venv
```

**4. Don't install dev dependencies**
```bash
pip install -r requirements.txt    # includes test deps
uv sync --no-dev                   # excludes dev/test groups
pip install --no-deps .            # install package without extras
```

**5. Clean Python cache**
```dockerfile
RUN find /app -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true \
    && find /app -name "*.pyc" -delete
```

**6. Measure and audit**
```bash
# Check image size
docker images myapp:latest

# Inspect individual layer sizes
docker history myapp:latest

# Use dive for detailed layer analysis
docker run --rm -it \
    -v /var/run/docker.sock:/var/run/docker.sock \
    wagoodman/dive myapp:latest
```

---

## 16. Complete Production Dockerfile: FastAPI + uv

```dockerfile
# syntax=docker/dockerfile:1
# Production Dockerfile for FastAPI application using uv for dependency management

# ── Stage 1: Builder ──────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

# Install uv — fast Python package manager
# Pin version for reproducibility
RUN pip install --no-cache-dir uv==0.4.10

# Copy only dependency specification files
# These change infrequently → cache this expensive layer
COPY pyproject.toml uv.lock ./

# Install production dependencies into an isolated virtual environment
# --frozen: fail if lock file is out of sync
# --no-dev: exclude development/test dependencies
# --no-cache: don't save pip cache in this layer (saves space)
RUN uv sync --frozen --no-dev --no-cache

# ── Stage 2: Runtime ──────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

# Install runtime system libraries only (not build tools)
# libpq5: PostgreSQL client library (needed if using psycopg2 binary)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libpq5 \
        tini \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user with explicit UID/GID
RUN groupadd --system --gid 1001 appgroup \
    && useradd --system --uid 1001 --gid appgroup \
               --no-create-home --shell /sbin/nologin \
               appuser

# Set working directory
WORKDIR /app

# Copy virtual environment from builder stage
# Only the installed packages — not uv, not build caches
COPY --from=builder /build/.venv /app/.venv

# Copy application source code with correct ownership
COPY --chown=appuser:appgroup src/ /app/src/

# Add virtual environment to PATH
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONPATH="/app"

# Switch to non-root user
USER appuser

# Expose the application port (documentation; actual publish via -p flag or ECS)
EXPOSE 8000

# Health check — use Python to avoid curl dependency
HEALTHCHECK \
    --interval=30s \
    --timeout=10s \
    --start-period=30s \
    --retries=3 \
    CMD python -c "\
import urllib.request, sys; \
req = urllib.request.Request('http://localhost:8000/health'); \
try: \
    urllib.request.urlopen(req, timeout=5); sys.exit(0) \
except: sys.exit(1)"

# Use tini as PID 1 — handles signal forwarding and zombie reaping
# tini ensures uvicorn receives SIGTERM for graceful shutdown
ENTRYPOINT ["/usr/bin/tini", "--"]

# Production: 2 workers, structured JSON logs
# Override in docker-compose for dev: --reload, --log-level debug
CMD ["uvicorn", "src.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "2", \
     "--log-config", "/app/src/log_config.json"]
```

### Companion: `pyproject.toml`

```toml
[project]
name = "myintegration"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.111.0",
    "uvicorn[standard]>=0.29.0",
    "sqlalchemy[asyncio]>=2.0.0",
    "asyncpg>=0.29.0",
    "redis>=5.0.0",
    "pydantic-settings>=2.0.0",
    "httpx>=0.27.0",
    "boto3>=1.34.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "httpx>=0.27.0",
    "ruff>=0.4.0",
    "mypy>=1.10.0",
]
```

### Build and Run

```bash
# Build production image
docker build -t myapp:latest .

# Build with specific stage target (for dev with fewer optimizations)
docker build --target runtime -t myapp:dev .

# Run production
docker run -d \
    -p 8000:8000 \
    -e DATABASE_URL="postgresql+asyncpg://user:pass@host:5432/db" \
    -e APP_ENV=production \
    --name myapp \
    myapp:latest

# Check it's healthy
docker ps   # STATUS column: "Up 2 minutes (healthy)"
```

---

## 17. Complete docker-compose: App + Postgres + Redis

```yaml
# docker-compose.yml
# Local development environment: FastAPI app + PostgreSQL + Redis
version: "3.9"

# ── Services ──────────────────────────────────────────────────────────────────
services:

  # ── Application ─────────────────────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime     # build only up to the runtime stage
    container_name: myapp_app
    ports:
      - "${APP_PORT:-8000}:8000"
    environment:
      APP_ENV: development
      LOG_LEVEL: debug
      # Use service names as hostnames (Docker internal DNS)
      DATABASE_URL: "postgresql+asyncpg://${POSTGRES_USER:-appuser}:${POSTGRES_PASSWORD:-devpass}@postgres:5432/${POSTGRES_DB:-appdb}"
      REDIS_URL: "redis://redis:6379/0"
      # Application settings
      APP_SECRET_KEY: "${APP_SECRET_KEY:-dev-only-insecure-key-change-in-prod}"
    # Load additional secrets from local file (gitignored)
    env_file:
      - .env.local        # optional, won't fail if missing
    volumes:
      # Hot reload: mount source code
      - ./src:/app/src:ro
    # Override CMD for development: add --reload
    command:
      - uvicorn
      - src.main:app
      - --host
      - "0.0.0.0"
      - --port
      - "8000"
      - --reload
      - --log-level
      - debug
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "-c",
             "import urllib.request; urllib.request.urlopen('http://localhost:8000/health', timeout=3)"]
      interval: 15s
      timeout: 5s
      start_period: 30s
      retries: 3
    networks:
      - appnet

  # ── PostgreSQL ───────────────────────────────────────────────────────────────
  postgres:
    image: postgres:16.3-alpine
    container_name: myapp_postgres
    environment:
      POSTGRES_USER: "${POSTGRES_USER:-appuser}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-devpass}"
      POSTGRES_DB: "${POSTGRES_DB:-appdb}"
      # Performance tuning for dev
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=en_US.UTF-8"
    ports:
      - "${POSTGRES_PORT:-5432}:5432"   # expose for local tools (DataGrip, psql)
    volumes:
      - postgres_data:/var/lib/postgresql/data
      # Init scripts run on first start
      - ./docker/postgres/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-appuser} -d ${POSTGRES_DB:-appdb}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
    restart: unless-stopped
    networks:
      - appnet

  # ── Redis ────────────────────────────────────────────────────────────────────
  redis:
    image: redis:7.2-alpine
    container_name: myapp_redis
    command: >
      redis-server
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --save ""
      --appendonly no
    ports:
      - "${REDIS_PORT:-6379}:6379"    # expose for local inspection
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    restart: unless-stopped
    networks:
      - appnet

  # ── Optional: Adminer (DB GUI) ───────────────────────────────────────────────
  adminer:
    image: adminer:4.8.1
    container_name: myapp_adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    environment:
      ADMINER_DEFAULT_SERVER: postgres
    networks:
      - appnet
    profiles:
      - tools     # only starts when: docker-compose --profile tools up

# ── Volumes ───────────────────────────────────────────────────────────────────
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

# ── Networks ──────────────────────────────────────────────────────────────────
networks:
  appnet:
    driver: bridge
```

### Companion `.env` File (commit this — no secrets)

```bash
# .env
# Non-secret configuration — committed to git
APP_PORT=8000
POSTGRES_PORT=5432
POSTGRES_DB=appdb
POSTGRES_USER=appuser
REDIS_PORT=6379
```

### Companion `.env.local` File (gitignored — secrets)

```bash
# .env.local — NOT committed to git (in .gitignore)
POSTGRES_PASSWORD=my_local_dev_password
APP_SECRET_KEY=local-dev-secret-key-32chars-minimum
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
```

### Usage Commands

```bash
# Start everything
docker-compose up -d

# Start with log tailing
docker-compose up

# Start only specific services
docker-compose up -d postgres redis

# Start with optional tools profile
docker-compose --profile tools up -d

# Watch logs
docker-compose logs -f app
docker-compose logs -f app postgres

# Run database migrations
docker-compose exec app alembic upgrade head

# Open a psql shell
docker-compose exec postgres psql -U appuser -d appdb

# Open Redis CLI
docker-compose exec redis redis-cli

# Rebuild app image (after Dockerfile changes)
docker-compose build app
docker-compose up -d app

# Stop and remove containers (keep volumes)
docker-compose down

# Stop and remove everything including volumes (DESTRUCTIVE)
docker-compose down -v
```

---

## 18. Common Failure Modes

### Failure 1: "Works locally, fails in Docker" — Environment Variables Missing

**Symptom:** `KeyError: 'DATABASE_URL'` or `ValidationError: field required`

**Cause:** Your local machine has `DATABASE_URL` in its environment (from `.zshrc`, exported manually). The container doesn't inherit host environment variables unless explicitly passed.

**Debug:**
```bash
# Check what env vars the container sees
docker exec myapp env | sort

# Compare with what your app expects
docker exec myapp python -c "import os; print(os.environ.get('DATABASE_URL', 'NOT SET'))"
```

**Fix:**
```yaml
# docker-compose.yml
services:
  app:
    env_file:
      - .env.local   # explicitly load the vars
```

### Failure 2: "Connection refused" to Postgres in Docker

**Symptom:** `psycopg2.OperationalError: could not connect to server: Connection refused`

**Cause A:** App uses `localhost` or `127.0.0.1` instead of the service name.
```python
# Wrong — localhost in container means the container itself, not postgres
DATABASE_URL = "postgresql://user:pass@localhost:5432/db"

# Right — use service name from docker-compose
DATABASE_URL = "postgresql://user:pass@postgres:5432/db"
```

**Cause B:** Postgres isn't ready yet (app starts before postgres is accepting connections).
```yaml
depends_on:
  postgres:
    condition: service_healthy   # waits for health check, not just process start
```

**Cause C:** Port not exposed correctly.
```bash
docker-compose exec app nc -zv postgres 5432   # test TCP connectivity
```

### Failure 3: Permission Denied on Mounted Volumes

**Symptom:** `PermissionError: [Errno 13] Permission denied: '/app/src/output.txt'`

**Cause:** Container user (uid=1001) can't write to a directory owned by root (host user).

**Debug:**
```bash
docker exec myapp ls -la /app/src/     # check ownership
docker exec myapp id                    # check container user UID
ls -la ./src/                           # check host ownership
```

**Fix:**
```bash
# Option 1: Change host directory permissions
chmod 777 ./src/output/

# Option 2: Match container UID to host UID in docker-compose
services:
  app:
    user: "${UID:-1000}:${GID:-1000}"   # use host user's UID

# Option 3: Use named volume (Docker manages permissions)
volumes:
  - output_data:/app/src/output
```

### Failure 4: Module Not Found Error

**Symptom:** `ModuleNotFoundError: No module named 'src'`

**Cause A:** WORKDIR doesn't match PYTHONPATH.
```dockerfile
# Wrong
WORKDIR /app/src
CMD ["uvicorn", "src.main:app"]   # tries to import src.src.main

# Right
WORKDIR /app
CMD ["uvicorn", "src.main:app"]
```

**Cause B:** `PYTHONPATH` not set.
```dockerfile
ENV PYTHONPATH="/app"
```

**Cause C:** Virtual env not activated.
```dockerfile
ENV PATH="/app/.venv/bin:$PATH"
```

### Failure 5: Container OOM Kill (Exit Code 137)

**Symptom:** Container exits with code 137, logs stop mid-processing.

**Debug:**
```bash
docker inspect --format '{{.State.OOMKilled}}' mycontainer   # returns "true"
docker stats mycontainer   # real-time memory usage
```

**Fix:**
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
```

### Failure 6: "No space left on device" During Build

**Cause:** Docker disk usage accumulated (old images, build cache, volumes).

```bash
# See disk usage
docker system df

# Clean up aggressively (removes ALL unused resources)
docker system prune -a --volumes

# More selective cleanup
docker image prune -a     # remove all unused images
docker volume prune       # remove unused volumes
docker builder prune      # remove build cache
```

---

## 19. Interview Angles

### Q: "Walk me through how you'd containerize a Python microservice."

**Great answer structure:**
1. Write a Dockerfile with multi-stage build — builder stage installs deps, runtime stage is clean
2. Use `python:3.12-slim` base — glibc compatibility, small enough, good defaults
3. Layer order: FROM → system deps → create user → copy requirements → `pip install` → copy code → USER → HEALTHCHECK → CMD
4. Non-root user with explicit UID for security and auditability
5. HEALTHCHECK so ECS/K8s can route traffic correctly
6. `.dockerignore` to exclude `.git`, `__pycache__`, `.env`, `tests/`
7. ENTRYPOINT/CMD in exec form for proper signal handling

### Q: "It works in Docker locally but fails in ECS. How do you debug?"

**Great answer — systematic approach:**
1. Check ECS task logs in CloudWatch — same as `docker logs`
2. Check if env vars are set — ECS task definition secrets/environment section
3. Check networking — can the ECS task reach RDS? Security group inbound rules
4. Check IAM — does the task role have permissions it needs?
5. Reproduce locally: `docker run` with the exact same environment variables and same image tag pushed to ECR
6. Check if the issue is timing — add `depends_on` equivalent in ECS (use startup probes, retry logic)

### Q: "How do you handle secrets in Docker?"

**Great answer:**
- **Never** in the Dockerfile (baked into image layers, visible in `docker history`)
- **Never** in `docker-compose.yml` committed to git
- Local dev: `.env.local` file loaded via `env_file`, gitignored
- Staging/prod: environment variables injected from AWS Secrets Manager at ECS task definition level
- The app reads `os.environ` — it doesn't know or care where the secret came from

### Q: "Why use multi-stage builds?"

**Answer:** Build dependencies (gcc, pip, uv, dev packages) shouldn't be in the production image. Multi-stage lets you use a fat builder image with all tools, then copy only the compiled artifacts to a slim runtime image. Result: smaller image (faster pulls, less ECR cost), smaller attack surface (no compiler to exploit), cleaner separation of concerns.

---

## 20. Practice Exercise

### Exercise: Dockerize a Broken Integration Service

**Setup:**
Create this project structure:
```
myservice/
├── src/
│   ├── __init__.py
│   ├── main.py          # FastAPI app
│   └── config.py        # pydantic-settings config
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env.local
```

**`src/main.py`:**
```python
from fastapi import FastAPI
from src.config import settings
import asyncpg

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok", "env": settings.app_env}

@app.get("/db-check")
async def db_check():
    conn = await asyncpg.connect(settings.database_url)
    result = await conn.fetchval("SELECT version()")
    await conn.close()
    return {"postgres_version": result}
```

**`src/config.py`:**
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_env: str = "development"
    database_url: str
    log_level: str = "info"

    class Config:
        env_file = ".env.local"

settings = Settings()
```

**Tasks:**
1. Write a `Dockerfile` using multi-stage build (uv or pip), non-root user, HEALTHCHECK
2. Write a `docker-compose.yml` with the app, PostgreSQL, and Redis
3. Write `.env.local` with the DB URL
4. Run `docker-compose up` and hit `/health` and `/db-check`
5. Intentionally break it (remove the `depends_on` condition), observe the race condition, fix it
6. Check the image size with `docker images` — if it's over 200MB, investigate with `docker history`
7. Try `docker exec -it myservice_app_1 bash` — confirm you're running as a non-root user
8. Add a bind mount for `./src:/app/src:ro` and change `main.py` while the container runs — verify hot reload works

**Expected outcome:**
- `GET /health` returns `{"status": "ok", "env": "development"}`
- `GET /db-check` returns the PostgreSQL version
- Image size under 150MB
- `whoami` inside container returns your non-root user

---

## Cross-References

- [02-cicd-github-actions.md](./02-cicd-github-actions.md) — Building and pushing Docker images in CI/CD pipelines
- [03-env-separation.md](./03-env-separation.md) — Environment-specific configuration management
- [04-secrets-config.md](./04-secrets-config.md) — Secrets management: from `.env` files to AWS Secrets Manager
- [05-kubernetes-basics.md](./05-kubernetes-basics.md) — Running Docker containers on Kubernetes
- [../E-AWS/01-compute.md](../E-AWS/01-compute.md) — Deploying containers on ECS Fargate
- [../E-AWS/03-iam-security.md](../E-AWS/03-iam-security.md) — IAM roles for ECS task execution
