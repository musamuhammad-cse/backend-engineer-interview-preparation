# AWS Services — Question Bank

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Type:** 150+ rapid-fire Q&A, architecture scenarios, debugging scenarios, incident response, STAR templates

---

## Table of Contents

1. Rapid-Fire Q&A (150+ questions)
2. Architecture Design Scenarios
3. Debugging Scenarios
4. Incident Response (STAR Templates)
5. Cost Optimization Puzzles

---

## 1. Rapid-Fire Q&A

### IAM & Security

**Q: What's the difference between IAM user access keys and STS temporary credentials?**
A: Access keys are long-term (no expiry). STS credentials expire (15 min–12 h). Always prefer STS for applications.

**Q: What is a trust policy?**
A: The IAM policy on a role that defines WHO can assume the role (Principal) and under what conditions.

**Q: Can a Lambda function access S3 without an IAM role?**
A: No. Lambda needs an execution IAM role with S3 permissions.

**Q: What's the difference between a permission boundary and SCP?**
A: Permission boundary limits max permissions for IAM users/roles. SCP limits max permissions for accounts in an organization.

**Q: How do you enforce MFA for AWS API calls in a specific account?**
A: Use an IAM policy condition: `"Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}`

**Q: What is the default action when no policy matches?**
A: Implicit Deny.

**Q: What overrides an Allow — an explicit Deny or SCP?**
A: Both. Explicit Deny always wins. SCP limits what can be allowed but doesn't grant.

**Q: Can you use IAM roles across accounts?**
A: Yes. Account A creates a role with a trust policy that allows Account B's user/role to assume it.

**Q: What is IMDSv2?**
A: Instance Metadata Service v2 — requires a session token (PUT request) before accessing metadata. Prevents SSRF attacks.

**Q: How do you enforce IMDSv2 across all EC2 instances?**
A: Set `MetadataOptions.HttpTokens` to `required` in launch templates. Use AWS Config rules to audit.

### VPC

**Q: What is a security group?**
A: Stateful, instance-level firewall. Allow rules only.

**Q: What is a NACL?**
A: Stateless, subnet-level firewall. Allow and Deny rules, evaluated by rule number.

**Q: Can a security group reference another security group?**
A: Yes. Allows traffic from instances in the referenced SG.

**Q: What is a NAT Gateway?**
A: Managed service in a public subnet that allows private subnets to access the internet (not inbound).

**Q: Is NAT Gateway HA?**
A: No. It's in a single AZ. Deploy one per AZ for HA.

**Q: What is VPC peering?**
A: Connects two VPCs via private IP. Not transitive.

**Q: What is Transit Gateway?**
A: Hub-and-spoke network hub for connecting VPCs and on-prem networks.

**Q: What is a Gateway Endpoint?**
A: VPC endpoint for S3 and DynamoDB. Uses prefix lists in route tables. Free.

**Q: What is PrivateLink?**
A: Interface endpoint that exposes an AWS service or your own service via ENI in your VPC.

**Q: How do you connect on-prem to AWS?**
A: Site-to-Site VPN, Direct Connect, or Transit Gateway with VPN/DX attachments.

**Q: What are VPC Flow Logs?**
A: Captures metadata (IP, port, protocol, action) about traffic traversing VPC network interfaces.

**Q: Flow Logs show REJECT for your ALB's traffic. What's wrong?**
A: Security group or NACL is blocking the traffic.

### EC2

**Q: What are the EC2 purchase options?**
A: On-Demand, Reserved (Standard/Convertible), Spot, Savings Plans, Dedicated Hosts.

**Q: What instance family would you use for a memory-intensive database?**
A: r5, r6i, r7g (memory optimized).

**Q: What's the difference between gp3 and gp2 EBS volumes?**
A: gp3 has baseline 3,000 IOPS regardless of size. gp2 scales IOPS with size (3 IOPS/GB). gp3 is cheaper for most.

**Q: Can you modify an EBS volume type without downtime?**
A: Yes. EBS volumes can be modified (size, type, IOPS) without detaching. Some changes require reboot.

**Q: What is an EBS snapshot?**
A: Incremental backup of an EBS volume. Stored in S3 (not directly accessible).

**Q: How do you warm up an EBS volume restored from snapshot?**
A: Use `fio` or `dd` to read the entire volume.

**Q: What happens to data on instance store when the instance stops?**
A: Data is lost. Instance store is ephemeral.

**Q: What is a placement group?**
A: Strategy for controlling instance placement: Cluster (low latency), Spread (distinct hardware), Partition (isolated racks).

**Q: How do EC2 user data scripts work?**
A: Run as root on first boot. Logs at `/var/log/cloud-init-output.log`.

### S3

**Q: What's the maximum object size in S3?**
A: 5 TB.

**Q: What's S3's consistency model?**
A: Strong consistency (as of December 2020) for all operations.

**Q: What's the difference between S3 Standard and Standard-IA?**
A: Standard-IA is cheaper but has minimum 30-day storage charge and retrieval fee.

**Q: What storage class would you use for archival data that must be retrievable within 12 hours?**
A: Glacier Deep Archive (retrieval ~12 hours, lowest cost).

**Q: How do you prevent accidental S3 bucket deletion?**
A: Enable versioning (with MFA delete), enforce via IAM/SCP policies.

**Q: What's the difference between OAI and OAC for CloudFront-S3?**
A: OAI is older (only GetObject). OAC is newer (supports KMS, PUT/POST, DELETE operations).

**Q: How do you generate a pre-signed URL for private S3 content?**
A: Use `CreatePresignedRequest` (SDK) with an expiry time. The URL includes a signature.

**Q: Can a pre-signed URL be used by anyone?**
A: Yes, anyone with the URL can access the object until it expires.

**Q: What is S3 Object Lock?**
A: WORM protection. Governance mode (overridable) or Compliance mode (cannot be overridden).

**Q: Can you change a bucket's region after creation?**
A: No. You must create a new bucket in the target region and replicate/copy objects.

### ALB & ASG

**Q: What's the difference between ALB and NLB?**
A: ALB: Layer 7, HTTP/HTTPS/gRPC, path/host routing, stickiness, Lambda targets. NLB: Layer 4, TCP/UDP/TLS, static IP, ultra-low latency.

**Q: How does ALB authenticate users?**
A: Via OIDC, Cognito, or Social login. ALB passes JWT claims in HTTP headers.

**Q: What is connection draining (deregistration delay)?**
A: ALB stops new requests to a deregistering target but waits for in-flight requests to complete.

**Q: What health check types does ASG support?**
A: EC2 status checks, ELB health checks, custom health checks.

**Q: What is a lifecycle hook?**
A: Pauses instance launch/termination for custom actions (drain, cleanup, register).

**Q: What is a warm pool?**
A: Pre-initialized instances ready to be quickly transitioned into the ASG.

**Q: What is predictive scaling?**
A: ML-based scaling that learns traffic patterns and pre-scales.

**Q: How does target tracking scaling work?**
A: Keeps a metric at a target value (e.g., CPU at 60%). Adjusts desired count up or down.

### RDS

**Q: What's the difference between Multi-AZ and Read Replicas?**
A: Multi-AZ: synchronous standby for HA (not readable). Read Replicas: async copies for read scaling.

**Q: What happens during an RDS Multi-AZ failover?**
A: DNS CNAME is updated to standby (< 120 seconds). Connections must reconnect.

**Q: How long can you retain automated backups?**
A: 1–35 days.

**Q: What is RDS Proxy?**
A: Connection pooling service for RDS. Reduces DB connections, handles Lambda cold starts.

**Q: What is Aurora's storage architecture?**
A: 6 copies across 3 AZs. Shared cluster volume. Writer and readers access the same storage.

**Q: How is Aurora different from RDS MySQL?**
A: Aurora has distributed storage (128 TB max), faster failover, lower replication lag, and separate engine.

**Q: How do you enforce SSL for RDS connections?**
A: Set `rds.force_ssl=1` (PostgreSQL) or require SSL in the connection string.

**Q: What is Performance Insights?**
A: RDS monitoring tool for database load analysis, top SQL queries, and wait events.

### Route 53

**Q: What record type do you use for a zone apex pointing to an ALB?**
A: ALIAS record (CNAME doesn't work at zone apex).

**Q: What routing policy would you use for blue-green deployment?**
A: Weighted routing policy.

**Q: What routing policy would you use for global latency optimization?**
A: Latency-based routing.

**Q: What routing policy would you use for DR?**
A: Failover routing (active-passive).

**Q: How does Route 53 health check work?**
A: Route 53 sends requests to the endpoint at a configurable interval (30s or 10s). If failed threshold reached, endpoint marked unhealthy.

**Q: What is a private hosted zone?**
A: DNS zone that resolves within VPCs only (no internet exposure).

**Q: Can you use Route 53 for on-prem DNS resolution?**
A: Yes, via Route 53 Resolver (inbound/outbound endpoints + resolver rules).

### CloudFront

**Q: What is Lambda@Edge?**
A: Lambda functions that run at CloudFront edge locations. Can inspect/modify requests and responses.

**Q: What CloudFront events does Lambda@Edge support?**
A: Viewer request, origin request, origin response, viewer response.

**Q: What is the difference between a Cache Policy and an Origin Request Policy?**
A: Cache policy defines the cache key (what affects caching). Origin request policy defines what is forwarded to the origin (without affecting caching).

**Q: How do you serve private content through CloudFront?**
A: Signed URLs (single file) or Signed Cookies (session/multiple files). Restrict by path, IP, and expiry.

**Q: What is OAC?**
A: Origin Access Control — restricts S3 bucket access to CloudFront only. Replaces OAI.

**Q: Can CloudFront cache POST requests?**
A: No. CloudFront only caches GET, HEAD, and OPTIONS requests.

**Q: What is geo-restriction?**
A: Allow or block countries from accessing your CloudFront distribution.

### Lambda

**Q: What causes a Lambda cold start?**
A: A new execution environment is created: download code, initialize runtime, run any init code outside handler.

**Q: How long do Lambda cold starts take?**
A: Python/Node ~100–300ms. Java ~500–2,000ms. Go ~50–200ms.

**Q: How do you reduce Lambda cold starts?**
A: Provisioned concurrency, keep deployment package small, SnapStart (Java/NET).

**Q: What is the difference between reserved and provisioned concurrency?**
A: Reserved = max concurrency (also guarantees availability). Provisioned = pre-warmed environments (eliminates cold starts).

**Q: What is the maximum Lambda timeout?**
A: 15 minutes (900 seconds).

**Q: What is the maximum Lambda ephemeral storage?**
A: 10,240 MB (10 GB). Default 512 MB.

**Q: Can a Lambda function access RDS in a VPC?**
A: Yes, but it adds cold start latency (ENI creation). Use RDS Proxy for connection pooling.

**Q: What is Lambda SnapStart?**
A: Takes a snapshot of the initialized runtime. Restores from snapshot — reduces cold start for Java/NET.

**Q: What is Lambda response streaming?**
A: Streams responses to the caller instead of buffering. Reduces time-to-first-byte.

**Q: How does Lambda handle throttling?**
A: Returns `429 TooManyRequestsException`. For async invocations, auto-retries for 6 hours with backoff.

### SQS

**Q: What's the difference between standard and FIFO queues?**
A: Standard: at-least-once, best-effort ordering, unlimited TPS. FIFO: exactly-once, strict ordering, 300/3,000 TPS.

**Q: What is a DLQ?**
A: Dead-letter queue — messages that exceed `maxReceiveCount` are moved here.

**Q: What is visibility timeout?**
A: When a consumer receives a message, it becomes invisible to others for this duration.

**Q: What happens if a consumer doesn't delete a message in time?**
A: The message becomes visible again after the visibility timeout expires.

**Q: What is long polling?**
A: `ReceiveMessage` waits up to 20s for messages. Reduces cost and empty responses.

**Q: How do you handle messages larger than 256 KB?**
A: Use the SQS Extended Client (stores message body in S3, sends reference in SQS).

**Q: Can Lambda process SQS messages out of order?**
A: If using a standard queue, yes. For FIFO, use a single message group ID.

**Q: What is `ChangeMessageVisibility`?**
A: Extends the visibility timeout for an in-flight message — useful if processing takes longer than expected.

### SNS

**Q: What is the difference between SNS and SQS?**
A: SNS is push (pub/sub), SQS is pull (queues). SNS has no persistence, SQS stores messages.

**Q: What is the fan-out pattern?**
A: SNS topic publishes one message to multiple SQS queues (each consumer gets their own queue).

**Q: Does SNS retry failed deliveries?**
A: For HTTP/HTTPS — yes (exponential backoff). For SQS/Lambda — no (but SQS/Lambda handle retries).

**Q: What is message filtering?**
A: SNS subscription filter policy — only deliver messages matching the filter conditions.

**Q: Can SNS send to a FIFO queue?**
A: Yes, using a FIFO topic. Name must end in `.fifo`.

### Secrets Manager

**Q: What's the difference between Secrets Manager and Parameter Store?**
A: Secrets Manager: automatic rotation, more expensive ($0.40/secret/month). Parameter Store: free (standard), no rotation.

**Q: How does Secrets Manager rotate RDS credentials?**
A: Using a Lambda function that changes the DB password and updates the secret.

**Q: What's the max secret size in Secrets Manager?**
A: 65 KB.

**Q: How do you reference a secret in an ECS task definition?**
A: Use the `secrets` configuration with `valueFrom` pointing to the secret ARN.

### ECS

**Q: What's the difference between Fargate and EC2 launch type?**
A: Fargate: serverless, no instance management, pay per task. EC2: you manage instances, pay per instance.

**Q: What's the difference between task role and execution role?**
A: Task role is used by your application (e.g., S3 access). Execution role is used by ECS agent (pull images, logs).

**Q: How do you scale ECS services?**
A: Application Auto Scaling with target tracking (CPU, memory, ALB request count).

**Q: What is Service Connect?**
A: ECS service mesh — DNS-based service discovery, traffic encryption, circuit breaking.

**Q: What is a Capacity Provider?**
A: Connects ECS to an ASG — ECS manages the ASG based on task placement needs.

### EKS

**Q: How does IRSA work?**
A: EKS OIDC provider allows K8s service accounts to assume IAM roles. Pods get temporary STS credentials.

**Q: What is the difference between Cluster Auto Scaler and Karpenter?**
A: CAS adds nodes based on pending pods. Karpenter provisions right-sized nodes and consolidates.

**Q: How does VPC CNI assign IPs to pods?**
A: Each pod gets a VPC IP address. Max pods per node depends on ENI limits. Prefix delegation increases capacity.

**Q: What is Bottlerocket?**
A: Immutable Linux OS optimized for containers. Atomic updates, smaller footprint.

**Q: What is the AWS Load Balancer Controller?**
A: K8s controller that provisions ALB (Ingress) and NLB (LoadBalancer services).

### Multi-Region & Well-Architected

**Q: What's the difference between active-passive and active-active DR?**
A: Active-passive: standby region is idle, failover promotes it. Active-active: both regions serve traffic.

**Q: How do you sync data across regions for active-active?**
A: Aurora Global Database (~1s lag), DynamoDB Global Tables, or application-level conflict resolution.

**Q: What are the 6 pillars of the Well-Architected Framework?**
A: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability.

**Q: What is the most common Well-Architected failure?**
A: Single-AZ deployment (no HA).

### General

**Q: What is the shared responsibility model?**
A: AWS secures the cloud (physical, hardware, virtualization). You secure what's in the cloud (data, IAM, OS, network config).

**Q: What's the difference between horizontal and vertical scaling in AWS?**
A: Horizontal: add more instances (ASG, ECS tasks). Vertical: increase instance size (preferred for RDS/ElastiCache).

**Q: What's the difference between CloudWatch and CloudTrail?**
A: CloudWatch: performance monitoring (metrics, logs, alarms). CloudTrail: audit API calls (who did what when).

**Q: What is X-Ray?**
A: Distributed tracing service — traces requests across microservices, maps service topology.

**Q: What is AWS Config?**
A: Resource inventory, configuration history, and compliance rules (e.g., "S3 buckets must be encrypted").

**Q: What is GuardDuty?**
A: Threat detection service — analyzes CloudTrail, VPC Flow Logs, and DNS logs for malicious activity.

**Q: What is WAF?**
A: Web application firewall — protects against SQL injection, XSS, bot traffic, and IP reputation lists.

**Q: What is AWS Shield?**
A: DDoS protection. Shield Standard (free, basic). Shield Advanced ($3,000/month, enhanced detection + cost protection).

**Q: How do you implement Blue-Green deployment on ECS?**
A: CodeDeploy with ECS. Blue task set (current) and Green task set (new). Traffic shifted gradually.

**Q: How do you implement Blue-Green deployment on EKS?**
A: Use the AWS Load Balancer Controller with weighted target groups, or Flagger/Argo Rollouts.

**Q: What's the best practice for tagging resources?**
A: Tag with Environment, Project, Team, Cost Center. Enable cost allocation tags.

**Q: What is AWS Organizations?**
A: Manage multiple AWS accounts centrally. SCPs for guardrails, consolidated billing.

**Q: How do you restrict which regions an account can use?**
A: SCP in Organizations: `Deny ec2:*` unless `aws:RequestedRegion` is in allowed list.

**Q: What is an AWS Service Catalog?**
A: Pre-approved product portfolio (CloudFormation templates) that users can deploy in their accounts.

### Pricing

**Q: What's the cheapest compute option for a stateless, fault-tolerant batch job?**
A: Spot Instances (up to 90% discount). Use with ASG or ECS Fargate Spot.

**Q: What's the difference between Reserved Instances and Savings Plans?**
A: RIs: commitment to specific instance type in specific region. Savings Plans: $/hr commitment, flexible across compute.

**Q: What free tier services are available for 12 months?**
A: 750 hours EC2 t2.micro, 5 GB S3 Standard, 750 hours RDS db.t2.micro, 1 million Lambda requests.

**Q: How do you monitor and forecast AWS costs?**
A: AWS Cost Explorer, Budgets (with alerts), Cost Anomaly Detection, Cost Allocation Tags.

---

## 2. Architecture Design Scenarios

### Scenario 1: Multi-tenant inventory SaaS

**Context:** You're building a scalable version of your Laravel multi-tenant inventory system on AWS.

**Requirements:**
- 5,000 tenants, 10–100 users per tenant
- Each tenant's data isolated (row-level via `organization_id`)
- < 200ms p95 response time
- Zero-downtime deployments
- Global expansion planned

**Design an AWS architecture.**

Consider:
- Compute: ECS Fargate (auto scaling, no instance management)
- API: ALB with path-based routing to API service
- Database: RDS PostgreSQL with multi-tenant data (row-level isolation via `organization_id`)
  - Single DB with RDS Proxy for connection pooling
  - Read replicas for heavy read workloads
  - Consider migration to Aurora for larger scale
- Cache: ElastiCache Redis (per-tenant cache keys: `org:{id}:*`)
- Files: S3 + CloudFront (tenant-specific prefixes: `orgs/{id}/assets/`)
- Queue: SQS for async tasks (inventory sync, report generation)
- Search: OpenSearch Service (Elasticsearch) for product search
- CI/CD: GitHub Actions → ECR → ECS Blue/Green via CodeDeploy

Key concerns:
- RDS connection limits per tenant (RDS Proxy pools)
- Hot tenant isolation (monitor `organization_id` query patterns)
- Data export per tenant (S3 exports via SQS + Lambda)

---

### Scenario 2: Trading platform (20K+ DAU)

**Context:** Real-time trading data streaming to users.

**Requirements:**
- Real-time price feed: < 50ms end-to-end
- Order execution: < 100ms
- Trade history: durable, auditable
- 99.99% uptime (53 min downtime/year max)

**Design an AWS architecture.**

Consider:
- Real-time data: WebSocket via API Gateway WebSocket API
  - Lambda for connection management (`$connect`, `$disconnect`, `$default`)
  - DynamoDB for connection store (`connectionId → userId`)
- Market data: ElastiCache Redis (pub/sub channels per symbol)
  - Data providers write to Redis → Lambda pushes to API Gateway connections
- Order execution: ALB → ECS Fargate (compute-optimized c7g instances)
  - EFS for task migration (stateless tasks)
- Trade history: DynamoDB (GSI on `userId + timestamp`)
- Durable orders: SQS FIFO (per user ordering)
- Pricing: Aurora for reference data (instruments, user profiles)
- DR: Multi-AZ + cross-region read replicas (Aurora Global Database)

Key concerns:
- WebSocket connections scale (API Gateway handles 1M+ connections)
- Lambda pushes to connections (throttle at 10 connections/sec per API)
- Order FIFO per user (SQS FIFO with `userId` as message group ID)
- Redis pub/sub fan-out per symbol

---

### Scenario 3: Chronos distributed scheduler on AWS

**Context:** Your Raft-based distributed scheduler.

**Requirements:**
- Multi-AZ deployment
- Leader election (Raft)
- Scheduled jobs with retries
- Job history for audit

**Design an AWS architecture.**

Consider:
- Compute: ECS Fargate (3 nodes for Raft quorum)
  - Task placement: spread across 3 AZs
  - Service auto scaling: reactive (CPU) + scheduled (known job peaks)
- Raft state: EFS (shared file system for Raft logs) or DynamoDB (Pessimistic locking)
  - EFS is simpler for Raft log files (append-only)
  - DynamoDB is serverless but Raft is file-based
- Job triggers: EventBridge (Scheduler) → SQS → Chronos workers
  - Schedule: `cron` or `rate` expressions via EventBridge
  - Durability: SQS (DLQ on failure)
- Job execution: ECS RunTask (one-off tasks)
  - Task definition matches job requirements (configurable CPU/memory)
  - Task role per job (least-privilege)
- Monitoring: CloudWatch metrics (jobs completed, failed, duration, queue depth) + X-Ray tracing
- Storage: S3 for job artifacts, CloudFront for viewing results

Key concerns:
- Raft leader election across AZs (network latency within AZ)
- Fargate task IP changes on restart (use Service Connect for stable DNS)
- Job idempotency (SQS at-least-once delivery)
- Rate limiting job execution (SQS + Lambda concurrency limit)

---

### Scenario 4: Serverless file processing pipeline

**Context:** Users upload CSV files via web app. System validates, transforms, and loads into RDS.

**Requirements:**
- Upload files up to 500 MB
- Validate schema and data types
- Transform (clean, normalize, enrich)
- Load into PostgreSQL tables
- Notify user on completion

**Design an AWS architecture.**

Consider:
- Upload: S3 (pre-signed URL from API) + CloudFront
- Trigger: S3 event notification → SQS queue
- Processing: Lambda (validate) → S3 (processed) → Lambda (transform) → S3 (final) → Lambda (load into RDS)
- Large file handling: For files > 256 KB SQS limit, S3 → SQS (object reference) → ECS Fargate (RunTask) for processing
- Notification: SNS → email/SMS, SES for rich email
- Error handling: DLQ per processing step + CloudWatch alarms
- State machine: Step Functions for orchestration (validates → transform → load → notify)
  - Each step retries with backoff
  - DLQ on failure after max retries

Key concerns:
- Lambda 15-min timeout vs 500 MB files (use ECS Fargate for heavy processing)
- SQS message size (store S3 reference, not file content)
- RDS connections from Lambda (use RDS Proxy)
- Cost: Lambda for validation, Fargate Spot for transformation

---

### Scenario 5: E-commerce order processing

**Context:** High-volume order system with inventory reservation, payment, shipping, and notifications.

**Requirements:**
- 10,000 orders/minute at peak
- Exactly-once order processing
- < 500ms order confirmation
- Graceful degradation on downstream failures

**Design an AWS architecture.**

Consider:
- API: ALB + ECS Fargate (order submission, auth)
- Order queue: SQS FIFO (per customer message group for ordering)
- Processing pipeline:
  1. Order service → SQS FIFO (order-created)
  2. Inventory service (reserve stock, SQS standard)
  3. Payment service (stripe/Plaid via Lambda)
  4. Shipping service (generate label, schedule pickup)
- Data: DynamoDB for orders (single-table design, `orderId` PK, GSI on `customerId + created`)
- Notifications: SNS (fan-out to email/SMS/push)
- Orchestration: Step Functions
  - Parallel: inventory + payment
  - Sequential: after both, shipping
  - Compensation: if payment fails, release inventory
- Dashboard: QuickSight or OpenSearch

Key concerns:
- SQS FIFO limit (3,000 TPS with batching) — at 10,000 orders/min (166/sec), a single FIFO queue works
- Step Function execution history limit (25,000 events — design concise steps)
- DynamoDB throttling (RCU/WCU planning)
- Payment provider retries (idempotency key)

---

## 3. Debugging Scenarios

### Scenario 1: ECS tasks failing to start

**Symptoms:** New ECS tasks transition from PROVISIONING → PENDING → STOPPED immediately.

**Debug steps:**
1. Check `StoppedReason` in ECS console/task definition
2. Check execution role — can it `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:BatchCheckLayerAvailability`?
3. Check VPC config — private subnets need NAT Gateway or VPC Endpoints (ECR, S3, CloudWatch)
4. Check security group — does the SG allow outbound HTTPS? (ECR pull)
5. Check task definition — correct image URI? account/region correct?
6. Check CloudWatch Logs group — does it exist? execution role has `logs:CreateLogStream`, `logs:PutLogEvents`?
7. Check ECR permissions — is the image in the same account/region?

**Root cause:** Missing ECR VPC Endpoint (or NAT Gateway) for private subnets. Tasks can't pull the image.

### Scenario 2: Lambda timeout with VPC configuration

**Symptoms:** Lambda timeout after 3 seconds (default timeout). Works without VPC config.

**Debug steps:**
1. Check Lambda timeout setting (increase if needed)
2. Check VPC config — Lambda in private subnet needs NAT Gateway for internet access
3. Check if the downstream resource (DB, API) is in the same VPC — use private DNS instead of public
4. Check if VPC Endpoints exist for any AWS services Lambda calls (S3, DynamoDB, SQS)
5. Check security group — allows outbound traffic?
6. Check NACL — allows ephemeral ports?
7. Check RDS Proxy — are connections being exhausted?

**Root cause:** Lambda in VPC cannot reach the internet (no NAT Gateway). Or it can reach the internet via NAT but the target service DNS resolves to a public IP that NAT can't reach from private subnet.

### Scenario 3: S3 → Lambda event not triggering

**Symptoms:** Files uploaded to S3 bucket, but Lambda doesn't trigger.

**Debug steps:**
1. Check S3 event notification configuration (bucket → events → destination)
2. Check S3 bucket policy — does it allow S3 to invoke Lambda?
3. Check Lambda resource-based policy — does it allow S3 to invoke?
4. Check Lambda trigger configuration in Lambda console (does it show S3 as trigger?)
5. Check if versioning is on and you uploaded a new version — event fires per version
6. Check if you're in a region that supports S3 event notifications (all do)
7. Check CloudWatch Logs for Lambda — any errors?
8. Test manually: upload a test file, check S3 event notification metrics in CloudWatch

**Root cause:** S3 event notification configuration missing or incorrect resource policy on Lambda.

### Scenario 4: ALB 503 errors

**Symptoms:** 503 Service Unavailable errors from ALB.

**Debug steps:**
1. Check target group health — are targets healthy?
2. Check health check path — does it return 200?
3. Check security group — does ALB SG allow traffic to target SG on health check port?
4. Check target group — correct port, correct protocol?
5. Check ASG — is desired count > 0? Are instances in service?
6. Check application logs — is the application running? Memory/CPU exhaustion?
7. Check ALB metrics — `HealthyHostCount`, `UnhealthyHostCount`, `RequestCount`, `TargetResponseTime`
8. Check if the application is overloaded — add more targets, scale out

**Root cause:** Target instances unhealthy (app crash, out of memory, port not listening). Or security group blocking health checks.

### Scenario 5: RDS CPU at 100%

**Symptoms:** Application slow, RDS CloudWatch CPUUtilization at 100%.

**Debug steps:**
1. Enable Performance Insights — find the top SQL queries by DB load
2. Check slow query log — compare with top queries
3. Check connections count — `DBConnections` metric
4. Check if there's a missing index (EXPLAIN ANALYZE on top queries)
5. Check if there's a sudden traffic spike (correlate with application metrics)
6. Check if a new deployment introduced an inefficient query
7. Check lock waits — `BlockedTransactions` metric
8. Consider scaling up (read replicas, larger instance)

**Root cause:** Missing index on a frequently scanned table. New JOIN query hit the table without an index.

### Scenario 6: SQS messages not being processed

**Symptoms:** Messages in SQS queue but never consumed. Queue depth increasing.

**Debug steps:**
1. Check Lambda event source mapping — is it enabled? Correct queue ARN?
2. Check Lambda execution role — has `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`?
3. Check DLQ — are messages being sent to DLQ instead?
4. Check Lambda function — any errors? (CloudWatch Logs)
5. Check batch size — too large? Lambda 15-min timeout with 10 messages?
6. Check Lambda reserved concurrency — is it 0? (blocks all invocations)
7. Check Lambda function timeout — long enough?
8. Check SQS visibility timeout — longer than Lambda processing time?

**Root cause:** Lambda concurrency set to 0 (accidentally). Or the Lambda function is crashing and messages return to queue after visibility timeout — retried infinitely.

### Scenario 7: Route 53 not resolving

**Symptoms:** Users can't reach the application. DNS returns `NXDOMAIN` or wrong IP.

**Debug steps:**
1. Check Route 53 hosted zone — does it have the correct record?
2. Check domain registrar — NS records point to Route 53?
3. Check TTL — recent changes propagate (use `dig` to check)
4. Check health checks — if failover routing, is primary healthy?
5. Check ALB/CloudFront — is the target healthy?
6. Check S3 static website — is bucket configured for hosting?
7. Check private hosted zone — is the VPC associated?
8. Check resolver rules — if using hybrid DNS, are rules correct?

**Root cause:** TTL not expired for old DNS records (wait and check). Or health check marked target unhealthy in failover routing.

---

## 4. Incident Response (STAR Templates)

### Template 1: Database migration with zero downtime

**Situation:** Your multi-tenant SaaS needed to add a new column to a large PostgreSQL table (10M+ rows) with zero downtime.

**Task:** Migrate schema without locking the table (long-running ALTER TABLE blocks writes).

**Action:**
```
1. Use pgroll / pg-osc (or manual approach):
   a. Create new column as nullable (ADD COLUMN IF NOT EXISTS — near-instant)
   b. Backfill data in batches (UPDATE ... WHERE pk BETWEEN x AND y) — 1,000 rows at a time
   c. Run backfill during low traffic (scheduled via ECS RunTask)
   d. Make column NOT NULL after backfill completes
2. Monitor: CloudWatch RDS connections, lock waits, replication lag
3. Rollback: Drop the column (instant)
4. Tested on read replica first — validated backfill speed
```

**Result:** Zero downtime migration. 15M+ records backfilled in 4 hours during off-peak. No locks, no blocked queries, no rollback needed.

---

### Template 2: 88% query reduction via caching

**Situation:** Application latency spiking under load. RDS CPU at 90%+ with a single query pattern consuming 60% of DB time.

**Task:** Reduce database load without application rewrite.

**Action:**
```
1. Identified the hot query: SELECT ... WHERE organization_id = ? AND status = 'active'
   — dashboard page, refreshed every 30s by 5,000+ tenants
2. Designed cache strategy:
   a. Cache key: `dashboard:{orgId}:summary`
   b. Cache TTL: 60 seconds (matching dashboard refresh + tolerance)
   c. Cache-aside pattern: app checks cache → miss reads DB → sets cache
   d. Invalidated on relevant data changes (event → SQS → cache invalidation)
3. Deployed ElastiCache Redis (one r6g.large, 2 AZ for HA)
4. Updated Laravel models to use cache decorator
5. Monitored: cache hit ratio, DB CPU, app latency
```

**Result:** 88% reduction in DB queries. RDS CPU dropped from 90% to 15%. P95 latency from 850ms to 45ms. Cache hit rate: 94%.

---

### Template 3: Multi-AZ failover during AZ outage

**Situation:** An AWS AZ experienced a power outage during peak business hours. Your application was multi-AZ but not fully tested for failover.

**Task:** Ensure business continuity with minimal impact.

**Action:**
```
1. Immediate actions:
   a. ALB automatically routed traffic to remaining AZs (healthy instances)
   b. RDS Multi-AZ failed over automatically to standby in healthy AZ (~90s)
   c. Confirmed DNS propagation via Route 53
2. Monitoring:
   a. Verified CloudWatch alarms for unhealthy hosts
   b. Checked ASG — scaled up in remaining AZs to compensate
   c. Verified SQS queues no impact (multi-AZ by design)
3. Recovery:
   a. After AZ restored, ASG rebalanced instances
   b. RDS Multi-AZ reestablished replication
   c. No data loss (RDS synchronous replication, auto backups)
4. Post-mortem:
   a. Reviewed and updated runbook
   b. Added Chaos Engineering tests (AZ failure simulation)
   c. Confirmed all new services deployed across 3 AZs
```

**Result:** 2 minutes of elevated latency during RDS failover. No data loss. Zero downtime for writes. Customer impact: minimal (dashboard auto-refresh caught up).

---

### Template 4: SQS consumer backlog recovery

**Situation:** A downstream API (shipping provider) went down for 2 hours. SQS queue depth grew to 500,000 messages.

**Task:** Process backlog without overwhelming the downstream API once it recovers.

**Action:**
```
1. Immediate:
   a. API provider notified us of outage → confirmed issue, not our code
   b. Kept messages in queue (did not purge — each order is important)
   c. Reduced Lambda batch size from 10 to 5 (lower impact on recovery)
2. Recovery:
   a. API came back — queue processing resumed automatically
   b. Messages were processing successfully (idempotent handler)
   c. DLQ received 0 messages (no poison pills)
3. Optimization:
   a. Increased Lambda concurrency (reserved) from 10 to 50
   b. Decreased visibility timeout to match faster processing
   c. Monitored queue depth → dropped from 500K to 0 in ~45 minutes
4. Prevention:
   a. Added circuit breaker: if API returns 5xx > 50%, stop processing
   b. Added alarm: QueueDepth > 10,000 triggers SNS notification
```

**Result:** All 500,000 messages processed within 45 minutes of API recovery. Zero data loss. No duplicate charges (idempotency keys).

---

## 5. Cost Optimization Puzzles

### Puzzle 1: Weekly batch processing

**Setup:** You have a batch job that processes 100 GB of data every Sunday from 2 AM–6 AM. Currently running on 4 `r5.xlarge` On-Demand instances.

**Problem:** Reduce cost.

**Solution:**
- Use Spot Instances for the batch job (4 `r5.xlarge` Spot = ~70% cheaper)
- Or use ECS Fargate Spot tasks (serverless, pay per second)
- Schedule EC2 instances via Instance Scheduler (stop outside batch window)
- Use S3 lifecycle policy for input/output data (Intelligent-Tiering)
- Right-size: test if `r5.large` (½ the cost) works with similar performance

**Savings:** 70–90% depending on spot availability.

### Puzzle 2: Development environment cost

**Setup:** Dev team has 5 EC2 instances running 24/7. Only used 8 AM–6 PM weekdays.

**Problem:** 70% of compute cost wasted.

**Solution:**
- Instance Scheduler (AWS Solution): stop at 6 PM, start at 8 AM
- Or use Lambda + EventBridge to start/stop instances on schedule
- Consider switching to `t3.large` (burstable) for dev workloads
- Or migrate dev to ECS Fargate (pay only when running)
- Use CloudWatch alarm to stop unused instances (no CPU for 2 hours → stop)

**Savings:** ~65% reduction.

### Puzzle 3: EBS snapshot storage

**Setup:** 50 EBS volumes (100 GB each) with daily snapshots. 35-day retention. Storage bill is $800/month for snapshots.

**Problem:** Reduce snapshot storage cost.

**Solution:**
- Snapshots are incremental — reviewing retention policy
- Keep daily for 7 days, weekly for 4 weeks (not 35 daily)
- Delete snapshots of terminated instances
- Use S3 Lifecycle transition to S3 Standard-IA for older snapshots (not directly — EBS snapshots are managed)
- Use AWS Backup (centralized backup management, can have cheaper tiered retention)
- Clean orphaned snapshots (no associated AMI or volume)

**Savings:** 40–50% reduction.

### Puzzle 4: S3 data transfer expense

**Setup:** Your application serves 10 TB/month of images from S3 to users globally. S3 data transfer out is $900/month.

**Problem:** Reduce data transfer cost.

**Solution:**
- Add CloudFront in front of S3 — CloudFront egress is ~$0.085/GB (vs S3 $0.09/GB)
- CloudFront also caches at edge locations — reduces S3 requests
- If content is cacheable, cache hit rate of 80%+ reduces both S3 GET requests and egress
- Use S3 Transfer Acceleration if data is uploaded from far regions (faster, not cheaper for egress)
- Compress images (WebP, AVIF) to reduce size by 30–50%
- Use signed URLs to prevent hotlinking

**Savings:** 15–50% depending on cache hit rate and compression.

### Puzzle 5: Lambda over-provisioned memory

**Setup:** A Lambda function processes CSV files. It's configured with 3,008 MB memory. Max memory used is 200 MB. You get 1M invocations/month.

**Problem:** Reduce cost.

**Solution:**
- Reduce memory to 256 MB (Lambda cost scales linearly with memory × duration)
- If duration stays the same at 256 MB, cost is ~8.5% of original (256/3008)
- However, CPU scales with memory — duration may increase slightly
- Test at 512 MB or 1,024 MB — likely sweet spot
- Also check: provisioned concurrency (not needed), ephemeral storage (512 MB default is fine)

**Savings:** 70–80% depending on final memory setting.

### Puzzle 6: Idle NAT Gateway

**Setup:** You have a NAT Gateway in each AZ for a pre-production environment that runs 24/7. NAT Gateway costs $32/month each × 3 = $96/month.

**Problem:** Reduce cost.

**Solution:**
- Replace NAT Gateways with a NAT Instance (t3.nano ~$5/month) for non-production
- Or use VPC Endpoints for AWS services (S3, ECR, DynamoDB) to eliminate internet need
- Schedule: stop NAT instance during non-business hours
- Pre-production may not need 3 AZs — use 1 NAT Gateway
- Consider using a single NAT Gateway for shared pre-prod VPC

**Savings:** $84–91/month (80–95% reduction).

---

> **Next topic in skill order:** Docker/Kubernetes/Terraform.
