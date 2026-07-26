# AWS Services — Senior Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Basic and intermediate AWS tiers  
> **Estimated time:** 10–12 hours

---

## Table of Contents

1. ECS (Elastic Container Service)
2. EKS (Elastic Kubernetes Service)
3. Lambda + API Gateway
4. SQS (Simple Queue Service)
5. SNS (Simple Notification Service)
6. Secrets Manager and Parameter Store
7. VPC Advanced
8. Multi-Region Architectures
9. Well-Architected Framework
10. Q&A

---

## 1. ECS (Elastic Container Service)

### Launch types

| Feature | Fargate | EC2 |
|---------|---------|-----|
| Server management | AWS manages all underlying instances | You manage EC2 instances (patching, scaling, instance type selection) |
| Pricing | Per vCPU + per GB memory (no idle cost) | Per EC2 instance (pay for idle capacity) |
| Scaling | Task-level auto scaling (add/remove tasks) | Instance-level + task-level (more complex) |
| GPU support | No (2024: limited preview) | Yes (p3/g4 instances) |
| Custom AMI | No | Yes |
| Daemon tasks | No (sidecars via task definition) | Yes (logging agents, monitoring) |
| Data volume | EFS only, ephemeral storage (20 GB default, up to 200 GB) | EBS, EFS, ephemeral, Docker volumes |

> **Trap:** Fargate tasks have a maximum of 10 GB ephemeral storage by default (up to 200 GB with `ephemeralStorage` setting). If you need more than 200 GB, use EC2 launch type or mount EFS.

### Task definition

The blueprint for running a container:

```json
{
  "family": "api-task",
  "taskRoleArn": "arn:aws:iam::123456789012:role/api-task-role",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/api:latest",
      "portMappings": [{
        "containerPort": 8080,
        "protocol": "tcp"
      }],
      "environment": [
        { "name": "DB_HOST", "value": "database.internal" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

Key role distinctions:
- **Task role** — IAM role the APPLICATION uses (S3 access, DynamoDB, etc.)
- **Execution role** — IAM role ECS uses to pull images from ECR, send logs to CloudWatch, fetch secrets

> **Trap:** Many engineers conflate task role and execution role. The execution role is ONLY for ECS agent actions (pull image, send logs). The task role is for your application's AWS API calls.

### Service vs Task

| | Service | Task (RunTask) |
|---|---|---|
| Persistence | Keeps desired count running | Run once, exit |
| Restart | Restarts on failure | Does not restart |
| Scheduling | Spread across AZs | Run on any available capacity |
| Load balancer | Yes (ALB/NLB) | No |
| Use case | Long-running apps (web servers, workers) | Batch jobs, migrations, one-off processes |

### Service auto scaling

ECS Service Auto Scaling uses the Application Auto Scaling service:

```yaml
# Terraform — ECS service with target tracking scaling
resource "aws_appautoscaling_target" "api" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "api-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70
    scale_in_cooldown  = 60
    scale_out_cooldown = 60
  }
}
```

> **Trap:** ECS service scaling adds/removes TASKS, not instances. On Fargate, this is sufficient. On EC2, you also need ASG scaling for the underlying instances, or use Capacity Providers.

### Capacity Providers

Connect ECS to the underlying infrastructure:
- **Fargate Capacity Provider** — default, no config needed
- **ASG Capacity Provider** — connect an ASG to ECS; ECS manages the ASG

Using Capacity Providers means ECS automatically manages instance scaling based on task placement needs:

```yaml
resource "aws_ecs_capacity_provider" "ec2" {
  name = "ec2-cp"
  auto_scaling_group_provider {
    auto_scaling_group_arn         = aws_autoscaling_group.ecs.arn
    managed_termination_protection = "ENABLED"
    managed_scaling {
      maximum_scaling_step_size = 10
      minimum_scaling_step_size = 1
      status                    = "ENABLED"
      target_capacity           = 80
    }
  }
}
```

### Task placement strategies

| Strategy | Description |
|----------|-------------|
| **binpack** | Pack tasks tightly (fewest instances). Lowest cost, highest risk. |
| **spread** | Spread across AZs and instances. Highest availability. |
| **distinctInstance** | Place each task on a different instance. |

For production, use `spread` across AZs for HA.

### Service Connect

- Service mesh for ECS (no sidecar needed)
- Service discovery via DNS (`svc-name.namespace`)
- Traffic encryption between services
- Circuit breaking, retries
- Integrates with CloudWatch metrics

---

## 2. EKS (Elastic Kubernetes Service)

### Architecture

```
EKS Control Plane (managed by AWS)
├── etcd (multi-AZ)
├── API server
├── Scheduler
└── Controller Manager

Data Plane (your responsibility)
├── Managed Node Group
│   └── EC2 instances (joined to cluster)
├── Self-managed Node Group
│   └── You manage everything
└── Fargate Profile
    └── Serverless pods (no node management)
```

### Control plane

AWS manages the control plane:
- Multi-AZ, auto-scaling, auto-updates
- API server endpoint (public, private, or both)
- AWS manages cert rotation, etcd backup
- You pay $0.10/hour per cluster

> **Trap:** EKS control plane is a flat $0.10/hour regardless of cluster size. For small clusters, this can be expensive relative to node costs. For chronos, a single-task Fargate ECS might be cheaper if K8s isn't needed.

### Node groups

| Type | Management | Patching | Scaling |
|------|-----------|----------|---------|
| **Managed Node Group** | AWS handles rollouts, security groups, labels | AWS Auto-updates AMI | Supports ASG, can use spot |
| **Self-managed** | You handle everything | You update AMIs | You manage ASG |

### Fargate profiles

Run pods without managing nodes:
- Each pod gets its own vCPU/memory allocation
- Pods run on Fargate infrastructure
- Network: each pod gets its own ENI
- You pay per pod (no node costs)

Use for: batch jobs, burst workloads, CI/CD runners. Not ideal for: daemonsets, privileged containers, sidecars requiring host network.

> **Trap:** Fargate profiles use a namespace selector. All pods in matching namespaces run on Fargate. You cannot selectively run some pods on Fargate and others on EC2 in the same namespace.

### AWS Load Balancer Controller

Previously known as AWS ALB Ingress Controller. Manages ALB/NLB for Kubernetes services:

- **Ingress** → ALB (Layer 7)
- **Service type LoadBalancer** → NLB (Layer 4)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip       # ip vs instance
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/success-codes: "200"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/*
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

> **Trap:** `target-type: ip` means ALB routes directly to pod IPs (requires VPC CNI). `target-type: instance` routes to node IP + NodePort. `ip` is preferred for better load distribution.

### IRSA (IAM Roles for Service Accounts)

Each K8s service account can assume an IAM role:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/api-role
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: api-service-account
```

The pod gets temporary credentials via the EKS OIDC provider. This is the recommended way to grant AWS permissions to pods — never use long-term access keys.

### VPC CNI (Container Network Interface)

- Each pod gets a VPC IP address (from the subnet CIDR)
- Native VPC networking — no overlay, no NAT
- Pods have the same IP reachability as EC2 instances

**Limitation:** Each node can only run as many pods as the max ENIs × IPs per ENI for that instance type. For `m5.large` (3 ENIs × 10 IPs = 30 max pods). Use prefix delegation to increase (each ENI gets a /28 prefix of 16 IPs).

> **Trap:** Running out of IP addresses in the subnet is a common EKS issue. Plan subnet size considering pods (not just nodes). Use a /16 per VPC and /20 per AZ for production.

### Cluster Auto Scaler vs Karpenter

| Feature | Cluster Auto Scaler | Karpenter |
|---------|-------------------|-----------|
| Scaling logic | Schedules pending pods to existing ASG or creates new nodes | Just-in-time provisioning, creates nodes of exact size needed |
| Node diversity | Single ASG per instance type | Multiple instance types per workload |
| Node termination | De-scale by identifying empty nodes | Consolidation (replace 3 nodes with 2 cheaper ones) |
| Setup | Requires ASGs configured | Manages its own instances via EC2 Fleet |

Karpenter is newer and generally preferred for:
- Faster scaling (seconds vs minutes)
- Better bin-packing (right-size instances)
- Automatic consolidation (cost optimization)

### Bottlerocket OS

- Linux-based OS optimized for container workloads
- Immutable filesystem (no SSH into running system — use `bottlerocket-admin` container)
- Smaller attack surface, smaller footprint (fewer packages)
- Atomic updates (dual root partition, A/B update)

---

## 3. Lambda + API Gateway

### Lambda execution environment lifecycle

```
Cold start:
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │ Init (CODE) │ → │ Init (RUNTIME) │ → │ Init (EXTENSIONS) │ → │ Handler invoked │
  │ Load code   │   │ Init runtime  │   │ Init layers/ │   │               │
  │ from S3     │   │ (Node, Python,│   │ extensions   │   │               │
  │             │   │  Java, etc.)  │   │             │   │               │
  └─────────────┘   └─────────────┘   └─────────────┘   └───────────────┘
                                                          │
Warm start:                                                │
  ┌──────────────────────────────────────────────────────────────┐
  │ Handler invoked (execution environment reused)               │
  └──────────────────────────────────────────────────────────────┘
```

Cold start duration varies by runtime:
- Python/Node: ~100–300ms
- Go (compiled): ~50–200ms
- Java: ~500–2,000ms (worse, but SnapStart helps)
- C# (.NET): ~300–800ms (SnapStart available)

> **Trap:** VPC Lambda cold starts are even slower (200–500ms extra) because AWS must create an ENI in your VPC. Use RDS Proxy to reduce DB connections, and provisioned concurrency for latency-sensitive VPC Lambdas.

### Concurrency

| Concept | Description |
|---------|-------------|
| **Reserved concurrency** | Guarantee a specific number of concurrent executions. Also sets a max (prevents runaway scaling). |
| **Provisioned concurrency** | Pre-initialize a number of environments — zero cold starts. Pay for pre-warmed capacity. |
| **Unreserved concurrency** | Remaining regional concurrency shared across all functions. |
| **Regional limit** | Default 1,000 concurrent executions per region (can request increase). |

```yaml
# Terraform — Lambda with provisioned concurrency and alias
resource "aws_lambda_function" "api" {
  function_name = "api-handler"
  role          = aws_iam_role.lambda.arn
  image_uri     = "123456789012.dkr.ecr.us-east-1.amazonaws.com/api:latest"
  package_type  = "Image"
  memory_size   = 1024
  timeout       = 30
  reserved_concurrent_executions = 100
  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }
  environment {
    variables = {
      DB_HOST = "database.internal"
    }
  }
}

resource "aws_lambda_alias" "prod" {
  name             = "prod"
  function_name    = aws_lambda_function.api.function_name
  function_version = aws_lambda_function.api.version

  routing_config {
    additional_version_weights = {
      "2" = 0.05  # 5% traffic to version 2
    }
  }
}

resource "aws_lambda_provisioned_concurrency_config" "prod" {
  function_name                     = aws_lambda_function.api.function_name
  qualifier                         = aws_lambda_alias.prod.name
  provisioned_concurrent_executions = 10
}
```

### Lambda SnapStart (Java/.NET)

- Takes a snapshot of the initialized execution environment
- Restores from snapshot on warm start (cold start drops from 2s to 200ms)
- Only available for Java 11+ and .NET 8+
- Not compatible with some frameworks (Spring Boot needs migration)
- Does NOT work with VPC Lambdas (caveat: limited preview)

### Lambda Response Streaming

- Lambda can stream responses instead of buffering them
- Reduces time-to-first-byte for large payloads
- Works with Node.js, Python, Java
- Integrated with API Gateway and CloudFront

### Lambda + API Gateway

Two API Gateway types:

| Feature | HTTP API | REST API |
|---------|----------|----------|
| Pricing | ~$1.00/M requests | ~$3.50/M requests |
| Latency | Lower | Higher |
| JWT/OIDC auth | Native | Via Cognito Authorizer |
| Lambda authorizer | Yes (simple) | Yes (custom) |
| API keys / usage plans | No | Yes |
| WAF integration | Limited | Full |
| Request/response transformation | No (via Lambda) | Yes (mapping templates) |
| Custom domain | Yes | Yes |

> **Trap:** Choose HTTP API unless you need REST API features (WAF, API keys, usage plans, mapping templates). HTTP API is cheaper and faster.

**Lambda proxy integration:**
```javascript
// Node.js Lambda with API Gateway proxy integration
exports.handler = async (event) => {
  const path = event.requestContext.http.path  // "/api/users"
  const method = event.requestContext.http.method  // "GET"
  const body = event.body ? JSON.parse(event.body) : {}
  const userId = event.pathParameters?.id

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id: userId, name: 'John' })
  }
}
```

### Lambda best practices

1. **Keep handlers lightweight** — move initialization outside handler (DB connections, SDK clients)
2. **Reuse connections** — create HTTP clients, DB connections, SDK clients in global scope
3. **Fail fast** — return early on validation errors, use DLQ for async invocations
4. **Minimize dependencies** — smaller deployment = faster cold start
5. **Use environment variables for config** — not code changes
6. **Enable X-Ray tracing** — debug performance issues
7. **Set appropriate timeout** — don't default 3s if your function takes 10s
8. **Monitor with CloudWatch metrics** — `Throttles`, `Invocations`, `Errors`, `Duration`, `ConcurrentExecutions`

---

## 4. SQS (Simple Queue Service)

### Standard vs FIFO

| Feature | Standard | FIFO |
|---------|----------|------|
| Throughput | Unlimited (nearly) | 300 TPS base (3,000 with batching) |
| Ordering | Best-effort | Strict FIFO |
| Exactly-once | No (at-least-once) | Yes (via deduplication ID) |
| Message group | N/A | Yes (per-group ordering within queue) |
| Use case | Decoupling, async processing | Order processing, financial transactions |

> **Trap:** Standard queues deliver at-least-once, meaning duplicate messages are possible. Your consumer must be **idempotent**. This is a very common interview topic.

### Dead-Letter Queue (DLQ)

Messages that cannot be processed are moved to a DLQ:

| Setting | Description |
|---------|-------------|
| `maxReceiveCount` | How many times a message can be received before going to DLQ |
| Redrive | Reprocess messages from DLQ back to source queue |

```yaml
# Terraform — SQS with DLQ
resource "aws_sqs_queue" "dlq" {
  name = "orders-dlq"
}

resource "aws_sqs_queue" "orders" {
  name = "orders-queue"
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3
  })
}
```

> **Trap:** DLQ is not automatic cleanup. Messages sit in DLQ indefinitely until you either (a) re-drive them, (b) set a retention period on the DLQ, or (c) process them manually. DLQ retention period defaults to 4 days.

### Visibility timeout

- When a consumer receives a message, it becomes invisible to other consumers
- Default: 30 seconds
- Max: 12 hours
- Consumer must call `DeleteMessage` within the visibility timeout
- If consumer fails, the message becomes visible again after timeout expires

```
Consumer receives message → message is hidden (30s default)
  ├── Consumer calls DeleteMessage → message deleted
  └── Consumer doesn't respond → message reappears after 30s
```

> **Trap:** What happens if the consumer takes longer than the visibility timeout? The message becomes visible to another consumer while the first consumer is still processing. Both process the message (duplicate processing). Solution: call `ChangeMessageVisibility` to extend the timeout as you process.

### Long polling vs Short polling

| Feature | Short polling | Long polling |
|---------|--------------|--------------|
| Behavior | Returns immediately (even if empty) | Waits up to 20 seconds for messages |
| Empty responses | Many — costs more (paid per request) | Fewer — costs less |
| Latency | Lower (but many empty responses) | Higher for first message, but fewer poll cycles |
| Setting | Default | Set `WaitTimeSeconds` to 1–20 |

**Always use long polling.** It reduces cost and eliminates "empty response" spam.

### Message batching

- `SendMessageBatch` — send up to 10 messages at once (max 256 KB total)
- `ReceiveMessage` — receive up to 10 messages at once
- `DeleteMessageBatch` — delete up to 10 messages at once

Batching reduces costs (10 messages in 1 request = 1 request cost).

### Large messages

SQS max message size is 256 KB. For larger messages:
1. `SendMessage` with reference to S3 object
2. Consumer downloads object from S3
3. AWS SDK has "Extended Client" that does this automatically

### SQS + Lambda integration

Lambda can poll SQS queues (event source mapping):

```yaml
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn = aws_sqs_queue.orders.arn
  function_name    = aws_lambda_function.order_processor.arn
  batch_size       = 10
  maximum_batching_window_in_seconds = 30
  enabled          = true
}
```

Lambda automatically:
1. Polls SQS for messages
2. Sends batch to Lambda function
3. Deletes messages if Lambda succeeds
4. Leaves messages visible if Lambda fails (retry by SQS)

> **Trap:** Lambda's SQS batch processing is all-or-nothing per batch. If one message in a batch fails, the entire batch fails and all messages are returned to the queue. Set `bisectBatchOnFunctionError` to split the batch on failure.

---

## 5. SNS (Simple Notification Service)

### Topics and subscriptions

```
Publisher → SNS Topic → Subscriptions (fan-out)
                         ├── SQS Queue
                         ├── Lambda Function
                         ├── Email (to admin)
                         ├── SMS
                         ├── HTTP/HTTPS endpoint
                         └── Platform Application (push notification)
```

### Message filtering

SNS can filter messages before delivering to specific subscriptions:

```json
{
  "event": "order_created",
  "customer_tier": "premium"
}
```

Subscription filter policy:
```json
{
  "event": ["order_created", "order_shipped"],
  "customer_tier": ["premium"]
}
```

Only messages matching both filter conditions are delivered to that subscription.

### Fan-out pattern

SNS + SQS fan-out is the most common pattern for event-driven architectures:

```
                    ┌── SQS Queue A (Order Service)
                    │
Event → SNS Topic ──┼── SQS Queue B (Notification Service)
                    │
                    └── SQS Queue C (Analytics Service)
```

Each consumer gets its own queue. Orders service publishes once; all three queues receive the message independently.

> **Trap:** SNS does not retry failed deliveries (except for HTTP/HTTPS, which retries with exponential backoff). Combine with SQS to add durability — SNS sends to SQS, SQS retries and has DLQ.

### FIFO topics

- Topic type: FIFO (name must end in `.fifo`)
- Requires SQS FIFO subscriber
- Strict ordering per message group
- Throughput matches SQS FIFO (300/3,000 TPS)

### SNS vs SQS vs EventBridge

| Feature | SNS | SQS | EventBridge |
|---------|-----|-----|-------------|
| Pattern | Pub/sub (push) | Point-to-point (pull) | Pub/sub with routing |
| Delivery | Push to subscribers | Pull by consumers | Push to targets |
| Persistence | None | Up to 14 days | None |
| Ordering | No | FIFO available | No |
| Filtering | Message attributes | No | Event patterns (content-based) |
| Schema registry | No | No | Yes (Schema Registry) |
| Best for | Broadcast events | Work queues | Complex event routing |

---

## 6. Secrets Manager and Parameter Store

### Secrets Manager vs Parameter Store

| Feature | Secrets Manager | Parameter Store |
|---------|-----------------|-----------------|
| Max secret size | 65 KB | 8 KB (standard), 8 KB (advanced — 8 KB param + 8 KB value) |
| Automatic rotation | Yes (via Lambda) | No |
| Cross-region replication | Yes | No (advanced tier supports cross-region) |
| Pricing | $0.40/secret/month + $0.05/10,000 API calls | Standard free; advanced $0.05/param/month |
| RDS integration | Native (rotate RDS credentials) | No |
| Secret binary | Yes (base64-encoded) | Yes (advanced tier) |

> **Trap:** Secrets Manager costs add up. If you don't need automated rotation, Parameter Store is sufficient. Many teams use both: Parameter Store for config, Secrets Manager for DB credentials.

### Rotation

Secrets Manager rotates secrets by invoking a Lambda function:
- **Single-user rotation** — same credentials, just change password
- **Alternating-users** — two sets of credentials, switch between them (zero-downtime)

RDS rotation Lambda is provided by AWS — you just configure a CloudFormation template.

```yaml
# Terraform — RDS password in Secrets Manager with rotation
resource "aws_secretsmanager_secret" "db" {
  name                    = "rds-password"
  recovery_window_in_days = 7
  rotation_lambda_arn     = aws_lambda_function.rds_rotation.arn
  rotation_rules {
    automatically_after_days = 30
  }
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    dbname   = "mydb"
    engine   = "postgres"
    host     = aws_db_instance.main.address
    password = random_password.db.result
    port     = 5432
    username = "admin"
  })
}
```

---

## 7. VPC Advanced

### VPC Endpoints (Gateway vs Interface)

| Type | Services | How it works | Cost |
|------|----------|-------------|------|
| Gateway | S3, DynamoDB | Prefix list in route table | Free |
| Interface (PrivateLink) | All other AWS services | ENI in subnet, DNS entries | $0.01/hour + $0.01/GB data processed |

**Gateway Endpoint for S3:**
- Add prefix list for S3 to route table
- All S3 traffic stays within AWS network (no NAT, no IGW)
- Only works for S3 in the same region

**Interface Endpoint:**
- Creates an ENI with a private IP in your subnet
- DNS resolution for the service points to the ENI
- Supports cross-region access (if service supports it)

> **Trap:** S3 Gateway Endpoints DO NOT support cross-region access. For cross-region S3 access from a VPC, use Interface Endpoint or NAT Gateway.

### Transit Gateway

Hub-and-spoke network topology for multiple VPCs and on-prem:

```
                    ┌── VPC A (Production)
                    │
On-prem ── VPN ── Transit Gateway ──┼── VPC B (Staging)
                    │
                    ├── VPC C (Shared Services)
                    │
                    └── VPC D (DR)
```

- Replace VPC peering mesh with a single hub
- Supports VPN (site-to-site) and Direct Connect
- Cross-region peering (TGW to TGW)
- Network segmentation via route tables per attachment

### PrivateLink

Expose a service privately across VPCs or accounts:

```
Service Provider:
  NLB (internal) → EC2 instances → Your app

Consumer:
  Interface VPC Endpoint → PrivateLink → NLB → EC2
```

Use for: exposing internal APIs to partner accounts, multi-tenant SaaS where each tenant has their own VPC.

### VPC Flow Logs

Captures metadata about IP traffic:

| Field | Example |
|-------|---------|
| srcaddr | 10.0.1.5 |
| dstaddr | 10.0.2.10 |
| srcport | 3389 (RDP) |
| dstport | 443 |
| protocol | 6 (TCP) |
| action | ACCEPT / REJECT |
| bytes | 1234 |
| packets | 2 |

Flow logs can be published to CloudWatch Logs or S3. Useful for:
- Security analysis (unexpected traffic, port scans)
- Troubleshooting connectivity
- Capacity planning
- Compliance auditing

> **Trap:** Flow logs do NOT capture traffic that never reaches a network interface (e.g., traffic blocked by security group before evaluation). You see ACCEPT for traffic that security group allowed, but you don't see blocked traffic at the SG level — you see it at the NACL level if blocked there.

---

## 8. Multi-Region Architectures

### Active-Passive (warm standby)

```
Route 53 → us-east-1 (Active)
             ├── ALB → ASG → EC2
             └── RDS (Primary)
          → us-west-2 (Passive — standby)
             ├── ALB → ASG → EC2 (minimal capacity)
             └── RDS Read Replica (promote on failover)
```

- RDS cross-region read replica (async replication)
- Route 53 DNS failover routing
- DR site at minimum capacity (scaled up during failover)
- RPO: seconds to minutes (async replication lag)
- RTO: minutes (DNS propagation + replica promotion + scale up)

### Active-Active

```
Route 53 latency-based / geoproximity routing
             │
      ┌──────┴──────┐
      │             │
  us-east-1     eu-west-1
  (read/write)  (read/write)
      │             │
  ┌───┴───┐     ┌───┴───┐
  │ RDS   │     │ RDS   │
  │ Multi-│     │ Multi-│
  │ AZ    │     │ AZ    │
  └───────┘     └───────┘
      │             │
      └─────────────┘
      Global Database (Aurora)
```

- More complex — conflict resolution, cross-region latency
- Aurora Global Database (~1s replication)
- DynamoDB Global Tables for NoSQL
- Requires careful data model design (last-writer-wins or custom conflict resolution)

### Multi-region trade-offs

| Factor | Active-Passive | Active-Active |
|--------|---------------|---------------|
| Complexity | Lower | Higher |
| Latency | Higher (all traffic to primary) | Lower (serve from closest region) |
| Cost | Lower (DR region at minimal capacity) | Higher (two full production deployments) |
| RPO | Minutes (async) | Seconds (bidirectional sync) |
| RTO | Minutes | Near-zero |
| Data conflicts | None | Possible (need merge strategy) |

---

## 9. Well-Architected Framework

### Six pillars

| Pillar | Key question | Common best practices |
|--------|-------------|----------------------|
| **Operational Excellence** | How do you run and monitor? | Infrastructure as Code (Terraform), immutable deployments, runbooks, change management |
| **Security** | How do you protect data and systems? | IAM least-privilege, encryption at rest and in transit, VPC isolation, security groups, WAF, detective controls (CloudTrail, GuardDuty) |
| **Reliability** | How do you recover from failure? | Multi-AZ, auto scaling, health checks, retries with exponential backoff, circuit breakers, disaster recovery plan |
| **Performance Efficiency** | How do you use resources efficiently? | Right-size instances, use managed services, auto scaling, caching, async processing |
| **Cost Optimization** | How do you minimize costs? | Right-sizing, reserved instances/savings plans, spot instances, lifecycle policies, delete unused resources |
| **Sustainability** | How do you minimize environmental impact? | Efficient resource utilization, scheduled scaling, newer hardware, optimize data storage |

### Most common review findings

1. **Single-AZ deployment** — no HA, AZ outage = full outage
2. **No automated backups** — no PITR, data loss on failure
3. **Over-provisioned resources** — large instances with low utilization
4. **No monitoring/alarms** — reactive instead of proactive operations
5. **IAM over-permission** — `Resource: "*"`, managed policies too broad
6. **Missing encryption** — S3 buckets public or unencrypted, unencrypted EBS
7. **No redundancy** — single load balancer, single NAT Gateway, single EC2
8. **Unrestricted egress** — overly permissive outbound security groups

---

## 10. Q&A

### Senior

**Q: You need to design a fault-tolerant microservices architecture on AWS. Walk through the key decisions.**
A: Start with: (1) Multi-AZ for everything, (2) Fargate ECS for compute (no instance management), (3) ALB + target groups per service, (4) SQS for async communication between services, (5) RDS Multi-AZ or Aurora for persistence, (6) ElastiCache Redis for caching and sessions, (7) CloudFront + S3 for static assets, (8) Route 53 with failover routing for DR, (9) Auto scaling on CPU/memory, (10) CloudWatch + X-Ray for observability.

Key decisions: Fargate vs EC2 (cost vs management overhead), SQS vs SNS for inter-service messaging, RDS vs Aurora vs DynamoDB for data layer, VPC layout (public vs private subnets).

**Q: How do you handle a Lambda concurrency spike that exhausts the regional limit?**
A: (1) Set reserved concurrency per function to prevent single-function monopolization, (2) Use SQS as a buffer (queue messages, Lambda processes at its own pace), (3) Request limit increase, (4) Use provisioned concurrency for predictable traffic, (5) Implement backpressure in your event sources.

**Q: Your ECS tasks need access to an S3 bucket, RDS, and an external API. How do you handle credentials?**
A: ECS task role for S3 (IAM role attached to the task). RDS credentials from Secrets Manager (retrieved at startup via the execution role). External API key stored in Secrets Manager or Parameter Store, retrieved at startup. No credentials in environment variables or code.

**Q: You need to migrate a monolithic Laravel app running on a single EC2 to a scalable architecture. Describe the process.**
A: Phase 1: Move database to RDS Multi-AZ (lift and shift), move sessions/files to ElastiCache and S3. Phase 2: Wrap the monolith in a Docker container, deploy to ECS Fargate (single-task service). Phase 3: Extract services (auth, inventory, orders) as separate ECS services with SQS between them. Phase 4: Replace direct RDS queries with API calls for extracted services.

**Q: Design a cost-optimized architecture for a batch processing system that runs every night.**
A: Use Spot Instances (up to 90% discount) in ASG with lifecycle hooks for graceful shutdown. SQS queue for job distribution. ECS Fargate Spot tasks for containerized processing. Use scheduled scaling to scale to 0 during the day. S3 for input/output data. CloudWatch Events (EventBridge) to trigger the pipeline at midnight.

### Trap questions

**Q: A FIFO SQS queue is processing orders. You need 10,000 TPS. Can FIFO handle this?**
A: Yes, FIFO supports up to 3,000 TPS with batching (300 TPS without). For 10,000 TPS, you can use multiple message group IDs (parallel consumers). If that's not enough, use standard SQS with idempotent consumers.

**Q: Can you attach the same security group to both your ALB and your EC2 instances?**
A: Yes, but it's not recommended. Use one SG for the ALB (allows 443 from 0.0.0.0/0) and a separate SG for EC2 instances (allows traffic from ALB SG only). This follows least-privilege.

**Q: You have an ECS Fargate service that keeps failing with "Task failed to start" — what do you check?**
A: (1) Check the task StoppedReason in ECS console. (2) Check execution role (can it pull the image from ECR?). (3) Check VPC config (subnets must have route to NAT or use VPC endpoints for ECR/S3/CloudWatch). (4) Check memory/cpu limits. (5) Check CloudWatch Logs for startup logs.

**Q: Can RDS Multi-AZ failover happen across regions?**
A: No. Multi-AZ is within a single region (different AZs). For cross-region HA, use Aurora Global Database or cross-region read replicas with manual promotion.

**Q: You set up an S3 event notification to trigger Lambda. Some file uploads don't trigger the Lambda. Why?**
A: (1) S3 event notifications are at-least-once but not in-order. (2) If two files are uploaded at nearly the same time, one notification may be dropped. (3) Use S3 event notifications only for non-critical processing. For reliable processing, use SQS (S3 → SQS → Lambda).

**Q: What happens if every Lambda in a VPC tries to connect to the internet at the same time?**
A: Lambda uses NAT Gateway or NAT Instance for internet access. NAT has bandwidth limits (NAT Gateway: up to 45 Gbps per AZ). If all Lambdas need internet, you need multiple NAT Gateways or consider NAT Gateway scaling. Better: use VPC endpoints for AWS services and avoid internet access from Lambdas.

### Follow-up questions

**Q: You mentioned Fargate vs EC2. When would you definitely use EC2?**
A: (1) GPU workloads, (2) Large storage needs (>200 GB per task), (3) Daemon services (monitoring agents that run as sidecars), (4) Cost optimization at very large scale (thousands of tasks).

**Q: How do you achieve blue-green deployments on ECS?**
A: Use CodeDeploy with ECS. Create two task set versions (blue = current, green = new). CodeDeploy shifts traffic 10% → 50% → 100% with built-in rollback on health check failure. Or use the AWS Load Balancer Controller for blue-green on EKS.

**Q: What monitoring would you set up for a production ECS service?**
A: (1) CPU + Memory utilization (alarm at 80%), (2) ALB 5xx errors, (3) SQS queue depth, (4) Task count (unexpected scaling events), (5) Custom application metrics (error rate, latency p50/p95/p99), (6) Container Insights for detailed visibility, (7) X-Ray traces for request flows.
