# Secrets and Configuration Management

**FDE Role Context:** "How do you handle customer credentials?" is one of the questions you will face in every technical interview and in every customer security review. The answer must be specific, defensible, and demonstrate operational maturity. This file gives you the complete production pattern — from why hardcoding is catastrophic to the exact boto3 code for Secrets Manager retrieval, rotation without downtime, and the audit trail that satisfies an enterprise security team.

---

## Table of Contents

1. [Why Hardcoding Secrets is Catastrophic](#1-why-hardcoding-secrets-is-catastrophic)
2. [Environment Variables: The Baseline](#2-environment-variables-the-baseline)
3. [.env Files: Local Development](#3-env-files-local-development)
4. [AWS Secrets Manager: Production Standard](#4-aws-secrets-manager-production-standard)
5. [AWS Parameter Store: Config vs Secrets](#5-aws-parameter-store-config-vs-secrets)
6. [Secret Injection Patterns](#6-secret-injection-patterns)
7. [Docker Secrets](#7-docker-secrets)
8. [ECS Secrets Integration](#8-ecs-secrets-integration)
9. [Rotating Secrets Without Downtime](#9-rotating-secrets-without-downtime)
10. [Audit Logging: Who Accessed What](#10-audit-logging-who-accessed-what)
11. [The Complete Production Pattern](#11-the-complete-production-pattern)
12. [Common Failure Modes](#12-common-failure-modes)
13. [Interview Angles](#13-interview-angles)
14. [Practice Exercise](#14-practice-exercise)

---

## 1. Why Hardcoding Secrets is Catastrophic

### The Git History Problem

```python
# A developer adds this to connect to the customer's ERP:
ERP_API_KEY = "sk_live_abc123XYZsuperSecret_customerProd"

# They realize the mistake and fix it:
ERP_API_KEY = os.environ["ERP_API_KEY"]

# They push the fix. But:
```

```bash
# The secret is permanently in git history
git log --all --full-history -- src/erp_client.py
git show abc1234:src/erp_client.py | grep API_KEY
# sk_live_abc123XYZsuperSecret_customerProd

# Anyone who ever had read access to this repo has this key.
# This includes:
# - All current and former developers
# - Anyone who forked or cloned the repo
# - GitHub's servers (if public even for a moment)
# - Automated scanners that monitor GitHub for leaked credentials
```

**The rule:** Once a secret is committed to git, it must be **rotated immediately**, regardless of how quickly you remove it. The git history exists forever.

### GitHub Automated Secret Scanning

GitHub automatically scans all public (and optionally private) repositories for patterns that match known secret formats. Within minutes of a push:
- AWS access keys → AWS notified, key disabled
- Stripe API keys → Stripe notified, key disabled
- Many other providers → similar automated response

This is a feature, but it means "I'll just delete the commit" is not a solution.

### The Docker Image Layer Problem

```dockerfile
# Developer hardcodes secret in Dockerfile (wrong approach)
RUN curl -H "Authorization: Bearer sk_live_secret" https://api.example.com/setup
```

```bash
# The secret is in the Docker image, in a specific layer
docker history myapp:latest
# IMAGE          CREATED        CREATED BY                          SIZE
# sha256:abc...  2 hours ago    RUN curl -H "Authorization: Bearer sk_...

docker run --rm myapp:latest sh -c "cat /proc/self/environ"
# Might expose env vars if they're in the final layer

# More dangerous: image layers are queryable
docker save myapp:latest | tar -xO | tar -xO layer.tar | grep "sk_live"
```

### The Log Problem

```python
# Developer logs configuration at startup for debugging (common mistake)
logger.info(f"Connecting to ERP with config: {config.__dict__}")
# Output: Connecting to ERP with config: {'api_key': 'sk_live_secret', 'url': '...'}

# This secret is now in:
# - CloudWatch Logs
# - Any log aggregation tool (Datadog, Splunk, ELK)
# - Log archives
# - Your SIEM
# - Incident tickets that include log excerpts
```

**Rules:**
1. Never log the Settings object directly
2. Use structured logging with an explicit allowlist of what to log
3. Mask secrets in logs even when debugging

---

## 2. Environment Variables: The Baseline

### Reading in Python

```python
import os

# Fail if missing (required secrets)
api_key = os.environ["ERP_API_KEY"]          # raises KeyError if not set

# Default value (optional config)
log_level = os.environ.get("LOG_LEVEL", "INFO")  # returns "INFO" if not set

# Check if set
if "FEATURE_X" in os.environ:
    enable_feature_x()

# All environment variables
all_env = dict(os.environ)   # be careful logging this!
```

### Setting in Different Contexts

```bash
# Shell (temporary, this session only)
export DATABASE_URL="postgresql+asyncpg://user:pass@host:5432/db"
python -m uvicorn src.main:app

# Inline (process-level, not exported)
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db python -m uvicorn src.main:app

# Systemd service unit
[Service]
Environment="DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db"
EnvironmentFile=/etc/myapp/secrets.env

# Docker (already covered in docker.md)
docker run -e DATABASE_URL="..." myapp:latest

# ECS task definition (preferred approach)
# Environment section: non-secret config
# Secrets section: from Secrets Manager (covered below)
```

### Validation at Startup (Not at Request Time)

```python
# WRONG: check at request time (silent failure until that route is hit)
@router.get("/orders")
async def get_orders():
    api_key = os.environ.get("ERP_API_KEY")
    if not api_key:
        raise HTTPException(500, "ERP_API_KEY not configured")
    ...

# RIGHT: validate at startup with pydantic-settings
# See src/config.py in the pydantic-settings section
# If ERP_API_KEY is missing, app fails to start with a clear error
# You know immediately, not when a customer hits the endpoint
```

---

## 3. .env Files: Local Development

### File Format

```bash
# .env — basic KEY=VALUE format
# Comments start with #
# No spaces around = (some parsers support spaces, but avoid for portability)
# Values don't need quotes for simple strings
# Quotes are needed for values with spaces or special characters

APP_ENV=development
LOG_LEVEL=DEBUG
APP_PORT=8000

# Multiline values need quotes
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
-----END RSA PRIVATE KEY-----"

# Values with special characters need quotes or escaping
DATABASE_URL="postgresql+asyncpg://user:p@$$w0rd!@localhost:5432/db"

# Boolean values — pydantic-settings understands these
FEATURE_FLAG=true     # or True, 1, yes, on
DEBUG_MODE=false      # or False, 0, no, off
```

### python-dotenv

```python
# Install: pip install python-dotenv

from dotenv import load_dotenv
import os

# Load .env file (does NOT override existing env vars)
load_dotenv()

# Load specific file
load_dotenv(".env.local")

# Override existing env vars (use sparingly)
load_dotenv(override=True)

# Multiple files (loaded in order, later files win)
load_dotenv(".env")
load_dotenv(".env.local")  # local overrides win
```

### The .gitignore Pattern

```bash
# .gitignore
# ── Secret env files (NEVER commit) ──────────────────────────────
.env.local
.env.*.local
.env.secrets
.env.production
.env.staging
*.env.secrets

# Committed env files (non-secret defaults only)
# .env and .env.staging and .env.development are committed
# But ONLY if they contain zero secrets
```

### Pre-commit Hook to Prevent Accidental Commits

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
        # Scans staged files for secrets before allowing commit

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: detect-aws-credentials
```

```bash
# Install and activate
pip install pre-commit
pre-commit install
# Now: git commit will scan for secrets before allowing the commit
```

---

## 4. AWS Secrets Manager: Production Standard

AWS Secrets Manager is the production store for secrets. Every secret has:
- A name (path-like: `/myapp/production/erp_api_key`)
- A value (JSON string or plain string)
- Versioning (current + previous versions during rotation)
- Access policy (who can read/write)
- Optional automatic rotation via Lambda

### Creating Secrets

```bash
# CLI: Create a secret (plain string)
aws secretsmanager create-secret \
  --name "/myapp/production/erp_api_key" \
  --description "ERP system API key for production" \
  --secret-string "sk_live_abc123XYZproductionKey" \
  --tags Key=Environment,Value=production \
         Key=Application,Value=myapp \
         Key=ManagedBy,Value=terraform

# CLI: Create a secret (JSON — for database credentials with multiple fields)
aws secretsmanager create-secret \
  --name "/myapp/production/database" \
  --description "Production database credentials" \
  --secret-string '{
    "username": "myapp_prod",
    "password": "SecureRandomP@ssw0rd!",
    "host": "myapp-prod.xxxxx.rds.amazonaws.com",
    "port": 5432,
    "dbname": "myapp_prod",
    "engine": "postgres"
  }'
```

### Retrieving Secrets in Python (boto3)

```python
# src/secrets.py
import json
import logging
from functools import lru_cache
from typing import Any

import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)


class SecretsManagerClient:
    """Thin wrapper around boto3 Secrets Manager with caching and error handling."""

    def __init__(self, region: str = "us-east-1"):
        self._client = boto3.client("secretsmanager", region_name=region)
        self._cache: dict[str, Any] = {}

    def get_secret(self, secret_name: str, use_cache: bool = True) -> str | dict:
        """
        Retrieve a secret value by name.

        Returns:
            dict if secret is valid JSON, str otherwise
        Raises:
            ValueError: if secret not found
            PermissionError: if IAM access denied
        """
        if use_cache and secret_name in self._cache:
            return self._cache[secret_name]

        logger.info(f"Retrieving secret: {secret_name}")
        try:
            response = self._client.get_secret_value(SecretId=secret_name)
        except ClientError as e:
            code = e.response["Error"]["Code"]
            msg = e.response["Error"]["Message"]

            if code == "ResourceNotFoundException":
                raise ValueError(f"Secret not found: {secret_name}") from e
            elif code == "AccessDeniedException":
                raise PermissionError(
                    f"Access denied to secret '{secret_name}'. "
                    f"Check the IAM role has secretsmanager:GetSecretValue permission "
                    f"for this resource ARN."
                ) from e
            elif code == "InvalidRequestException":
                raise ValueError(f"Invalid request for secret '{secret_name}': {msg}") from e
            elif code == "DecryptionFailure":
                raise ValueError(
                    f"Cannot decrypt secret '{secret_name}' — "
                    f"check KMS key permissions for the IAM role"
                ) from e
            else:
                raise

        # Secrets can be strings or binary
        if "SecretString" in response:
            raw = response["SecretString"]
        else:
            raw = response["SecretBinary"].decode("utf-8")

        # Try to parse as JSON (for structured secrets like DB credentials)
        try:
            parsed = json.loads(raw)
            result = parsed
        except json.JSONDecodeError:
            result = raw

        if use_cache:
            self._cache[secret_name] = result

        return result

    def get_secret_string(self, secret_name: str) -> str:
        """Retrieve a secret that is a plain string."""
        value = self.get_secret(secret_name)
        if isinstance(value, dict):
            raise ValueError(
                f"Secret '{secret_name}' is a JSON object, use get_secret() instead"
            )
        return value

    def get_database_url(self, secret_name: str, db_name: str | None = None) -> str:
        """
        Retrieve database credentials and construct a SQLAlchemy connection URL.
        Secret must be a JSON object with keys: username, password, host, port, dbname
        """
        creds = self.get_secret(secret_name)
        if not isinstance(creds, dict):
            raise ValueError(
                f"Secret '{secret_name}' must be a JSON object with database credentials"
            )
        dbname = db_name or creds["dbname"]
        url = (
            f"postgresql+asyncpg://{creds['username']}:{creds['password']}"
            f"@{creds['host']}:{creds.get('port', 5432)}/{dbname}"
        )
        return url

    def clear_cache(self):
        """Force re-fetch of all secrets (use after rotation)."""
        self._cache.clear()


# Module-level singleton
_secrets_client: SecretsManagerClient | None = None

def get_secrets_client(region: str = "us-east-1") -> SecretsManagerClient:
    global _secrets_client
    if _secrets_client is None:
        _secrets_client = SecretsManagerClient(region=region)
    return _secrets_client
```

### Using Secrets Manager in FastAPI Startup

```python
# src/main.py
from fastapi import FastAPI
from src.config import settings
from src.secrets import get_secrets_client

app = FastAPI()

@app.on_event("startup")
async def startup():
    """Load secrets at startup — fail fast if secrets are unavailable."""
    if not settings.is_development:
        # In non-dev environments, load secrets from Secrets Manager
        # and inject into settings (or validate they're accessible)
        secrets = get_secrets_client(region=settings.aws_region)
        try:
            # Verify critical secrets are accessible
            # (actual values are injected by ECS task definition — see section 8)
            secrets.get_secret(f"/myapp/{settings.app_env}/erp_api_key")
            logger.info("Secrets Manager connectivity verified")
        except (ValueError, PermissionError) as e:
            logger.critical(f"Cannot access required secrets: {e}")
            raise SystemExit(1)
```

### Secret Versioning

Every update to a secret creates a new version. During rotation, two versions coexist:
- `AWSCURRENT`: the current secret
- `AWSPREVIOUS`: the previous secret (retained so in-flight operations complete)

```bash
# List versions of a secret
aws secretsmanager list-secret-version-ids \
  --secret-id "/myapp/production/erp_api_key"

# Retrieve a specific version
aws secretsmanager get-secret-value \
  --secret-id "/myapp/production/erp_api_key" \
  --version-stage AWSCURRENT

aws secretsmanager get-secret-value \
  --secret-id "/myapp/production/erp_api_key" \
  --version-stage AWSPREVIOUS
```

---

## 5. AWS Parameter Store: Config vs Secrets

Parameter Store is simpler than Secrets Manager. Use it for non-secret configuration values.

### SecureString vs String

```bash
# String: plain text, for non-sensitive config
aws ssm put-parameter \
  --name "/myapp/production/log_level" \
  --value "INFO" \
  --type String

aws ssm put-parameter \
  --name "/myapp/production/allowed_origins" \
  --value "https://app.example.com,https://api.example.com" \
  --type String

# SecureString: encrypted, for sensitive values
aws ssm put-parameter \
  --name "/myapp/production/erp_api_key" \
  --value "sk_live_secret" \
  --type SecureString \
  --key-id "alias/myapp-secrets"   # KMS key alias
```

### Hierarchical Naming

```
/myapp/
  production/
    database_url          ← SecureString
    erp_api_key           ← SecureString
    secret_key            ← SecureString
    log_level             ← String (not sensitive)
    allowed_origins       ← String
  staging/
    database_url          ← SecureString
    erp_api_key           ← SecureString
    ...
```

### Retrieving from Parameter Store in Python

```python
import boto3

ssm = boto3.client("ssm", region_name="us-east-1")

# Get single parameter
response = ssm.get_parameter(
    Name="/myapp/production/erp_api_key",
    WithDecryption=True    # REQUIRED for SecureString
)
value = response["Parameter"]["Value"]

# Get multiple parameters by path (efficient batch retrieval)
response = ssm.get_parameters_by_path(
    Path="/myapp/production/",
    Recursive=True,
    WithDecryption=True
)
params = {p["Name"]: p["Value"] for p in response["Parameters"]}
log_level = params["/myapp/production/log_level"]
```

### Secrets Manager vs Parameter Store: When to Use Each

| Feature | Secrets Manager | Parameter Store |
|---|---|---|
| Primary use | Database passwords, API keys | App configuration, feature flags |
| Automatic rotation | Yes (Lambda-based) | No |
| Cost | $0.40/secret/month | Free (standard tier) |
| Versioning | Full rotation history | Versioned (less sophisticated) |
| Binary secrets | Yes | No (string only) |
| Cross-account | Easy via resource policy | Complex |
| ECS native integration | Yes | Yes |
| KMS encryption | Yes (optional) | Yes (required for SecureString) |

**Decision rule:**
- Password, API key, private key → Secrets Manager
- App settings, feature flags, service URLs → Parameter Store (free tier)

---

## 6. Secret Injection Patterns

### Pattern 1: ECS Secrets Injection (Preferred for Production)

The ECS task definition injects secrets as environment variables at task launch time. The secret value is fetched from Secrets Manager/Parameter Store by the ECS agent (not your application code).

**What this means:**
- Your application code reads `os.environ["ERP_API_KEY"]` — it doesn't know or care where it came from
- The secret fetch happens in the ECS execution role context, not your app's task role
- Secret value is not stored on disk — it's injected directly into the process environment

```json
// ECS Task Definition — container definition
{
  "name": "myapp",
  "environment": [
    // Non-sensitive config: plain environment vars
    {"name": "APP_ENV", "value": "production"},
    {"name": "LOG_LEVEL", "value": "INFO"}
  ],
  "secrets": [
    // Sensitive values: fetched from Secrets Manager at task launch
    {
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/database_url-AbCdEf"
    },
    {
      "name": "ERP_API_KEY",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/erp_api_key-XyZaBc"
    },
    {
      "name": "SECRET_KEY",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/secret_key-PqRsTu"
    },
    // Parameter Store SecureString
    {
      "name": "ALLOWED_ORIGINS",
      "valueFrom": "arn:aws:ssm:us-east-1:123456789:parameter/myapp/production/allowed_origins"
    }
  ]
}
```

**Required IAM permissions for ECS execution role:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameters",
        "ssm:GetParameter"
      ],
      "Resource": [
        "arn:aws:ssm:us-east-1:123456789:parameter/myapp/production/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:us-east-1:123456789:key/the-kms-key-id"
    }
  ]
}
```

### Pattern 2: Fetch at Application Startup (for Lambda, K8s, etc.)

When ECS-native injection isn't available:

```python
# src/startup.py
import os
import json
import boto3
from botocore.exceptions import ClientError

def load_secrets_into_env(env: str, region: str = "us-east-1"):
    """
    Fetch all secrets for the given environment and inject into process env.
    Called once at application startup, before Settings() is instantiated.
    """
    client = boto3.client("secretsmanager", region_name=region)
    secret_name = f"/myapp/{env}/all-secrets"

    try:
        response = client.get_secret_value(SecretId=secret_name)
        secrets = json.loads(response["SecretString"])

        for key, value in secrets.items():
            os.environ[key] = value
            print(f"Injected secret: {key} (length: {len(value)})")  # don't log value!

    except ClientError as e:
        print(f"FATAL: Cannot load secrets from Secrets Manager: {e}")
        raise SystemExit(1)


# In Lambda handler or app entrypoint:
import os
if os.environ.get("APP_ENV") not in ("development", "test"):
    load_secrets_into_env(os.environ["APP_ENV"])

# Now instantiate Settings — it will find the env vars
from src.config import settings
```

### Pattern 3: Don't Pass Secrets as Function Arguments

```python
# WRONG: secret flows through call stack, might be logged, might be in stack traces
def sync_orders(db_conn, erp_api_key: str):
    response = requests.get(erp_url, headers={"Authorization": f"Bearer {erp_api_key}"})

sync_orders(db, settings.erp_api_key)  # secret is now a positional arg

# RIGHT: use the settings singleton inside the function that needs it
def sync_orders(db_conn):
    # Import inside function to make the dependency explicit and testable
    from src.config import settings
    response = requests.get(erp_url, headers={"Authorization": f"Bearer {settings.erp_api_key}"})

# OR: create a client class that holds the key internally
class ERPClient:
    def __init__(self, api_key: str):
        self._api_key = api_key  # private attribute

    def get_orders(self):
        # api_key never leaves this class
        response = requests.get(self._url, headers={"Authorization": f"Bearer {self._api_key}"})
        return response.json()

# Instantiate once at startup from settings
erp_client = ERPClient(api_key=settings.erp_api_key)
```

---

## 7. Docker Secrets

For local development, Docker Compose supports secrets without environment variables.

```yaml
# docker-compose.yml with secrets
version: "3.9"

secrets:
  # Read from a file on the host (file NOT in the image)
  db_password:
    file: ./secrets/db_password.txt
  erp_api_key:
    file: ./secrets/erp_api_key.txt

services:
  app:
    build: .
    secrets:
      - db_password
      - erp_api_key
    # Secrets are mounted at /run/secrets/<secret_name>
    # NOT as environment variables
    # Your app must read the file:
    environment:
      # Tell the app WHERE to find the secret (not the secret itself)
      DB_PASSWORD_FILE: /run/secrets/db_password
      ERP_API_KEY_FILE: /run/secrets/erp_api_key
```

```python
# Reading a Docker secret from a file
import os

def read_secret_from_file(env_var: str, fallback_env_var: str | None = None) -> str:
    """
    Read a secret from a file path specified by env_var,
    or fall back to reading directly from fallback_env_var.
    Supports both Docker secrets and plain env vars.
    """
    secret_file = os.environ.get(env_var)
    if secret_file and os.path.exists(secret_file):
        with open(secret_file) as f:
            return f.read().strip()

    if fallback_env_var:
        value = os.environ.get(fallback_env_var)
        if value:
            return value

    raise ValueError(f"Secret not available: neither {env_var} file nor {fallback_env_var} env var is set")

# Usage:
db_password = read_secret_from_file("DB_PASSWORD_FILE", "DB_PASSWORD")
```

---

## 8. ECS Secrets Integration

### Setting Up the Complete ECS Secret Flow

Step 1: Create the secret in Secrets Manager:
```bash
aws secretsmanager create-secret \
  --name "/myapp/production/erp_api_key" \
  --secret-string "sk_live_the_actual_key" \
  --description "ERP API key for production environment"
```

Step 2: Create the ECS execution role with permission:
```json
// ecs-execution-role-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    // Standard ECS execution permissions
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    // Secrets Manager access (scoped to this app's secrets)
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/*"
    }
  ]
}
```

Step 3: Reference in task definition:
```json
{
  "family": "myapp-production-task",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/myapp-task-role",
  "containerDefinitions": [{
    "name": "myapp",
    "secrets": [
      {
        "name": "ERP_API_KEY",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/erp_api_key-AbCdEf"
      }
    ]
  }]
}
```

**Important:** The ARN in `valueFrom` must include the suffix (`-AbCdEf`) which is Secrets Manager's internal ID, not just the name. Use the full ARN from `aws secretsmanager describe-secret --secret-id /myapp/production/erp_api_key --query ARN`.

### Specific JSON Field from Secret

If your secret is a JSON object with multiple fields, you can reference a specific key:

```json
// Secret value: {"username": "myapp", "password": "secret123"}
"secrets": [
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:...secret/myapp/production/database-AbCdEf:password::"
    //                                                                  ^^^^^^^^
    //                                          JSON key to extract: password
  },
  {
    "name": "DB_USERNAME",
    "valueFrom": "arn:aws:...secret/myapp/production/database-AbCdEf:username::"
  }
]
```

---

## 9. Rotating Secrets Without Downtime

### The Problem

You need to change a credential (e.g., ERP API key was exposed) without:
1. Downtime (can't stop the service)
2. 500 errors during rotation (in-flight requests using old key must complete)

### Zero-Downtime Rotation Pattern

```
Phase 1: Add new secret, both versions valid
──────────────────────────────────────────────
Old key: sk_live_OLD   → still accepted by ERP
New key: sk_live_NEW   → also accepted by ERP (parallel period)
Secrets Manager: AWSCURRENT=NEW, AWSPREVIOUS=OLD

Your app: reads AWSCURRENT (new key) → works ✓

Phase 2: Remove old key
──────────────────────────────────────────────
After ~1 hour (all in-flight requests complete):
ERP: revoke OLD key
Secrets Manager: only AWSCURRENT=NEW remains
```

### AWS Secrets Manager Automatic Rotation

For RDS databases, Secrets Manager handles rotation automatically:

```bash
# Enable automatic rotation for an RDS secret
aws secretsmanager rotate-secret \
  --secret-id "/myapp/production/database" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRDSRotation" \
  --rotation-rules AutomaticallyAfterDays=30
```

AWS provides Lambda functions for RDS rotation. The Lambda:
1. Creates a new DB user password
2. Updates the RDS user's password
3. Updates the Secrets Manager secret
4. Verifies the new password works
5. Marks old version as AWSPREVIOUS

### Manual Rotation Pattern (for External API Keys)

```python
# scripts/rotate_erp_api_key.py
"""
Zero-downtime rotation for external API keys.
Run this when the vendor gives you a new key.
"""
import boto3
import time
import httpx

def rotate_api_key(
    secret_name: str,
    new_key: str,
    service_url: str,
    validation_endpoint: str,
    region: str = "us-east-1"
):
    """
    Rotate an API key with zero downtime.

    Steps:
    1. Update Secrets Manager with new key (old becomes PREVIOUS)
    2. Wait for ECS to pick up new key (task restart or cache TTL)
    3. Verify new key works
    4. Notify that old key can now be revoked
    """
    client = boto3.client("secretsmanager", region_name=region)

    print(f"Step 1: Updating secret '{secret_name}' with new key...")
    client.update_secret(
        SecretId=secret_name,
        SecretString=new_key
    )
    print("✓ Secret updated in Secrets Manager (AWSCURRENT=new, AWSPREVIOUS=old)")

    print("Step 2: Waiting for ECS tasks to restart with new secret...")
    print("  (ECS injects secrets at task launch, so new tasks use new key)")
    print("  Trigger a rolling deploy or wait for natural task replacement...")

    # For immediate cutover: force ECS service update
    ecs = boto3.client("ecs", region_name=region)
    ecs.update_service(
        cluster="myapp-production",
        service="myapp-production-service",
        forceNewDeployment=True
    )
    print("✓ ECS rolling deploy triggered")

    print("Step 3: Waiting for deployment stability...")
    waiter = ecs.get_waiter("services_stable")
    waiter.wait(
        cluster="myapp-production",
        services=["myapp-production-service"],
        WaiterConfig={"Delay": 15, "MaxAttempts": 40}
    )
    print("✓ Deployment stable")

    print("Step 4: Validating new key works...")
    response = httpx.get(
        f"{service_url}{validation_endpoint}",
        headers={"Authorization": f"Bearer {new_key}"}
    )
    response.raise_for_status()
    print(f"✓ New key validated: {response.status_code}")

    print("\n✅ Rotation complete!")
    print("NEXT STEP: Revoke the old API key in the ERP vendor portal")
    print("Wait ~1 hour before revoking to allow any in-flight requests to complete")


if __name__ == "__main__":
    import sys
    new_key = sys.argv[1]  # pass new key as argument (from vault/1Password)
    rotate_api_key(
        secret_name="/myapp/production/erp_api_key",
        new_key=new_key,
        service_url="https://myapp.example.com",
        validation_endpoint="/health"
    )
```

---

## 10. Audit Logging: Who Accessed What

### AWS CloudTrail for Secrets Manager

Every `GetSecretValue` call is logged in CloudTrail:

```json
// CloudTrail event for secret access
{
  "eventTime": "2024-01-15T10:23:45Z",
  "eventSource": "secretsmanager.amazonaws.com",
  "eventName": "GetSecretValue",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAXXXXXXXXX:ecs-task-abc123",
    "arn": "arn:aws:sts::123456789:assumed-role/myapp-task-role/ecs-task-abc123"
  },
  "requestParameters": {
    "secretId": "/myapp/production/erp_api_key"
  },
  "responseElements": null,  // value is NOT logged — only metadata
  "sourceIPAddress": "10.0.1.45",
  "requestID": "uuid-uuid-uuid"
}
```

**What you can audit:**
- Which IAM role accessed which secret
- When it was accessed
- From what IP/service
- Whether the access succeeded or was denied

**What's NOT logged:**
- The secret value itself (intentionally — CloudTrail logs go to S3, audit trail, etc.)

### Querying Audit Logs

```bash
# Find all accesses to production secrets in the last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --start-time "2024-01-15T00:00:00Z" \
  --end-time "2024-01-16T00:00:00Z" \
  --query 'Events[?contains(CloudTrailEvent, `production`)]'

# Via CloudWatch Log Insights (if CloudTrail logs are in CW)
fields eventTime, userIdentity.arn, requestParameters.secretId
| filter eventSource = "secretsmanager.amazonaws.com"
| filter eventName = "GetSecretValue"
| sort eventTime desc
| limit 100
```

### Detecting Unauthorized Access

```bash
# CloudWatch Metric Filter + Alarm for unauthorized secret access

# 1. Create metric filter on CloudTrail logs
aws logs put-metric-filter \
  --log-group-name "aws-cloudtrail-logs" \
  --filter-name "UnauthorizedSecretsAccess" \
  --filter-pattern '{ ($.eventSource = "secretsmanager.amazonaws.com") && ($.errorCode = "AccessDeniedException") }' \
  --metric-transformations \
    metricName="UnauthorizedSecretsAccess",metricNamespace="Security",metricValue=1

# 2. Create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "UnauthorizedSecretsAccess" \
  --metric-name "UnauthorizedSecretsAccess" \
  --namespace "Security" \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789:security-alerts"
```

---

## 11. The Complete Production Pattern

Here is the complete, authoritative secrets pattern for an FDE-deployed service. This is what you describe in interviews and customer security reviews.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  COMPLETE SECRETS PATTERN FOR FDE SERVICES                              │
│                                                                         │
│  DEVELOPMENT                                                            │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  .env.local (gitignored, on developer's machine only)            │  │
│  │  → pydantic-settings reads → Settings object                     │  │
│  │  Never enters git, never enters Docker image                     │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  STAGING / PRODUCTION                                                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  AWS Secrets Manager                                              │  │
│  │  /myapp/{env}/database_url  ──┐                                  │  │
│  │  /myapp/{env}/erp_api_key   ──┤ ECS execution role reads         │  │
│  │  /myapp/{env}/secret_key    ──┤ at task launch time              │  │
│  │                               ↓                                  │  │
│  │  ECS task environment vars (injected by ECS agent)               │  │
│  │                               ↓                                  │  │
│  │  pydantic-settings reads os.environ → Settings object            │  │
│  │                                                                   │  │
│  │  Your application code: os.environ["ERP_API_KEY"]                │  │
│  │  → IDENTICAL to dev — no Secrets Manager calls in app code       │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  AUDIT                                                                  │
│  Every secret access logged in CloudTrail                              │
│  Alarm on unauthorized access → SNS → PagerDuty                        │
│                                                                         │
│  ROTATION                                                               │
│  Secrets Manager automatic rotation (RDS: every 30 days)              │
│  Manual rotation procedure with zero downtime (external APIs)          │
│  Old key valid during ~1hr parallel period, then revoked               │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Answerable Customer Question

When a customer's CISO asks "How do you protect our API credentials?":

> "Customer credentials are stored exclusively in AWS Secrets Manager, scoped to the specific environment where they're used. In production, only the ECS execution role — which is a machine identity, not a human — has IAM permission to read those secrets. The permission is scoped to the exact ARN of each secret, not a wildcard. No developer can read production secrets without a documented break-glass procedure that triggers a CloudTrail alert. Secrets are injected directly into the container environment at task launch; the application reads them as environment variables and never calls Secrets Manager directly. All secret access is logged in CloudTrail, which we retain for 90 days, and we have an automated alarm that fires within 5 minutes of any unauthorized access attempt. We rotate credentials on a defined schedule — 30 days for database passwords via Secrets Manager's automatic rotation, and a documented zero-downtime procedure for external API keys."

---

## 12. Common Failure Modes

### Failure 1: "The secret ARN is not found" on ECS Task Launch

```
Error: ResourceNotFoundException: Secrets Manager can't find the specified secret.
```

**Cause A:** ARN is wrong in task definition — missing or wrong suffix
```bash
# Get the correct full ARN:
aws secretsmanager describe-secret \
  --secret-id "/myapp/production/erp_api_key" \
  --query ARN
# Returns: arn:aws:secretsmanager:us-east-1:123456789:secret:/myapp/production/erp_api_key-AbCdEf
# Use this full ARN in the task definition, not just the name
```

**Cause B:** Wrong region in ARN (secret in us-east-1, task running in us-west-2)

**Cause C:** Secret is in a different account, cross-account role setup needed

### Failure 2: Access Denied on GetSecretValue

```
Error: AccessDeniedException: User: arn:aws:sts::123456789:assumed-role/ecsTaskExecutionRole/...
is not authorized to perform: secretsmanager:GetSecretValue
```

**Check:**
```bash
# Verify the execution role has the permission
aws iam get-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name SecretsManagerAccess

# Check if resource matches the secret ARN
# Resource: "arn:aws:...:secret:/myapp/production/*"
# Does the secret ARN match this pattern?
```

**Common mistake:** Using `*` in the Resource only works if it includes the `-AbCdEf` suffix. Better: use the full ARN.

### Failure 3: Pydantic Validation Error at Startup

```
pydantic_core.ValidationError: 1 validation error for Settings
erp_api_key
  Field required [type=missing, input_url=...]
```

**Cause:** The ECS secrets injection didn't work, so `ERP_API_KEY` env var was never set.

**Debug:**
```bash
# Check ECS task stopped reason
aws ecs describe-tasks \
  --cluster myapp-production \
  --tasks <task-id> \
  --query 'tasks[0].stoppedReason'

# Check ECS service events
aws ecs describe-services \
  --cluster myapp-production \
  --services myapp-production-service \
  --query 'services[0].events[:5]'
```

### Failure 4: Secret Cache Serving Stale Value After Rotation

```python
# Problem: you cached the old secret value, rotation happened, app still uses old value

# In the SecretsManagerClient above, clear_cache() exists for this
# But if you're running multiple tasks, each has its own cache

# Solution 1: Short TTL cache (acceptable for credentials that rotate rarely)
import time

class SecretsManagerClient:
    CACHE_TTL_SECONDS = 300  # 5 minutes

    def get_secret(self, name: str):
        now = time.time()
        if name in self._cache:
            value, cached_at = self._cache[name]
            if now - cached_at < self.CACHE_TTL_SECONDS:
                return value
        # Cache miss or expired — fetch fresh
        value = self._fetch(name)
        self._cache[name] = (value, now)
        return value

# Solution 2: Use ECS task injection — no in-process caching needed
# Secrets are baked into env vars at task start; rotation triggers new task
```

### Failure 5: Secret Value Contains Special Characters Breaking URL Parsing

```python
# Password: "p@$$w0rd!&more" breaks PostgreSQL URL parsing
# postgresql+asyncpg://user:p@$$w0rd!&more@host:5432/db
#                               ^^^^^^^^^^^
# The @ is interpreted as end of password, host starts at $w0rd!&more

# Fix: URL-encode the password
from urllib.parse import quote_plus

def build_db_url(username: str, password: str, host: str, port: int, dbname: str) -> str:
    return f"postgresql+asyncpg://{quote_plus(username)}:{quote_plus(password)}@{host}:{port}/{dbname}"

# Or: store the URL already encoded in Secrets Manager
# Or: store username and password separately, build URL in code
```

---

## 13. Interview Angles

### Q: "How do you handle customer credentials in your integration service?"

**Great answer (the full production pattern):**

"Customer credentials — like their ERP API key or webhook signing secret — are stored in AWS Secrets Manager with a path like `/myapp/production/erp_api_key`. Access is controlled via IAM: only the ECS task execution role has `secretsmanager:GetSecretValue` permission, scoped to that exact ARN. ECS injects the secret as an environment variable at task launch — my application code reads `os.environ['ERP_API_KEY']` without knowing or caring about Secrets Manager. In development, I use a `.env.local` file that's gitignored. The application code is identical between environments — pydantic-settings reads from env vars regardless of where they came from.

For rotation: I have a documented procedure for external API keys that uses the old/new parallel window. For database credentials, Secrets Manager's automatic rotation handles it every 30 days. All secret access is logged in CloudTrail, and I have a CloudWatch alarm that fires on any unauthorized access attempt within 5 minutes."

### Q: "A developer accidentally committed an API key to git. What do you do right now?"

**Answer — this must be immediate and specific:**
1. Rotate the key immediately — don't wait. Contact the vendor or go to their dashboard.
2. The exposed key must be treated as compromised regardless of how quickly you removed the commit.
3. Check CloudTrail/the vendor's access logs for unauthorized use of the key.
4. Remove from git history (git filter-branch or BFG Repo Cleaner), force-push.
5. Review who had access to that repo during the exposure window.
6. Post-mortem: add pre-commit hooks (gitleaks) to prevent recurrence.
7. Notify the customer if it was their credential.

### Q: "What's the difference between environment variables and Secrets Manager? Why use Secrets Manager?"

**Answer:**
- Environment variables are the mechanism (how the app reads the config), not the storage
- Secrets Manager is the secure storage backend that populates those env vars
- Direct env vars in task definitions: value is visible in the ECS console and API responses — anyone with ECS describe permissions sees the value
- Secrets Manager: value is never visible in task definition (only the ARN is), requires explicit IAM permission to read the value, all reads are audited

---

## 14. Practice Exercise

### Exercise: Build the Complete Secrets Pipeline

**Step 1: Local Setup**
1. Create `src/config.py` with a `Settings` class using pydantic-settings
2. Include fields: `database_url`, `erp_api_key`, `secret_key` (all required), `log_level` (default: INFO)
3. Create `.env.local` with fake values
4. Verify: `python -c "from src.config import settings; print(settings.app_env)"`

**Step 2: Pre-commit Protection**
1. Install `pre-commit` and `gitleaks`
2. Add `.pre-commit-config.yaml` with gitleaks hook
3. Test: try to `git commit` a file containing a fake AWS key pattern (`AKIA` + 16 chars)
4. Verify the commit is blocked with a clear error

**Step 3: Secrets Manager (requires AWS credentials)**
1. Create a secret in Secrets Manager: `/myapp/dev-exercise/erp_api_key` with value `test-key-123`
2. Write Python code to retrieve it using boto3 (use the `SecretsManagerClient` class above)
3. Test error handling: try to retrieve a nonexistent secret, verify the error message is clear

**Step 4: Rotation Simulation**
1. Update the secret with a new value: `test-key-456`
2. Verify `get_secret_value` returns the new value
3. Verify `AWSPREVIOUS` still has the old value for 1 version
4. Write code that tries the current key, falls back to previous key if current fails

**Step 5: Audit**
1. Enable CloudTrail in your test AWS account (if available)
2. Make 3 `GetSecretValue` calls
3. Query CloudTrail to find those calls

**Validation:**
- Settings class fails fast at startup if any required env var is missing
- Pre-commit hook blocks commits with fake secrets
- boto3 code retrieves the secret and handles errors gracefully
- You can explain the complete chain: code → env var → ECS task def → Secrets Manager → IAM

---

## Cross-References

- [03-env-separation.md](./03-env-separation.md) — pydantic-settings configuration per environment
- [02-cicd-github-actions.md](./02-cicd-github-actions.md) — GitHub Actions secrets (OIDC, repository secrets)
- [01-docker.md](./01-docker.md) — Docker secrets and env_file patterns
- [../E-AWS/01-compute.md](../E-AWS/01-compute.md) — ECS task definitions: environment vs secrets sections
- [../E-AWS/03-iam-security.md](../E-AWS/03-iam-security.md) — IAM policies for Secrets Manager access, execution role vs task role
- [../E-AWS/05-cloudwatch-ops.md](../E-AWS/05-cloudwatch-ops.md) — CloudTrail audit logging and CloudWatch alarms
