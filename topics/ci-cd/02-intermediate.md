# CI/CD — Intermediate Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** CI/CD basics (workflows, triggers, jobs)  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Deployment Strategies
2. Multi-Environment Pipelines
3. Docker Image Build and Push
4. Database Migrations in Pipelines
5. Approval Gates
6. GitHub Actions vs Other CI/CD Tools
7. Q&A

---

## 1. Deployment Strategies

### Rolling Update

Gradually replaces old pods/instances with new ones. Default in K8s.

```
Time ──────────────────────────────────►
┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐     ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
│v1│ │v1│ │v1│ │v1│ │v1│ ──► │v2│ │v2│ │v2│ │v2│ │v2│
└──┘ └──┘ └──┘ └──┘ └──┘     └──┘ └──┘ └──┘ └──┘ └──┘
```

**Pros:** No additional infrastructure cost, gradual rollback possible  
**Cons:** Both versions coexist (incompatible DB schemas are risky), slower rollout

```yaml
# K8s RollingUpdate
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

### Blue-Green Deployment

Two identical environments. Blue = current. Green = new. Switch traffic.

```
Before:               After:
┌──────┐              ┌──────┐
│ Blue │◄─ Traffic    │ Blue │  (idle)
│ (v1) │              │ (v1) │
└──────┘              └──────┘
                      ┌──────┐
                      │ Green│◄─ Traffic
                      │ (v2) │
                      └──────┘
```

**Pros:** Instant switch, easy rollback (switch back), full environment testing  
**Cons:** Double infrastructure cost, DB schema compatibility needed

```yaml
# ECS Blue/Green with CodeDeploy
deployment_controller:
  type: CODE_DEPLOY

# K8s — two Deployments + Service selector switch
# or Istio: VirtualService with weights
```

### Canary Deployment

Gradually shift small percentage of traffic to new version.

```
Traffic:  v1: 100% → v1: 95%, v2: 5% → v1: 80%, v2: 20% → ... → v2: 100%
```

**Pros:** Real user validation before full rollout, minimal blast radius  
**Cons:** Requires traffic routing control (service mesh, API gateway), slower rollout

```yaml
# Istio VirtualService — canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
    - api-service
  http:
    - route:
        - destination:
            host: api-service
            subset: v1
          weight: 95
        - destination:
            host: api-service
            subset: v2
          weight: 5
```

### A/B Testing

Route based on user attributes (header, cookie, geography):

```yaml
# Route users with "X-Experiment: new-checkout" header to v2
http:
  - match:
      - headers:
          X-Experiment:
            exact: "new-checkout"
    route:
      - destination:
          host: api-service
          subset: v2
  - route:
      - destination:
          host: api-service
          subset: v1
```

### Recreate

Stop all old instances, start all new instances.

**Pros:** Simple, no version coexistence  
**Cons:** Downtime

```yaml
strategy:
  type: Recreate
```

### Strategy comparison

| Strategy | Zero-downtime | Rollback speed | Cost | Complexity |
|----------|---------------|----------------|------|------------|
| Rolling | Yes | Gradual | Same | Low |
| Blue-green | Yes | Instant | 2x | Medium |
| Canary | Yes | Instant | Same + routing | High |
| Recreate | No | Slow | Same | Low |

> **Trap:** Blue-green and canary require DB schema backward-compatible. You CANNOT drop a column in the new version if the old version still reads it. Schema changes must be additive (add before use, remove after old code is gone).

---

## 2. Multi-Environment Pipelines

### Pipeline flow

```
Develop ──► Build & Test ──► Deploy to Dev ──► Tests ──► Deploy to Staging ──► Approve ──► Deploy to Production
```

### GitHub Actions — multi-environment

```yaml
name: Deploy

on:
  push:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build-image.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        id: build-image
        run: |
          TAG=${{ github.sha }}
          docker build -t api:$TAG .
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://dev.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to dev"

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to staging"

  e2e-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Run E2E tests
        run: |
          echo "Running Cypress/Playwright tests against staging"

  deploy-production:
    needs: e2e-tests
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to production"
```

### Environment protection rules

In GitHub → Settings → Environments → Add environment rules:

- **Required reviewers** — specific users/teams must approve
- **Wait timer** — delay before deployment
- **Deployment branches** — restrict which branches can deploy
- **Environment secrets** — secrets only available in this environment

### GitLab CI — multi-environment

```yaml
stages:
  - build
  - deploy
  - test
  - deploy-production

build:
  stage: build
  script:
    - docker build -t api:$CI_COMMIT_SHA .
    - echo $CI_COMMIT_SHA > image-tag.txt
  artifacts:
    paths: [image-tag.txt]

.deploy: &deploy
  stage: deploy
  script:
    - echo "Deploying $(cat image-tag.txt) to $CI_ENVIRONMENT_NAME"

deploy:dev:
  extends: .deploy
  environment:
    name: development
  only:
    - develop

deploy:staging:
  extends: .deploy
  environment:
    name: staging
  only:
    - main

e2e:
  stage: test
  script:
    - echo "Running E2E tests against staging"
  environment:
    name: staging

deploy:production:
  extends: .deploy
  stage: deploy-production
  environment:
    name: production
  when: manual  # requires manual approval
  only:
    - main
```

---

## 3. Docker Image Build and Push

### GitHub Actions — Docker build to ECR

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: inventory-api

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-ecr
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate image tags
        id: meta
        run: |
          echo "tags=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}" >> $GITHUB_OUTPUT
          echo "tags=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest" >> $GITHUB_OUTPUT

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Image tagging strategies

| Strategy | Example | Use case |
|----------|---------|----------|
| Git SHA | `api:abc123de` | Traceable, immutable |
| Semantic version | `api:1.2.3` | Release tracking |
| Git tag | `api:v1.2.3` | Git tag = Docker tag |
| Branch + SHA | `api:main-abc123de` | Per-branch images |
| `latest` | `api:latest` | Avoid in production (mutable) |

> **Trap:** Docker image `latest` tag is mutable. If you deploy using `:latest`, you can't tell which version is running. Always use Git SHA or semantic version for deployments.

---

## 4. Database Migrations in Pipelines

### Migration strategy

**Order matters:**

```
1. Run backward-compatible migrations  (add columns, add tables)
2. Deploy new application code
3. Run cleanup migrations              (remove old columns, remove unused tables)
```

### Pipeline integration

```yaml
jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Run migrations
        run: |
          php artisan migrate --force
        env:
          DB_HOST: ${{ secrets.DB_HOST }}
          DB_DATABASE: ${{ secrets.DB_DATABASE }}
          DB_USERNAME: ${{ secrets.DB_USERNAME }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    steps:
      - name: Deploy new code
        run: |
          echo "Deploying new version"
```

### Backward-compatible migration example

```php
// OK: adding a nullable column — old code ignores it
Schema::table('orders', function (Blueprint $table) {
    $table->string('shipping_method')->nullable();
});

// OK: adding a column with default — old code uses default
Schema::table('orders', function (Blueprint $table) {
    $table->boolean('is_priority')->default(false);
});

// DANGEROUS: removing a column — old code crashes if it reads it
Schema::table('orders', function (Blueprint $table) {
    $table->dropColumn('legacy_field');
});

// DANGEROUS: renaming a column — old code uses old name
Schema::table('orders', function (Blueprint $table) {
    $table->renameColumn('old_name', 'new_name');
});

// DANGEROUS: change column type — may lock table, break old code
Schema::table('orders', function (Blueprint $table) {
    $table->string('status')->change();  // changes VARCHAR length
});
```

### Migration strategies for zero-downtime

| Strategy | Description |
|----------|-------------|
| **Expand-Migrate-Contract** | Add column → Deploy → Remove old column |
| **Dual writes** | Write to old + new columns/schemas simultaneously |
| **Feature flags** | Gate new features behind flags (enable after old code is gone) |
| **Shadow tables** | Create new table alongside old, migrate data, switch |

> **Trap:** Migrations that run in the pre-deploy step use the OLD codebase to connect to the DB. If your migration requires the NEW class definitions (e.g., custom casts, accessors), run migrations as a standalone step, not through the app.

---

## 5. Approval Gates

### Manual approval in GitHub Actions

```yaml
jobs:
  deploy-production:
    needs: [deploy-staging, e2e-tests]
    environment:
      name: production
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

With `environment: production` and "Required reviewers" configured, the deployment waits for approval.

### Deployments with GitHub Deployment API

```yaml
- name: Create deployment
  uses: actions/github-script@v7
  with:
    script: |
      const { data: deployment } = await github.rest.repos.createDeployment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        ref: context.ref,
        environment: 'production',
        auto_merge: false,
        required_contexts: [],
      });
      // Wait for status check from external system
      // Then create deployment status
```

### GitLab CI approval

```yaml
deploy:production:
  stage: deploy
  script:
    - echo "Deploying to production"
  when: manual          # requires manual click to run
  environment:
    name: production
  only:
    - main
```

### Multi-step approval

For complex releases:

```
Developer commit → CI passes → Deploy to Staging → E2E tests → 
  PM Approval → Security Review → Deploy to Canary (5%) → 
     Monitoring Check → Deploy to 100%
```

Each gate can be implemented as a separate environment or a manual step.

---

## 6. GitHub Actions vs Other CI/CD Tools

| Feature | GitHub Actions | GitLab CI | Jenkins | CircleCI | AWS CodePipeline |
|---------|---------------|-----------|---------|----------|-----------------|
| Setup complexity | Low (SaaS) | Low (SaaS) | High (self-host) | Low (SaaS) | Medium (AWS) |
| Build minutes | 2,000/mo free | 400/mo free | Unlimited (your infra) | 6,000/mo free | 1 pipeline free |
| Plugin/marketplace | Actions marketplace | CI templates | Largest plugin ecosystem | Orbs | Limited |
| Matrix builds | Native | `parallel:` | Via plugins | Native | Manual |
| Self-hosted runners | Yes | Yes | Yes | Yes | No |
| OIDC auth | AWS, Azure, GCP | AWS, GCP | Via plugins | Via plugins | AWS native |
| Kubernetes support | Actions Runner Controller | GitLab Agent | Jenkins X | CircleCI runner | CodePipeline + EKS |
| Deployment strategies | Basic | Basic | Via plugins | Basic | Native (ECS, EKS, Lambda) |

### When to use which

- **GitHub Actions** — if you use GitHub for code, easy integration, large marketplace
- **GitLab CI** — if you use GitLab, Auto DevOps, built-in registry
- **Jenkins** — if you need maximum flexibility, complex pipelines, or are migrating legacy systems
- **CircleCI** — if you want fast builds with good caching and parallelization
- **AWS CodePipeline** — if you're all-in on AWS and want native service integration

---

## 7. Q&A

### Intermediate

**Q: What's the difference between blue-green and canary deployments?**
A: Blue-green = two full environments, switch traffic instantly. Canary = gradual traffic shift to new version (5% → 20% → 100%). Blue-green = instant rollback, double cost. Canary = gradual validation, needs routing control.

**Q: How do you handle database migrations in a blue-green deployment?**
A: Schema changes must be backward-compatible. Add columns before deploy, remove columns after all old instances are gone. Run migrations between blue being live and green being created.

**Q: How do you pass artifacts between jobs in GitHub Actions?**
A: `actions/upload-artifact` in the producer job, `actions/download-artifact` in the consumer job. Set `needs:` to ensure the producer runs first.

**Q: What's the purpose of environment protection rules?**
A: Require approvals, restrict branches, add wait timers before production deployments. Prevent accidental or unauthorized deployments.

**Q: How do you build Docker images in CI without using `docker build` directly?**
A: Use `docker/build-push-action@v5` (GitHub Actions). Supports BuildKit, caching, multiple registries, and layer caching.

**Q: What's the best Docker image tag for traceable deployments?**
A: Git SHA (e.g., `api:abc123de`). Immutable, traceable to commit, supports rollback. Optionally add semantic version (e.g., `api:1.2.3-abc123de`).

### Trap questions

**Q: A blue-green deployment fails because the new version can't connect to the database. What's the likely cause?**
A: Security group. The green instances are in a different security group than blue, and the DB SG doesn't allow traffic from the green SG. Update the DB SG.

**Q: You deploy a canary (5% traffic). Metrics look good for 10 minutes. You ramp to 100%. Immediately, error rate spikes. What happened?**
A: The canary may not have received representative traffic (e.g., no users with complex queries). Or the load test pattern didn't match real patterns. Increase canary duration and use session-affine routing.

**Q: A migration takes 30 minutes and locks the `orders` table. Production is down. How do you prevent this?**
A: (1) Use online migration tools (pt-osc, gh-ost, pgroll). (2) Use additive migrations only. (3) Run migrations during low traffic. (4) Test migration on staging with full dataset.

**Q: You need to rollback a deployment. What's the fastest strategy?**
A: Blue-green: switch traffic back to blue (instant). Canary: shift traffic back to 100% v1 (instant). Rolling: `kubectl rollout undo deployment/api` (gradual). Recreate: rebuild and deploy old version (slowest).

### Follow-up questions

**Q: How do you ensure the same artifact is deployed to staging and production?**
A: Build once in CI, upload as artifact. Both staging and production jobs download the SAME artifact. Never rebuild for production.

**Q: How do you handle secrets rotation in CI/CD?**
A: Use OIDC (no long-lived secrets). If you need static secrets, rotate them in the secrets manager, update the CI secret, and validate by running a test deployment.

**Q: What metrics do you track for deployment health?**
A: DORA metrics: deployment frequency, lead time, change failure rate, MTTR. Additionally: error rate, p50/p95/p99 latency, deployment duration, rollback time.
