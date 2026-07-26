# CI/CD — Basic Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — ground-level CI/CD concepts  
> **Estimated time:** 4–6 hours

---

## Table of Contents

1. CI vs CD Concepts
2. GitHub Actions Basics
3. GitLab CI Basics
4. Triggers and Events
5. Artifacts and Caching
6. Environment Variables and Secrets
7. Q&A

---

## 1. CI vs CD Concepts

### Continuous Integration (CI)

**What:** Every code push is automatically built and tested.

**Why:**
- Catch bugs early (before they reach production)
- Ensure code compiles/passes tests
- Maintain code quality (linting, static analysis)
- Reduce integration risk (merge conflicts)

**CI pipeline:**
```
Code Push → Checkout → Install Dependencies → Lint → Unit Tests → Build
```

### Continuous Delivery (CD)

**What:** Every CI-passing build is automatically deployable. Deployment to production may be manual.

**Why:**
- Reduce deployment risk (deploy more often, smaller changes)
- Automated, repeatable deployments
- Fast feedback loop

**CD pipeline (extends CI):**
```
... → Build → Package → Deploy to Staging → Integration Tests → Manual Approval → Deploy to Production
```

### Continuous Deployment

**What:** Every CI-passing build is automatically deployed to production. No manual approval.

**Why:**
- Fastest path to production
- Requires high confidence in tests and monitoring
- Common in SaaS companies with mature testing

### CI/CD Pipeline Stages

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Source  │ │   Build  │ │   Test   │ │  Scan    │ │ Package  │ │  Deploy  │
│ (checkout)│ │ (compile)│ │(unit/int)│ │(security)│ │(artifact)│ │(env)     │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

> **Trap:** CI and CD are often conflated. CI = testing every commit. CD = deploying every commit (or on demand). You can have CI without CD (many teams do this).

---

## 2. GitHub Actions Basics

### Workflow structure

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  APP_ENV: testing
  DB_DATABASE: inventory_test

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: inventory_test
          POSTGRES_USER: app
          POSTGRES_PASSWORD: password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: pdo_pgsql, mbstring, bcmath
          tools: composer

      - name: Cache Composer dependencies
        uses: actions/cache@v4
        with:
          path: vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: |
            ${{ runner.os }}-composer-

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Run lint
        run: vendor/bin/pest --lint

      - name: Run tests
        run: vendor/bin/pest --parallel
        env:
          DB_HOST: localhost
          DB_PASSWORD: password

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          tools: composer
      - run: composer install --no-interaction --prefer-dist
      - name: PHPStan
        run: vendor/bin/phpstan analyse --level=max src/
```

### Workflow components

| Component | Description |
|-----------|-------------|
| `name` | Workflow name (displayed in Actions tab) |
| `on` | Trigger event(s) — push, PR, schedule, etc. |
| `env` | Environment variables available to all jobs |
| `jobs` | Set of jobs that run in parallel by default |
| `job.name` | Display name for the job |
| `job.runs-on` | Runner type (ubuntu-latest, windows-latest, self-hosted) |
| `job.services` | Docker containers for dependencies (DB, Redis, etc.) |
| `job.steps` | Sequential tasks within a job |
| `step.uses` | Reusable action from marketplace |
| `step.run` | Shell command |

### Runner types

| Runner | Description |
|--------|-------------|
| `ubuntu-latest` | Linux (default) — most common |
| `windows-latest` | Windows — for .NET, Windows-specific builds |
| `macos-latest` | macOS — for iOS, macOS builds |
| `self-hosted` | Your own infrastructure — custom config, no usage limits |

### Common actions

```yaml
# Checkout code
- uses: actions/checkout@v4

# Setup language
- uses: actions/setup-node@v4
- uses: actions/setup-python@v5
- uses: actions/setup-go@v5

# Cache dependencies
- uses: actions/cache@v4

# Upload/download artifacts
- uses: actions/upload-artifact@v4
- uses: actions/download-artifact@v4

# Docker
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
- uses: docker/build-push-action@v5

# AWS
- uses: aws-actions/configure-aws-credentials@v4

# GitHub
- uses: actions/create-release@v1
- uses: actions/labeler@v5
```

### Matrix builds

Test across multiple versions/configurations:

```yaml
jobs:
  test:
    strategy:
      matrix:
        php: ['8.1', '8.2', '8.3']
        laravel: ['10', '11']
        os: [ubuntu-latest, ubuntu-22.04]
      fail-fast: false  # continue even if one fails

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
      - run: composer require laravel/framework:^${{ matrix.laravel }}
      - run: vendor/bin/pest
```

### Conditional execution

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production"

  notify:
    needs: deploy
    if: always()  # always run (even if deploy failed)
    steps:
      - run: echo "Deployment status: ${{ job.status }}"
```

Common conditions:

| Expression | Description |
|------------|-------------|
| `github.ref == 'refs/heads/main'` | Main branch only |
| `github.event_name == 'pull_request'` | PR events only |
| `startsWith(github.ref, 'refs/tags/')` | Tag pushes |
| `success()` | All previous steps succeeded |
| `failure()` | Any previous step failed |
| `always()` | Always run (regardless of success/failure) |
| `cancelled()` | Workflow was cancelled |

---

## 3. GitLab CI Basics

### `.gitlab-ci.yml` structure

```yaml
stages:
  - build
  - test
  - deploy

variables:
  APP_ENV: testing
  DB_DATABASE: inventory_test

cache:
  paths:
    - vendor/

before_script:
  - composer install --no-interaction --prefer-dist

build:
  stage: build
  script:
    - php artisan optimize
  artifacts:
    paths:
      - bootstrap/cache/
    expire_in: 1 hour

test:phpunit:
  stage: test
  script:
    - vendor/bin/pest --parallel
  services:
    - name: postgres:16-alpine
      alias: postgres
  variables:
    DB_HOST: postgres
    DB_PASSWORD: password

test:phpstan:
  stage: test
  script:
    - vendor/bin/phpstan analyse src/ --level=max

deploy:staging:
  stage: deploy
  script:
    - echo "Deploy to staging"
  environment:
    name: staging
  only:
    - develop

deploy:production:
  stage: deploy
  script:
    - echo "Deploy to production"
  environment:
    name: production
  when: manual
  only:
    - main
```

### Key differences from GitHub Actions

| Aspect | GitHub Actions | GitLab CI |
|--------|---------------|-----------|
| Config location | `.github/workflows/*.yml` | `.gitlab-ci.yml` (root) |
| Job parallelism | Default (all jobs run in parallel) | Ordered by `stages` |
| Dependencies | `needs:` | Implicit by stage |
| Services | `job.services` | `job.services` |
| Cache | `actions/cache` action | `cache:` keyword |
| Matrix | `strategy.matrix` | `parallel:` with variables |
| Self-hosted | Self-hosted runners | GitLab Runners |
| Auto DevOps | No | Yes (templates for common apps) |

---

## 4. Triggers and Events

### GitHub Actions events

```yaml
on:
  push:                              # Branch/tag push
    branches: [main, develop]        # filter by branch
    paths:                           # filter by file paths
      - 'src/**'
      - '!docs/**'

  pull_request:                      # PR events
    types: [opened, synchronize, reopened]

  pull_request_target:               # PR with access to secrets (careful!)
    types: [labeled]

  schedule:                          # Cron job
    - cron: '0 3 * * *'             # Every day at 3 AM

  workflow_dispatch:                 # Manual trigger
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production

  release:                           # GitHub Release
    types: [published]

  workflow_run:                      # Another workflow completes
    workflows: ["Build"]
    types: [completed]
```

### GitLab CI triggers

```yaml
# Push to branches
on: push to main

# Merge request
on: merge request to main

# Schedule
on: schedule at '0 3 * * *'

# Manual
on: when: manual

# Pipeline trigger (via API)
on: pipeline trigger from API

# Multi-project pipeline
on: pipeline from upstream project
```

---

## 5. Artifacts and Caching

### Artifacts (pass files between jobs)

```yaml
# Upload in one job
jobs:
  build:
    steps:
      - run: php artisan optimize
      - uses: actions/upload-artifact@v4
        with:
          name: build-artifacts
          path: |
            bootstrap/cache/
            public/build/
          retention-days: 5

# Download in another job
  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-artifacts
      - run: echo "Deploying..."
```

### Caching (speed up dependency installation)

```yaml
# Composer cache
- uses: actions/cache@v4
  with:
    path: ~/.cache/composer
    key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
    restore-keys: |
      ${{ runner.os }}-composer-

# NPM cache
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# Docker layer cache
- uses: docker/setup-buildx-action@v3
- uses: actions/cache@v4
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```

> **Trap:** Cache keys should include the lockfile hash. If you don't, old cache persists even after dependency changes.

---

## 6. Environment Variables and Secrets

### GitHub Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # production environment with protection rules
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Login to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_URL }}
```

**Secret types:**
- Repository secrets: available across all workflows in the repo
- Environment secrets: only available for specific environment deployments
- Organization secrets: shared across repos in the organization
- Dependabot secrets: for Dependabot PRs

### OIDC (no long-lived secrets)

GitHub Actions can authenticate to AWS via OIDC — no need to store AWS credentials:

```yaml
permissions:
  id-token: write   # needed for OIDC
  contents: read    # needed for checkout

jobs:
  deploy:
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: us-east-1
```

Configure the IAM role with a trust policy that allows GitHub's OIDC provider:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": "repo:myorg/inventory:ref:refs/heads/main"
    }
  }
}
```

---

## 7. Q&A

### Basic

**Q: What's the difference between CI and CD?**
A: CI = build and test every commit. CD = deploy every commit (or on demand) to production. CD extends CI with deployment steps.

**Q: What's a GitHub Actions workflow?**
A: An automated process defined in YAML, triggered by events (push, PR, schedule). Contains jobs with steps.

**Q: What's the difference between `actions/upload-artifact` and `actions/cache`?**
A: Artifacts pass files between jobs in a single workflow. Cache stores dependencies across workflow runs (keyed by lockfile hash).

**Q: How do you run tests with a PostgreSQL database in GitHub Actions?**
A: Use `services:` to spin up a PostgreSQL container. Set environment variables for the test runner to connect to `localhost`.

**Q: What's a matrix build?**
A: Test across multiple combinations (PHP versions, Laravel versions, OS). `strategy.matrix` defines the combinations.

**Q: What's `if:` used for in GitHub Actions?**
A: Conditional execution. Skip steps/jobs based on branch, event type, or previous step outcomes.

**Q: How do you handle secrets in GitHub Actions?**
A: Repository/Environment secrets. Accessed via `${{ secrets.SECRET_NAME }}`. Never print secrets in logs.

### Trap questions

**Q: A GitHub Actions workflow with `on: push` runs twice for the same commit. Why?**
A: If you push to a branch that has an open PR, both `push` and `pull_request` events fire. Use `on: pull_request` with `types: [opened, synchronize]` instead—or filter by branch.

**Q: A job runs but doesn't complete because it times out after 6 hours. What's the limit?**
A: GitHub Actions has a 360-minute (6 hour) execution time limit per job. For longer jobs, use self-hosted runners.

**Q: A workflow step fails but the job continues. Why?**
A: By default, GitHub Actions uses `set -e` (exit on error). Some commands (like `curl`) may not fail on HTTP errors. Use `set -euo pipefail` explicitly or check exit codes.

**Q: You reference a secret but the workflow shows `***` in logs. Is the secret working?**
A: GitHub masks secrets in logs (shows `***`). This is expected. However, if the command doesn't actually receive the secret value, test by using it in a non-output way (like a shell command).

**Q: A workflow is queued but never runs. What's the cause?**
A: (1) Concurrent workflow limit reached (60 concurrent jobs across all workflows). (2) Minutes exhausted (free tier: 2,000 min/month). (3) Self-hosted runner offline.

### Follow-up questions

**Q: How do you promote an artifact from staging to production?**
A: Build once in CI, upload as artifact. In CD, download the same artifact and deploy. Don't rebuild in CD — that introduces risk of building with different deps.

**Q: What's the difference between GitHub-hosted and self-hosted runners?**
A: GitHub-hosted: managed by GitHub, clean environment per job, minutes billing. Self-hosted: your own machines, reusable environments, no minutes limit, can access internal networks.

**Q: How do you handle database migrations in CI/CD?**
A: Migrations run in the CD step before the new code deploys (deployment order: run migrations → deploy new code). For zero-downtime, migrations must be backward-compatible (add columns before querying them, remove columns after old code is gone).
