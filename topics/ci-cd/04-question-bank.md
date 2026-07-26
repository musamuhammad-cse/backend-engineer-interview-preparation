# CI/CD — Question Bank

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Type:** 120+ rapid-fire Q&A, pipeline design scenarios, debugging, deployment failure response

---

## Table of Contents

1. Rapid-Fire Q&A (120+ questions)
2. Pipeline Design Scenarios
3. Debugging Scenarios
4. Deployment Failure Response (STAR Templates)

---

## 1. Rapid-Fire Q&A

### CI/CD Concepts

**Q: What's the difference between CI and CD?**
A: CI = build and test every commit. CD = deploy every commit (or on demand) to production.

**Q: What's continuous deployment vs continuous delivery?**
A: Continuous delivery = deployable state but manual approval for production. Continuous deployment = auto-deploy to production after CI passes.

**Q: What are DORA metrics?**
A: Deployment frequency, lead time for changes, change failure rate, MTTR.

**Q: What's a rollout vs a release?**
A: Rollout = code change in production. Release = feature is accessible to users. They should be decoupled (feature flags).

**Q: What's the ideal deployment frequency for an elite team?**
A: On-demand (multiple deployments per day).

**Q: What's the ideal lead time for changes?**
A: Less than 1 hour.

### GitHub Actions

**Q: What are the components of a GitHub Actions workflow?**
A: `name`, `on` (triggers), `env`, `jobs` (with `runs-on`, `steps`, `needs`).

**Q: How do you pass data between jobs in the same workflow?**
A: `actions/upload-artifact` and `actions/download-artifact`. Outputs via `$GITHUB_OUTPUT`.

**Q: How do you cache dependencies in GitHub Actions?**
A: `actions/cache@v4` with a key based on the lockfile hash.

**Q: What's the difference between `actions/checkout` and fetching the code manually?**
A: `actions/checkout` handles auth, submodules, depth, and works with the GitHub token.

**Q: How do you run a job conditionally?**
A: `if:` expression (e.g., `if: github.ref == 'refs/heads/main'`).

**Q: What's a matrix build?**
A: Test across multiple versions/configurations using `strategy.matrix`.

**Q: How do you stop the workflow if one matrix combination fails?**
A: Default is stop all (`fail-fast: true`). Set `fail-fast: false` to continue.

**Q: What's the difference between `on: push` and `on: pull_request`?**
A: `push` = code pushed to a branch. `pull_request` = a PR is opened/updated (runs on merge commit).

**Q: How do you trigger a workflow manually?**
A: `workflow_dispatch` event with optional `inputs`.

**Q: How do you schedule a workflow?**
A: `schedule` with cron syntax: `- cron: '0 3 * * *'`.

**Q: What's a self-hosted runner?**
A: Runner you manage on your own infrastructure. Has access to internal networks, bigger resources, no minute limits.

**Q: How do you authenticate to AWS from GitHub Actions?**
A: OIDC (best practice). Or store AWS access keys in secrets.

**Q: What's the job execution time limit?**
A: 360 minutes (6 hours) for GitHub-hosted runners. 6 hours for each job.

**Q: What's the workflow run time limit?**
A: 35 days (workflow must complete within 35 days of triggering).

**Q: What's an environment in GitHub Actions?**
A: Named target (production, staging) with protection rules, secrets, and deployment tracking.

**Q: How do approval gates work in environments?**
A: Configure "required reviewers" in environment settings. Deployments wait for approval.

**Q: What's the difference between repository secrets and environment secrets?**
A: Repository secrets available to all workflows. Environment secrets only to jobs targeting that environment.

### GitLab CI

**Q: What's the main difference between GitLab CI stages and GitHub Actions jobs?**
A: GitLab stages run sequentially (stage 1 → stage 2). GitHub Actions jobs are parallel by default.

**Q: How do you define manual jobs in GitLab CI?**
A: `when: manual` on the job.

**Q: How do you pass artifacts between GitLab CI jobs?**
A: `artifacts:` keyword in the producing job, downloaded in dependent jobs.

**Q: What's a GitLab Runner?**
A: Agent that runs GitLab CI jobs. Can be shared, group, or specific.

### Deployment Strategies

**Q: What's a rolling update?**
A: Gradual replacement of old instances/versions. K8s default.

**Q: What's a blue-green deployment?**
A: Two identical environments. Blue = current, green = new. Switch traffic when ready.

**Q: What's a canary deployment?**
A: Gradually shift traffic (5% → 20% → 100%). Real-world validation before full rollout.

**Q: What's the fastest rollback strategy?**
A: Blue-green (switch traffic back) and canary (shift traffic back) are instant. Rolling requires waiting for gradual replacement.

**Q: What's the main risk of blue-green deployments?**
A: Double infrastructure cost. DB schema must be backward-compatible.

**Q: What's the main risk of canary deployments?**
A: Uneven traffic distribution (non-representative canary). Requires traffic routing.

**Q: What's the safest deployment strategy for critical infrastructure?**
A: Canary with automated metrics-based rollback (Flagger).

**Q: What's a recreate strategy?**
A: Stop all old instances, start new. Causes downtime.

### Pipeline Security

**Q: How do you prevent secrets from leaking in CI logs?**
A: GitHub Actions masks secrets in logs. Never echo or print secrets. Use OIDC.

**Q: What's OIDC in CI/CD?**
A: OpenID Connect — CI tool gets short-lived tokens from a cloud provider. No long-lived secrets.

**Q: What's an SBOM?**
A: Software Bill of Materials — list of all dependencies and their versions in a build.

**Q: What's the difference between SAST and DAST?**
A: SAST = source code analysis (static). DAST = running application analysis (dynamic).

**Q: What is supply chain security?**
A: Securing the software pipeline: dependencies, build tools, CI/CD, artifacts, deployment.

**Q: What's cosign used for?**
A: Container image signing (verifies image integrity and origin).

**Q: What's Dependabot?**
A: GitHub's automated dependency update tool. Creates PRs for vulnerable/outdated dependencies.

### GitOps

**Q: What is GitOps?**
A: Git is the single source of truth for both application code AND infrastructure/configuration. Agents reconcile cluster state to match Git.

**Q: What's the difference between Argo CD and Flux?**
A: Argo CD: web UI, push-based sync, SSO. Flux: controller-only, pull-based reconciliation, smaller footprint.

**Q: What are the benefits of GitOps?**
A: Audit trail, easy rollback (Git revert), drift detection, disaster recovery, multi-cluster management.

**Q: What's drift detection?**
A: GitOps agent detects when cluster state differs from Git and either alerts or auto-reconciles.

**Q: How does CI integrate with GitOps?**
A: CI builds image + updates Git manifest → GitOps agent detects change → syncs to cluster.

### Terraform in CI/CD

**Q: How do you run Terraform in CI/CD?**
A: `terraform init` → `terraform plan` (in PR) → `terraform apply` (on merge to main).

**Q: What's Atlantis?**
A: Terraform CI/CD tool. `atlantis plan` on PR comments → `atlantis apply` after approval.

**Q: How do you manage Terraform state in CI?**
A: Remote backend (S3 + DynamoDB). Never local state in CI.

**Q: How do you review Terraform changes in PRs?**
A: `terraform plan` output should be commented on PR by CI. Reviewers verify the plan before merging.

### General

**Q: What's the difference between a build artifact and a Docker image?**
A: Build artifact = compiled files (JAR, ZIP, PHAR). Docker image = container image with app + runtime.

**Q: Why should you build artifacts once for staging and production?**
A: Ensure you deploy EXACTLY what was tested in staging. Rebuilding introduces risk.

**Q: What's the purpose of a CI gating check?**
A: Block merge if tests/security scans fail. Ensures main branch is always deployable.

**Q: What's trunk-based development?**
A: Short-lived branches (< 1 day), merge to main frequently, no long-lived feature branches.

**Q: What's the difference between push-based and pull-based deployments?**
A: Push = CI pushes to target (needs credentials). Pull = target pulls from registry (GitOps).

**Q: What's a pre-commit hook?**
A: Script that runs before `git commit` — lint, format, secret scanning.

**Q: How do you handle failing tests in CI?**
A: Block merge. Investigate and fix. Re-run CI. Never merge with red tests.

**Q: What's a flaky test?**
A: Test that sometimes passes, sometimes fails without code changes. Should be quarantined and fixed separately.

**Q: How do you measure deployment quality?**
A: Change failure rate (% of deployments causing incidents), MTTR, error budget burn rate.

**Q: What's an error budget?**
A: SLA minus actual uptime = allowed error budget. Can use error budget to decide when to slow down releases.

---

## 2. Pipeline Design Scenarios

### Scenario 1: Laravel monolith CI/CD

**Requirements:**
- PHP 8.2, PostgreSQL, Redis
- Pest tests (4,000+)
- NPM for frontend (Vite)
- Deploy to ECS Fargate
- Dev/staging/prod environments

**Pipeline design:**
```yaml
name: CI/CD

on:
  pull_request:
    paths: ['src/**', 'tests/**', 'composer.*']
  push:
    branches: [main, develop]

jobs:
  lint:
    runs: [php:8.2]
    steps: [Pint, PHPStan]

  test:
    needs: lint
    runs: [php:8.2]
    services: [postgres, redis]
    steps: [Composer install, Pest parallel]

  frontend:
    runs: [node:20]
    steps: [NPM ci, Vite build]

  build:
    needs: [test, frontend]
    steps: [Docker build, Push to ECR]

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: development

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging

  e2e:
    needs: deploy-staging
    steps: [Playwright tests]

  deploy-prod:
    needs: e2e
    if: github.ref == 'refs/heads/main'
    environment: production
    # requires manual approval
```

### Scenario 2: Microservices (20 services)

**Requirements:**
- 20 Go/PHP services
- Shared CI template
- Independent deploy per service
- Monorepo (shared tooling)

**Pipeline design:**
- Single `.github/workflows/ci.yml` with matrix per service
- Change detection: only build/test changed services
- Shared composite action for deploy
- Argo CD + Helm per service
- DORA dashboard

```yaml
jobs:
  detect-changes:
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api: 'services/api/**'
            orders: 'services/orders/**'
            inventory: 'services/inventory/**'

  build:
    needs: detect-changes
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect-changes.outputs.changed) }}
    steps:
      - uses: ./.github/actions/build-service
        with:
          service: ${{ matrix.service }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect-changes.outputs.changed) }}
    steps:
      - uses: ./.github/actions/deploy-service
        with:
          service: ${{ matrix.service }}
```

### Scenario 3: Terraform + application deployment

**Requirements:**
- Terraform for infrastructure
- Docker for application
- CI/CD for both

**Pipeline design:**
```
Infrastructure changes:
  PR → terraform plan (comment on PR) → merge → terraform apply

Application changes:
  PR → CI (test, build) → merge → build image → update Helm values →
  Argo CD sync → deploy
```

### Scenario 4: Database migration pipeline

**Requirements:**
- Zero-downtime DB migrations
- Automatic rollback on failure
- Backward-compatible schema changes

**Pipeline design:**
```
1. CI: Build + Test (with pre-migration state)
2. CD Pre-Deploy: Run backward-compatible migrations
   - Add columns (nullable), add tables
   - Monitor for errors, lock time, replication lag
3. CD Deploy: Deploy new application code
   - Rolling update with readiness check
4. CD Post-Deploy (after X hours): Cleanup migrations
   - Remove old columns, remove unused tables
5. Rollback plan:
   - If deploy fails: rollback code (old code works with new schema)
   - If migration fails: manual rollback via pre-migration snapshot
```

### Scenario 5: GitOps migration

**Requirements:**
- Migrate from Jenkins push-based to Argo CD GitOps
- Zero-downtime migration

**Migration steps:**
```
1. Deploy Argo CD alongside existing deployment (no conflict)
2. Create Git repo with current K8s manifests (argocd bootstrap)
3. Set Argo CD to sync (creates same resources — no-op)
4. Add "managed-by: argocd" annotation
5. Remove Jenkins deploy job (Argo CD now manages)
6. Set up CI to update Git manifests (not deploy directly)
7. Test drift detection (manually change resource → Argo CD reverts)
8. Enable `selfHeal: true`
```

---

## 3. Debugging Scenarios

### Scenario 1: CI passes but deploy fails

**Symptom:** All CI tests pass. Deployment to ECS fails with "Task failed to start."

**Debug:**
1. Check ECS stopped reason (role permissions? VPC? image not found?)
2. Compare CI and ECS environments: different platform architecture? (CI is x86, image was built for ARM)
3. Check ECR image — does the tag exist? Was it pushed?
4. Check AWS credentials — does deploy role have correct permissions?
5. Check task definition — correct container port? correct command?

**Root cause:** Docker built with `--platform linux/arm64` (M1 Mac CI runner) but ECS uses x86 EC2 instances.

### Scenario 2: Canary deployment fails silently

**Symptom:** Canary deploys (5% traffic). Metrics look normal. After 1 hour ramp to 100%, error rate spikes.

**Debug:**
1. Check if canary received representative traffic (maybe only health checks hit it)
2. Check canary's readiness — was it truly ready when traffic was sent?
3. Check downstream services — canary connects to different DB/API?
4. Check session affinity — users were pinned to old version
5. Check feature flag — canary had a flag enabled that wasn't tested

**Root cause:** Canary used percentage-based routing without session stickiness. New users (with simpler queries) hit the canary. Returning users (with complex queries) hit the old version. At 100%, returning users hit the new code with complex queries → timeouts.

### Scenario 3: GitHub Actions minutes exhausted

**Symptom:** Workflow starts but hangs or fails with "This workflow has been disabled."

**Debug:**
1. Check billing — free tier minutes at 0 (2,000 min/month)
2. Check if workflows are using larger runners (charged at 2x minutes)
3. Check for unnecessary matrix builds (testing 8 PHP versions when 3 suffice)
4. Check for inefficient caching (Docker not using layer cache)

**Solution:**
- Optimize CI (faster tests, better caching)
- Use self-hosted runners (no minute limits)
- Request paid plan

### Scenario 4: Deployment rollback fails

**Symptom:** New version crashes. You run `kubectl rollout undo` but it fails.

**Debug:**
1. Check if old image still exists in registry (ECR lifecycle policy may have deleted)
2. Check if old Helm release was deleted (`helm list --all`)
3. Check if rollback requires old PVC/PV (deleted by StatefulSet)
4. Check if rollback tries to use a ConfigMap that doesn't exist (deleted by Helm)

**Root cause:** ECR lifecycle policy set to keep only 10 most recent images. After 11 deployments, oldest image is deleted. Rollback to version 1 fails because image no longer exists.

**Prevention:** Set ECR lifecycle policy to keep minimum 100 images or tag-protect production images.

### Scenario 5: Pipeline takes too long

**Symptom:** CI takes 45 minutes. Developers wait too long to merge.

**Debug:**
1. Check slowest step (test suite? Docker build? lint?)
2. Check test parallelism (using `—parallel`? sharding?)
3. Check dependency caching (Composer lock cached?)
4. Check Docker layer caching (BuildKit cache?)
5. Check if all tests run on every commit (slow tests only on merge?)

**Optimizations:**
- Parallel test sharding (`—parallel —processes=4`)
- Run unit tests (fast) on every commit, integration tests on PR merge
- Docker layer caching (GitHub Actions cache or ECR caching)
- Skip unnecessary matrix combinations
- Use bigger runners (8-core, 16-core)

---

## 4. Deployment Failure Response (STAR Templates)

### Template 1: Failed production deploy

**Situation:** Deploying a new version of the inventory API to ECS. After rolling update completes, error rate jumps from 0.1% to 15%. P95 latency increases 5x.

**Task:** Respond to the incident and restore service.

**Action:**
```
1. Detect: CloudWatch alarm triggers (5xx errors > 5%). On-call engineer acknowledges.
2. Assess: Check error logs — new query pattern missing an index. New query was added in this release.
3. Rollback: Run `kubectl rollout undo deployment/api` — rollback to previous version (rolling update, ~2 min).
4. Verify: Error rate drops to 0.1%. Latency normalizes.
5. Fix: Add missing index to staging DB. Run migration in production. Add migration to pipeline.
6. Deploy: Fixed version deployed successfully. Index migration ran before deploy.
7. Post-mortem: Add migration step to CI/CD checklist. Add index check to review process.
```

**Result:** MTTR: 8 minutes. Root cause: missing DB index on new query. Fix: added index migration to pre-deploy step.

---

### Template 2: Canary auto-rollback

**Situation:** Using Flagger for canary releases. A new version is deployed to 10% traffic. After 2 minutes, Flagger automatically rolls back.

**Task:** Investigate and fix the canary failure.

**Action:**
```
1. Check Flagger events: `kubectl describe canary/api` — "metrics check failed: request-success-rate < 99%"
2. Check metrics: Prometheus shows error rate at 5% on canary pods vs 0.1% on primary.
3. Check canary logs: New version has a bug in the payment API call (wrong authentication header).
4. Fix: Correct the header, rebuild image, update Helm values.
5. Deploy: Flagger detects new image, starts new canary analysis.
6. Verify: Canary at 10% → metrics pass → auto-promote to 50% → 100%.
```

**Result:** Automated rollback prevented full outage. Bug caught before reaching 100% traffic. MTTR: 15 minutes (including fix).

---

### Template 3: Failed DB migration

**Situation:** A migration to add a NOT NULL column to `orders` table fails because existing rows have NULL values. Migration runs in pre-deploy step.

**Task:** Handle the failure without downtime.

**Action:**
```
1. Migration fails: `ALTER COLUMN SET NOT NULL` fails on rows with NULL values.
2. Assess: Migration script didn't handle existing NULLs. Need to backfill first.
3. Immediate: Pipeline stops (migration failed = deploy blocked). No downtime — old code still running.
4. Fix: Create new migration:
   a. UPDATE orders SET shipping_method = 'default' WHERE shipping_method IS NULL;
   b. ALTER COLUMN shipping_method SET NOT NULL;
5. Deploy: Run new migration (fast — few rows needed backfill). Then deploy code.
6. Prevention: Add data integrity check to migration review. Always backfill before adding NOT NULL.
```

**Result:** Zero downtime (deploy was blocked, no bad code reached production). Migration fixed and deployed successfully.

---

### Template 4: Secret rotation incident

**Situation:** GitHub security alert detects an AWS access key committed to a private repo. The key has S3 read/write access.

**Task:** Rotate the key and prevent recurrence.

**Action:**
```
1. Immediate: Rotate the compromised key in AWS IAM. Deactivate old key, generate new.
2. Assess: Check GitHub audit log — who pushed the key? When? Was the repo public or private? Any forks?
3. Check: Scan CloudTrail for unauthorized S3 access using the compromised key.
   - No suspicious activity found (key was only in CI, rotated within 5 min of alert).
4. Update: Update GitHub secrets with new key. Update any services using the old key.
5. Prevention:
   a. Enable pre-commit secret scanning (detect-secrets, git-secrets).
   b. Migrate to OIDC (remove long-lived AWS keys from CI).
   c. Add branch protection rules (prevent force push to main).
   d. Run secret scanning on all repos.
6. Post-mortem: Developer accidentally committed `.env` file with AWS keys. `.env` was in `.gitignore` but was force-added.
```

**Result:** No data breach. Key rotated within 5 minutes. Migrated to OIDC — no more static AWS keys in CI.
