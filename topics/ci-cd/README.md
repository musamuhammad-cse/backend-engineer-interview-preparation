# CI/CD — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** GitHub Actions for Laravel/ECS deployment, multi-environment pipelines (dev/staging/prod), Docker image builds + ECR push, automated testing on PRs  
> **Note:** CI/CD is expected knowledge for senior roles. The senior signal is **designing deployment pipelines for zero-downtime**, **handling rollbacks**, **canary deployments**, **pipeline security** (secrets management, supply chain security), and **measuring deployment velocity** (lead time, deployment frequency, MTTR).

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud | 20 min/section |
| 2 | Write a GitHub Actions pipeline from scratch for your Laravel app | 30 min |
| 3 | Answer the section's Q&A without looking, then diff | 20 min/section |
| 4 | Design a deployment pipeline for a microservices architecture | 15 min |

**The senior signal is deployment strategy design, pipeline reliability, and security.** Knowing YAML syntax is table stakes; designing for zero-downtime, rollback, and compliance is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | CI vs CD concepts, GitHub Actions (workflow, jobs, steps, triggers, matrices), GitLab CI (stages, jobs, `.gitlab-ci.yml`), build artifacts, basic test pipelines, environment variables | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Deployment strategies (rolling, blue-green, canary, recreate), multi-environment pipelines, Docker image build + ECR push, database migrations in pipelines, approval gates, environments with protection rules, GitHub Actions vs GitLab CI vs Jenkins vs CircleCI | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Pipeline as code best practices, security (supply chain, SBOM, signed commits, secrets rotation), zero-downtime deployment for microservices, canary releases with Flagger/Argo Rollouts, feature flags, GitOps (Argo CD, Flux), deployment metrics (DORA), Terraform in CI/CD with Atlantis, SAST/DAST scanning, artifact repository strategies | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 120+ interview questions, pipeline design scenarios, debugging, deployment failure response | Ongoing drill |

---

## Coverage map

### CI/CD fundamentals
- CI: Build, test, lint on every push
- CD: Deploy automatically after CI passes
- Pipeline stages: checkout → build → test → scan → package → deploy
- Triggers: push, PR, schedule, manual, webhook
- Artifacts: build output passed between jobs (JARs, Docker images, ZIPs)

### GitHub Actions
- Workflows: YAML files in `.github/workflows/`
- Events: `push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`
- Runners: GitHub-hosted (Ubuntu, Windows, macOS), self-hosted
- Jobs: `runs-on`, `steps`, `needs`, `if`, `strategy.matrix`
- Actions: marketplace actions, Docker actions, JavaScript actions, composite actions
- Environments: `environment:` with protection rules and secrets
- Contexts: `github`, `env`, `secrets`, `vars`, `matrix`, `needs`, `steps`
- OIDC: authenticate to AWS without long-lived secrets
- Caching: `actions/cache` for dependencies, `docker/build-push-action` caching

### GitLab CI
- `.gitlab-ci.yml` at repo root
- Stages: sequential groups of jobs
- Jobs: `script`, `image`, `services`, `artifacts`, `only/except`, `rules`
- Runners: shared, group, specific
- DAG: `needs` for parallel job execution
- Environments: `environment: name` with manual approvals
- Auto DevOps: built-in CI/CD templates

### Deployment strategies
- Rolling: gradual replacement (K8s default)
- Blue-green: two environments, switch traffic
- Canary: small percentage of traffic to new version
- Recreate: stop all, start all (downtime)
- A/B testing: route based on user attributes

### Pipeline security
- Secrets management: GitHub secrets, OIDC, HashiCorp Vault
- Supply chain security: SBOM, software signing (cosign), SLSA levels
- SAST: static analysis (CodeQL, SonarQube)
- DAST: dynamic analysis (OWASP ZAP)
- Dependency scanning: Dependabot, Snyk, Trivy
- Container scanning: Trivy, Clair, Docker Scout
- Signed commits: GPG commit signing

### GitOps
- Argo CD: declarative GitOps for K8s
- Flux: GitOps with reconciliation
- Principle: Git is the single source of truth
- Drift detection: auto-reconcile or alert

### Deployment metrics (DORA)
- Deployment frequency: How often you deploy
- Lead time for changes: Time from commit to production
- Change failure rate: Percentage of deployments causing incidents
- Time to restore service (MTTR): How long to recover from failure

### Tools comparison
- GitHub Actions: native to GitHub, large ecosystem, good for GitHub-centric teams
- GitLab CI: integrated with GitLab, Auto DevOps, built-in registry
- Jenkins: most flexible, plugin ecosystem, self-hosted complexity
- CircleCI: fast, config caching, good DX
- AWS CodePipeline: native AWS integration, deployment to ECS/EKS
- ArgoCD: K8s-native GitOps

---

## Study order recommendation

Focus on GitHub Actions (your daily tool), then deployment strategies (senior signal), then GitOps and pipeline security.

```
Week 1:  01-basic.md          + Write a GitHub Actions pipeline for Laravel
Week 2:  02-intermediate.md   + Multi-environment pipeline with approvals
Week 3:  03-senior.md         + GitOps/Argo CD, pipeline security
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** Architecture patterns.
