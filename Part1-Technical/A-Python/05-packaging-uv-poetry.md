# 05 — Packaging with uv and Poetry: Reproducible Python Environments

> **Related files:** [01-async-await.md](./01-async-await.md) | [../B-FastAPI/01-async-endpoints-di.md](../B-FastAPI/01-async-endpoints-di.md)

---

## Why Packaging Matters for FDE Work

You'll be deploying your integration into customer environments where:
- The customer's infosec team controls what Python versions are allowed
- The customer's IT has a proxy that blocks PyPI — you need a private mirror
- The CI pipeline runs on a bare Ubuntu 22.04 with Python 3.10, but your laptop has Python 3.12
- The deployment needs to be deterministic — "it worked in staging" must mean "it will work in prod"
- Build times in CI need to be fast — a 3-minute `pip install` is unacceptable for rapid iteration

Understanding `pyproject.toml`, virtual environments, `uv`, and `poetry` at a deep level means the difference between a smooth deployment and a four-hour dependency resolution session on a customer call.

---

## Part 1: pyproject.toml — The Unified Standard

`pyproject.toml` is the PEP 517/518/621 standard for Python project configuration. It replaces `setup.py`, `setup.cfg`, `requirements.txt` (for project metadata at minimum), and `tox.ini` (for tool config).

### Complete Annotated pyproject.toml

```toml
[build-system]
# This tells pip/uv HOW to build the project
# hatchling, setuptools, flit, poetry all implement PEP 517
requires = ["hatchling"]
build-backend = "hatchling.build"

# ─── Project metadata (PEP 621) ───────────────────────────────────────────────

[project]
name = "logistics-integration"
version = "1.2.0"
description = "Logistics API integration service for 3PL customers"
readme = "README.md"
license = { file = "LICENSE" }
authors = [
  { name = "FDE Team", email = "fde@company.com" }
]

# CRITICAL: specify the Python version your code runs on
# This prevents installing on incompatible Python versions
requires-python = ">=3.11,<3.13"
# ">=" means "at least this version"
# "<3.13" means "not yet tested on 3.13 — prevent accidental breakage"

# Runtime dependencies — what's needed to run in production
dependencies = [
  "fastapi>=0.111.0,<0.113",  # version range: at least 0.111, but not 0.113+ (breaking changes)
  "uvicorn[standard]>=0.29.0",
  "pydantic>=2.7.0,<3",       # stay on v2, don't accidentally upgrade to v3
  "httpx>=0.27.0",
  "asyncpg>=0.29.0",
  "python-jose[cryptography]>=3.3.0",
  "tenacity>=8.2.0",           # retry library
  "structlog>=24.1.0",         # structured logging
]

# Optional extras — groups of optional deps
[project.optional-dependencies]
# Install with: pip install logistics-integration[aws]
aws = [
  "aioboto3>=12.4.0",
  "boto3>=1.34.0",
]
# Install with: pip install logistics-integration[dev]
dev = [
  "pytest>=8.0.0",
  "pytest-asyncio>=0.23.0",
  "pytest-cov>=5.0.0",
  "respx>=0.21.0",
  "httpx>=0.27.0",
  "ruff>=0.4.0",
  "mypy>=1.10.0",
  "pre-commit>=3.7.0",
]

# ─── Tool configurations ──────────────────────────────────────────────────────

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
  "slow: tests taking >5 seconds",
  "integration: requires external services",
  "unit: fast isolated tests",
]
# Treat these warnings as errors
filterwarnings = [
  "error::RuntimeWarning",      # catch 'coroutine was never awaited'
  "error::DeprecationWarning",  # catch deprecated API usage
]

[tool.ruff]
target-version = "py311"
line-length = 100
select = [
  "E",   # pycodestyle errors
  "F",   # pyflakes
  "I",   # isort
  "N",   # pep8-naming
  "UP",  # pyupgrade
  "B",   # flake8-bugbear (common bugs)
  "ASYNC",  # flake8-async (async antipatterns)
]
ignore = ["E501"]  # line length handled by formatter

[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true

[tool.coverage.run]
source = ["src"]
omit = ["tests/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
```

### Understanding Dependency Specifiers

```toml
# Exact version (rare — only for tools, not libraries)
"black==24.4.2"

# Minimum version (most common for libraries)
"fastapi>=0.111.0"

# Compatible release — "~=X.Y" means ">=X.Y, ==X.*"
# So ~=0.111.0 means >=0.111.0, <0.112
"fastapi~=0.111.0"

# Upper bound (prevent accidentally getting breaking changes)
"pydantic>=2.0,<3"

# Extras (optional dependencies of a package)
"uvicorn[standard]"     # includes websockets, httptools, watchfiles
"boto3[crt]"            # includes awscrt for faster S3/CloudFront
"pydantic[email]"       # includes email-validator

# Environment markers (rarely needed, but useful for platform-specific deps)
"pywin32>=306; sys_platform == 'win32'"
```

---

## Part 2: uv — The Modern Way

`uv` is a Python package installer and project manager written in Rust by Astral (same team as `ruff`). It's 10-100x faster than pip and is replacing pipenv/virtualenv/pip-tools for most use cases.

### Installing uv

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# With pip (if you already have Python)
pip install uv

# Verify
uv --version  # uv 0.2.x
```

### Starting a New Project with uv

```bash
# Initialize a new project
uv init logistics-integration
cd logistics-integration

# What uv creates:
# logistics-integration/
# ├── .python-version      ← pins Python version for this project
# ├── pyproject.toml       ← project config (uv fills in [build-system])
# ├── README.md
# └── src/
#     └── logistics_integration/
#         └── __init__.py

# .python-version contents:
cat .python-version
# 3.12.3

# The virtual environment is created automatically in .venv/
# uv uses the Python version from .python-version
```

### Adding Dependencies

```bash
# Add a runtime dependency
uv add fastapi
# This:
# 1. Resolves the dependency tree
# 2. Installs into .venv/
# 3. Updates pyproject.toml
# 4. Updates uv.lock (the lockfile)

# Add with version constraint
uv add "pydantic>=2.7.0,<3"

# Add with extras
uv add "uvicorn[standard]"

# Add a dev dependency
uv add --dev pytest pytest-asyncio respx

# Add to a named group
uv add --group test pytest pytest-asyncio
uv add --group linting ruff mypy

# Remove a dependency
uv remove httpx

# Upgrade a dependency
uv add "httpx>=0.28.0"  # or
uv lock --upgrade-package httpx
```

### uv.lock — The Lockfile

`uv.lock` is generated by uv and should **always be committed to git**. It records the exact version of every package (including transitive dependencies) for perfectly reproducible installs.

```
# Example uv.lock (simplified)
version = 1
requires-python = ">=3.11"

[[package]]
name = "fastapi"
version = "0.111.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pydantic", version = "2.7.4" },
    { name = "starlette", version = "0.37.2" },
]
sdist = { url = "...", hash = "sha256:abc123..." }
wheels = [
    { url = "...", hash = "sha256:def456..." },
]

[[package]]
name = "pydantic"
version = "2.7.4"
...
```

**Why lockfiles matter:** Without a lockfile, `pip install -r requirements.txt` might install slightly different versions on Tuesday vs Wednesday (if a new patch release drops). With `uv sync --frozen`, you always get the exact same environment.

### Key uv Commands

```bash
# Install all dependencies from lockfile (production use)
uv sync --frozen  # fails if uv.lock doesn't match pyproject.toml

# Install with dev dependencies
uv sync --group dev

# Install all optional groups
uv sync --all-groups

# Run a command in the project's virtual environment
uv run python -m pytest
uv run uvicorn app.main:app --reload
uv run python src/integration/main.py

# Run a one-off script without modifying the environment
uv run --with httpx python -c "import httpx; print(httpx.__version__)"

# Show the dependency tree
uv tree

# Show outdated packages
uv lock --upgrade --dry-run

# Pin a specific Python version for this project
uv python pin 3.11.9

# Install a Python version (uv manages Python installations too)
uv python install 3.11.9 3.12.3

# Create a lockfile without installing
uv lock

# Export to requirements.txt (for tools that need it)
uv export --format requirements-txt > requirements.txt
uv export --no-dev --format requirements-txt > requirements-prod.txt
```

### uv in Docker (Multi-Stage Build)

This is the pattern you'll use for all FDE deployments:

```dockerfile
# ─── Stage 1: Builder ────────────────────────────────────────────────────────
FROM python:3.11-slim as builder

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# Set environment variables
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/app/.venv \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Copy dependency files first (for layer caching)
COPY pyproject.toml uv.lock ./

# Install dependencies (no dev deps, frozen = exact lockfile versions)
RUN uv sync --frozen --no-dev --no-install-project

# Copy source code
COPY src/ ./src/

# Install the project itself
RUN uv sync --frozen --no-dev

# ─── Stage 2: Runtime ────────────────────────────────────────────────────────
FROM python:3.11-slim as runtime

# Don't run as root
RUN addgroup --system app && adduser --system --group app

# Copy only the virtual environment and source from builder
COPY --from=builder --chown=app:app /app/.venv /app/.venv
COPY --from=builder --chown=app:app /app/src /app/src

WORKDIR /app
USER app

# Put the venv on PATH
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH="/app/src"

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD python -c "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"

EXPOSE 8000
CMD ["uvicorn", "integration.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Build and run:
```bash
# Build (layer caching makes rebuilds fast)
docker build -t logistics-integration:latest .

# Run
docker run -p 8000:8000 \
  -e DATABASE_URL=postgresql://... \
  -e CARRIER_API_KEY=... \
  logistics-integration:latest
```

**Why this Dockerfile pattern is fast:**
1. `uv` itself is 10-100x faster than pip
2. The lockfile (`uv.lock`) is copied before source code — Docker layer caching means dependencies aren't reinstalled if only source code changed
3. Multi-stage build means the final image doesn't include uv, build tools, or test dependencies

---

## Part 3: Poetry — Full Project Management

Poetry is an alternative to uv that's been around longer (2018 vs 2023) and has a larger ecosystem of integrations. Many existing projects use Poetry. You need to be fluent in both.

### Installing Poetry

```bash
# Install Poetry (the officially recommended way)
curl -sSL https://install.python-poetry.org | python3 -

# Add to PATH (follow installer instructions)
# Usually: export PATH="$HOME/.local/bin:$PATH"

poetry --version  # Poetry (version 1.8.x)
```

### Starting a New Project with Poetry

```bash
# Create new project with scaffold
poetry new logistics-integration
cd logistics-integration

# Structure:
# logistics-integration/
# ├── pyproject.toml
# ├── README.md
# ├── logistics_integration/
# │   └── __init__.py
# └── tests/
#     └── __init__.py

# Initialize in an existing directory
poetry init
# Interactive wizard — fills in pyproject.toml
```

### Managing Dependencies with Poetry

```bash
# Add runtime dependency
poetry add fastapi
poetry add "pydantic>=2.7.0,<3"
poetry add "uvicorn[standard]"

# Add dev dependency (goes in [tool.poetry.dev-dependencies])
poetry add --group dev pytest pytest-asyncio ruff

# Add optional group
poetry add --group aws "aioboto3>=12.4.0"

# Remove dependency
poetry remove requests

# Upgrade dependency
poetry add "httpx@latest"

# Show all installed packages
poetry show

# Show dependency tree
poetry show --tree

# Check for outdated packages
poetry show --outdated
```

### poetry.lock File

Same concept as `uv.lock`. Generated by poetry, must be committed to git:

```bash
# Install exactly what's in poetry.lock (CI/CD, production)
poetry install --no-root --only main

# Install with dev dependencies
poetry install

# Regenerate the lock file (after manual pyproject.toml edits)
poetry lock

# Update all packages to their latest allowed versions
poetry update

# Update only a specific package
poetry update httpx
```

### Poetry's pyproject.toml Format

Poetry uses its own section format (different from PEP 621, but actively being migrated):

```toml
[tool.poetry]
name = "logistics-integration"
version = "1.2.0"
description = "Logistics API integration service"
authors = ["FDE Team <fde@company.com>"]
readme = "README.md"
packages = [{ include = "integration", from = "src" }]

[tool.poetry.dependencies]
python = ">=3.11,<3.13"
fastapi = ">=0.111.0,<0.113"
uvicorn = { version = ">=0.29.0", extras = ["standard"] }
pydantic = ">=2.7.0,<3"
httpx = ">=0.27.0"
asyncpg = ">=0.29.0"

# Optional dependency groups
aioboto3 = { version = ">=12.4.0", optional = true }

[tool.poetry.extras]
aws = ["aioboto3"]

[tool.poetry.group.dev.dependencies]
pytest = ">=8.0.0"
pytest-asyncio = ">=0.23.0"
pytest-cov = ">=5.0.0"
respx = ">=0.21.0"
ruff = ">=0.4.0"
mypy = ">=1.10.0"

[tool.poetry.group.test.dependencies]
pytest = ">=8.0.0"
pytest-asyncio = ">=0.23.0"
moto = { version = ">=5.0.0", extras = ["s3", "sqs"] }
fakeredis = ">=2.23.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### Poetry Virtual Environment Management

```bash
# Poetry creates a virtualenv automatically
# By default in: ~/.cache/pypoetry/virtualenvs/

# To keep it in the project directory (recommended for IDE integration):
poetry config virtualenvs.in-project true
# Now .venv/ is created inside the project directory

# Activate the virtualenv manually
poetry shell
# Or run without activating
poetry run python -m pytest
poetry run uvicorn app.main:app --reload

# Show where the virtualenv is
poetry env info
poetry env info --path  # just the path

# List all virtualenvs for this project
poetry env list

# Remove and recreate
poetry env remove python3.11
poetry install
```

### Poetry in Docker

```dockerfile
# Using Poetry in Docker — slightly more complex than uv
FROM python:3.11-slim as builder

# Install Poetry
ENV POETRY_VERSION=1.8.3 \
    POETRY_HOME="/opt/poetry" \
    POETRY_VENV="/opt/poetry-venv" \
    POETRY_NO_INTERACTION=1 \
    POETRY_VIRTUALENVS_IN_PROJECT=1 \
    POETRY_VIRTUALENVS_CREATE=1

RUN python3 -m venv $POETRY_VENV \
    && $POETRY_VENV/bin/pip install -U pip setuptools \
    && $POETRY_VENV/bin/pip install poetry==${POETRY_VERSION}

ENV PATH="${POETRY_VENV}/bin:${PATH}"

WORKDIR /app

# Copy dependency files
COPY pyproject.toml poetry.lock ./

# Install only production dependencies
RUN poetry install --no-root --only main

# Copy source code
COPY src/ ./src/

# Install the project
RUN poetry install --only main

# ─── Runtime stage ───────────────────────────────────────────────────────────
FROM python:3.11-slim as runtime

RUN addgroup --system app && adduser --system --group app

COPY --from=builder --chown=app:app /app/.venv /app/.venv
COPY --from=builder --chown=app:app /app/src /app/src

WORKDIR /app
USER app

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH="/app/src"

EXPOSE 8000
CMD ["uvicorn", "integration.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Part 4: Virtual Environments Deep Dive

### What a Virtual Environment Actually Is

A virtual environment is just a directory with:
1. A Python interpreter (or symlink to one)
2. A `site-packages/` directory where packages are installed
3. Scripts that set `PATH` and `PYTHONPATH` to use this directory

```
.venv/
├── bin/
│   ├── python → python3.11     ← symlink to system Python
│   ├── python3 → python
│   ├── activate                ← source this to activate
│   ├── uvicorn                 ← installed package scripts
│   └── pytest
├── include/
│   └── python3.11/
├── lib/
│   └── python3.11/
│       └── site-packages/
│           ├── fastapi/        ← installed packages
│           ├── pydantic/
│           └── ...
└── pyvenv.cfg                  ← Python version this was created with
```

When you run `source .venv/bin/activate`:
- `PATH` is prepended with `.venv/bin/`
- `python` now refers to `.venv/bin/python`
- `pip install X` installs to `.venv/lib/python3.11/site-packages/`

### Why This Matters for Customer Deployments

```bash
# Customer's server has Python 3.9 as system Python
# Your code requires Python 3.11

# uv manages Python installations
uv python install 3.11.9
uv python pin 3.11.9  # creates .python-version

# uv uses the pinned version for the project
uv sync  # uses Python 3.11.9 even if system has 3.9

# Check what Python the venv uses
.venv/bin/python --version  # Python 3.11.9
```

### .python-version File

```bash
# Pin the Python version for the entire project
# Used by uv, pyenv, and many CI systems
echo "3.11.9" > .python-version

# With pyenv installed, this automatically activates 3.11.9 when you cd into the directory
# With uv, it controls which Python version uv uses for the project
```

---

## Part 5: Managing Dependencies for Different Environments

### Dev vs Test vs Production

```toml
# pyproject.toml (uv style)
[project]
dependencies = [
    # Production: what runs in prod
    "fastapi>=0.111.0",
    "httpx>=0.27.0",
    "pydantic>=2.7.0",
    "asyncpg>=0.29.0",
]

[dependency-groups]
dev = [
    # Development: also includes test
    { include-group = "test" },
    "ruff>=0.4.0",
    "mypy>=1.10.0",
    "ipython>=8.24.0",
    "pre-commit>=3.7.0",
]
test = [
    # Testing only
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=5.0.0",
    "respx>=0.21.0",
    "moto[s3,sqs]>=5.0.0",
    "fakeredis>=2.23.0",
    "factory-boy>=3.3.0",
]
```

```bash
# Install for development (everything)
uv sync --all-groups

# Install for testing (test group only, not dev linting tools)
uv sync --group test

# Install for production (no groups)
uv sync --no-group dev --no-group test
# Or simply:
uv sync --only-group default  # just the [project] dependencies
```

### Handling Conflicting Dependencies

Sometimes two packages need incompatible versions of a shared dependency:

```bash
# Check for conflicts
uv lock --check
# or
uv tree --invert  # shows which packages require each dep

# Example conflict: package A needs pydantic<2, package B needs pydantic>=2
# Solution 1: pin the package that's causing the conflict
uv add "old-package==1.2.3"  # use the last version that was compatible

# Solution 2: use package overrides (uv 0.2+)
# In pyproject.toml:
[tool.uv]
override-dependencies = [
  "pydantic>=2.7.0,<3",  # force this version even if transitive deps want otherwise
]

# Solution 3: isolate in separate virtual env (last resort)
# Create a separate env for the conflicting package and call it as a subprocess
```

---

## Part 6: Private Package Mirrors

When you're in a customer environment with restricted internet access:

```bash
# Configure uv to use a private mirror
# In pyproject.toml:
[tool.uv]
index-url = "https://nexus.customer.internal/repository/pypi-proxy/simple/"

# Or in uv.toml (project-level) or ~/.config/uv/uv.toml (global):
[pip]
index-url = "https://nexus.customer.internal/repository/pypi-proxy/simple/"

# Or via environment variable:
UV_INDEX_URL="https://nexus.customer.internal/repository/pypi-proxy/simple/" uv sync

# For multiple indexes (e.g., PyPI + private):
[[tool.uv.extra-index-url]]
url = "https://nexus.customer.internal/repository/pypi-proxy/simple/"

# Authentication for private index
UV_INDEX_URL="https://user:password@nexus.customer.internal/..."

# Or use .netrc file (better, doesn't expose creds in env)
# ~/.netrc:
# machine nexus.customer.internal login myuser password mypassword
```

---

## Part 7: CI/CD Pipeline Configuration

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]  # test on multiple Python versions
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v2
        with:
          version: "0.2.x"
          enable-cache: true  # caches .venv between runs
          cache-dependency-glob: "uv.lock"
      
      - name: Set up Python
        run: uv python install ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: uv sync --all-groups --frozen
      
      - name: Run linting
        run: uv run ruff check .
      
      - name: Run type checking
        run: uv run mypy src/
      
      - name: Run tests
        run: uv run pytest tests/ -v --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build-docker:
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t logistics-integration:${{ github.sha }} .
      
      - name: Test Docker image starts
        run: |
          docker run -d --name test-app \
            -e DATABASE_URL=sqlite:///test.db \
            -p 8000:8000 \
            logistics-integration:${{ github.sha }}
          sleep 5
          curl -f http://localhost:8000/health || exit 1
          docker stop test-app
```

---

## Part 8: Comparing uv vs Poetry

| Feature | uv | Poetry |
|---|---|---|
| Speed | 10-100x faster than pip | 5-20x faster than pip (v1.8+) |
| Lock file format | `uv.lock` (TOML) | `poetry.lock` (TOML) |
| Python version management | Built-in (uv python install) | External (pyenv) |
| pyproject.toml format | PEP 621 standard | Poetry-specific sections |
| Dependency groups | PEP 735 `[dependency-groups]` | `[tool.poetry.group.X]` |
| Workspace support | Yes (uv workspaces) | Limited |
| Publish to PyPI | Not yet (use twine) | Built-in (poetry publish) |
| Maturity | Newer (2023), fast-moving | Established (2018), stable |
| IDE integration | Growing | Excellent (VSCode extension) |
| Docker usage | Simple, single binary | Requires installing Python pkg manager |

**Which to use:**
- **New projects:** `uv` — it's the future, much faster, simpler Docker usage
- **Existing Poetry projects:** stay with Poetry, migration isn't worth it unless you have a specific pain point
- **In an interview:** know both, express preference for `uv` for new work while respecting existing tooling

---

## Part 9: Troubleshooting Common Issues

### "Module not found" in production but works locally

```bash
# Problem: installed package locally but not in the right environment
# Diagnosis:
uv run python -c "import mypackage; print(mypackage.__file__)"
# vs
which python
python -c "import mypackage"

# Solution: always use `uv run` or activate the venv first
uv run uvicorn main:app  # correct
uvicorn main:app          # might use wrong Python

# In Docker: ensure PYTHONPATH and PATH are set correctly
ENV PATH="/app/.venv/bin:$PATH"
```

### Lock file conflicts (two developers)

```bash
# Developer A and Developer B both added packages simultaneously
# Git conflict in uv.lock

# Solution: accept either version and regenerate
git checkout --theirs uv.lock  # or --ours
uv lock  # regenerate the lockfile cleanly
git add uv.lock
git commit -m "Regenerate uv.lock after merge"
```

### Python version mismatch in CI

```bash
# Error: "requires Python >=3.11 but you have 3.9"
# In .python-version, ensure you have the right version:
cat .python-version
# 3.11.9

# In GitHub Actions, use the setup-uv action which reads .python-version
- uses: astral-sh/setup-uv@v2
- run: uv sync  # uses .python-version automatically

# In Docker, pin the base image version:
FROM python:3.11.9-slim  # exact version, not just "python:3.11"
```

### Slow installs in CI

```bash
# Problem: uv.lock is committed but CI reinstalls from scratch every time

# Solution 1: cache the .venv directory (GitHub Actions)
- uses: astral-sh/setup-uv@v2
  with:
    enable-cache: true
    cache-dependency-glob: "uv.lock"

# Solution 2: cache the uv download cache
- name: Cache uv
  uses: actions/cache@v4
  with:
    path: ~/.cache/uv
    key: ${{ runner.os }}-uv-${{ hashFiles('uv.lock') }}
```

---

## FDE Context Callouts

> **FDE Context: Python Version Constraints in Customer Environments**
>
> Before you start a customer project, ask: "What Python version is approved by your infosec team?" You'll often get "3.8" or "3.9" even though it's 2024. Test your code on that version before the first deployment. With `uv`, you can test multiple Python versions in CI easily. This conversation should happen in week 1, not on deployment day.

> **FDE Context: Lock Files Are Deployment Safety**
>
> A production deployment should use `uv sync --frozen` or `poetry install --no-update` — the exact versions in the lockfile. Never run `uv sync` (without `--frozen`) in production — it might resolve to slightly different versions. The mantra: "If it passed tests with these exact versions, deploy these exact versions."

> **FDE Context: The uv Docker Pattern Is Your Default**
>
> The multi-stage Dockerfile with uv is your starting template for every FDE project. It gives you: (a) fast builds due to layer caching, (b) a small final image (no build tools), (c) reproducible builds from the lockfile, and (d) a non-root user for security. Copy-paste this template at the start of every project.

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| `uv sync` works, Docker build fails | `uv.lock` not copied before `uv sync` in Dockerfile | Copy `pyproject.toml uv.lock ./` before `RUN uv sync` |
| CI installs different versions than local | Not using `--frozen` flag | Use `uv sync --frozen` in CI |
| `poetry install` takes 3+ minutes | No caching in CI | Cache `~/.cache/pypoetry` directory |
| Package installed but not found at runtime | Wrong PYTHONPATH in Docker | Set `PYTHONPATH=/app/src` and `PATH=/app/.venv/bin:$PATH` |
| Dependency conflicts after `uv add` | New package incompatible with existing tree | Run `uv tree` to see conflict, use override or find compatible version |
| `.venv` not used in IDE | VSCode using wrong Python interpreter | Cmd+Shift+P → "Python: Select Interpreter" → choose `.venv/bin/python` |
| `No module named 'app'` | Running uvicorn from wrong directory | Use `uvicorn src.app.main:app` or set `PYTHONPATH` |

---

## Interview Angle

**Q: "What's the difference between a lock file and a requirements.txt?"**

Great answer: "A requirements.txt typically lists direct dependencies with loose version ranges. A lock file (`uv.lock`, `poetry.lock`) records the exact version of every package in the entire dependency tree — including transitive dependencies — along with their hashes for security verification. If I add `fastapi>=0.111` to my requirements.txt, different teammates get different versions of starlette (a fastapi dependency) unless they all install at the same time. A lock file ensures everyone — and every CI run and every production deployment — gets byte-for-byte identical packages."

**Q: "Why use multi-stage Docker builds?"**

Great answer: "Multi-stage builds separate the build environment from the runtime environment. The build stage has uv, build tools, and dev dependencies. The runtime stage has only the virtual environment and source code. This means: (1) the final image is much smaller — no uv binary, no gcc, no test packages, (2) build secrets like API keys used during build don't leak into the final image, (3) the attack surface of the deployed container is minimal. The virtual environment is just a directory with Python files — it copies cleanly between stages."

**Q: "How do you handle a customer environment that doesn't have internet access?"**

Great answer: "First, I use `uv export --format requirements-txt > requirements.txt` to generate a traditional requirements file. Then I use `uv export` with the exact hashes to create a requirements file for an air-gapped install. For the actual installation, I either: (a) configure uv/pip to point to the customer's internal Nexus/Artifactory mirror, or (b) use `pip download` to download all packages with their wheels before the deployment and then `pip install --no-index --find-links ./wheels` offline. The lockfile is critical here — it tells me exactly what I need to download."

---

## Practice Exercise

**Task 1:** Create a new project called `shipment-tracker` using uv that has:
- Python 3.11 as minimum version
- FastAPI, httpx, pydantic v2 as runtime dependencies  
- pytest, respx, ruff, mypy as dev dependencies
- A separate `test` group with: moto (for AWS mocking), fakeredis
- A `Makefile` with targets: `make install`, `make test`, `make lint`, `make docker-build`

**Task 2:** Write a Dockerfile for this project using the multi-stage uv pattern. The final image should run uvicorn and not include any dev dependencies.

**Task 3:** Create a `.github/workflows/ci.yml` that: runs tests on Python 3.11 and 3.12, caches the uv virtual environment, runs ruff linting, builds the Docker image, and runs a health check against the built image.

**Task 4:** Simulate a version conflict: add a package that requires `pydantic<2` and show how to resolve it using `[tool.uv].override-dependencies`.

**Task 5:** Export the lockfile's production dependencies to `requirements.txt` (no dev deps, with hashes for security). Write a comment explaining why you'd provide this file even though you have a lockfile.

---

*Previous: [04-error-handling.md](./04-error-handling.md) | Next: [06-pandas-data-wrangling.md](./06-pandas-data-wrangling.md)*
