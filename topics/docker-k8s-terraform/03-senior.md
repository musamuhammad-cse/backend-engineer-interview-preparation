# Docker, Kubernetes & Terraform — Senior Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** K8s basics (Pods, Deployments, Services, Ingress)  
> **Estimated time:** 10–12 hours

---

## Table of Contents

1. K8s Production Patterns
2. Helm
3. Service Mesh (Istio)
4. Horizontal Pod Autoscaler
5. Pod Disruption Budgets
6. Network Policies
7. RBAC
8. Terraform Deep Dive
9. Terraform State and Backends
10. Terraform Modules and Composition
11. Terraform in CI/CD
12. Production EKS with IaC
13. Q&A

---

## 1. K8s Production Patterns

### Sidecar Pattern

A helper container that runs alongside the main application in the same Pod.

**Use cases:** logging agent, metrics collector, proxy, config reloader

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: api:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: logs
              mountPath: /var/log/app
        - name: fluentd
          image: fluent/fluentd:v1.16
          volumeMounts:
            - name: logs
              mountPath: /var/log/app
            - name: fluentd-config
              mountPath: /fluentd/etc
      volumes:
        - name: logs
          emptyDir: {}
        - name: fluentd-config
          configMap:
            name: fluentd-config
```

### Ambassador Pattern

A proxy container that mediates traffic between the application and external services.

```yaml
containers:
  - name: api
    image: api:latest
    env:
      - name: DB_HOST
        value: localhost  # connects to ambassador
      - name: DB_PORT
        value: "5432"
  - name: db-ambassador
    image: envoyproxy/envoy:v1.30
    ports:
      - containerPort: 5432
    # Envoy handles TLS, circuit breaking, retries, observability
```

### Init Containers

Run to completion before the main containers start:

```yaml
spec:
  initContainers:
    - name: migrate
      image: api:latest
      command: ["php", "artisan", "migrate", "--force"]
      env:
        - name: DB_HOST
          value: postgres.default
  containers:
    - name: api
      image: api:latest
```

Init containers share volumes with main containers. They must exit successfully (code 0) or the Pod never starts.

> **Trap:** Init containers are NOT restarted on completion. If you need to run a job every deployment, use a Helm pre-upgrade hook or a separate Job.

### Graceful Shutdown (PreStop Hook)

```yaml
spec:
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                # Signal application to stop accepting new connections
                kill -SIGTERM $(pidof php-fpm)
                # Wait for in-flight requests to complete
                sleep 10
```

The Pod receives a SIGTERM (from `kubelet`), then has `terminationGracePeriodSeconds` (default 30s) to shut down. After that, SIGKILL.

### Resource Patterns

| Pattern | Description |
|---------|-------------|
| Always set requests & limits | Prevents noisy neighbor, ensures predictable scheduling |
| Requests should match typical usage | Requests for scheduling, limits for crash protection |
| CPU limits can cause throttling | Consider not setting CPU limits (Burstable QoS) if throttling is measured |
| Memory limits are critical | Without them, a leak can OOM-kill other pods |

---

## 2. Helm

Helm is the Kubernetes package manager. Charts are packages of pre-configured K8s resources.

### Chart structure

```
mychart/
├── Chart.yaml          # metadata (name, version, dependencies)
├── values.yaml         # default values
├── charts/             # sub-chart dependencies (pulled on dependency update)
└── templates/          # Go template YAML files
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── _helpers.tpl    # named templates (macros)
    └── NOTES.txt       # displayed after install
```

### Chart.yaml

```yaml
apiVersion: v2
name: api
description: API service for inventory management
type: application
version: 0.1.0
appVersion: "1.2.3"
dependencies:
  - name: postgresql
    version: "12.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  host: api.example.com
  path: /v1/?
  annotations:
    kubernetes.io/ingress.class: alb

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

env:
  APP_ENV: production
  APP_DEBUG: "false"
```

### Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}
  labels:
    {{- include "api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
```

### Helm commands

```bash
# Install a release
helm install my-api ./api-chart --values prod-values.yaml

# Upgrade (rolls out changes)
helm upgrade my-api ./api-chart --values prod-values.yaml

# Rollback to revision
helm rollback my-api 2

# List releases
helm list -n production

# Template rendering (no cluster interaction)
helm template ./api-chart --values prod-values.yaml

# Package chart
helm package ./api-chart -d ./packages

# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency update ./api-chart
```

### Helm best practices

- **Separate values per environment** — `dev-values.yaml`, `staging-values.yaml`, `prod-values.yaml`
- **Pin chart versions** — use exact versions in `Chart.lock`
- **Use `include` for names/labels** — consistency via `_helpers.tpl`
- **Test your charts** — `helm test` runs pod-based tests
- **Manage dependencies** — `helm dependency update` downloads sub-charts

### Helm Hooks

Run actions at certain points in the release lifecycle:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    helm.sh/hook: pre-upgrade      # runs before upgrade
    helm.sh/hook-weight: "1"       # ordering
    helm.sh/hook-delete-policy: before-hook-creation  # clean up before re-run
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: api:latest
          command: ["php", "artisan", "migrate", "--force"]
      restartPolicy: Never
```

Available hooks: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`

> **Trap:** Helm hooks are Jobs. If a pre-upgrade Job fails (DB migration fails), the upgrade fails and the release is NOT updated. This is safer than running migrations in init containers.

---

## 3. Service Mesh (Istio)

### What a service mesh provides

- **Traffic management** — routing, retries, timeouts, circuit breaking
- **Observability** — mTLS metrics, tracing, access logs
- **Security** — mTLS between all services, authorization policies
- **Resilience** — fault injection, mirroring, retries, circuit breaking

### Istio architecture

```
┌─────────────────────────┐
│     Istio Control Plane │
│  (istiod)               │
│  - Pilot (routing)      │
│  - Citadel (certs)      │
│  - Galley (config)      │
└─────────────────────────┘
          │
          ▼ xDS protocol
┌─────────────────────────┐
│      Pod                │
│  ┌───────────────────┐  │
│  │ App Container     │  │
│  │ (localhost:8080)  │  │
│  └─────────┬─────────┘  │
│            │            │
│  ┌─────────▼─────────┐  │
│  │ Envoy Proxy       │  │
│  │ (sidecar)         │  │
│  │ - Inbound rules   │  │
│  │ - Outbound rules  │  │
│  │ - mTLS            │  │
│  │ - Metrics         │  │
│  └───────────────────┘  │
└─────────────────────────┘
```

### Key Istio resources

```yaml
# VirtualService — routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
    - api-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: api-service
            subset: v2
          weight: 100
    - route:
        - destination:
            host: api-service
            subset: v1
          weight: 90
        - destination:
            host: api-service
            subset: v2
          weight: 10
      retries:
        attempts: 3
        retryOn: connect-failure,refused-stream,503
      timeout: 10s
---
# DestinationRule — circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: "1.0"
    - name: v2
      labels:
        version: "2.0"
```

> **Trap:** Istio adds ~5–15ms latency per hop (Envoy proxy overhead) and significant resource usage (Envoy sidecars). For latency-sensitive apps (trading platform), evaluate if service mesh overhead is acceptable.

---

## 4. Horizontal Pod Autoscaler (HPA)

Automatically scales the number of pods based on metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

### How HPA works

```
HPA controller polls metrics API every 15s (default)
  │
  ├── desiredReplicas = ceil[currentReplicas × (currentValue / targetValue)]
  │
  ├── Scale up: no cooldown, faster response
  └── Scale down: cooldown period (default 5 min) prevents thrashing
```

### Custom metrics with Prometheus adapter

HPA can use custom metrics from Prometheus via the `prometheus-adapter`:

```yaml
metrics:
  - type: Object
    object:
      metric:
        name: http_requests_total
      describedObject:
        apiVersion: v1
        kind: Service
        name: api-service
      target:
        type: Value
        value: "10000"
```

### Horizontal vs Vertical scaling

| | HPA (horizontal) | VPA (vertical) |
|---|---|---|
| Unit | Pod count | Pod size (CPU/memory) |
| Response | Add/remove pods | Restart pod with new resources |
| Model | Stateless only | Stateful support |
| Configuration | Deployment | StatefulSet, DaemonSet |

---

## 5. Pod Disruption Budgets (PDB)

Ensures a minimum number of pods are available during voluntary disruptions (node drain, cluster upgrades, auto-scaling down):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2          # at least 2 pods must always be running
  selector:
    matchLabels:
      app: api
```

Or as a percentage:

```yaml
spec:
  maxUnavailable: 25%      # at most 25% of pods can be down during disruption
```

> **Trap:** PDB only covers VOLUNTARY disruptions (node drain, cluster updates). Involuntary disruptions (node crash, AZ outage) are NOT protected by PDB. You need multi-AZ deployment for that.

---

## 6. Network Policies

Pod-level firewall (ingress and egress). Not enforced unless a CNI that supports it is installed (Calico, Cilium, AWS VPC CNI).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: production
          podSelector:
            matchLabels:
              app: ingress-gateway
      ports:
        - port: 8080
          protocol: TCP
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: production
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
          protocol: TCP
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - port: 443
          protocol: TCP
```

> **Trap:** NetworkPolicies are allow-lists — by default, all traffic is allowed. Once ANY NetworkPolicy selects a pod, ALL traffic to/from that pod is DENIED unless explicitly allowed by SOME policy.

---

## 7. RBAC

Role-Based Access Control for K8s resources:

```yaml
# Role (within namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
# ClusterRole (cluster-wide)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: read-pods
subjects:
  - kind: User
    name: dev@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Principle of least privilege:** CI/CD systems get minimal permissions per namespace. Humans get view-only in production, edit in dev.

---

## 8. Terraform Deep Dive

### Core concepts

Terraform is Infrastructure as Code (IaC): declare desired state, Terraform converges to it.

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_s3_bucket" "state" {
  bucket = "inventory-terraform-state"
}

resource "aws_dynamodb_table" "state_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Terraform workflow

```
$ terraform init        # Initialize providers, modules, backend
$ terraform fmt         # Format code
$ terraform validate    # Validate syntax + logic
$ terraform plan        # Show changes without applying
$ terraform apply       # Execute changes
$ terraform destroy     # Delete all managed resources
```

### Lifecycle rules

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    create_before_destroy = true                    # create new before destroying old
    prevent_destroy       = true                    # prevent accidental deletion
    ignore_changes        = [password, tags]        # ignore changes made outside Terraform
  }
}
```

### Meta-arguments

```hcl
# count — create multiple instances of a resource
resource "aws_iam_user" "developer" {
  count = length(var.developers)
  name  = var.developers[count.index]
}

# for_each — create from map
resource "aws_s3_bucket" "data" {
  for_each = var.buckets
  bucket   = each.value.name
  tags     = each.value.tags
}

# depends_on — explicit dependency
resource "aws_iam_role_policy" "s3_access" {
  depends_on = [aws_iam_role.ecs_task]
}
```

### Functions

```hcl
# String
lower(var.name)
format("%s-%s", var.environment, var.name)

# Collection
length(var.items)
element(var.list, 0)
merge(var.default_tags, var.extra_tags)

# Encoding
filebase64("${path.module}/config.json")
jsonencode(local.config)

# Filesystem
file("${path.module}/policy.json")
templatefile("${path.module}/user_data.sh", {
  db_host = aws_db_instance.main.address
})

# Network
cidrsubnet("10.0.0.0/16", 8, 1)   # → "10.0.1.0/24"

# Conditional
var.environment == "production" ? "db.r6g.large" : "db.t4g.medium"
```

---

## 9. Terraform State and Backends

### Remote state with S3 + DynamoDB

```hcl
terraform {
  backend "s3" {
    bucket         = "inventory-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

| Component | Purpose |
|-----------|---------|
| **S3 bucket** | Store state file (encrypted at rest) |
| **DynamoDB table** | Locking — prevents concurrent modifications |
| **State file** | JSON mapping of resources to real-world infrastructure |

> **Trap:** NEVER commit state files to Git! State files contain secrets (DB passwords, API keys, private keys). Always use remote backends.

### State commands

```bash
terraform state list                                     # list all resources
terraform state show aws_s3_bucket.state                 # show resource detail
terraform state mv aws_s3_bucket.old aws_s3_bucket.new  # rename resource
terraform state rm aws_s3_bucket.old                     # remove from state (not delete)
terraform import aws_s3_bucket.existing my-bucket        # import existing resource
```

### Workspaces

Manage multiple environments:

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new production
terraform workspace select production

# Reference current workspace
locals {
  env = terraform.workspace
}
```

```hcl
# backend key per workspace
terraform {
  backend "s3" {
    key = "env:/${terraform.workspace}/terraform.tfstate"
  }
}
```

---

## 10. Terraform Modules and Composition

### Module structure

```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
├── ecs-service/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── rds/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

### VPC module

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
}

variable "environment" {
  description = "Environment name (tags)"
  type        = string
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-public-${count.index}"
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name        = "${var.environment}-private-${count.index}"
    Environment = var.environment
  }
}

# Internet Gateway, NAT Gateway, Route Tables...
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

### Using modules

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
  environment          = var.environment
}

module "rds" {
  source = "./modules/rds"

  engine         = "postgres"
  engine_version = "16.3"
  instance_class = var.environment == "production" ? "db.r6g.large" : "db.t4g.medium"
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.private_subnet_ids
}

module "ecs_api" {
  source = "./modules/ecs-service"

  name        = "api"
  image       = "${var.ecr_repository}:${var.image_tag}"
  cpu         = 1024
  memory      = 2048
  port        = 8080
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnet_ids
}
```

### Module sources

| Source | Example | Use case |
|--------|---------|----------|
| Local path | `./modules/vpc` | Internal modules |
| Git | `git::https://github.com/org/terraform-modules.git//vpc` | Shared across teams |
| Terraform Registry | `hashicorp/consul/aws` | Public modules |
| S3 | `s3::https://s3-us-east-1.amazonaws.com/bucket/modules.zip` | Private artifact |

### Module best practices

- **Pin module versions** — use Git tags/branches, not `latest`
- **Semantic versioning** — breaking changes = major version bump
- **Small, focused modules** — VPC, RDS, ECS service, IAM roles (don't create a "monolith" module)
- **Document inputs/outputs** — each variable and output needs a description
- **Composable, not coupled** — modules should work independently

---

## 11. Terraform in CI/CD

### GitHub Actions example

```yaml
name: Terraform
on:
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
        working-directory: terraform/environments/production
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Atlantis

Terraform automation for pull requests:

1. Developer opens PR with Terraform changes
2. PR comment: `atlantis plan` → shows plan output in PR
3. Review plan, comment `atlantis apply` → auto-apply after merge

### Terragrunt

Wrapper for Terraform to reduce code duplication:

```hcl
# terragrunt.hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "inventory-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
}
EOF
}
```

---

## 12. Production EKS with IaC

### EKS module

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "inventory-${var.environment}"
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  # EKS managed node groups
  eks_managed_node_groups = {
    main = {
      desired_size = 3
      min_size     = 3
      max_size     = 10

      instance_types = ["m6i.large", "m6a.large", "m7i.large"]
      capacity_type  = "ON_DEMAND"

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          volume_type = "gp3"
          volume_size = 100
        }
      }

      tags = {
        Environment = var.environment
        NodeGroup   = "main"
      }
    }

    spot = {
      desired_size = 0
      min_size     = 0
      max_size     = 20

      instance_types = ["c6i.large", "c6a.large", "c7i.large"]
      capacity_type  = "SPOT"

      tags = {
        Environment = var.environment
        NodeGroup   = "spot"
      }
    }
  }

  # Karpenter
  enable_karpenter                 = true
  karpenter_enable_spot_termination = true

  # IRSA for K8s services
  enable_irsa = true

  # Cluster security group
  cluster_endpoint_public_access           = false
  cluster_endpoint_private_access          = true
  cluster_endpoint_public_access_cidrs     = []

  tags = {
    Environment = var.environment
  }
}
```

### Karpenter provisioning

```hcl
resource "kubectl_manifest" "karpenter_provisioner" {
  yaml_body = <<YAML
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand", "spot"]
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64"]
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["c", "m", "r"]
      nodeClassRef:
        name: default
  limits:
    cpu: 200
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  role: "inventory-prod-karpenter"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "inventory-production"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "inventory-production"
  tags:
    Environment: production
YAML
}
```

---

## 13. Q&A

### Senior

**Q: How would you design a multi-tenant K8s cluster?**
A: (1) Namespaces per tenant (resource quotas, network policies). (2) RBAC per namespace (tenant admins limited to their namespace). (3) Priority classes to ensure critical workloads get resources. (4) Node taints + tolerations to separate tenants on dedicated nodes (if needed). (5) Cost allocation via labels + Kubecost.

**Q: What's the difference between Terraform's `count` and `for_each`?**
A: `count` works with lists (indexed access). `for_each` works with maps/sets of strings (keyed access). `for_each` is preferred because elements can be removed from the middle without shifting indexes.

**Q: How do you handle Terraform state corruption?**
A: (1) Use remote backends with locking (S3 + DynamoDB). (2) State backups (S3 versioning). (3) `terraform state pull` → inspect → `terraform state push` (last resort). (4) Import resources into a fresh state as a fallback.

**Q: How do you implement blue-green deployment on K8s?**
A: (1) Two Deployments (blue = current, green = new). (2) Service selector switches between them (`version: blue` → `version: green`). (3) Or use Istio VirtualService with weights. (4) Or use Flagger (GitOps progressive delivery).

**Q: What's the sidecar pattern and when would you use it?**
A: A helper container in the same Pod as the main app. Use for: logging (Fluentd), monitoring (Prometheus exporter), service mesh (Envoy), TLS termination, config reloader.

**Q: How do you handle secrets in Terraform?**
A: (1) Never hardcode. (2) Use `sensitive = true` on outputs. (3) Reference AWS Secrets Manager: `data.aws_secretsmanager_secret_version.db`. (4) Use Terraform Cloud's sensitive variable encryption. (5) State files WILL contain secrets — use remote backends with encryption.

### Trap questions

**Q: You apply `terraform destroy` and it deletes an S3 bucket with production data. How does this happen?**
A: No `prevent_destroy = true` lifecycle on the bucket resource. Terraform follows the state file — if the resource is in state and you destroy, it's gone. Always set `prevent_destroy = true` on data-critical resources and use `terraform plan` review.

**Q: A Helm upgrade fails halfway. Pods are in a bad state. What do you do?**
A: `helm rollback <release> <revision>`. Helm tracks each revision. Rollback to the previous working revision restores the prior state.

**Q: HPA is configured to scale based on CPU at 70%. CPU spikes to 90% but HPA doesn't scale. Why?**
A: (1) Metrics server not installed/running. (2) HPA not matching the deployment selector. (3) No resource requests set on containers (HPA needs CPU requests to calculate utilization). (4) Cooldown period preventing scale-up.

**Q: Two Pods in the same namespace can reach each other via Service name. Then you apply a NetworkPolicy and they can't. Why?**
A: NetworkPolicy is an allow-list. Once any policy selects a pod, ALL traffic is denied unless explicitly allowed. You need ingress/egress rules in the policy to allow the traffic.

**Q: Terraform plan shows changes to a resource you didn't modify. What happened?**
A: (1) Someone modified the resource outside Terraform ("drift"). (2) Provider version upgrade changed defaults. (3) A referenced resource changed (e.g., module input changed). Run `terraform plan` in CI to catch drift.

### Follow-up questions

**Q: You mentioned Karpenter. How does it decide which instance type to launch?**
A: Karpenter evaluates instance types based on pod resource requests + constraints (architecture, instance category, GPU). It launches the cheapest instance type that fits the pending pods.

**Q: How would you debug a Pod in CrashLoopBackOff?**
A: (1) `kubectl describe pod` → recent events show errors. (2) `kubectl logs pod --previous` (logs from the crashed container). (3) Check resource limits (OOMKilled?). (4) Check init containers (failed migration?). (5) Check ConfigMap/Secret (missing keys?).

**Q: Terraform workspaces vs Terraform Cloud — when would you use each?**
A: Workspaces: simple env separation within a single backend (for smaller teams). Terraform Cloud: remote execution, VCS integration, sentinel policies, team collaboration, audit (for larger orgs).
