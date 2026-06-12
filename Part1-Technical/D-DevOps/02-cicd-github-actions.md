# CI/CD with GitHub Actions: From Code Push to ECS Deploy

**FDE Role Context:** Every integration service you build needs to ship reliably. You will not manually SSH into servers to deploy. GitHub Actions is the CI/CD backbone you'll set up for your own projects and often help customers configure. The goal: every push to `main` runs tests, builds a Docker image, pushes to ECR, and deploys to ECS — automatically, with zero human steps in the happy path. You need to understand this pipeline deeply enough to debug it when it breaks at 11pm before a customer go-live.

---

## Table of Contents

1. [GitHub Actions Fundamentals](#1-github-actions-fundamentals)
2. [Workflow Triggers](#2-workflow-triggers)
3. [Job Structure Deep Dive](#3-job-structure-deep-dive)
4. [Secrets and Environments](#4-secrets-and-environments)
5. [Caching: pip, uv, and Docker Layers](#5-caching-pip-uv-and-docker-layers)
6. [Matrix Builds](#6-matrix-builds)
7. [Artifacts: Test Reports and Coverage](#7-artifacts-test-reports-and-coverage)
8. [Docker Build and Push to ECR](#8-docker-build-and-push-to-ecr)
9. [ECS Deployment](#9-ecs-deployment)
10. [Complete Production-Grade Pipeline](#10-complete-production-grade-pipeline)
11. [Branch Protection Rules](#11-branch-protection-rules)
12. [Reusable Workflows and Composite Actions](#12-reusable-workflows-and-composite-actions)
13. [Debugging Failed Workflows](#13-debugging-failed-workflows)
14. [Common Failure Modes](#14-common-failure-modes)
15. [Interview Angles](#15-interview-angles)
16. [Practice Exercise](#16-practice-exercise)

---

## 1. GitHub Actions Fundamentals

### The Hierarchy

```
Repository
└── .github/
    └── workflows/
        ├── ci.yml           ← one workflow file
        ├── deploy.yml       ← another workflow file
        └── cron-jobs.yml    ← another

Workflow (.yml file)
└── Jobs (parallel by default)
    ├── test
    │   └── Steps (sequential within a job)
    │       ├── uses: actions/checkout@v4
    │       ├── run: pip install -r requirements.txt
    │       └── run: pytest
    └── build
        └── Steps
            ├── uses: docker/build-push-action@v5
            └── uses: aws-actions/amazon-ecs-deploy-task-definition@v1
```

**Key concepts:**
- **Workflow**: a YAML file in `.github/workflows/` — triggered by events
- **Job**: a group of steps that run on the same runner (VM). Jobs run in parallel unless you add `needs:`
- **Step**: a single task within a job — either `uses:` (an Action) or `run:` (shell command)
- **Action**: a reusable, versioned piece of logic — `actions/checkout@v4`, `aws-actions/configure-aws-credentials@v4`
- **Runner**: the VM that executes the job — `ubuntu-latest`, `windows-latest`, `macos-latest`, or self-hosted

### The Runner Context

Every job gets a fresh, clean VM. Nothing persists between jobs (unless you use artifacts or caches).

```
Job "test" starts:
  → Fresh Ubuntu VM provisioned
  → Your code is NOT there yet (you must checkout)
  → Python is NOT installed (you must set it up)
  → No packages installed

Step: actions/checkout@v4
  → Clones your repo into /home/runner/work/your-repo/your-repo/

Step: actions/setup-python@v5
  → Installs Python 3.12 on the runner VM

Step: run: pip install -r requirements.txt
  → Installs your packages

Step: run: pytest
  → Runs tests

Job ends → VM is destroyed
```

This statelessness is a feature: reproducible builds. It's also a cost: every job re-downloads and re-installs everything (unless you cache).

---

## 2. Workflow Triggers

```yaml
on:
  # Trigger on push to specific branches
  push:
    branches:
      - main
      - develop
    # Only trigger if these paths changed (ignore docs-only changes)
    paths:
      - 'src/**'
      - 'tests/**'
      - 'Dockerfile'
      - 'requirements*.txt'
      - 'pyproject.toml'

  # Trigger on PRs targeting specific branches
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize    # new commits pushed to PR branch
      - reopened

  # Scheduled trigger (cron syntax)
  schedule:
    - cron: '0 6 * * 1-5'   # 6am UTC, Monday-Friday (nightly build)

  # Manual trigger (button in GitHub UI or API)
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production
        default: staging
      image_tag:
        description: 'Docker image tag to deploy (leave empty for latest)'
        required: false
        type: string

  # Triggered by another workflow completing
  workflow_run:
    workflows: ["CI"]
    types:
      - completed
    branches:
      - main
```

### Practical Trigger Patterns

```yaml
# Pattern 1: CI on PRs, CD on main merge
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# In the job, check if it's a push (deploy) or PR (test only)
jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

```yaml
# Pattern 2: Separate CI and CD workflows
# .github/workflows/ci.yml — triggers on PR
on:
  pull_request:
    branches: [main]

# .github/workflows/cd.yml — triggers on push to main
on:
  push:
    branches: [main]
```

```yaml
# Pattern 3: Path filtering (only run tests when code changes, not docs)
on:
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '*.md'
      - '.gitignore'
```

---

## 3. Job Structure Deep Dive

```yaml
jobs:
  test:
    name: "Run Tests"

    # Runner OS — ubuntu-latest is cheapest and most common
    runs-on: ubuntu-latest

    # Default shell/working directory for all run steps
    defaults:
      run:
        shell: bash
        working-directory: ./

    # Environment variables available to all steps in this job
    env:
      PYTHONPATH: src
      DATABASE_URL: "postgresql://testuser:testpass@localhost:5432/testdb"

    # Service containers — started alongside job runner
    # Use for integration tests that need real databases
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U testuser"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: --health-cmd "redis-cli ping" --health-interval 5s --health-timeout 3s --health-retries 5

    steps:
      # Step 1: Checkout code
      - name: Checkout repository
        uses: actions/checkout@v4

      # Step 2: Setup Python with caching
      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"              # cache pip packages

      # Step 3: Install dependencies
      - name: Install dependencies
        run: |
          pip install uv
          uv sync --frozen

      # Step 4: Lint
      - name: Lint with ruff
        run: |
          uv run ruff check src/ tests/
          uv run ruff format --check src/ tests/

      # Step 5: Type check
      - name: Type check with mypy
        run: uv run mypy src/

      # Step 6: Run tests with coverage
      - name: Run tests
        run: |
          uv run pytest \
            --cov=src \
            --cov-report=xml:coverage.xml \
            --cov-report=term-missing \
            --junitxml=test-results.xml \
            -v
        env:
          # Step-level env var (overrides job-level)
          APP_ENV: test

      # Step 7: Upload coverage (even if tests fail)
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        if: always()
        with:
          files: coverage.xml
          fail_ci_if_error: false

      # Step 8: Upload test results as artifact
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()   # upload even if tests fail
        with:
          name: test-results
          path: test-results.xml
          retention-days: 30
```

### Conditional Steps

```yaml
steps:
  # Only run on main branch push
  - name: Deploy to prod
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh

  # Skip on draft PRs
  - name: Integration tests
    if: github.event.pull_request.draft == false
    run: pytest tests/integration/

  # Run even if previous steps failed
  - name: Cleanup
    if: always()
    run: ./cleanup.sh

  # Run only on failure
  - name: Notify team of failure
    if: failure()
    uses: slackapi/slack-github-action@v1
    with:
      payload: '{"text": "Build failed on ${{ github.ref }}"}'
```

### Job Dependencies

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: pytest

  build:
    runs-on: ubuntu-latest
    needs: test          # only runs if test succeeds
    steps:
      - run: docker build .

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build         # only runs if build succeeds
    steps:
      - run: ./deploy-staging.sh

  deploy-prod:
    runs-on: ubuntu-latest
    needs: [test, build, deploy-staging]   # all three must succeed
    if: github.ref == 'refs/heads/main'
    steps:
      - run: ./deploy-prod.sh
```

---

## 4. Secrets and Environments

### Repository Secrets

Set in: GitHub → Repository → Settings → Secrets and variables → Actions

```yaml
steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1
```

### Environment Secrets (with Protection Rules)

Environments allow different secrets per environment AND protection rules (required reviewers, wait timers).

```yaml
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://myapp.example.com    # shown in GitHub deployment UI
    # This job will pause and wait for required reviewer approval
    # before running (configured in GitHub environment settings)
    steps:
      - name: Deploy
        run: ./deploy.sh
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
          API_KEY: ${{ secrets.PROD_API_KEY }}
```

### OIDC Authentication (Best Practice — No Long-Lived Keys)

Instead of storing AWS access keys as secrets, use OIDC to get temporary credentials:

```yaml
permissions:
  id-token: write    # REQUIRED for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy-role
          role-session-name: GitHubActions-${{ github.run_id }}
          aws-region: us-east-1
          # No access key or secret key needed!
```

**AWS IAM setup for OIDC:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
      }
    }
  }]
}
```

**FDE Context:** OIDC is the right answer when a customer asks "how do you authenticate to AWS from CI?" Static access keys are a credential leak waiting to happen. OIDC gives short-lived credentials scoped to specific repos and branches.

---

## 5. Caching: pip, uv, and Docker Layers

### Python Dependency Cache

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"   # built-in pip cache by setup-python
    cache-dependency-path: "requirements*.txt"
```

For `uv`:
```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v3
  with:
    version: "0.4.10"
    enable-cache: true          # caches uv's package cache
    cache-dependency-glob: "uv.lock"
```

### Manual Cache with `actions/cache`

```yaml
- name: Cache uv packages
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/uv
      .venv
    key: venv-${{ runner.os }}-${{ hashFiles('uv.lock') }}
    restore-keys: |
      venv-${{ runner.os }}-

- name: Install dependencies (only if cache miss)
  run: uv sync --frozen
```

### Docker Layer Cache

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push with layer caching
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
    cache-from: type=gha          # read cache from GitHub Actions cache
    cache-to: type=gha,mode=max   # write all layers to cache
```

### Cache Key Strategy

```yaml
# Cache based on lock file hash — busts when deps change
key: pip-${{ runner.os }}-py3.12-${{ hashFiles('requirements.txt', 'requirements-dev.txt') }}

# Fallback key (restore closest available cache even if exact miss)
restore-keys: |
  pip-${{ runner.os }}-py3.12-
  pip-${{ runner.os }}-
```

---

## 6. Matrix Builds

Test across multiple Python versions and OS:

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
        os: [ubuntu-latest, macos-latest]
        # Exclude specific combinations
        exclude:
          - os: macos-latest
            python-version: "3.11"
        # Add extra variables to specific matrix entries
        include:
          - python-version: "3.12"
            os: ubuntu-latest
            run-coverage: true     # only upload coverage from this combination

      fail-fast: false    # don't cancel all jobs when one fails (see all failures)
      max-parallel: 3     # limit concurrent jobs (GitHub has limits)

    runs-on: ${{ matrix.os }}
    name: "Test Python ${{ matrix.python-version }} on ${{ matrix.os }}"

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Run tests
        run: pytest tests/

      - name: Upload coverage
        if: matrix.run-coverage == true
        uses: codecov/codecov-action@v4
```

---

## 7. Artifacts: Test Reports and Coverage

```yaml
- name: Run tests
  run: |
    pytest \
      --junitxml=reports/junit.xml \
      --cov=src \
      --cov-report=xml:reports/coverage.xml \
      --cov-report=html:reports/htmlcov

# Always upload even if tests fail (you need the report to debug failures)
- name: Upload test reports
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: test-reports-${{ github.run_id }}
    path: reports/
    retention-days: 30

# Publish JUnit test results as PR check (shows pass/fail per test)
- name: Publish test results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()
  with:
    files: reports/junit.xml
    comment_mode: always
```

---

## 8. Docker Build and Push to ECR

### Login to ECR

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
    aws-region: ${{ env.AWS_REGION }}

- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2

- name: Set image tag
  id: vars
  run: |
    SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
    echo "image_tag=${SHORT_SHA}" >> $GITHUB_OUTPUT
    echo "ecr_image=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${SHORT_SHA}" >> $GITHUB_OUTPUT
```

### Build, Tag, and Push

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    push: ${{ github.ref == 'refs/heads/main' }}  # only push on main branch
    tags: |
      ${{ steps.vars.outputs.ecr_image }}
      ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
    labels: |
      git.sha=${{ github.sha }}
      git.branch=${{ github.ref_name }}
      build.number=${{ github.run_number }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
    build-args: |
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
      GIT_SHA=${{ github.sha }}
```

### Image Scanning

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ steps.vars.outputs.ecr_image }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1     # fail the build if CRITICAL/HIGH vulns found

- name: Upload Trivy scan results to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: trivy-results.sarif
```

---

## 9. ECS Deployment

### Update ECS Task Definition and Deploy

```yaml
- name: Download current task definition
  run: |
    aws ecs describe-task-definition \
      --task-definition ${{ env.ECS_TASK_DEFINITION_FAMILY }} \
      --query taskDefinition \
      > task-definition.json

- name: Update image in task definition
  id: task-def
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: task-definition.json
    container-name: ${{ env.CONTAINER_NAME }}
    image: ${{ steps.vars.outputs.ecr_image }}
    # Optionally inject environment variables
    environment-variables: |
      GIT_SHA=${{ github.sha }}
      DEPLOYED_AT=${{ github.event.head_commit.timestamp }}

- name: Deploy to ECS service
  uses: aws-actions/amazon-ecs-deploy-task-definition@v1
  with:
    task-definition: ${{ steps.task-def.outputs.task-definition }}
    service: ${{ env.ECS_SERVICE }}
    cluster: ${{ env.ECS_CLUSTER }}
    wait-for-service-stability: true      # wait until deployment completes
    wait-for-minutes: 10                  # timeout after 10 minutes
    # For blue/green deployment with CodeDeploy:
    # codedeploy-appspec: appspec.json
    # codedeploy-application: myapp
    # codedeploy-deployment-group: myapp-dg
```

### Blue/Green Deployment on ECS (with CodeDeploy)

```yaml
- name: Deploy blue/green to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v1
  with:
    task-definition: ${{ steps.task-def.outputs.task-definition }}
    service: ${{ env.ECS_SERVICE }}
    cluster: ${{ env.ECS_CLUSTER }}
    wait-for-service-stability: true
    codedeploy-appspec: |
      version: 0.0
      Resources:
        - TargetService:
            Type: AWS::ECS::Service
            Properties:
              TaskDefinition: <TASK_DEFINITION>
              LoadBalancerInfo:
                ContainerName: myapp
                ContainerPort: 8000
```

---

## 10. Complete Production-Grade Pipeline

This is the complete, working pipeline for a Python/FastAPI service deployed to ECS.

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'tests/**'
      - 'Dockerfile'
      - 'pyproject.toml'
      - 'uv.lock'
      - '.github/workflows/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/**'
      - 'tests/**'
      - 'Dockerfile'
      - 'pyproject.toml'
      - 'uv.lock'

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECS_CLUSTER: myapp-cluster
  ECS_SERVICE: myapp-service
  CONTAINER_NAME: myapp
  ECS_TASK_DEFINITION_FAMILY: myapp-task
  PYTHON_VERSION: "3.12"

permissions:
  id-token: write       # for OIDC AWS auth
  contents: read
  checks: write         # for test result publishing
  pull-requests: write  # for PR comments

jobs:
  # ──────────────────────────────────────────────────────────────────────────
  # JOB 1: Lint and Type Check
  # ──────────────────────────────────────────────────────────────────────────
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "0.4.10"
          enable-cache: true
          cache-dependency-glob: "uv.lock"

      - name: Install dependencies
        run: uv sync --frozen

      - name: Ruff lint
        run: uv run ruff check src/ tests/ --output-format=github

      - name: Ruff format check
        run: uv run ruff format --check src/ tests/

      - name: Mypy type check
        run: uv run mypy src/ --ignore-missing-imports

  # ──────────────────────────────────────────────────────────────────────────
  # JOB 2: Tests (unit + integration with real Postgres)
  # ──────────────────────────────────────────────────────────────────────────
  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: lint

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U testuser"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

    env:
      DATABASE_URL: "postgresql+asyncpg://testuser:testpass@localhost:5432/testdb"
      REDIS_URL: "redis://localhost:6379/0"
      APP_ENV: test
      APP_SECRET_KEY: "test-secret-key-not-for-production"

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "0.4.10"
          enable-cache: true
          cache-dependency-glob: "uv.lock"

      - name: Install dependencies (including dev)
        run: uv sync --frozen

      - name: Run database migrations
        run: uv run alembic upgrade head

      - name: Run tests
        run: |
          uv run pytest \
            tests/ \
            --cov=src \
            --cov-report=xml:reports/coverage.xml \
            --cov-report=term-missing \
            --junitxml=reports/junit.xml \
            --tb=short \
            -v
        env:
          PYTHONPATH: .

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: reports/junit.xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        if: always()
        with:
          files: reports/coverage.xml
          fail_ci_if_error: false

      - name: Upload reports artifact
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-reports
          path: reports/
          retention-days: 14

  # ──────────────────────────────────────────────────────────────────────────
  # JOB 3: Build and Push Docker Image to ECR
  # (only on push to main — not on PRs)
  # ──────────────────────────────────────────────────────────────────────────
  build:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    outputs:
      image-tag: ${{ steps.vars.outputs.image_tag }}
      ecr-image: ${{ steps.vars.outputs.ecr_image }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          role-session-name: GitHubActions-build-${{ github.run_id }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image variables
        id: vars
        run: |
          SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          REGISTRY="${{ steps.login-ecr.outputs.registry }}"
          IMAGE_TAG="${SHORT_SHA}"
          ECR_IMAGE="${REGISTRY}/${{ env.ECR_REPOSITORY }}:${IMAGE_TAG}"
          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT
          echo "ecr_image=${ECR_IMAGE}" >> $GITHUB_OUTPUT
          echo "registry=${REGISTRY}" >> $GITHUB_OUTPUT
          echo "Building: ${ECR_IMAGE}"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push to ECR
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ steps.vars.outputs.ecr_image }}
            ${{ steps.vars.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
            org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
            org.opencontainers.image.source=https://github.com/${{ github.repository }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: false   # disable SBOM generation (reduces image manifest complexity)

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.vars.outputs.ecr_image }}
          format: table
          severity: CRITICAL
          exit-code: 0   # don't fail build (set to 1 to enforce)

  # ──────────────────────────────────────────────────────────────────────────
  # JOB 4: Deploy to Staging
  # ──────────────────────────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
      url: https://staging.myapp.example.com

    env:
      ECS_SERVICE: myapp-staging-service
      ECS_CLUSTER: myapp-staging-cluster

    steps:
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_DEPLOY_ROLE_ARN }}
          role-session-name: GitHubActions-staging-${{ github.run_id }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_TASK_DEFINITION_FAMILY }}-staging \
            --query taskDefinition \
            > task-definition.json
          echo "Current task def downloaded"

      - name: Update container image in task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build.outputs.ecr-image }}
          environment-variables: |
            GIT_SHA=${{ github.sha }}
            DEPLOY_ENV=staging

      - name: Deploy to ECS staging
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 10

      - name: Run smoke tests against staging
        run: |
          STAGING_URL="https://staging.myapp.example.com"
          echo "Testing health endpoint..."
          curl -f --retry 5 --retry-delay 5 "${STAGING_URL}/health"
          echo "Staging smoke tests passed!"

  # ──────────────────────────────────────────────────────────────────────────
  # JOB 5: Deploy to Production (with manual approval gate)
  # ──────────────────────────────────────────────────────────────────────────
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production            # environment protection rules apply here
      url: https://myapp.example.com
    # GitHub will pause here and wait for a required reviewer to approve

    steps:
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_DEPLOY_ROLE_ARN }}
          role-session-name: GitHubActions-prod-${{ github.run_id }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Download current prod task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_TASK_DEFINITION_FAMILY }} \
            --query taskDefinition \
            > task-definition.json

      - name: Update container image in task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build.outputs.ecr-image }}
          environment-variables: |
            GIT_SHA=${{ github.sha }}
            DEPLOY_ENV=production

      - name: Deploy to ECS production
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 15

      - name: Verify production health
        run: |
          sleep 10  # brief settle time
          curl -f --retry 3 --retry-delay 10 "https://myapp.example.com/health"
          echo "Production deployment verified!"

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "✅ Production deployment successful",
              "attachments": [{
                "color": "good",
                "fields": [
                  {"title": "Repository", "value": "${{ github.repository }}", "short": true},
                  {"title": "Image Tag", "value": "${{ needs.build.outputs.image-tag }}", "short": true},
                  {"title": "Deployed By", "value": "${{ github.actor }}", "short": true}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "🚨 Production deployment FAILED",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Repository", "value": "${{ github.repository }}", "short": true},
                  {"title": "Run URL", "value": "https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## 11. Branch Protection Rules

Set in: GitHub → Repository → Settings → Branches → Add rule

### Required Settings for a Production Repository

```
Branch name pattern: main

✅ Require a pull request before merging
   Required approving reviews: 1
   Dismiss stale pull request approvals when new commits are pushed: ✅
   Require review from Code Owners: ✅ (if you have CODEOWNERS file)

✅ Require status checks to pass before merging
   Require branches to be up to date before merging: ✅
   Status checks that must pass:
     - lint
     - test
     - build (if applicable)

✅ Require signed commits (if your team uses GPG signing)

✅ Include administrators (even admins must follow the rules)

✅ Do not allow bypassing the above settings
```

### CODEOWNERS File

```
# .github/CODEOWNERS
# Global owners for all files
*                @myorg/platform-team

# Infrastructure changes require DevOps review
/terraform/       @myorg/devops-team
/.github/         @myorg/devops-team
/Dockerfile       @myorg/devops-team

# Security-sensitive files require security review
/src/auth/        @myorg/security-team
/src/config.py    @myorg/security-team
```

---

## 12. Reusable Workflows and Composite Actions

### Reusable Workflow

When you have the same CI logic across multiple repositories, extract it:

```yaml
# .github/workflows/reusable-test.yml
# Called FROM other workflows, not triggered directly

on:
  workflow_call:
    inputs:
      python-version:
        required: false
        type: string
        default: "3.12"
      run-integration:
        required: false
        type: boolean
        default: true
    secrets:
      CODECOV_TOKEN:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run pytest tests/unit/
      - name: Integration tests
        if: inputs.run-integration
        run: uv run pytest tests/integration/
```

```yaml
# .github/workflows/ci.yml — in another repo, calls the reusable workflow
jobs:
  test:
    uses: myorg/shared-workflows/.github/workflows/reusable-test.yml@main
    with:
      python-version: "3.12"
      run-integration: true
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

### Composite Action

```yaml
# .github/actions/setup-python-uv/action.yml
name: "Setup Python with uv"
description: "Setup Python and install dependencies using uv"

inputs:
  python-version:
    description: Python version to use
    default: "3.12"
  install-dev:
    description: Install dev dependencies
    default: "true"

runs:
  using: "composite"
  steps:
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}

    - name: Install uv
      shell: bash
      run: pip install uv==0.4.10

    - name: Install dependencies
      shell: bash
      run: |
        if [ "${{ inputs.install-dev }}" = "true" ]; then
          uv sync --frozen
        else
          uv sync --frozen --no-dev
        fi
```

Usage:
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-python-uv
    with:
      python-version: "3.12"
      install-dev: "true"
```

---

## 13. Debugging Failed Workflows

### Enable Debug Logging

```bash
# Set these repository secrets to enable verbose logging
ACTIONS_RUNNER_DEBUG: true
ACTIONS_STEP_DEBUG: true
```

### Re-run Failed Jobs

```bash
# Via GitHub CLI
gh run list --repo myorg/myrepo
gh run rerun <run-id> --failed    # re-run only failed jobs
gh run rerun <run-id>             # re-run entire workflow

# Watch a run in real-time
gh run watch <run-id>

# View logs
gh run view <run-id> --log
gh run view <run-id> --log-failed  # only failed jobs
```

### Common Debugging Patterns

```yaml
# Add a debug step to dump context
- name: Debug context
  run: |
    echo "github.ref: ${{ github.ref }}"
    echo "github.event_name: ${{ github.event_name }}"
    echo "github.sha: ${{ github.sha }}"
    echo "github.actor: ${{ github.actor }}"
    env | sort    # dump all environment variables (careful with secrets!)

# Test network connectivity (useful for debugging service containers)
- name: Test Postgres connection
  run: |
    sudo apt-get install -y postgresql-client
    PGPASSWORD=testpass psql -h localhost -U testuser -d testdb -c "SELECT version();"

# Check if secrets are set (without revealing values)
- name: Verify secrets
  run: |
    if [ -z "${{ secrets.AWS_DEPLOY_ROLE_ARN }}" ]; then
      echo "ERROR: AWS_DEPLOY_ROLE_ARN secret is not set"
      exit 1
    else
      echo "AWS_DEPLOY_ROLE_ARN is set (length: ${#AWS_DEPLOY_ROLE_ARN})"
    fi
  env:
    AWS_DEPLOY_ROLE_ARN: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
```

---

## 14. Common Failure Modes

### Failure 1: "Permission denied" on AWS Operations

**Symptom:** `AccessDeniedException` when deploying to ECS

**Cause A:** OIDC trust policy is too restrictive — doesn't allow this repo/branch
```json
// Wrong: too restrictive
"StringEquals": {
  "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
}

// Right: allow all branches (or be explicit about which branches deploy)
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
}
```

**Cause B:** IAM role doesn't have the right permissions for ECS deployment
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::*:role/ecsTaskExecutionRole"
    }
  ]
}
```

### Failure 2: Tests Pass Locally but Fail in CI

**Cause A:** Different Python version
```yaml
# Pin to exact same version as local
python-version: "3.12.4"   # not just "3.12"
```

**Cause B:** Service container not ready when tests start
```yaml
services:
  postgres:
    options: >-
      --health-cmd "pg_isready -U testuser"
      --health-interval 5s
      --health-retries 10   # increased retries
```

**Cause C:** Missing environment variable
```yaml
env:
  DATABASE_URL: "postgresql+asyncpg://testuser:testpass@localhost:5432/testdb"
  # Service containers in GitHub Actions are on localhost
  # NOT on the service name (unlike docker-compose)
```

### Failure 3: Docker Build Fails with "no space left on device"

```yaml
- name: Free disk space
  run: |
    sudo rm -rf /usr/share/dotnet
    sudo rm -rf /opt/ghc
    sudo rm -rf "/usr/local/share/boost"
    sudo rm -rf "$AGENT_TOOLSDIRECTORY"
    df -h   # verify space freed
```

### Failure 4: ECS Deploy Times Out

**Symptom:** `Timeout waiting for service stability` after 10 minutes

**Cause A:** New task is failing health checks — new container crashes or health check endpoint wrong
```bash
# Check ECS events
aws ecs describe-services \
  --cluster myapp-cluster \
  --services myapp-service \
  --query 'services[0].events[:5]'

# Check CloudWatch logs for new task
aws logs tail /ecs/myapp --since 10m
```

**Cause B:** Not enough ECS capacity (CPU/memory limit on cluster)

**Cause C:** Task can't pull image from ECR (IAM permission issue on execution role)

### Failure 5: Cache Not Working (Building Full Dependencies Every Run)

**Debug:**
```yaml
- name: Check cache hit
  run: |
    echo "Cache hit: ${{ steps.cache.outputs.cache-hit }}"
```

**Common cause:** Cache key includes timestamp or non-deterministic value
```yaml
# Wrong — always misses because date changes
key: pip-${{ github.run_id }}

# Right — only misses when lock file changes
key: pip-${{ runner.os }}-${{ hashFiles('uv.lock') }}
```

---

## 15. Interview Angles

### Q: "Walk me through your CI/CD pipeline."

**Great answer:**
1. PR opened → lint + tests run (with service containers for real Postgres integration tests)
2. Tests use `uv sync --frozen` with cache keyed on `uv.lock` hash — most runs skip package install
3. All PRs require lint + test to pass before merge (branch protection rules)
4. Merge to main → build job: OIDC to AWS, ECR login, `docker build-push-action` with layer cache, push `sha-abc123` and `latest` tags
5. Deploy to staging: download current task def, render new image tag, `ecs-deploy-task-definition`, wait for stability
6. Smoke test staging — hit `/health`
7. Deploy to production: same pattern, but `production` environment requires manual reviewer approval in GitHub

### Q: "How do you handle secrets in GitHub Actions?"

**Great answer:**
- **Never** in workflow YAML (even in `env:` is visible in logs if echoed)
- Short-lived credentials via OIDC (no static keys at all for AWS)
- Repository secrets for service API keys (Slack webhook, etc.)
- Environment-scoped secrets for prod vs staging differences
- `${{ secrets.MY_SECRET }}` is masked in logs — GitHub replaces with `***`
- For debugging: check if a secret is set without printing it — check `${#SECRET_VALUE}` > 0

### Q: "A deployment failed at 11pm. What do you do?"

**Answer:**
1. Check GitHub Actions run — which job failed? What step?
2. `gh run view <run-id> --log-failed` — get the exact error
3. If ECS deploy timed out: `aws ecs describe-services` for events, `aws logs tail` for task logs
4. If it was a bad deploy that succeeded but app is broken: roll back by re-deploying the previous image tag (know the previous tag from ECR or ECS task history)
5. `aws ecs update-service --force-new-deployment` with old task definition
6. Notify stakeholders via Slack, create incident ticket

---

## 16. Practice Exercise

### Build a Complete Pipeline for a FastAPI Service

**Step 1:** Create `.github/workflows/ci.yml` that:
- Triggers on push to `main` and PRs targeting `main`
- Job 1 (`lint`): ruff check + format check + mypy
- Job 2 (`test`): pytest with a Postgres service container, uploads junit results

**Step 2:** Extend with `.github/workflows/cd.yml` that:
- Triggers only on push to main (not PRs)
- Uses OIDC to authenticate to AWS (you can simulate this with `secrets.AWS_ACCESS_KEY_ID` if no OIDC)
- Builds Docker image, tags with short SHA, pushes to ECR
- Updates ECS service with new task definition

**Step 3:** Add matrix testing:
- Run tests on Python 3.11 and 3.12
- Only upload coverage from Python 3.12

**Step 4:** Add caching:
- Cache uv packages keyed on `uv.lock` hash
- Add Docker layer caching with `type=gha`
- Verify cache hit rate by checking step durations across multiple runs

**Step 5:** Add environment protection:
- Create `staging` and `production` environments in GitHub
- Add yourself as required reviewer for `production`
- Make the deploy job target the `production` environment — verify it pauses for approval

**Validation:**
- Push a code change → full pipeline runs end-to-end in under 5 minutes (with cache)
- Open a PR with a failing test → merge is blocked
- Merge to main → image appears in ECR, ECS service updates with new task

---

## Cross-References

- [01-docker.md](./01-docker.md) — Dockerfile, multi-stage builds, and image optimization
- [03-env-separation.md](./03-env-separation.md) — Environment-specific configuration and secrets
- [04-secrets-config.md](./04-secrets-config.md) — AWS Secrets Manager and secrets patterns
- [../E-AWS/01-compute.md](../E-AWS/01-compute.md) — ECS Fargate: clusters, services, task definitions
- [../E-AWS/03-iam-security.md](../E-AWS/03-iam-security.md) — IAM roles for CI/CD deployment
- [../E-AWS/05-cloudwatch-ops.md](../E-AWS/05-cloudwatch-ops.md) — CloudWatch logs and debugging deployed services
