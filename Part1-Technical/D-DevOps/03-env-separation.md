# Environment Separation: Dev, Staging, and Production

**FDE Role Context:** Every enterprise customer will ask "can we test this in staging before it hits production?" Your answer must be an unambiguous yes, backed by a clean architecture they can inspect. Environment separation is also a compliance requirement — SOC 2, ISO 27001, HIPAA all mandate that production data cannot flow freely into development environments. As an FDE, you are the engineer who sets up and explains this architecture. Shaky answers here erode trust fast.

---

## Table of Contents

1. [Why Environment Separation Matters](#1-why-environment-separation-matters)
2. [The Three-Environment Minimum](#2-the-three-environment-minimum)
3. [The 12-Factor App Methodology](#3-the-12-factor-app-methodology)
4. [Config Management: Loading the Right Values](#4-config-management-loading-the-right-values)
5. [pydantic-settings: The Production Config Pattern](#5-pydantic-settings-the-production-config-pattern)
6. [Database Per Environment](#6-database-per-environment)
7. [Secrets Per Environment](#7-secrets-per-environment)
8. [Feature Flags](#8-feature-flags)
9. [Promoting Code Through Environments](#9-promoting-code-through-environments)
10. [The Hotfix Path](#10-the-hotfix-path)
11. [UAT With Customers](#11-uat-with-customers)
12. [Environment-Specific Docker Compose](#12-environment-specific-docker-compose)
13. [Environment-Specific ECS Deployments](#13-environment-specific-ecs-deployments)
14. [Common Failure Modes](#14-common-failure-modes)
15. [Interview Angles](#15-interview-angles)
16. [Practice Exercise](#16-practice-exercise)

---

## 1. Why Environment Separation Matters

### The Concrete Risks Without Separation

**Risk 1: Dev experiments corrupt production data**
```
Developer Alice is testing a new database migration. Her script has a bug that
deletes all records older than 30 days. Without a separate dev DB, Alice just
deleted 3 years of customer transaction history from production.
```

**Risk 2: Testing reveals real credentials**
```
The team is testing an email integration. Without a separate staging environment,
"test emails" go to real customers at 2am when the developer is running load tests.
```

**Risk 3: Compliance violations**
```
HIPAA requires PHI (Protected Health Information) never be used in development/testing
without de-identification. If dev pulls a production DB dump, you've created a HIPAA
violation. SOC 2 auditors will find it.
```

**Risk 4: "Works in dev, breaks in prod" with no systematic testing layer**
```
Without a staging environment that mirrors production, every prod deployment is
a gamble. When it breaks, you have no controlled environment to reproduce the issue.
```

### The FDE Pitch to Skeptical Customers

"Some of our smaller customers want to skip staging to move faster. Here's the conversation:"

> Customer: "Can we just push straight to prod? We're a small team and staging feels like overhead."
>
> You: "I understand the instinct. Here's the risk: our integration touches your ERP's production database. If there's a bug in our data transformation code, it writes corrupted records directly to live data. A staging environment is a 2-hour setup that protects you from a potential 2-day rollback incident. And once we go live, your own team will want to test config changes before they affect live orders. The protection is worth it."

---

## 2. The Three-Environment Minimum

```
┌─────────────────────────────────────────────────────────────────┐
│  ENVIRONMENT ARCHITECTURE                                       │
│                                                                 │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐   │
│  │  DEVELOPMENT  │    │   STAGING     │    │  PRODUCTION   │   │
│  │               │    │   (UAT)       │    │               │   │
│  │ localhost /   │    │ AWS (mirrored │    │ AWS (live     │   │
│  │ dev cluster   │    │  prod config) │    │  traffic)     │   │
│  │               │    │               │    │               │   │
│  │ Dev DB        │    │ Staging DB    │    │ Prod DB       │   │
│  │ (Docker)      │    │ (RDS t3.micro)│    │ (RDS m6i, HA) │   │
│  │               │    │               │    │               │   │
│  │ Dev secrets   │    │ Staging       │    │ Prod secrets  │   │
│  │ (.env.local)  │    │ secrets       │    │ (Secrets Mgr) │   │
│  │               │    │ (Secrets Mgr) │    │               │   │
│  └───────────────┘    └───────────────┘    └───────────────┘   │
│         │                    │                    │            │
│    Developers           Customer UAT         Live users        │
│    run locally          + final testing      + real data       │
└─────────────────────────────────────────────────────────────────┘

Code flows only in one direction: dev → staging → prod
Data NEVER flows backwards: prod data NEVER goes to staging/dev
```

### Environment Characteristics

| Aspect | Development | Staging | Production |
|---|---|---|---|
| Purpose | Building + unit testing | Integration + UAT | Live traffic |
| Who accesses | Developers | Devs + customer UAT team | End users |
| Deployment trigger | Manual / on PR | On merge to main | Manual approval |
| Database | Docker PostgreSQL, reset freely | RDS, can reset with migration | RDS, Multi-AZ, never reset |
| Secrets | `.env.local` files | AWS Secrets Manager | AWS Secrets Manager |
| Scale | Minimal | ~50% of prod capacity | Full capacity + autoscaling |
| Monitoring | Logs to console | CloudWatch | CloudWatch + alerting |
| Downtime acceptable | Yes | Brief, with notice | No |
| Data | Test/synthetic only | Anonymized or synthetic | Real customer data |

### When You Need More Than Three

For larger customers or complex integrations, you might need:

```
feature-branch → dev → integration → staging → prod
                              ↑
                    shared integration testing
                    (multiple teams' services)
```

But start with three. Most FDE deployments run well on three.

---

## 3. The 12-Factor App Methodology

The canonical reference for building portable, environment-agnostic applications. For FDE work, the most critical factors:

### Factor III: Config — Store Config in the Environment

```python
# WRONG: hardcoded config
DATABASE_URL = "postgresql://user:prod_pass@prod-db.example.com:5432/mydb"

# WRONG: config file per environment (brittle, secrets in git)
# config_prod.py, config_staging.py, config_dev.py

# RIGHT: config from environment variables
import os
DATABASE_URL = os.environ["DATABASE_URL"]
# The caller (ECS task definition, .env.local, k8s ConfigMap) provides the value
# The code is identical across all environments
```

### Factor XI: Logs — Treat Logs as Event Streams

```python
# WRONG: write to files
logging.basicConfig(filename='/var/log/myapp.log')

# RIGHT: write to stdout/stderr (the container runtime/ECS/CloudWatch handles routing)
logging.basicConfig(stream=sys.stdout, level=logging.INFO)
```

### Factor I: Codebase — One codebase, many deploys

```
ONE git repository → deployed to dev, staging, prod
Only configuration (environment variables) differs between environments
The same Docker image artifact is deployed to staging and then promoted to prod
(never rebuild the image per environment)
```

**This is a critical FDE point:** The same Docker image that passes staging tests is the image that goes to production. You never rebuild. Rebuilding introduces risk that the prod build behaves differently from the tested staging build.

---

## 4. Config Management: Loading the Right Values

### The Loading Hierarchy

```
Priority (highest to lowest):
1. Actual environment variables (set on OS / ECS task definition)
2. .env.local file (developer's personal overrides, gitignored)
3. .env file (shared non-secret defaults, optionally committed)
4. Default values in code (fallbacks for optional config)
```

### File Structure

```
project/
├── .env                    # committed — non-secret defaults
├── .env.local              # gitignored — personal overrides + secrets
├── .env.staging            # committed — staging defaults (no secrets)
├── .env.production         # committed — prod defaults (no secrets, secrets come from AWS)
├── .gitignore              # must include .env.local, *.env.secrets
└── src/
    └── config.py           # pydantic-settings Settings class
```

```bash
# .env (committed — safe non-secrets)
APP_ENV=development
LOG_LEVEL=debug
APP_PORT=8000
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8000

# .env.local (gitignored — NEVER committed)
# Developer-specific secrets and overrides
DATABASE_URL=postgresql+asyncpg://appuser:localpass@localhost:5432/appdb
REDIS_URL=redis://localhost:6379/0
APP_SECRET_KEY=local-dev-key-not-for-prod-32chars!!
EXTERNAL_API_KEY=dev_key_abc123

# .env.staging (committed — staging-specific non-secrets)
APP_ENV=staging
LOG_LEVEL=info
ALLOWED_ORIGINS=https://staging.myapp.example.com
# DATABASE_URL comes from Secrets Manager in ECS task def — NOT here
```

```bash
# .gitignore
.env.local
.env.*.local
*.env.secrets
.env.production.local
```

---

## 5. pydantic-settings: The Production Config Pattern

`pydantic-settings` is the standard for Python apps. It:
1. Reads from environment variables automatically
2. Validates types and required fields at startup (fail fast, not silently)
3. Provides type hints and IDE completion
4. Supports `.env` file loading
5. Allows nested settings for complex configs

### Basic Settings Class

```python
# src/config.py
from pydantic import Field, field_validator, model_validator
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Literal
import secrets


class DatabaseSettings(BaseSettings):
    """Database connection settings."""
    url: str = Field(
        description="PostgreSQL connection URL",
        examples=["postgresql+asyncpg://user:pass@host:5432/db"]
    )
    pool_size: int = Field(default=10, ge=1, le=50)
    max_overflow: int = Field(default=5, ge=0, le=20)
    pool_timeout: int = Field(default=30, ge=5, le=120)
    echo_sql: bool = Field(default=False)

    model_config = SettingsConfigDict(env_prefix="DATABASE_")


class RedisSettings(BaseSettings):
    url: str = Field(default="redis://localhost:6379/0")
    max_connections: int = Field(default=10)

    model_config = SettingsConfigDict(env_prefix="REDIS_")


class Settings(BaseSettings):
    """
    Application settings loaded from environment variables.

    Priority: env vars > .env.local > .env > defaults
    All secrets come from env vars (injected by ECS/K8s from Secrets Manager)
    """

    # ── App Identity ──────────────────────────────────────────────────────────
    app_env: Literal["development", "test", "staging", "production"] = Field(
        default="development",
        description="Current deployment environment"
    )
    app_name: str = Field(default="myapp")
    app_version: str = Field(default="unknown")

    # ── Security ──────────────────────────────────────────────────────────────
    secret_key: str = Field(
        min_length=32,
        description="Application secret key for JWT/session signing"
    )
    allowed_origins: list[str] = Field(
        default=["http://localhost:3000", "http://localhost:8000"]
    )

    # ── Database ──────────────────────────────────────────────────────────────
    database_url: str = Field(
        description="Database connection URL (injected from Secrets Manager in non-dev)"
    )
    database_pool_size: int = Field(default=10, ge=1, le=50)
    database_echo_sql: bool = Field(default=False)

    # ── Redis ─────────────────────────────────────────────────────────────────
    redis_url: str = Field(default="redis://localhost:6379/0")

    # ── External Integrations ─────────────────────────────────────────────────
    erp_api_url: str = Field(description="ERP system API base URL")
    erp_api_key: str = Field(description="ERP system API key")
    erp_timeout_seconds: int = Field(default=30, ge=5, le=300)

    # ── Observability ─────────────────────────────────────────────────────────
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"] = Field(
        default="INFO"
    )
    log_format: Literal["json", "text"] = Field(default="json")
    sentry_dsn: str | None = Field(default=None)

    # ── Feature Flags ─────────────────────────────────────────────────────────
    feature_new_sync_engine: bool = Field(default=False)
    feature_bulk_operations: bool = Field(default=False)

    # ── AWS ───────────────────────────────────────────────────────────────────
    aws_region: str = Field(default="us-east-1")
    s3_bucket_name: str | None = Field(default=None)

    # ── Settings Config ───────────────────────────────────────────────────────
    model_config = SettingsConfigDict(
        # Load from .env files in priority order
        env_file=[".env", ".env.local"],
        env_file_encoding="utf-8",
        # Ignore extra fields in env (don't fail on unknown vars)
        extra="ignore",
        # Case-insensitive matching: DATABASE_URL and database_url both work
        case_sensitive=False,
    )

    # ── Validators ────────────────────────────────────────────────────────────
    @field_validator("secret_key")
    @classmethod
    def validate_secret_key(cls, v: str, info) -> str:
        # Prevent using dev default in production
        if v == "dev-only-insecure-key-change-in-prod":
            import os
            env = os.environ.get("APP_ENV", "development")
            if env == "production":
                raise ValueError(
                    "Default development secret_key must not be used in production"
                )
        return v

    @model_validator(mode="after")
    def validate_production_settings(self) -> "Settings":
        if self.app_env == "production":
            # Enforce stricter settings in production
            if self.database_echo_sql:
                raise ValueError("database_echo_sql must be False in production")
            if not self.sentry_dsn:
                import warnings
                warnings.warn("sentry_dsn not configured in production — errors won't be tracked")
        return self

    # ── Computed Properties ───────────────────────────────────────────────────
    @property
    def is_production(self) -> bool:
        return self.app_env == "production"

    @property
    def is_development(self) -> bool:
        return self.app_env == "development"

    @property
    def cors_origins(self) -> list[str]:
        return self.allowed_origins


# Singleton — loaded once at import time
# If any required field is missing, this raises at startup (not mid-request)
settings = Settings()
```

### Using Settings in FastAPI

```python
# src/main.py
from src.config import settings
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    # Disable docs in production (optional — depends on your policy)
    docs_url="/docs" if not settings.is_production else None,
    redoc_url="/redoc" if not settings.is_production else None,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.on_event("startup")
async def startup():
    print(f"Starting {settings.app_name} in {settings.app_env} environment")
```

### Settings for Testing

```python
# tests/conftest.py
import pytest
from unittest.mock import patch

@pytest.fixture(autouse=True)
def test_settings(monkeypatch):
    """Override settings for test environment."""
    monkeypatch.setenv("APP_ENV", "test")
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://testuser:testpass@localhost:5432/testdb")
    monkeypatch.setenv("SECRET_KEY", "test-secret-key-exactly-32-chars!!")
    monkeypatch.setenv("ERP_API_URL", "http://localhost:9999")  # mock server
    monkeypatch.setenv("ERP_API_KEY", "test-key")
    # Re-import settings after patching env
    from importlib import reload
    import src.config
    reload(src.config)
    from src.config import settings
    return settings
```

---

## 6. Database Per Environment

### The Architecture

```
dev:      Docker PostgreSQL on localhost:5432 (appdb)
staging:  AWS RDS PostgreSQL t3.micro (myapp-staging.xxxxx.rds.amazonaws.com)
prod:     AWS RDS PostgreSQL m6i.large, Multi-AZ (myapp-prod.xxxxx.rds.amazonaws.com)
```

**Why not share databases?**
1. A developer running `DELETE FROM orders WHERE status='pending'` to clean up tests would delete staging orders
2. Migrations run in dev might break staging if they're not tested first
3. Performance testing in staging would impact production queries on a shared DB
4. Compliance: dev team members often can't have access to production data

### Migration Management Per Environment

```bash
# src/alembic/env.py — alembic configuration
from src.config import settings

def get_url():
    return settings.database_url

# Run migrations per environment:
# Dev:
APP_ENV=development alembic upgrade head

# Staging:
APP_ENV=staging alembic upgrade head
# (staging DATABASE_URL is injected from env or Secrets Manager)

# Prod (run via CI/CD pipeline step, not manually):
APP_ENV=production alembic upgrade head
```

### Data Policies Per Environment

```
Development:
- Seed data: scripts in /seeds/ that populate test data
- Reset: docker-compose down -v && docker-compose up recreates clean DB
- Never use production data dumps — use anonymized/synthetic data

Staging:
- Seed data: similar to dev seeds + realistic test scenarios
- Reset: allowed, but coordinate with UAT team
- Can use anonymized production data if needed for realistic testing
  (with PII removed/randomized — never real customer names/emails)
- Weekly reset during early integration phase

Production:
- Never reset
- Point-in-time recovery enabled (automated backups)
- Schema changes only via tested Alembic migrations
- Data changes only via application layer (never direct SQL by developers)
```

### Synthetic Data Generator

```python
# scripts/seed_dev_data.py
"""Generate realistic synthetic data for development/staging environments."""
import asyncio
import random
from faker import Faker
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

fake = Faker()

async def seed_data(db_url: str):
    engine = create_async_engine(db_url)
    async with AsyncSession(engine) as session:
        # Create synthetic companies
        for _ in range(10):
            company = Company(
                name=fake.company(),
                erp_system="SAP",
                # Never use real company names or data
                created_at=fake.date_time_this_year(),
            )
            session.add(company)

        # Create synthetic orders with realistic patterns
        for _ in range(500):
            order = Order(
                order_number=f"ORD-{random.randint(10000, 99999)}",
                status=random.choice(["pending", "processing", "shipped", "delivered"]),
                amount=round(random.uniform(100, 50000), 2),
                # etc.
            )
            session.add(order)

        await session.commit()
    print(f"Seeded development data into {db_url.split('@')[1]}")  # don't log credentials

if __name__ == "__main__":
    from src.config import settings
    asyncio.run(seed_data(settings.database_url))
```

---

## 7. Secrets Per Environment

### The Pattern

```
Development:
  .env.local (gitignored, on developer's machine)
  └── POSTGRES_PASSWORD=local_dev_only
  └── EXTERNAL_API_KEY=dev_api_key_from_vendor_sandbox

Staging:
  AWS Secrets Manager (region: us-east-1)
  └── /myapp/staging/database_url
  └── /myapp/staging/erp_api_key        ← vendor's sandbox/test key
  └── /myapp/staging/secret_key

Production:
  AWS Secrets Manager (region: us-east-1)
  └── /myapp/production/database_url
  └── /myapp/production/erp_api_key     ← vendor's production key
  └── /myapp/production/secret_key
  Access: only the ECS task role + break-glass admin role
```

### Loading Secrets at Startup

```python
# src/secrets.py
"""Load secrets from AWS Secrets Manager for non-development environments."""
import json
import boto3
from botocore.exceptions import ClientError
from src.config import settings
import logging

logger = logging.getLogger(__name__)


def load_secret(secret_name: str, region: str = "us-east-1") -> dict | str:
    """
    Retrieve a secret from AWS Secrets Manager.
    Returns parsed JSON dict if value is JSON, otherwise returns raw string.
    """
    client = boto3.client("secretsmanager", region_name=region)
    try:
        response = client.get_secret_value(SecretId=secret_name)
        secret = response.get("SecretString") or response.get("SecretBinary", b"").decode()
        try:
            return json.loads(secret)
        except json.JSONDecodeError:
            return secret
    except ClientError as e:
        error_code = e.response["Error"]["Code"]
        if error_code == "ResourceNotFoundException":
            raise ValueError(f"Secret '{secret_name}' not found in Secrets Manager")
        elif error_code == "AccessDeniedException":
            raise PermissionError(f"IAM role lacks permission to read '{secret_name}'")
        raise


def inject_secrets_into_env():
    """
    For local development simulation: load secrets from Secrets Manager
    and inject into environment variables so Settings picks them up.
    Only needed when running locally against staging Secrets Manager.
    Usually secrets are injected at the ECS task level.
    """
    if settings.is_development:
        return   # dev uses .env.local

    import os
    secret_path = f"/myapp/{settings.app_env}"

    db_secret = load_secret(f"{secret_path}/database")
    os.environ["DATABASE_URL"] = (
        f"postgresql+asyncpg://{db_secret['username']}:{db_secret['password']}"
        f"@{db_secret['host']}:{db_secret['port']}/{db_secret['dbname']}"
    )
    logger.info(f"Loaded database credentials from Secrets Manager ({secret_path}/database)")
```

---

## 8. Feature Flags

Feature flags let you deploy code that isn't yet activated, and enable/disable features per environment.

### Simple Boolean Flag via Environment Variables

```python
# src/config.py (in Settings class)
class Settings(BaseSettings):
    # Feature flags — controlled per environment via env vars
    feature_new_sync_engine: bool = Field(
        default=False,
        description="Enable new high-performance sync engine (in testing)"
    )
    feature_bulk_import: bool = Field(
        default=False,
        description="Enable bulk import endpoint (staging only for now)"
    )
    feature_webhooks_v2: bool = Field(
        default=False,
        description="New webhook delivery system"
    )
```

```python
# src/routers/sync.py
from src.config import settings

@router.post("/sync")
async def sync_orders(request: SyncRequest):
    if settings.feature_new_sync_engine:
        # New implementation (tested in staging, not yet in prod)
        return await new_sync_engine.sync(request)
    else:
        # Stable implementation
        return await legacy_sync_engine.sync(request)
```

Environment variable configuration:
```bash
# dev .env.local — developer testing the new feature
FEATURE_NEW_SYNC_ENGINE=true

# staging ECS task definition — all features on for UAT
FEATURE_NEW_SYNC_ENGINE=true
FEATURE_BULK_IMPORT=true
FEATURE_WEBHOOKS_V2=true

# production ECS task definition — only proven features on
FEATURE_NEW_SYNC_ENGINE=false    # still validating in staging
FEATURE_BULK_IMPORT=true         # validated, shipping
FEATURE_WEBHOOKS_V2=false        # not ready
```

### LaunchDarkly (Enterprise Feature Flags)

For more sophisticated flagging (percentage rollouts, per-customer flags, A/B testing):

```python
# pip install launchdarkly-server-sdk
import ldclient
from ldclient.config import Config

ldclient.set_config(Config(settings.launchdarkly_sdk_key))
ld_client = ldclient.get()

def is_feature_enabled(flag_key: str, user_key: str, default: bool = False) -> bool:
    """Check if a feature flag is enabled for a specific user/customer."""
    context = ldclient.Context.builder(user_key).build()
    return ld_client.variation(flag_key, context, default)

# Usage:
if is_feature_enabled("new-sync-engine", customer_id):
    return await new_sync_engine.sync(request)
```

LaunchDarkly allows:
- Enable for specific customers only (pilot rollout)
- Percentage-based rollout (10% → 50% → 100%)
- Kill switch: turn off in prod without redeploying
- Different values per environment (managed in LaunchDarkly dashboard, not code)

**FDE Context:** For your own integration service, simple boolean env vars are usually sufficient. Customers running their own apps may use LaunchDarkly or similar — you should know the concept.

---

## 9. Promoting Code Through Environments

### The Immutable Artifact Pattern

```
┌──────────────────────────────────────────────────────────────────┐
│  PROMOTION PIPELINE                                              │
│                                                                  │
│  1. Developer pushes to feature branch                          │
│     → CI runs tests                                              │
│                                                                  │
│  2. PR merged to main                                            │
│     → CI builds Docker image: myapp:sha-abc1234                 │
│     → Image pushed to ECR                                        │
│     → SAME IMAGE deployed to staging                            │
│                                                                  │
│  3. Staging validation passes (UAT sign-off)                    │
│     → SAME IMAGE (sha-abc1234) deployed to production           │
│     → No rebuild, no changes between staging and prod           │
│                                                                  │
│  Key: The artifact that was tested IS the artifact that ships.  │
└──────────────────────────────────────────────────────────────────┘
```

### Git Branching Strategy

```
main branch        ← always deployable, reflects production
  │
  ├── feature/sync-engine-v2    ← developer work
  ├── feature/bulk-import
  └── hotfix/order-null-pointer ← emergency prod fix (see hotfix section)
```

Simple trunk-based development:
1. Work in short-lived feature branches
2. PRs require CI pass + 1 reviewer approval
3. Merge to main → auto-deploy to staging
4. Manual approval gate → deploy to production

Alternatively (GitFlow — more overhead but clearer):
```
main    ← production code only
develop ← integration branch, deploys to staging
  └── feature/* branches merge to develop
  └── release/* branch for final staging freeze before prod
  └── hotfix/* branches from main
```

### GitHub Actions Promotion

```yaml
# CI/CD flow:
# push to main → deploy-staging (automatic)
# manual approval → deploy-prod

jobs:
  deploy-staging:
    needs: build
    environment: staging
    # Automatic — no approval required
    steps:
      - name: Deploy to ECS staging
        # ... ECS deploy steps

  deploy-prod:
    needs: deploy-staging
    environment:
      name: production
      # GitHub pauses here for manual approval
      # Configured in: Settings → Environments → production → Protection rules
      # Required reviewers: [platform-team]
    steps:
      - name: Deploy to ECS production
        # Uses SAME image tag that was deployed to staging
        # ${{ needs.build.outputs.image-tag }} ← this is the key
```

---

## 10. The Hotfix Path

A hotfix is an emergency fix for a production bug that cannot wait for the normal dev → staging → prod cycle.

**This path is rare, risky, and must be controlled.**

### When to Use a Hotfix

- Production is actively broken for users
- A security vulnerability has been exploited or disclosed
- Data corruption is ongoing

### Hotfix Procedure

```
1. Create hotfix branch FROM MAIN (not develop):
   git checkout main
   git checkout -b hotfix/order-null-pointer-crash

2. Make the minimal fix — only the bug, nothing else

3. Write a test that fails on main and passes on the fix

4. Push branch → CI runs automatically

5. Code review (at least one reviewer, even in emergency)

6. Merge to main → auto-deploys to staging

7. Abbreviated staging validation:
   - Does the specific bug reproduce in staging? No → good.
   - Does the health check pass? Yes → good.
   - 15-minute smoke test (not full UAT)

8. Deploy to production with explicit approval
   (skip the normal 24-hour staging soak)

9. AFTER INCIDENT: backport to develop if using GitFlow
   git checkout develop && git merge hotfix/order-null-pointer-crash

10. Postmortem: why did this bug make it to prod?
    Add a regression test. Fix the process gap.
```

### Emergency Rollback (Faster Than Hotfix)

Before even writing a hotfix, consider rolling back:

```bash
# Find the previous known-good task definition revision
aws ecs describe-task-definition \
  --task-definition myapp-task \
  --query 'taskDefinition.revision'
# Returns: 47 (current)

# Previous good revision: 46
# Redeploy it directly (no code change needed)
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-service \
  --task-definition myapp-task:46 \
  --force-new-deployment

# This takes ~2 minutes vs writing + deploying a fix
```

---

## 11. UAT With Customers

UAT (User Acceptance Testing) is when your customer's team validates that the integration works in staging before it goes live.

### Setting Up Staging Access for Customers

```
1. Staging URL: https://staging-api.myapp.example.com
   - Protected by HTTP Basic Auth or VPN (not open to internet)
   - Or: behind an allowlist of the customer's office IP ranges

2. Staging credentials for the customer:
   - Separate staging API keys (not prod keys)
   - Read-only database access if needed for their testing team
   - Dashboard or admin panel at https://staging.myapp.example.com/admin

3. Staging data:
   - Pre-populate with realistic test scenarios
   - Document the test data: "Order ORD-12345 is in 'pending' state"
   - Provide a test data reset mechanism if needed

4. Communication:
   - Dedicated Slack channel: #uat-[customer-name]
   - Scheduled UAT sessions (don't leave them to discover issues alone)
   - Known issues list: what's working, what's not yet built
```

### UAT Test Plan Template

```markdown
# UAT Test Plan: [Customer] Integration

**Staging URL:** https://staging-api.myapp.example.com
**Test Period:** [date range]
**Customer Contact:** [name, email]
**FDE Contact:** [your name]

## Scope
What we're testing in this UAT cycle:
- [ ] Order ingestion from ERP → our system
- [ ] Status updates from our system → ERP
- [ ] Error handling (malformed records, network timeouts)
- [ ] Performance: 1000 orders/hour ingestion

## Not In Scope (will be tested in next UAT cycle)
- Bulk historical data import
- Real-time webhook notifications

## Test Scenarios

### Scenario 1: Happy Path Order Sync
**Given:** A new order is created in the ERP system (test order: ORD-TEST-001)
**When:** The sync runs (every 5 minutes in staging)
**Then:** The order appears in our system within 6 minutes

### Scenario 2: Error Handling — Missing Required Field
**Given:** An order with missing `ship_to_address` is in ERP (test order: ORD-ERR-001)
**When:** The sync runs
**Then:** Our system creates an error record, the ERP is notified, the order is NOT created

### Scenario 3: Performance
**Given:** 500 orders pending in ERP
**When:** Initial sync runs
**Then:** All 500 orders processed within 30 minutes with no data loss

## Sign-Off
UAT is complete when:
- All test scenarios pass
- Customer team lead signs off in this document
- Zero P1/P2 bugs outstanding (P3 bugs acceptable with tracking)

Customer sign-off: _________________ Date: _________
FDE sign-off: _________________ Date: _________
```

---

## 12. Environment-Specific Docker Compose

### Dev docker-compose.yml (local development)

```yaml
# docker-compose.yml — local development
version: "3.9"

services:
  app:
    build:
      context: .
      target: runtime
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src:ro    # hot reload
    command:
      - uvicorn
      - src.main:app
      - --host
      - "0.0.0.0"
      - --port
      - "8000"
      - --reload
    env_file:
      - .env
      - .env.local           # secrets — gitignored
    environment:
      APP_ENV: development
      LOG_LEVEL: DEBUG
      DATABASE_ECHO_SQL: "true"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  postgres_data:
```

### Production-Check: Running Against Staging Secrets Locally

```yaml
# docker-compose.staging-check.yml
# Use to test your local build against staging infrastructure
# Run: docker-compose -f docker-compose.staging-check.yml up

version: "3.9"

services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      APP_ENV: staging
      # These vars pull from your local AWS credentials → Secrets Manager
      AWS_DEFAULT_REGION: us-east-1
    env_file:
      - .env.staging-check   # contains real staging env vars (gitignored!)
    # No postgres/redis services — connects to real staging AWS resources
```

---

## 13. Environment-Specific ECS Deployments

In ECS, each environment is a separate ECS cluster or at minimum separate ECS services and task definitions.

### ECS Resource Naming Convention

```
Cluster: myapp-{env}           myapp-staging, myapp-production
Service: myapp-{env}-service   myapp-staging-service, myapp-production-service
Task Def: myapp-{env}-task     myapp-staging-task, myapp-production-task
Log Group: /ecs/myapp-{env}    /ecs/myapp-staging, /ecs/myapp-production
```

### Environment-Specific Task Definitions

```json
// Staging Task Definition
{
  "family": "myapp-staging-task",
  "containerDefinitions": [{
    "name": "myapp",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:PLACEHOLDER",
    "cpu": 256,
    "memory": 512,
    "environment": [
      {"name": "APP_ENV", "value": "staging"},
      {"name": "LOG_LEVEL", "value": "DEBUG"},
      {"name": "DATABASE_POOL_SIZE", "value": "5"},
      {"name": "FEATURE_NEW_SYNC_ENGINE", "value": "true"},
      {"name": "FEATURE_BULK_IMPORT", "value": "true"}
    ],
    "secrets": [
      {
        "name": "DATABASE_URL",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/staging/database_url"
      },
      {
        "name": "SECRET_KEY",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/staging/secret_key"
      },
      {
        "name": "ERP_API_KEY",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/staging/erp_api_key"
      }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapp-staging",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
```

```json
// Production Task Definition (same structure, different values)
{
  "family": "myapp-production-task",
  "containerDefinitions": [{
    "name": "myapp",
    "cpu": 1024,       // 4x more CPU
    "memory": 2048,    // 4x more memory
    "environment": [
      {"name": "APP_ENV", "value": "production"},
      {"name": "LOG_LEVEL", "value": "INFO"},
      {"name": "DATABASE_POOL_SIZE", "value": "20"},
      {"name": "FEATURE_NEW_SYNC_ENGINE", "value": "false"},  // not yet
      {"name": "FEATURE_BULK_IMPORT", "value": "true"}        // validated
    ],
    "secrets": [
      // Different ARNs — production secrets
      {
        "name": "DATABASE_URL",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/database_url"
      }
      // ... etc
    ]
  }]
}
```

---

## 14. Common Failure Modes

### Failure 1: "It works in staging but fails in prod"

**Root cause analysis framework:**

1. **Config difference:** Did you miss an environment variable in the prod task definition?
   ```bash
   # Compare env vars between staging and prod task definitions
   aws ecs describe-task-definition --task-definition myapp-staging-task \
     --query 'taskDefinition.containerDefinitions[0].environment' | sort > staging-env.json
   aws ecs describe-task-definition --task-definition myapp-production-task \
     --query 'taskDefinition.containerDefinitions[0].environment' | sort > prod-env.json
   diff staging-env.json prod-env.json
   ```

2. **Scale difference:** Staging has 1 task, prod has 5 tasks — race condition only visible at scale
   ```
   Symptom: Works with 1 concurrent user, fails under load
   Fix: Run load tests in staging with production-level concurrency
   ```

3. **Data difference:** Production has edge cases that staging test data doesn't cover
   ```
   Symptom: Works for all staged orders, fails for specific real orders
   Fix: Better anonymized production data in staging, more comprehensive test cases
   ```

4. **Credentials difference:** Staging uses a different external API key with different permissions
   ```
   Symptom: API calls succeed in staging, return 403 in prod
   Fix: Verify the production API key has all required scopes/permissions
   ```

### Failure 2: Developer Accidentally Points at Production

```python
# Add an explicit check in startup to prevent this
@app.on_event("startup")
async def startup_safety_check():
    if settings.is_production:
        # Verify we're running in a context that should be production
        import os
        hostname = os.environ.get("HOSTNAME", "")
        if "local" in hostname or "dev" in hostname:
            raise RuntimeError(
                "Production config loaded on a non-production host! "
                "Check your .env.local file — it should NOT set APP_ENV=production"
            )
```

### Failure 3: Missing Required Env Var (Silent Failure)

```python
# BAD — silent default means misconfigured prod uses empty string
erp_api_url: str = ""

# GOOD — required field, pydantic raises at startup if not set
erp_api_url: str = Field(
    description="ERP system API base URL — REQUIRED"
)
# If ERP_API_URL is not in environment, app crashes with clear error at startup
# "1 validation error for Settings\nerp_api_url\n  Field required"
```

### Failure 4: Secrets Rotated in Production, Not Updated in Staging

```
Scenario: Your team rotated the ERP API key in production (good security practice).
But the ERP vendor also required staging to use a new key format.
Staging was not updated → staging UAT fails because it's using old key format.

Solution: Document secrets per environment, use a secrets rotation checklist.
When vendor changes API key format, update BOTH staging and prod Secrets Manager entries.
```

---

## 15. Interview Angles

### Q: "How do you handle environment separation for a new customer integration?"

**Great answer structure:**
1. Three environments minimum: dev (Docker Compose locally), staging (AWS, mirrors prod config), prod
2. Same Docker image artifact promoted from staging to prod — never rebuild
3. Config via pydantic-settings reading environment variables — code is identical, only env vars change
4. Separate database instances per environment — never share data
5. Secrets in AWS Secrets Manager per environment (`/myapp/staging/` vs `/myapp/production/`)
6. Feature flags in environment variables for safe gradual rollout
7. GitHub Actions deploys to staging on every main branch merge, production requires manual approval gate
8. UAT session with customer in staging before production go-live — I create a formal test plan

### Q: "A customer says they found a bug in production but they can't reproduce it in staging. What do you do?"

**Answer:**
1. First: check if the prod and staging environments have the same config — compare task definition env vars, secrets versions
2. Check if the bug is data-dependent — staging might not have the specific data shape that triggers it
3. Look at prod logs at the time of the bug — can I see the actual input that triggered it?
4. If I can identify the problematic input pattern, add it to staging test data and reproduce
5. If still can't reproduce, add more logging around the suspect area, deploy to staging, create the conditions
6. Never debug directly in production — always reproduce in a lower environment first

### Q: "Why shouldn't developers have direct access to the production database?"

**Answer:**
1. Accidental data modification — a `WHERE` clause typo in a `DELETE` can destroy data
2. Compliance — SOC 2 and HIPAA require access to be minimal, audited, and justified
3. Encourages bypassing the application layer for fixes, which creates undocumented state changes
4. Even read-only access exposes production PII/business data that shouldn't be on developer laptops

The answer: all data changes go through the application layer. Emergency prod access requires a break-glass procedure with audit logging, and should result in a proper code fix immediately after.

---

## 16. Practice Exercise

### Exercise: Build a Three-Environment Setup

**Setup:** Use the FastAPI app from the [docker.md exercise](./01-docker.md).

**Task 1: Settings Class**
Create `src/config.py` with a pydantic-settings `Settings` class that:
- Has fields for: `app_env`, `database_url`, `redis_url`, `secret_key`, `log_level`, `feature_new_endpoint`
- Validates that `secret_key` is at least 32 chars
- Validates that `database_echo_sql` is False when `app_env == "production"`
- Has a computed property `is_production`

**Task 2: Environment Files**
Create:
- `.env` — committed, non-secret defaults
- `.env.local` — gitignored, your local secrets
- Add `.env.local` to `.gitignore`

**Task 3: Three docker-compose Files**
- `docker-compose.yml` — dev with Postgres + Redis + hot reload
- `docker-compose.staging-test.yml` — simulates staging (no hot reload, no debug SQL, `APP_ENV=staging`)
- Document in a README: "run `docker-compose up` for dev, `docker-compose -f docker-compose.staging-test.yml up` for staging simulation"

**Task 4: Feature Flag**
Add a feature flag `feature_new_endpoint` (default: False).
Create a route `/v2/orders` that only works when the flag is True.
Set `FEATURE_NEW_ENDPOINT=true` in your dev `.env.local`.
Verify the endpoint returns 404 when the flag is False, 200 when True.

**Task 5: Startup Validation**
Add a startup event that:
- Logs which environment is starting
- Logs which feature flags are enabled
- Raises `RuntimeError` if `app_env == "production"` and `secret_key` is the default dev value

**Validation:**
```bash
# Dev mode: hot reload works, debug SQL logs, feature flag from .env.local
docker-compose up

# Staging simulation: no reload, no debug SQL
docker-compose -f docker-compose.staging-test.yml up

# Verify startup logs show correct environment
curl http://localhost:8000/health   # {"status": "ok", "env": "development"}
```

---

## Cross-References

- [01-docker.md](./01-docker.md) — Docker fundamentals and docker-compose
- [02-cicd-github-actions.md](./02-cicd-github-actions.md) — CI/CD pipeline with environment-gated deployments
- [04-secrets-config.md](./04-secrets-config.md) — Deep dive on secrets management with AWS Secrets Manager
- [../E-AWS/01-compute.md](../E-AWS/01-compute.md) — ECS Fargate: environment-specific task definitions and services
- [../E-AWS/03-iam-security.md](../E-AWS/03-iam-security.md) — IAM access control per environment
- [../E-AWS/05-cloudwatch-ops.md](../E-AWS/05-cloudwatch-ops.md) — Monitoring per environment, environment-specific log groups
