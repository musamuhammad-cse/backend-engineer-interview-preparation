# CI/CD — Senior Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** CI/CD intermediate (multi-environment pipelines, deployment strategies)  
> **Estimated time:** 10–12 hours

---

## Table of Contents

1. Pipeline as Code Best Practices
2. Pipeline Security
3. Zero-Downtime Microservices Deployments
4. Canary Releases with Flagger
5. Feature Flags
6. GitOps (Argo CD, Flux)
7. DORA Metrics and Deployment Velocity
8. Terraform in CI/CD with Atlantis
9. SAST/DAST Scanning
10. Q&A

---

## 1. Pipeline as Code Best Practices

### Keep pipelines DRY

```yaml
# .github/actions/deploy/action.yml — reusable composite action
name: Deploy to ECS
description: Deploy a Docker image to ECS Fargate
inputs:
  environment:
    required: true
    type: string
  image-tag:
    required: true
    type: string
  aws-role-arn:
    required: true
    type: string

runs:
  using: composite
  steps:
    - name: Configure AWS
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ inputs.aws-role-arn }}
        aws-region: us-east-1
    - name: Render task definition
      id: render
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: .aws/task-definition.json
        container-name: api
        image: ${{ inputs.image-tag }}
    - name: Deploy to ECS
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.render.outputs.task-definition }}
        service: api-${{ inputs.environment }}
        cluster: inventory-${{ inputs.environment }}
```

Then use in workflows:

```yaml
deploy-staging:
  steps:
    - uses: ./.github/actions/deploy
      with:
        environment: staging
        image-tag: ${{ needs.build.outputs.image-tag }}
        aws-role-arn: arn:aws:iam::123456789012:role/deploy-staging
```

### Pipeline structure conventions

```
.github/workflows/
├── ci.yml                    # Build + test (runs on all branches)
├── cd-staging.yml            # Deploy to staging (on push to main)
├── cd-production.yml          # Deploy to production (manual approval)
├── security-scan.yml          # Weekly dependency scan
├── cleanup.yml                # Cleanup old artifacts
```

### Minimum required CI checks

```yaml
# Every PR must pass:
jobs:
  lint:        # Code style (Pint, ESLint, Prettier)
  type-check:  # Static analysis (PHPStan, TypeScript, mypy)
  security:    # SAST (CodeQL, SonarQube)
  test:        # Unit + integration tests
  build:       # Compiles/succeeds (Docker build)
```

Use branch protection rules to require all checks pass before merge.

### Pipeline failure design

- **Fail fast** — run fast checks first (lint, type-check before integration tests)
- **Notify on failure** — Slack, email, PagerDuty for production deployments
- **Auto-retry flaky tests** — retry up to 3 times, but flag for investigation
- **Rollback on deploy failure** — `helm rollback` or switch traffic back
- **Pipeline timeouts** — set max runtime per job (prevent runaway builds)

---

## 2. Pipeline Security

### Supply chain security

| Threat | Mitigation |
|--------|------------|
| Compromised dependency | `dependabot`, lockfiles, SBOM generation |
| Compromised CI runner | Self-hosted runners in isolated VPC |
| Secret leakage | OIDC, secret scanners, never log secrets |
| Malicious PR | `pull_request_target` restrictions, `pull_request` for untrusted forks |
| Image tampering | Image signing (cosign), image scanning (Trivy) |

### SBOM (Software Bill of Materials)

Generate SBOM for every build:

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    path: ./
    format: cyclonedx-json
    artifact-name: sbom-${{ github.sha }}
```

### Image signing with cosign

```yaml
- name: Install cosign
  uses: sigstore/cosign-installer@v3

- name: Sign image
  run: |
    cosign sign --key awskms:///alias/signing-key \
      ${{ env.REGISTRY }}/${{ env.REPOSITORY }}@${{ steps.build-image.outputs.digest }}
```

### Container vulnerability scanning

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: HIGH,CRITICAL
    exit-code: 1  # fail on HIGH/CRITICAL

- name: Upload scan results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy-results.sarif
```

### `pull_request_target` security

`pull_request_target` runs in the context of the base branch (has access to secrets). It's dangerous:

```yaml
# SAFE — use pull_request for untrusted fork PRs
on: pull_request

# DANGEROUS — only use if you know what you're doing
on: pull_request_target
```

If you need `pull_request_target` (e.g., to comment on PRs), never check out the PR's code in a way that runs it:

```yaml
# SAFER — use pull_request_target only for label/comment actions
on: pull_request_target
jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v5
```

### Least-privilege for CI credentials

```yaml
permissions:
  contents: read           # only read code
  id-token: write           # OIDC (needed for AWS auth)
  pull-requests: write      # to comment on PRs
  actions: read             # to read workflow metadata
  checks: write             # to create check runs
```

### OIDC for AWS authentication

Never store AWS credentials in secrets. Use OIDC:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: us-east-1
```

The IAM role trust policy restricts which repos/branches can assume the role:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:sub": [
      "repo:myorg/inventory:ref:refs/heads/main",
      "repo:myorg/inventory:ref:refs/heads/develop"
    ]
  }
}
```

---

## 3. Zero-Downtime Microservices Deployments

### Dependency deployment order

```
1. Backward-compatible DB migrations
2. Libraries/packages (if shared)
3. Downstream services (what your service depends on)
4. Your service
5. Upstream services (what depends on your service)
```

### Service mesh with gradual rollout

Using Istio for fine-grained traffic control:

```yaml
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
            subset: stable
          weight: 100
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: api-service
            subset: candidate
          weight: 100
```

### Health gate during deployment

CI/CD should check:

1. **Deployment started** — new pods/batch entering service
2. **Readiness** — health check passing for all new instances
3. **Error rate** — < 0.1% 5xx errors on new instances
4. **Latency** — p99 not increased by > 10%
5. **Traffic shift** — gradual with monitoring at each step
6. **Rollback** — automated if metrics degrade

```yaml
- name: Monitor rollout
  run: |
    # Check rollout status every 30s
    while true; do
      ROLLOUT=$(kubectl rollout status deployment/api --timeout=10s 2>&1)
      if echo "$ROLLOUT" | grep -q "successfully"; then
        break
      fi
      # Check error rate
      ERRORS=$(curl -s prometheus.example.com/api/v1/query?query=rate(http_requests_total{status=~"5.."}[5m]))
      if [ "$ERRORS" -gt "10" ]; then
        echo "Error rate too high — rolling back"
        kubectl rollout undo deployment/api
        exit 1
      fi
      sleep 30
    done
```

---

## 4. Canary Releases with Flagger

Flagger automates canary releases with metrics-based promotion:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: api
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  service:
    port: 8080
  ingressRef:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: api
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
    webhooks:
      - name: load-test
        url: http://load-test.flux.svc.cluster.local/
        timeout: 5s
```

Flagger workflow:
1. New image pushed → Flagger creates canary deployment (v2)
2. Shifts traffic: 10% → 20% → 30% → ... → 100%
3. At each step, checks metrics (error rate, latency)
4. If metrics degrade → rollback instantly
5. If all steps pass → promote canary to primary

---

## 5. Feature Flags

### Why feature flags in CI/CD

- Decouple deployment from release (deploy code that's off by default)
- Gradual rollout to specific users/segments
- Instant rollback (toggle off, no redeploy)
- A/B testing and experimentation

### LaunchDarkly integration

```yaml
- name: Enable feature for testing
  run: |
    curl -X PATCH https://app.launchdarkly.com/api/v2/flags/default/production/new-checkout \
      -H "Authorization: ${{ secrets.LAUNCHDARKLY_SDK_KEY }}" \
      -d '{"instructions": [{"kind": "updateFallthroughVariation", "variationId": "enabled"}]}'
```

### Feature flag in application code

```php
// Laravel with feature flags
if (FeatureFlags::isEnabled('new-checkout', $user)) {
    return $this->newCheckout($request);
}
return $this->legacyCheckout($request);
```

### Pipeline integration

```
1. Deploy code with feature flag OFF (default)
2. Enable flag for internal testing
3. Enable flag for 5% of users
4. Monitor metrics
5. Enable flag for 100%
6. Remove old code + flag in next deployment
```

---

## 6. GitOps (Argo CD, Flux)

### GitOps principles

1. **Declarative** — entire system described in Git
2. **Versioned** — Git is the single source of truth
3. **Pulled** — agent in the cluster pulls from Git (not pushed)
4. **Reconciled** — agent ensures cluster state matches Git

### Argo CD architecture

```
Git Repository              Argo CD                Kubernetes Cluster
┌──────────────┐           ┌───────────┐           ┌──────────────┐
│ manifests/   │           │           │ reconcile │              │
│  deployment  │── watch ─►│ Argo CD   │──────────►│ Deployment   │
│  service     │           │ Server    │           │ Service      │
│  configmap   │           │           │           │ ConfigMap    │
└──────────────┘           └───────────┘           └──────────────┘
                                │
                          (diff detected)
```

### Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/inventory-deploy
    targetRevision: main
    path: helm/api
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### CI + GitOps workflow

```
CI Pipeline:                  GitOps:
1. Build image                1. Argo CD watches Git
2. Push image to ECR          2. Detects manifest change
3. Update K8s manifest        3. Syncs to cluster
   (image tag in Git)          4. Reports sync status
4. Commit + push to Git
```

### Advantages of GitOps

- **Audit trail** — every change is a Git commit
- **Easy rollback** — `git revert` or `git checkout`
- **Drift detection** — Argo CD alerts if cluster state diverges
- **Multi-cluster** — one Git repo → many clusters
- **Disaster recovery** — entire cluster can be recreated from Git

### Flux

Similar to Argo CD but with key differences:

| Aspect | Argo CD | Flux |
|--------|---------|------|
| Architecture | Client-server | Controller-only |
| UI | Web UI | CLI |
| Sync | Push-based (Argo triggers sync) | Pull-based (reconciliation loop) |
| Multi-cluster | Excellent | Good |
| SSO | Built-in | Via OIDC proxy |

---

## 7. DORA Metrics and Deployment Velocity

### The four key metrics

| Metric | Definition | Elite performer | Target |
|--------|------------|----------------|---------|
| **Deployment frequency** | How often you deploy to production | On-demand (multiple/day) | At least weekly |
| **Lead time for changes** | Time from commit to production | < 1 hour | < 1 day |
| **Change failure rate** | % of deployments causing incidents | 0–5% | < 15% |
| **MTTR** | Time to restore service | < 1 hour | < 24 hours |

### Measuring with GitHub Actions

```yaml
# Track lead time
- name: Record deployment timestamp
  run: |
    echo "Deploying commit ${{ github.sha }} at $(date -u +%Y-%m-%dT%H:%M:%SZ)"
    # Store in data store (DynamoDB, custom DB)
    aws dynamodb put-item \
      --table-name deployment-metrics \
      --item '{
        "deployId": {"S": "'${{ github.run_id }}'"},
        "commitSha": {"S": "'${{ github.sha }}'"},
        "timestamp": {"S": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"},
        "environment": {"S": "production"},
        "status": {"S": "success"}
      }'
```

### Improving deployment velocity

1. **Reduce batch size** — smaller PRs = faster review + safer deployment
2. **Automate everything** — no manual steps (except production approval)
3. **Feature flags** — decouple deployment from release
4. **Parallel testing** — run tests in parallel (matrix, sharding)
5. **Fast feedback** — CI finishes in < 10 minutes
6. **Deploy on merge** — main branch is always deployable
7. **Self-service** — teams deploy their own services without waiting

---

## 8. Terraform in CI/CD with Atlantis

### Atlantis workflow

```
1. Open PR with Terraform changes
2. Atlantis comments: `atlantis plan` output
3. Review the plan
4. Comment: `atlantis apply`
5. Atlantis applies the changes
```

### Atlantis server config

```yaml
# atlantis.yaml (in repo)
version: 3
projects:
  - name: production
    dir: terraform/environments/production
    workspace: production
    terraform_version: 1.8.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars"]
      enabled: true
  - name: staging
    dir: terraform/environments/staging
    workspace: staging
    autoplan:
      enabled: true
```

### GitHub Actions for Terraform (without Atlantis)

```yaml
name: Terraform

on:
  pull_request:
    paths: [terraform/**]
  push:
    branches: [main]
    paths: [terraform/**]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.8.0

      - name: Terraform Init
        run: terraform init
        working-directory: terraform/environments/production

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate
        working-directory: terraform/environments/production

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color
        working-directory: terraform/environments/production

      - name: Comment Plan on PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const output = `#### Terraform Plan \`production\`
            <details><summary>Show Plan</summary>
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            </details>`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

---

## 9. SAST/DAST Scanning

### SAST (Static Application Security Testing)

Scan source code for vulnerabilities:

```yaml
- name: CodeQL Initialization
  uses: github/codeql-action/init@v3
  with:
    languages: javascript, typescript

- name: CodeQL Analysis
  uses: github/codeql-action/analyze@v3
```

Other SAST tools:
- **SonarQube** — code quality + security hotspots
- **Semgrep** — custom rule-based scanning
- **PHPStan/Pint** — PHP-specific static analysis
- **ESLint** with security plugins — JS/TS

### DAST (Dynamic Application Security Testing)

Test running application for vulnerabilities:

```yaml
- name: Deploy to staging
  run: echo "Deploying..."

- name: Run OWASP ZAP DAST
  uses: zaproxy/action-full-scan@v0
  with:
    target: https://staging.example.com
    rules_file_name: .zap/rules.tsv
    cmd_options: '-a'
```

### Dependency scanning

```yaml
- name: Check for vulnerabilities
  uses: dependabot/dependabot-core@v2
  # Or use GitHub's built-in Dependabot alerts

- name: Snyk scan
  uses: snyk/actions/php@master
  with:
    command: monitor
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

### Secret scanning

GitHub's built-in secret scanner detects:
- AWS access keys
- GitHub tokens
- npm/GitHub PAT
- Slack tokens
- Private keys

> **Trap:** If a secret is pushed to Git (even if you delete it in a later commit), assume it's compromised. Rotate the secret immediately. Git history retains the secret.

---

## 10. Q&A

### Senior

**Q: How would you design a CI/CD pipeline for a microservices architecture with 20+ services?**
A: (1) Monorepo or polyrepo based on team structure and coupling. (2) CI per service (build + test only changed services). (3) Shared CI templates (reusable composite actions). (4) Artifact per service (Docker image). (5) GitOps with Argo CD (Git manifest = desired state). (6) Canary releases with Flagger for critical services. (7) DORA metrics dashboard to track velocity.

**Q: What's the difference between GitOps and traditional CI/CD push-based deployments?**
A: Traditional: CI/CD pushes to cluster (requires cluster credentials in CI). GitOps: CI pushes to Git, agent in cluster pulls changes. GitOps is more secure (no cluster creds in CI), provides drift detection, and simpler rollback (Git revert).

**Q: How do you handle secrets rotation across CI/CD?**
A: (1) Use OIDC (no static secrets). (2) For API keys: store in Secrets Manager, rotate via Lambda, CI retrieves at deploy time. (3) For K8s: use External Secrets Operator to sync from vault. (4) Automate rotation: schedule + Slack notification.

**Q: How do you enforce pipeline security across all teams?**
A: (1) Organization-wide required checks (CodeQL, lint, tests). (2) Branch protection rules (no direct push to main, required reviews). (3) OIDC with repo/branch-restricted IAM roles. (4) SBOM generation on every build. (5) Container scanning with fail-on-critical. (6) Secret scanning pre-commit hooks. (7) Regular audit of CI permissions.

**Q: You have a monolithic Laravel app that takes 45 minutes to test. How do you improve?**
A: (1) Parallelize tests (Pest parallel, sharding). (2) Run only affected tests (test impact analysis). (3) Split into smaller modules (eventually microservices). (4) Use caching (Composer, Docker layers). (5) Optimize test suite (slow tests → integration vs unit). (6) Run CI on self-hosted runners (faster).

### Trap questions

**Q: A GitHub Actions workflow fails intermittently with "Process exited with code 137". What is it?**
A: Code 137 = SIGKILL (OOM). The runner ran out of memory. Use a larger runner (`runs-on: ubuntu-latest-8-cores`) or optimize memory usage.

**Q: You push to main, CI passes, CD deploys, and the app crashes. Tests passed in CI. What went wrong?**
A: (1) Different environment (CI DB config vs production config). (2) Missing migration. (3) Feature flag default wrong. (4) Dependency version drift (CI doesn't use lockfile). (5) Hardcoded config that differs between environments.

**Q: An engineer pushes a secret to a public repo. What should happen?**
A: (1) GitHub detects and alerts. (2) Rotate the secret immediately (in all places it was used). (3) Notify security team. (4) Scan for usage in logs, commits, and artifacts. (5) Add pre-commit secret scanning. (6) Post-mortem.

**Q: Canary deployment looks fine at 10% but fails at 50%. What might explain this?**
A: (1) Load balancer routing distribution isn't uniform. (2) Certain user segments hit the canary at different rates. (3) The canary doesn't handle peak traffic well. (4) A downstream service rate-limits at higher volume. Increase canary time at each step and use consistent routing.

### Follow-up questions

**Q: You mentioned DORA metrics. How would you improve lead time for changes?**
A: (1) Smaller PRs (review faster, deploy faster). (2) Automate tests (no waiting for manual QA). (3) Trunk-based development (no long-lived branches). (4) Feature flags (deploy incomplete features, release later). (5) Optimize CI (parallel, caching). (6) Empower teams to deploy independently.

**Q: How would you set up a CI/CD pipeline for Terraform?**
A: (1) PR → `terraform plan` commented on PR. (2) Merge to main → `terraform apply`. (3) State in S3 + DynamoDB. (4) Multiple workspaces (dev/staging/prod). (5) `terraform fmt -check` + `terraform validate`. (6) Atlantis for larger teams.

**Q: What's the difference between `pull_request` and `pull_request_target`?**
A: `pull_request` runs in the context of the merge commit (no access to secrets, safe for forks). `pull_request_target` runs in the context of the base branch (has access to secrets, but runs code from the merge commit — dangerous if you check out the PR code).
