# Docker, Kubernetes & Terraform — Question Bank

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Type:** 150+ rapid-fire Q&A, debugging scenarios, architecture prompts, code puzzles

---

## Table of Contents

1. Rapid-Fire Q&A (150+ questions)
2. Debugging Scenarios
3. Architecture Design Prompts
4. Terraform Code Puzzles
5. Dockerfile Challenges

---

## 1. Rapid-Fire Q&A

### Docker

**Q: What's the difference between an image and a container?**
A: Image = read-only template (layers). Container = running instance (writable layer on top).

**Q: What Dockerfile instruction would you use to set the user for the container?**
A: `USER www-data`

**Q: What's the difference between CMD and ENTRYPOINT?**
A: CMD = default command (overridable). ENTRYPOINT = main command (not easily overridden). Combined: `ENTRYPOINT ["/app"]` + `CMD ["serve"]` → `/app serve`.

**Q: What's a multi-stage build?**
A: Multiple FROM statements in one Dockerfile. First stage builds the app, final stage only copies the binary. Reduces final image size.

**Q: How do you reduce Docker image size?**
A: Multi-stage builds, Alpine base, chain RUN commands, `.dockerignore`, remove build dependencies, use distroless/scratch.

**Q: What's the difference between `docker run` and `docker compose up`?**
A: `docker run` = single container. `docker compose up` = multi-service stack defined in YAML.

**Q: What's a bind mount vs a named volume?**
A: Bind mount = host directory. Named volume = Docker-managed (`/var/lib/docker/volumes/`). Prefer named volumes for production.

**Q: What's the purpose of `.dockerignore`?**
A: Exclude files from build context (smaller, faster builds; no secrets leaked).

**Q: What's the default Docker network driver?**
A: `bridge` (single host, internal network). Containers can communicate via IP (not DNS unless custom bridge).

**Q: What's the difference between bridge and host network modes?**
A: Bridge = isolated network stack (port mapping required). Host = shares host network (no isolation, no port mapping needed).

**Q: What happens when you run `docker run --rm`?**
A: Container is automatically removed when it stops. Useful for one-off jobs.

**Q: What's the max size of a Docker image layer?**
A: No hard limit, but each layer is the diff between filesystem states. Keep layers small for caching efficiency.

**Q: How does Docker layer caching work?**
A: Each instruction creates a layer. If the instruction text and context haven't changed, the cached layer is reused. Place frequently changed instructions last.

**Q: What's the difference between `COPY` and `ADD`?**
A: `COPY` = copy files. `ADD` = copy + auto-extract tar/zip + remote URL (prefer COPY except when extraction needed).

**Q: What's Docker BuildKit?**
A: Enhanced build engine (performance, secrets, SSH mounts). Enable with `DOCKER_BUILDKIT=1`.

**Q: How do you pass secrets to a Docker build without leaving them in the image?**
A: BuildKit `--secret` flag + `RUN --mount=type=secret` mount.

**Q: What's the difference between `docker exec` and `docker attach`?**
A: `exec` = run new process. `attach` = connect to container's main process (stdin/stdout/stderr).

### Kubernetes

**Q: What's the smallest deployable unit in Kubernetes?**
A: Pod (one or more containers sharing network/storage/lifecycle).

**Q: What's the difference between a Pod and a Deployment?**
A: Pod = single instance. Deployment = manages Pods (scaling, rolling updates, self-healing). Always use Deployments in production.

**Q: How does a Service route traffic to Pods?**
A: Label selector matches pod labels. kube-proxy creates iptables/IPVS rules for load balancing.

**Q: What Service types are available?**
A: ClusterIP (internal), NodePort (static port on nodes), LoadBalancer (cloud LB), ExternalName (DNS alias).

**Q: What's the difference between readiness and liveness probes?**
A: Readiness = ready for traffic (remove from Service). Liveness = alive (restart container). Liveness failures restart; readiness doesn't.

**Q: What's a ConfigMap?**
A: Non-confidential configuration data (env vars, config files). Stored in etcd.

**Q: How are Secrets different from ConfigMaps?**
A: Secrets are base64-encoded (not encrypted by default). Use for sensitive data. Enable KMS encryption for etcd.

**Q: How do you store secrets outside K8s?**
A: External Secrets Operator (sync from AWS Secrets Manager), Secrets Store CSI Driver, sealed-secrets (encrypted in Git).

**Q: What's a PersistentVolumeClaim?**
A: Request for storage by a Pod. Binds to a PersistentVolume.

**Q: What access modes does a PV support?**
A: ReadWriteOnce (single node), ReadOnlyMany (multiple nodes read), ReadWriteMany (multiple nodes R/W).

**Q: What's a Namespace?**
A: Virtual cluster within a K8s cluster. Used for isolation (environments, teams).

**Q: Do Namespaces provide network isolation?**
A: No. Use NetworkPolicies for network isolation.

**Q: What's a DaemonSet?**
A: Runs one Pod per node. Use for logging, monitoring, networking.

**Q: What's a StatefulSet?**
A: Stateful workload (stable identity, ordered deployment, per-pod PVC). For databases, queues.

**Q: What's a Job vs a CronJob?**
A: Job = run once. CronJob = schedule (cron format).

**Q: What's the default deployment strategy?**
A: RollingUpdate. Gradual replacement with `maxUnavailable` and `maxSurge` controls.

**Q: What's a Recreate strategy?**
A: Delete all pods, then create. Causes downtime.

**Q: How do you rollback a Deployment?**
A: `kubectl rollout undo deployment/<name>` or `kubectl rollout undo deployment/<name> --to-revision=2`.

**Q: What's the HPA?**
A: HorizontalPodAutoscaler — scales pod count based on CPU, memory, or custom metrics.

**Q: What's a PodDisruptionBudget?**
A: Ensures minimum available pods during voluntary disruptions (node drain, upgrades).

**Q: What's the difference between requests and limits?**
A: Requests = scheduling guarantee. Limits = throttling/OOM protection.

**Q: What happens when a pod exceeds its memory limit?**
A: OOM-killed by kernel. K8s restarts per restartPolicy.

**Q: What's a QoS class?**
A: Guaranteed (limits == requests), Burstable (requests < limits), BestEffort (no limits/requests).

**Q: What's a ServiceAccount?**
A: Identity for Pods to authenticate with K8s API. Used with RBAC.

**Q: What's RBAC?**
A: Role-Based Access Control. Role (namespace) or ClusterRole (cluster-wide) + RoleBinding/ClusterRoleBinding.

**Q: What's an Ingress?**
A: HTTP/HTTPS routing to Services. Path-based, host-based routing, TLS termination.

**Q: What's needed for Ingress to work on EKS?**
A: AWS Load Balancer Controller (watches Ingress resources, provisions ALBs).

**Q: What's a NetworkPolicy?**
A: Pod-level firewall. Allow-list — once applied, all traffic denied unless explicitly allowed.

**Q: What's an Init Container?**
A: Runs before main containers. Must complete successfully. Used for migrations, setup.

**Q: What's a Sidecar?**
A: Helper container in the same Pod. Logging, proxy, monitoring.

**Q: What's Helm?**
A: K8s package manager. Charts = pre-configured K8s resources. Templating + release management.

**Q: What's a Helm hook?**
A: Runs Jobs at specific points (pre-upgrade, post-install, etc.). DB migrations, validation.

**Q: What's the difference between Helm and Kustomize?**
A: Helm = templating + package management. Kustomize = native YAML patching (overlays).

**Q: What's a Service Mesh?**
A: Infrastructure layer for service-to-service communication (Istio, Linkerd). Traffic management, mTLS, observability.

**Q: What does Istio's sidecar (Envoy) provide?**
A: mTLS, circuit breaking, retries, traffic routing, metrics, tracing.

**Q: What's kubelet?**
A: Node agent. Registers node, manages Pod lifecycle, reports status to API server.

**Q: What's etcd?**
A: Distributed key-value store. Single source of truth for cluster state.

**Q: What's kube-proxy?**
A: Network proxy. Maintains iptables/IPVS rules for Service traffic forwarding.

**Q: What's containerd?**
A: Container runtime (OCI-compatible). Used by Docker and K8s directly.

**Q: How does the K8s scheduler work?**
A: Filters nodes (resource fit, port conflicts, taints) → scores nodes (resource utilization, affinity) → selects best node.

**Q: What's a taint and toleration?**
A: Taint = node attribute that repels pods. Toleration = pod attribute that allows scheduling on tainted nodes.

**Q: What's node affinity?**
A: Pod constraint to prefer/require certain nodes (node labels). `requiredDuringScheduling` vs `preferredDuringScheduling`.

**Q: What's pod anti-affinity?**
A: Avoid scheduling pods on the same node (spread for HA).

**Q: What's a Headless Service?**
A: `clusterIP: None`. Returns pod IPs directly (not load-balanced). Used for StatefulSet DNS.

**Q: How does DNS work in K8s?**
A: CoreDNS (cluster addon). Pod DNS: `pod-ip.namespace.pod.cluster.local`. Service DNS: `service.namespace.svc.cluster.local`.

**Q: What's a Kubernetes Operator?**
A: Custom controller extending K8s API. Manages complex stateful apps (PostgreSQL Operator, Prometheus Operator).

### Terraform

**Q: What's Terraform?**
A: Infrastructure as Code tool. Declarative configuration → desired state. Plans and applies changes.

**Q: What's the Terraform workflow?**
A: `init` → `plan` → `apply`. `destroy` tears down.

**Q: What's the difference between `terraform plan` and `terraform apply`?**
A: Plan = preview changes. Apply = execute changes (requires approval).

**Q: What's Terraform state?**
A: JSON file mapping resources to infrastructure. Source of truth for what Terraform manages.

**Q: Where should you store state?**
A: Remote backend (S3 + DynamoDB locking). Never in Git.

**Q: What's state locking?**
A: Prevents concurrent modifications (DynamoDB table stores lock). Fail on conflict.

**Q: What's a Terraform module?**
A: Reusable, composable unit of infrastructure. Input variables + output values.

**Q: How do you use modules?**
A: `module "vpc" { source = "./modules/vpc" ... }`. Source can be local, Git, registry.

**Q: What's `count` vs `for_each`?**
A: `count` = list (indexed). `for_each` = map/set (keyed). Prefer `for_each` for stability.

**Q: What's `depends_on`?**
A: Explicit dependency (Terraform auto-detects most dependencies via references, but some need explicit declaration).

**Q: What's `prevent_destroy`?**
A: Lifecycle meta-argument. Prevents accidental deletion of critical resources (DB, S3).

**Q: What's `create_before_destroy`?**
A: Creates replacement before destroying old. Zero-downtime replacement.

**Q: What's `ignore_changes`?**
A: Ignoces changes to specified attributes made outside Terraform.

**Q: What's a provider?**
A: Plugin for managing a specific platform (AWS, GCP, Azure, Kubernetes, Helm).

**Q: What's a data source?**
A: Reads data from provider without creating resources. `data.aws_vpc.main`.

**Q: What's a local value?**
A: `locals { name = "${var.env}-vpc" }`. Derived expressions used within the module.

**Q: What's a variable?**
A: Input parameter. `variable "instance_type" { type = string }`.

**Q: What's an output?**
A: `output "vpc_id" { value = aws_vpc.main.id }`. Exposes resource attributes.

**Q: What's a Terraform workspace?**
A: Separate state instances for same configuration (dev, staging, prod).

**Q: What's Terraform Cloud?**
A: Managed Terraform service. Remote execution, state management, VCS integration, policy as code.

**Q: What's Sentinel?**
A: Policy as code for Terraform Cloud. Enforce rules (e.g., "all S3 buckets must be encrypted").

**Q: What's Terragrunt?**
A: Terraform wrapper for DRY configuration (remote state, provider generation, dependency management).

**Q: How do you import existing infrastructure into Terraform?**
A: `terraform import <resource>.<name> <id>`. Adds to state. Must write matching config manually.

**Q: What's `terraform taint`?**
A: Marks resource for recreation on next `apply`. Deprecated in favor of `terraform apply -replace`.

**Q: What's the difference between Terraform and CloudFormation?**
A: Terraform = multi-cloud, HCL, state management, modular. CloudFormation = AWS-only, YAML/JSON, drift detection.

**Q: What's the difference between Terraform and Pulumi?**
A: Terraform = HCL (DSL). Pulumi = general-purpose languages (TypeScript, Python, Go, C#).

**Q: How do you handle environment-specific configs?**
A: Workspaces, directory layout (`environments/dev`, `environments/prod`), Terragrunt, Terraform Cloud workspaces.

**Q: How do you test Terraform code?**
A: `terraform validate` (syntax), `terraform plan` (logic), Terratest (Go integration tests), `conftest` (policy), `tfsec` (security).

**Q: What's `terraform fmt`?**
A: Format HCL files to canonical style. Run in CI to enforce consistency.

**Q: What's a `terraform refresh`?**
A: Updates state file to match real-world infrastructure. Does NOT change resources.

---

## 2. Debugging Scenarios

### Scenario 1: ImagePullBackOff

**Symptom:** Pod stuck in `ImagePullBackOff`.

**Debug steps:**
1. `kubectl describe pod <name>` → event shows specific error
2. Check image name and tag (typo? wrong registry?)
3. Check image pull policy (`imagePullPolicy` — `IfNotPresent` vs `Always`)
4. Check authentication (does node have pull access to ECR/Docker Hub?)
5. Check network (can node reach the registry?)
6. For ECR: check IAM role (node/IRSA) has `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`
7. Check if image exists in registry (manually pull on a node)

**Root cause:** ECR IAM role missing, or image tag doesn't exist.

### Scenario 2: CrashLoopBackOff

**Symptom:** Pod starts, crashes, restarts, crashes again.

**Debug steps:**
1. `kubectl logs <pod> --previous` → logs from the crashed instance
2. `kubectl describe pod` → exit code (code 137 = OOMKilled, code 139 = segfault)
3. Check resource limits (OOMKilled? Increase memory limit)
4. Check app configuration (env vars, ConfigMap, Secrets — missing keys?)
5. Check init containers (did migration fail?)
6. Check probe config (liveness probe failing on startup?)

**Root cause:** Missing environment variable causing panic on boot.

### Scenario 3: Pod stuck in Pending

**Symptom:** Pod never transitions to Running.

**Debug steps:**
1. `kubectl describe pod` → events show "0/1 nodes available"
2. Check resource requests — no node has enough CPU/memory
3. Check taints — nodes have taints pod doesn't tolerate
4. Check node selector / affinity — no matching nodes
5. Check PVC pending — PVC not bound to PV
6. Check if nodes have enough IP addresses (EKS VPC CNI)

**Root cause:** Node pool exhausted — CPU requests exceed available capacity. Add nodes or reduce requests.

### Scenario 4: Service not reachable

**Symptom:** Can't reach the service from another Pod.

**Debug steps:**
1. Check Service exists: `kubectl get svc`
2. Check Endpoints: `kubectl get endpoints <svc>` — should show pod IPs
3. If endpoints empty: selector doesn't match pod labels
4. Check pod readiness: `kubectl get pods` — are pods Running + Ready?
5. Check NetworkPolicy blocking traffic
6. Check DNS resolution: `kubectl exec -it <pod> -- nslookup <svc>`

**Root cause:** Selector mismatch — Service has `app: api` label, Pod has `app: api-v2`. Labels must match exactly.

### Scenario 5: Terraform plan shows unexpected changes

**Symptom:** `terraform plan` shows changes to resources that weren't modified.

**Debug steps:**
1. Check for drift — did someone modify resource in AWS console?
2. Check provider version — new version may have different defaults
3. Check for interpolated values that changed (module input variable changed)
4. Run `terraform refresh` to update state to real-world
5. Check `ignore_changes` lifecycle — was it removed?
6. Check if resource was replaced (new AMI ID for launch template)

**Root cause:** Someone manually added a tag to an S3 bucket in AWS console. Terraform wants to remove it to match config.

### Scenario 6: Helm upgrade fails on pre-upgrade hook

**Symptom:** Helm upgrade fails. Error: "pre-upgrade hook failed".

**Debug steps:**
1. `helm list --failed` → check revision
2. `kubectl get jobs` → find the hook job
3. `kubectl logs job/<name>` → see migration/script error
4. Fix the issue (SQL error, network issue)
5. `helm rollback <release> <previous-revision>` → restore working state
6. Re-run upgrade after fix

**Root cause:** DB migration script in pre-upgrade hook hit a duplicate column error.

### Scenario 7: Container keeps being OOMKilled

**Symptom:** Pod restarts, exit code 137.

**Debug steps:**
1. Check memory limits vs actual usage (`kubectl top pod`)
2. Check for memory leaks in application (heap profiles)
3. Increase memory limits (but also increase requests proportionally)
4. Add memory monitoring (Prometheus + Grafana)
5. Consider VPA (Vertical Pod Autoscaler) for automatic memory adjustment
6. Check if limits are too tight (e.g., PHP process needs more memory for peak)

**Root cause:** Application memory leak in a background worker. Fix the leak, not just increase limits.

### Scenario 8: ECS task vs EKS pod decision

**Symptom:** Team debating ECS vs EKS for a new microservice.

**Decision framework:**

| Factor | ECS | EKS |
|--------|-----|-----|
| Team K8s expertise | Low | High |
| Existing investment | Non-K8s | Already have K8s |
| Service mesh needed | No (Service Connect) | Yes (Istio, Linkerd) |
| Portability | AWS-only | Multi-cloud |
| Operational complexity | Low | High |

**Recommendation:** Use ECS Fargate if the team has no K8s expertise and the app is a simple stateless service. Use EKS if you need portability, service mesh, or already have K8s investment.

---

## 3. Architecture Design Prompts

### Prompt 1: Laravel inventory SaaS on EKS

**Requirements:**
- Multi-tenant Laravel app (5,000+ tenants)
- PostgreSQL (Aurora)
- Redis for cache/sessions/queues
- S3 for file uploads
- Zero-downtime deployments

**Design:**
- EKS cluster (3 m6i.large nodes, auto scaling 3–10)
- Deployment: Laravel FPM + Nginx sidecar
  - ConfigMap for app config (env vars)
  - Secrets from External Secrets Operator (RDS credentials)
  - HPA based on CPU + request count
- Service: ClusterIP (internal) + Ingress (ALB) for external
- MySQL: RDS Aurora (Multi-AZ)
- Redis: ElastiCache (internal, no sidecar needed)
- Storage: S3 via IRSA (IAM roles for service accounts)
- CronJobs: scheduler, cleanup tasks
- Helm chart: reusable chart with env-specific values
- Deployment: blue-green with Istio VirtualService + Flagger

### Prompt 2: Chronos distributed scheduler on EKS

**Requirements:**
- Go-based Raft scheduler (3 nodes for quorum)
- StatefulSet for stable identity
- Job execution via CronJob

**Design:**
- StatefulSet: 3 replicas, `podManagementPolicy: Parallel`
- Headless Service: `chronos-0.chronos.default.svc.cluster.local`
- Data: PVC per pod (Raft logs + snapshots)
- Anti-affinity: spread across nodes and AZs
- HPA: not needed (fixed quorum size)
- Job execution: Controller pod creates CronJob resources dynamically
- Monitoring: Prometheus metrics + Grafana
- Config: ConfigMap for Raft config, Secrets for TLS certs

### Prompt 3: CI/CD pipeline with Terraform + Helm

**Requirements:**
- GitHub repository
- Terraform for infrastructure
- Helm for app deployments
- Dev/staging/prod environments

**Design:**
- Directory structure:
  ```
  terraform/
    environments/
      dev/     (main.tf, terraform.tfvars)
      staging/ (main.tf, terraform.tfvars)
      prod/    (main.tf, terraform.tfvars)
    modules/
      vpc/
      rds/
      eks/
      ecr/
  helm/
    api-chart/
    worker-chart/
  ```
- GitHub Actions:
  - `on: push to main, paths: terraform/**` → Terraform plan/apply
  - `on: push to main, paths: helm/**` → Helm upgrade with image tag
- ECR: push image with Git SHA tag
- Helm: `helm upgrade --install --values environments/prod/values.yaml`

### Prompt 4: Multi-region EKS with Terraform

**Requirements:**
- Active-passive DR
- Terraform manages both regions
- State stored centrally (S3 in us-east-1)

**Design:**
- Terraform backend: `key = "eks-${var.region}/terraform.tfstate"`
- Modules per region:
  ```
  modules/
    eks-cluster/     (called for us-east-1 and us-west-2)
    networking/      (VPC, subnets)
    monitoring/      (Prometheus, Grafana per region)
  ```
- Active region: Route 53 failover routing → primary ALB
- DR region: scaled to minimum, Route 53 health check fails → failover

### Prompt 5: Terraform state migration

**Requirements:**
- Currently local state (git repo)
- Migrate to S3 + DynamoDB

**Steps:**
1. Create S3 bucket + DynamoDB table (via separate Terraform or manually)
2. Add backend config to Terraform:
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "new-state-bucket"
       key            = "terraform.tfstate"
       region         = "us-east-1"
       dynamodb_table = "terraform-locks"
       encrypt        = true
     }
   }
   ```
3. `terraform init -migrate-state` — copies local state to S3
4. Remove local state file (`terraform.tfstate`) from Git
5. Add `.gitignore` entry for state files
6. Verify: `terraform state list` returns same resources

---

## 4. Terraform Code Puzzles

### Puzzle 1: Fix state locking issue

**Problem:** `terraform apply` fails with: `Error: Error acquiring the state lock`.

**Solution:**
```bash
# Force unlock (only if no other process is running)
terraform force-unlock <lock-id>

# Or manually delete lock from DynamoDB
aws dynamodb delete-item \
  --table-name terraform-locks \
  --key '{"LockID": {"S": "bucket-name/env-name/terraform.tfstate-md5"}}'
```

### Puzzle 2: Resource naming inconsistency

**Problem:** Resources are named inconsistently. Fix using locals.

```hcl
# BAD: repeating name pattern
resource "aws_s3_bucket" "logs" {
  bucket = "myapp-prod-logs"
  tags = { Environment = "prod" }
}
resource "aws_s3_bucket" "backups" {
  bucket = "myapp-prod-backups"
  tags = { Environment = "prod" }
}
resource "aws_s3_bucket" "artifacts" {
  bucket = "myapp-prod-artifacts"
  tags = { Environment = "prod" }
}

# GOOD: use locals
locals {
  name_prefix = "myapp-${var.environment}"
}
resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs"
  tags   = local.tags
}
resource "aws_s3_bucket" "backups" {
  bucket = "${local.name_prefix}-backups"
  tags   = local.tags
}
```

### Puzzle 3: Dynamic security group rules

**Problem:** Need to create multiple ingress rules from a list of ports.

```hcl
variable "ingress_ports" {
  type = list(object({
    port     = number
    protocol = string
    cidr     = string
  }))
  default = [
    { port = 443, protocol = "tcp", cidr = "0.0.0.0/0" },
    { port = 80,  protocol = "tcp", cidr = "0.0.0.0/0" },
    { port = 22,  protocol = "tcp", cidr = "10.0.0.0/8" },
  ]
}

resource "aws_security_group_rule" "ingress" {
  for_each = { for idx, rule in var.ingress_ports : idx => rule }

  type        = "ingress"
  from_port   = each.value.port
  to_port     = each.value.port
  protocol    = each.value.protocol
  cidr_blocks = [each.value.cidr]
  security_group_id = aws_security_group.main.id
}
```

### Puzzle 4: Conditional resource creation

**Problem:** Only create certain resources in production.

```hcl
variable "environment" {
  type = string
}

resource "aws_waf_web_acl" "production" {
  count = var.environment == "production" ? 1 : 0
  # WAF config...
}

# Reference conditionally
# If count = 0, this would fail... use try():
output "waf_arn" {
  value = try(aws_waf_web_acl.production[0].arn, null)
}
```

### Puzzle 5: Terraform workspace references

**Problem:** Use workspace name in resource naming and conditional logic.

```hcl
locals {
  env = terraform.workspace
}

resource "aws_db_instance" "main" {
  instance_class = local.env == "production" ? "db.r6g.xlarge" : "db.t4g.small"
  allocated_storage = local.env == "production" ? 500 : 20
  tags = {
    Environment = local.env
    Name        = "app-db-${local.env}"
  }
}
```

### Puzzle 6: Remote backend per workspace

**Problem:** Each workspace should have its own state file.

```hcl
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "env:/${terraform.workspace}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

## 5. Dockerfile Challenges

### Challenge 1: Optimize this Dockerfile

```dockerfile
FROM php:8.2-fpm
RUN apt-get update
RUN apt-get install -y libpq-dev
RUN docker-php-ext-install pdo_pgsql
COPY . /app
WORKDIR /app
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
RUN composer install --no-dev
RUN php artisan optimize
EXPOSE 9000
CMD ["php-fpm"]
```

**Fix:**
```dockerfile
FROM php:8.2-fpm-alpine AS runtime
RUN apk add --no-cache postgresql-dev \                         # fewer layers, alpine is smaller
    && docker-php-ext-install pdo_pgsql
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer  # multi-stage
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-interaction  # cache deps separately
COPY . .
RUN php artisan optimize
USER www-data
EXPOSE 9000
CMD ["php-fpm"]
```

### Challenge 2: Minimal Go Dockerfile

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/chronos ./cmd/chronos

FROM scratch
COPY --from=build /app/chronos /chronos
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/chronos"]
```

**Questions:**
- Why `CGO_ENABLED=0`? → Static binary (no libc dependencies). Runs on scratch.
- Why copy `ca-certificates.crt`? → If app makes HTTPS calls (S3 API calls).
- Why scratch? → Minimum attack surface, small image (~5 MB for Go binary).

### Challenge 3: Docker Compose for local dev with hot reload

```yaml
services:
  app:
    build:
      context: .
      target: runtime
    volumes:
      - .:/app          # bind mount for hot reload
      - /app/vendor     # exclude vendor from bind mount
    environment:
      APP_DEBUG: "true"
      DB_HOST: postgres
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
```

**Questions:**
- Why exclude `/app/vendor` from bind mount? → Mac/Windows filesystem performance. Vendor is in the image.
- Why use `condition: service_healthy`? → Wait for DB to accept connections before starting app.

---

> **Next topic in skill order:** CI/CD.
