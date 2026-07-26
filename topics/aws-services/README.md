# AWS Services — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS (Laravel on EC2/ECS, RDS PostgreSQL, ElastiCache Redis, S3 for assets, SQS for async jobs), trading platform (real-time data, Lambda, API Gateway), Chronos (containerized on ECS/Fargate), CI/CD with GitHub Actions deploying to AWS  
> **Note:** AWS is the dominant cloud platform in FAANG interviews. The senior signal is **architecting for failure**, **cost-aware design**, **IAM least-privilege**, and **understanding trade-offs between services** (e.g., SQS vs SNS vs Kinesis, ECS vs EKS, RDS vs Aurora vs DynamoDB).

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Sketch the architecture diagram on paper (whiteboard practice) | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what kills senior loops | 5 min |

**The senior signal is cross-service architecture decisions — knowing when to use which service and why.** Memorizing service features is table stakes; trade-off analysis under constraints (cost, latency, durability, operational overhead) is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | AWS global infrastructure (regions, AZs, edge locations), IAM (users, groups, roles, policies, trust relationships), VPC fundamentals (subnets, route tables, NAT gateway, security groups vs NACLs, VPC peering), EC2 (instance types, AMIs, EBS, user data, key pairs), S3 (buckets, objects, storage classes, versioning, bucket policies), shared responsibility model | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | ELB (ALB vs NLB vs CLB, target groups, stickiness, health checks), Auto Scaling (launch template, scaling policies, lifecycle hooks), RDS (Multi-AZ, read replicas, automated backups, parameter groups), Route 53 (record types, routing policies, health checks), CloudFront (distributions, origins, behaviors, signed URLs/cookies, Lambda@Edge), CloudWatch (metrics, logs, alarms, dashboards), cost management (reserved instances, savings plans, cost explorer) | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | ECS (cluster, task definition, service, Fargate vs EC2 launch type, service auto scaling, task placement), EKS (control plane, node groups, Fargate profiles, AWS Load Balancer Controller, IRSA), Lambda (cold starts, concurrency, reserved/concurrency, VPC networking, layers, Lambda SnapStart, Lambda + API Gateway HTTP/REST APIs), SQS (standard vs FIFO, dead-letter queues, delay queues, visbility timeout, long polling, batching), SNS (topics, subscriptions, message filtering, fan-out), Secrets Manager (rotation, RDS integration, vs Parameter Store), VPC advanced (VPC endpoints, private link, transit gateway, flow logs), multi-region architectures, Well-Architected Framework | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 150+ interview questions, architecture scenarios, debugging scenarios, cost optimization puzzles, real-world incident response prompts | Ongoing drill |

---

## Coverage map

### Global infrastructure
- Regions, Availability Zones (AZs), edge locations
- Data residency, sovereignty, compliance (SOC, PCI DSS)
- Service endpoints: public, regional, VPC, PrivateLink
- AWS global vs regional services

### Identity and Access Management (IAM)
- IAM users, groups, roles
- IAM policies: managed vs inline, policy structure (Effect, Action, Resource, Condition)
- Trust policies (who can assume a role)
- IAM Roles for Service Accounts (IRSA) on EKS
- Permissions boundaries, SCPs in Organizations
- Least-privilege principle: specific ARNs, condition keys, NotAction
- Instance profiles, service-linked roles

### Virtual Private Cloud (VPC)
- VPC CIDR, subnets (public, private, isolated), route tables
- Internet Gateway (IGW), NAT Gateway, NAT Instance
- Security groups (stateful, allow rules only)
- Network ACLs (stateless, allow/deny rules, numbered)
- VPC peering, transit gateway, VPC endpoints (Gateway + Interface)
- PrivateLink (powered by NLB + ENI)
- Flow logs, traffic mirroring

### EC2
- Instance types: general purpose, compute optimized, memory optimized, storage optimized, GPU
- Amazon Machine Images (AMI), marketplace, shared AMIs
- EBS: gp2, gp3, io1, io2, st1, sc1; snapshots, encryption, multi-attach
- Instance store (ephemeral storage)
- Security groups, key pairs (SSH)
- User data, metadata service (IMDSv1 vs IMDSv2)
- Placement groups: cluster, spread, partition
- Dedicated hosts, dedicated instances
- Elastic IP, ENI

### S3
- Buckets (globally unique name), objects (key, version, metadata)
- Storage classes: S3 Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier, Glacier Deep Archive
- Versioning, object locking (WORM)
- S3 bucket policies vs ACLs vs IAM policies
- Pre-signed URLs, multipart upload, S3 Transfer Acceleration
- Static website hosting, S3 event notifications
- S3 Object Lambda, S3 Select, S3 Batch Operations
- Replication (CRR, SRR), cross-region replication

### Elastic Load Balancing
- ALB: HTTP/HTTPS, path-based routing, host-based routing, WebSocket, HTTP/2
- NLB: TCP/UDP/TLS, static IP, ultra-low latency
- Target groups, health checks, deregistration delay
- Stickiness (session affinity), cross-zone load balancing
- Connection draining

### Auto Scaling
- Launch template, launch configuration (deprecated)
- Scaling policies: target tracking, step scaling, simple scaling
- Scheduled scaling, predictive scaling
- Lifecycle hooks, cooldown periods
- Warm pools, instance refresh

### RDS
- Supported engines: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora
- Multi-AZ (synchronous standby), read replicas (async, cross-region)
- Automated backups, manual snapshots, PITR
- Parameter groups, option groups
- Enhanced Monitoring, Performance Insights
- RDS Proxy (connection pooling)
- Aurora: storage (6 copies across 3 AZs), Serverless v2, Global Database

### Route 53
- DNS concepts: TTL, authoritative vs recursive DNS
- Record types: A, AAAA, CNAME, ALIAS, MX, TXT, NS, SOA
- Routing policies: simple, weighted, latency-based, failover, geolocation, geoproximity, multi-value
- Health checks (endpoint, calculated, CloudWatch alarm)
- Private hosted zones, Route 53 Resolver
- DNS failover architecture

### CloudFront
- Distributions: web vs RTMP
- Origins: S3, ALB, EC2, custom, Lambda@Edge origin
- Behaviors: path patterns, TTL, cache policies, origin request policies
- Signed URLs, signed cookies, OAI/OAC
- Geo-restriction, WAF integration
- Lambda@Edge (viewer request/response, origin request/response)
- Field-level encryption, real-time logs

### CloudWatch
- Metrics: standard (every 5 min), detailed (1 min), custom metrics
- Logs: log groups, log streams, metric filters, insights, subscription filters
- Alarms: state (OK, ALARM, INSUFFICIENT_DATA), actions (SNS, Auto Scaling)
- Dashboards: cross-region, cross-account
- Contributor Insights, ServiceLens, Synthetics (canaries)
- Container Insights, Lambda Insights

### ECS (Elastic Container Service)
- Cluster: EC2 vs Fargate launch type, Capacity Providers
- Task definition: CPU, memory, IAM role, port mappings, env vars, secrets
- Service: desired count, deployment (rolling, blue/green), service auto scaling
- Task placement strategies: binpack, spread, random
- Service Connect, Cloud Map
- ECS Exec, EFS integration

### EKS (Elastic Kubernetes Service)
- Control plane (managed by AWS), node groups (managed, self-managed)
- Fargate profiles (serverless pods)
- AWS Load Balancer Controller (ALB + NLB for k8s services)
- IRSA (IAM Roles for Service Accounts)
- Cluster auto-scaler, Karpenter
- EBS CSI driver, EFS CSI driver
- VPC CNI (max pods per node, prefix delegation)
- Managed node groups, Bottlerocket OS

### Lambda
- Function configuration: memory (128–10,240 MB), timeout (15 min max), ephemeral storage (512–10,240 MB)
- Triggers: S3, SQS, SNS, API Gateway, DynamoDB Streams, EventBridge
- Concurrency: reserved (min 2), provisioned, unreserved pool
- Cold starts, SnapStart (for Java/.NET), Lambda response streaming
- VPC networking: ENI, Hyperplane ENI, NAT requirement
- Lambda layers, Lambda extensions, Lambda Runtime API
- Lifecycle: init → invoke → shutdown
- Best practices: keep handlers lightweight, reuse connections, fail fast

### SQS (Simple Queue Service)
- Standard: at-least-once, best-effort ordering, unlimited TPS
- FIFO: exactly-once, strict ordering, 3,000 TPS (with batching)
- Dead-letter queues (DLQ): max receives, redrive, redrive allowlist
- Delay queues, visibility timeout, long polling, short polling
- Message batching (max 10), message size limit (256 KB)
- SQS extended client (large messages via S3)
- Cost: request-based pricing (1 request = 1–10 messages)

### SNS (Simple Notification Service)
- Topics: standard vs FIFO
- Subscriptions: SQS, Lambda, email, SMS, HTTP/HTTPS, Platform Application (push)
- Message filtering (by attributes), fan-out pattern
- Message durability: replication across AZs
- Raw message delivery vs JSON structure
- FIFO topics (with SQS FIFO subscriber)

### Secrets Manager
- Automatic secret rotation (with Lambda)
- Rotation strategies: single-user, alternating-users
- RDS, Redshift, DocumentDB, and custom DB credentials rotation
- Secrets vs Parameter Store: pricing, rotation, cross-region replication
- Retrieval via AWS SDK, AWS CLI, Secrets Manager agent

### Well-Architected Framework
- Six pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability
- AWS Well-Architected Tool, lenses (Serverless, SaaS, IoT)
- Common review findings: no backups, single-AZ, no fault tolerance, over-provisioned resources, missing IAM least-privilege

### Pricing models
- On-demand, reserved instances (Standard, Convertible, Scheduled), savings plans
- Spot instances: interruption handling, spot fleet, EC2 Spot price
- Data transfer costs: internet, inter-AZ, inter-region
- Free tier, consolidated billing, cost explorer, budgets, anomaly detection

### Migration
- AWS MGN (Application Migration Service), DMS (Database Migration Service)
- DataSync, Snowball/Snowmobile
- Server Migration Service (SMS)

---

## Study order recommendation

AWS is broad — focus on the services you use most (S3, RDS, Lambda, SQS, SNS, ECS, API Gateway) and the architecture principles that span them.

```
Week 1:  01-basic.md          + IAM policy writing practice
Week 2:  02-intermediate.md   + Load balancer / ASG design
Week 3:  03-senior.md         + ECS vs EKS trade-off analysis, Lambda deep dive
Week 4+: 04-question-bank.md daily drill + architecture diagram practice
```

**Next topic in skill order:** Docker/Kubernetes/Terraform.
