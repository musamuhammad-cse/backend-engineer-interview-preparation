# AWS Services — Intermediate Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Basic tier (regions/AZs, IAM, VPC, EC2, S3)  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Elastic Load Balancing (ALB vs NLB vs CLB)
2. Auto Scaling
3. RDS (Relational Database Service)
4. Route 53
5. CloudFront
6. CloudWatch
7. Cost Management
8. Q&A

---

## 1. Elastic Load Balancing (ALB vs NLB vs CLB)

### Load balancer types

| Feature | ALB (Application) | NLB (Network) | CLB (Classic — legacy) |
|---------|-------------------|---------------|------------------------|
| OSI layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 4/7 |
| Protocols | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS | HTTP, HTTPS, TCP, SSL |
| Target type | IP, instance, Lambda, ALB | IP, instance, ALB | EC2 instance |
| Fixed IP | No (use NLB + ALB for fixed IP + Layer 7) | Yes (static IP per AZ) | No |
| Idle timeout | 60s (configurable 1–4,000s) | 350s (not configurable) | 60s (configurable) |
| Path/host-based routing | Yes | No | No |
| Sticky sessions | Yes (cookie-based) | No (source IP) | Yes (cookie-based) |
| HTTP/2 | Yes | N/A | No |
| WebSocket | Yes | N/A | No |
| gRPC | Yes (HTTP/2) | N/A | No |
| IP whitelisting | Security group | NACL (NLB has no SG) | Security group |

> **Trap:** NLB does not have security groups. You must rely on NACLs at the subnet level for network filtering. This is a common interview gotcha.

### ALB components

**Listener** — checks for connection requests on a port (e.g., 443 with HTTPS protocol)  
**Listener rules** — conditions (host header, path pattern, HTTP header, query string) + actions (forward, redirect, fixed response)  
**Target group** — routing to targets (EC2 instances, IP addresses, Lambda functions, private IPs)  
**Health checks** — HTTP/HTTPS path (e.g., `/health`), interval, threshold, timeout

```yaml
# Terraform — ALB with path-based routing
resource "aws_lb" "api" {
  name               = "api-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_target_group" "api" {
  name        = "api-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = aws_vpc.main.id
  health_check {
    path                = "/health"
    interval            = 30
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_lb_target_group" "admin" {
  name        = "admin-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = aws_vpc.main.id
}

resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
  condition {
    path_pattern { values = ["/api/*", "/v1/*"] }
  }
}

resource "aws_lb_listener_rule" "admin" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 200
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.admin.arn
  }
  condition {
    host_header { values = ["admin.example.com"] }
  }
}
```

> **Trap:** ALB can route directly to Lambda functions. No need for an intermediary EC2 instance. The Lambda must return a JSON-compatible response that ALB translates to HTTP.

### ALB stickiness

- **Load balancer generated cookie** — ALB generates `AWSALB` cookie
- **Application-based cookie** — your app sets a custom cookie, ALB reads it
- Duration: 1 second to 7 days

Stickiness is useful for session-affine workloads but breaks the "stateless" ideal. Prefer storing session state in Redis/ElastiCache.

> **Trap:** Sticky sessions can cause uneven load distribution. If one instance becomes hot, new sessions are still routed to it. Use sparingly.

### ALB authentication

ALB can authenticate users before forwarding to the target:

- **OIDC** — Google, Auth0, Okta, etc.
- **Amazon Cognito** — user pools, identity pools
- **Social login** — via Cognito

This offloads auth from your application. The ALB passes user claims in HTTP headers (`x-amzn-oidc-accesstoken`, `x-amzn-oidc-identity`, `x-amzn-oidc-claims`).

> **Trap:** The `x-amzn-oidc-claims` header is base64-encoded JWT. Your app must decode it to get user info.

### NLB use cases

- **Static IP** — when clients need to whitelist specific IPs
- **TCP/UDP workloads** — gaming, streaming, IoT
- **TLS termination at NLB** — for end-to-end encryption (NLB terminates TLS, forwards to targets)
- **PrivateLink** — NLB is the service provider endpoint for PrivateLink

### Connection draining / Deregistration delay

- ALB: `deregistration_delay.timeout_seconds` (default 300s)
- NLB: `connection_termination` (default false)

When a target is deregistering (e.g., during deployment), the LB stops sending new requests but waits for in-flight requests to complete within the timeout.

---

## 2. Auto Scaling

### Components

| Component | Description |
|-----------|-------------|
| **Launch template** | Instance config: AMI, instance type, key pair, SG, user data, IMDSv2 |
| **Auto Scaling Group (ASG)** | Scopes instances: VPC/subnets, min/max/desired, health checks, scaling policies |
| **Scaling policy** | Defines WHEN and HOW to scale |

### Launch template vs Launch configuration

| Feature | Launch Template | Launch Configuration |
|---------|----------------|---------------------|
| Versioning | Yes | No (create new for each change) |
| Multiple instance types | Yes (mix) | No |
| T2/T3 unlimited | Yes | No |
| IMDSv2 enforcement | Yes | No |
| Status | Current | Legacy |

> **Trap:** Always use launch templates. Launch configurations are deprecated.

### ASG health checks

By default, ASG only checks EC2 status (`status check failed = unhealthy`). You can configure:

- **ELB health check** — ASG uses ALB/NLB health check results to determine instance health
- **Custom health check** — external system reports health via AWS SDK

```yaml
# Terraform — ASG with ALB health check
resource "aws_autoscaling_group" "api" {
  name               = "api-asg"
  launch_template {
    id      = aws_launch_template.api.id
    version = "$Latest"
  }
  min_size         = 2
  max_size         = 10
  desired_capacity = 2
  vpc_zone_identifier = aws_subnet.private[*].id
  health_check_type         = "ELB"
  health_check_grace_period = 300
  target_group_arns         = [aws_lb_target_group.api.arn]
}
```

### Scaling policies

| Type | Description | Best for |
|------|-------------|----------|
| **Target tracking** | Keep metric at target value (e.g., CPU at 60%) | Steady, predictable workloads |
| **Step scaling** | Add/remove instances based on metric size | Spiky workloads, more control |
| **Simple scaling** | Single adjustment based on alarm | Legacy, avoid |
| **Scheduled scaling** | Scale at specific times (e.g., 9 AM Mon–Fri) | Known traffic patterns |
| **Predictive scaling** | ML-based, learns traffic patterns and pre-scales | Cyclical workloads |

**Target tracking example:**
```yaml
resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.api.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}
```

> **Trap:** Target tracking has a cooldown period (default 300s) during which scaling actions triggered by the same policy are suppressed. If your app has bursty traffic, you may need step scaling + shorter cooldown.

### Lifecycle hooks

Pause instance launch or termination so you can run custom actions:

- **Launch** — e.g., register with CMDB, validate config
- **Terminate** — e.g., drain connections, move state, in-flight request handling

```yaml
resource "aws_autoscaling_lifecycle_hook" "terminate" {
  name                   = "graceful-shutdown"
  autoscaling_group_name = aws_autoscaling_group.api.name
  lifecycle_transition   = "autoscaling:EC2_INSTANCE_TERMINATING"
  heartbeat_timeout      = 300
  default_result         = "CONTINUE"
}
```

Lambda processes the SNS/SQS notification from the lifecycle hook.

### Warm pools

Pre-initialized instances that are ready to be quickly transitioned into service:
- Reduces scale-up time (no waiting for boot/install)
- Instances can be in states: Stopped, Running (Hibernate), Running (No health check)

> **Trap:** Warm pools cost money for idle instances. Worth it only if you need sub-30-second scale-up.

### Instance refresh

Rolling replacement of instances in an ASG:
- Similar to deployment — replaces instances gradually
- Supports min healthy percentage and warmup time
- Can be used with launch template updates

---

## 3. RDS (Relational Database Service)

### Supported engines

| Engine | Managed by AWS | MySQL compatible | PostgreSQL compatible |
|--------|---------------|-----------------|----------------------|
| MySQL | Yes | Yes | No |
| PostgreSQL | Yes | No | Yes |
| MariaDB | Yes | Yes | No |
| Oracle | Yes | No | No |
| SQL Server | Yes | No | No |
| Amazon Aurora | Yes | Yes (MySQL) | Yes (PostgreSQL) |
| Amazon RDS on VMware | Yes | No | No |

> **Trap:** "Amazon Aurora" is not the same as "RDS MySQL." Aurora is a separate engine with different architecture (distributed storage, 6 copies across 3 AZs), though it's MySQL/PostgreSQL-compatible.

### Multi-AZ vs Read Replicas

| Feature | Multi-AZ | Read Replicas |
|---------|----------|---------------|
| Purpose | High availability (failover) | Read scaling |
| Replication | Synchronous | Asynchronous |
| Number of copies | 1 standby (in different AZ) | Up to 15 (MySQL), 15 (PG), 5 (Aurora) |
| Failover | Automatic (60–120s DNS update) | Manual (promote to standalone) |
| Cross-region | No | Yes |
| Cost | 2x (primary + standby) | 1x per replica (plus storage/IO) |
| Can be used for reads? | No (standby not directly readable) | Yes |

**Multi-AZ failover process:**
1. Primary becomes unavailable (AZ outage, instance failure, network issue)
2. AWS detects the failure (health check)
3. DNS CNAME is updated to point to standby
4. Standby is promoted to primary
5. Application resumes with 60–120s DNS propagation delay

> **Trap:** RDS Multi-AZ is NOT for read scaling — you cannot read from the standby. Use Read Replicas for read traffic. Aurora handles this differently (read from any of 6 storage copies).

### Automated backups vs Manual snapshots

| Feature | Automated Backup | Manual Snapshot |
|---------|-----------------|-----------------|
| Trigger | Automatic (backup window) | Manual |
| Retention | 1–35 days (user configurable) | Indefinite (no expiry) |
| PITR | Yes (within retention period) | Yes (can restore from snapshot) |
| Deletion | Deleted with DB instance | Persists after DB instance deletion |
| Cross-region copy | No | Yes |
| Cost | Included in instance cost | Charged for snapshot storage |

### Parameter groups

- RDS engine configuration (e.g., `max_connections`, `innodb_buffer_pool_size`, `timezone`)
- Default parameter group with RDS-optimized defaults
- Custom parameter group for your tuning
- Apply changes immediately or during maintenance window
- Some parameters require a reboot (`static`) — others apply immediately (`dynamic`)

> **Trap:** `max_connections` in RDS is limited by instance memory. The formula for MySQL/RDS: `{DBInstanceClassMemory/12582880}`. A `db.r5.large` (16 GB RAM) gets about 1,349 connections by default. This is a common bottleneck.

### RDS Proxy

- Connection pooling for RDS (reduces connections to DB)
- Handles Lambda cold starts (many concurrent connections are pooled)
- IAM authentication (no password in Lambda env)
- Auto-scaling, failover-aware

```yaml
resource "aws_rds_proxy" "main" {
  name                   = "rds-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_subnet_ids         = aws_subnet.private[*].id
  auth {
    auth_scheme = "SECRETS"
    secret_arn  = aws_secretsmanager_secret.db.arn
  }
}

resource "aws_rds_proxy_target_group" "main" {
  db_proxy_name = aws_rds_proxy.main.name
  connection_pool_config {
    max_connections_percent      = 100
    max_idle_connections_percent = 50
  }
}
```

> **Trap:** RDS Proxy costs ~$15/month + data processing. Use it if you have many short-lived connections (Lambda, serverless) or connection management issues.

### Aurora

**Key differences from standard RDS:**

| Feature | Standard RDS | Aurora |
|---------|-------------|--------|
| Storage | EBS (gp3/io2) | Distributed (6 copies across 3 AZs) |
| Max storage | 16 TB (gp3) | 128 TB |
| Failover time | 60–120s | ~30s |
| Read replicas | Up to 15 | Up to 15 Aurora Replicas |
| Replication lag | Might be seconds | Low (~10ms) |
| Backup | Automated up to 35 days | Automated up to 35 days, no performance impact |
| Storage cost | Per GB provisioned | Per GB consumed (IO-based) |

Aurora uses a **cluster volume** — a shared, distributed storage layer:
- 6 copies across 3 AZs (2 copies per AZ)
- Writer node in one AZ
- Reader nodes can read from same storage (no replication lag)

### RDS Security

- **Encryption at rest:** AES-256 via KMS. Encrypts data files, backups, snapshots, replicas.
- **Encryption in transit:** SSL/TLS (enforce with `rds.force_ssl=1` for PostgreSQL)
- **Network isolation:** Private subnet, security groups
- **IAM authentication:** Use IAM tokens instead of passwords (IAM DB Auth)
- **Secrets Manager:** Rotate RDS credentials automatically

---

## 4. Route 53

### DNS concepts (AWS-specific)

Route 53 is AWS's managed DNS service. It handles:
- Domain registration (buy domains)
- DNS resolution (translate names to IPs)
- Health checking (monitor endpoints)

### Record types

| Type | Value | Use case |
|------|-------|----------|
| A | IPv4 address | `example.com → 1.2.3.4` |
| AAAA | IPv6 address | `example.com → 2001:db8::1` |
| CNAME | Canonical name | `www.example.com → example.com` (cannot be at zone apex) |
| ALIAS | AWS resource | `example.com → ALB DNS name` (CAN be at zone apex) |
| MX | Mail servers | Routing email |
| TXT | Text (up to 4,096 bytes) | SPF, DKIM, domain verification |
| NS | Name servers | Delegation |

> **Trap:** CNAME cannot be used at the zone apex (`example.com` without `www`). Use ALIAS record for zone apex pointing to AWS resources (ALB, CloudFront, S3).

### Routing policies

| Policy | Description | Use case |
|--------|-------------|----------|
| **Simple** | Single record, route to one value | Basic, no HA needed |
| **Weighted** | Split traffic by weight (0–255) | Canary deployments, A/B testing |
| **Latency-based** | Route to lowest latency region | Global user base |
| **Failover** | Active-passive with health checks | DR setup |
| **Geolocation** | Route based on user location (country/continent) | Content localization, compliance |
| **Geoproximity** | Route based on location + bias (shift traffic to one region) | Traffic shifting |
| **Multi-value answer** | Return multiple healthy records (up to 8) | Simple HA without load balancer |

**Weighted routing for canary:**
```yaml
resource "aws_route53_record" "canary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"
  set_identifier = "canary"
  weighted_routing_policy {
    weight = 5
  }
  alias {
    name                   = aws_lb.canary.dns_name
    zone_id                = aws_lb.canary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "stable" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"
  set_identifier = "stable"
  weighted_routing_policy {
    weight = 95
  }
  alias {
    name                   = aws_lb.stable.dns_name
    zone_id                = aws_lb.stable.zone_id
    evaluate_target_health = true
  }
}
```

### Health checks

Route 53 health checks monitor endpoint health and route DNS only to healthy endpoints:

- **Endpoint health check** — HTTP/HTTPS/TCP to a specific IP/domain
- **Calculated health check** — combine multiple health checks (AND/OR, pass/fail threshold)
- **CloudWatch alarm health check** — route based on CloudWatch alarm state

Health check interval: 30s (standard) or 10s (fast). Standard costs are per health check; fast costs more.

> **Trap:** Route 53 health checks originate from multiple global IP ranges. Your firewall/SG must ALLOW these ranges for the health check to work. The IGW/NAT must also allow inbound.

### Private hosted zones

- DNS for internal VPC resources (e.g., `db.internal.example.com` → RDS private IP)
- Associated with one or more VPCs
- No internet exposure

**Hybrid DNS:** Route 53 Resolver can resolve on-premises DNS and AWS DNS bi-directionally:
- Inbound endpoints — on-prem queries AWS DNS
- Outbound endpoints — AWS queries on-prem DNS
- Resolver rules — route specific domains to specific DNS resolvers

### DNS failover architecture

```
CloudFront / Route 53
       │
       ├── us-east-1 (Primary)
       │   ├── ALB
       │   ├── ASG (EC2)
       │   └── RDS Multi-AZ
       │
       └── us-west-2 (DR/Standby)
           ├── ALB
           ├── ASG (EC2)
           └── RDS Read Replica (promote on failover)
```

Route 53 failover policy routes to primary. If health checks fail, DNS resolves to secondary (standby).

---

## 5. CloudFront

### Distribution types

- **Web** — HTTP/HTTPS content (websites, APIs, streaming via HLS/DASH)
- **RTMP** — Legacy Adobe Flash (deprecated)

### Origins

| Origin type | Use case |
|-------------|----------|
| S3 bucket | Static assets, file downloads, images |
| ALB / EC2 | Dynamic content, APIs |
| Custom origin (HTTP server) | Any existing HTTP server on-prem or in cloud |
| Lambda@Edge | Serverless content transformation |

### Cache behaviors

A distribution can have multiple cache behaviors, each matching URL path patterns:

- **Path pattern**: `/images/*`, `/api/*`, default (*)
- **Origin**: which origin this path forwards to
- **Viewer protocol policy**: HTTP-only, redirect-to-HTTPS, HTTPS-only
- **Allowed HTTP methods**: GET/HEAD (for static), GET/HEAD/OPTIONS/PUT/POST/PATCH/DELETE (for API)
- **Cache policy**: TTL, cache key (which headers/cookies/query strings to include)
- **Origin request policy**: what headers/cookies/query strings to forward to origin

### Cache policies vs Origin request policies

CloudFront v2 (2021+) introduced Cache Policies and Origin Request Policies to replace "Forward Headers/Cookies/Query Strings."

| Policy | Purpose |
|--------|---------|
| **Cache policy** | Controls WHAT is cached and FOR HOW LONG. Defines cache key (which headers/cookies/query strings affect caching) |
| **Origin request policy** | Controls WHAT is forwarded to the origin. Headers/cookies/query strings that the origin needs but should not affect cache |

> **Trap:** If you include `Authorization` header in the cache key, every unique Authorization header creates a separate cache entry. This will destroy your cache hit rate. Cache based on a derived value (e.g., user group) or use Origin Request Policy to forward it without affecting caching.

### Signed URLs and Signed Cookies

Used for private content distribution:

| Method | When to use |
|--------|-------------|
| **Signed URL** | Single file access (e.g., PDF download link) |
| **Signed cookie** | Multiple files or session-based access (e.g., user subscription) |

Signed URLs/Cookies can restrict:
- Expiration time
- IP ranges
- Path pattern

### OAI (Origin Access Identity) and OAC (Origin Access Control)

**OAI (older):** Creates a special CloudFront identity that can access S3 buckets. You grant `s3:GetObject` to the OAI principal.

**OAC (newer, 2022+):** Replaces OAI with more features:
- Supports S3 server-side encryption with KMS
- Supports PUT/POST requests (for upload via CloudFront)
- Supports dynamic requests (DELETE, PATCH, etc.)

> **Trap:** When using CloudFront with S3, ALWAYS restrict S3 bucket access to CloudFront only (OAI/OAC). Never make the bucket public. This prevents direct S3 access bypassing CloudFront.

### Lambda@Edge

Lambda functions that run at CloudFront edge locations:

| Event type | When it runs | Use case |
|------------|-------------|----------|
| **Viewer request** | Before CloudFront checks cache | URL rewriting, redirect, auth check |
| **Origin request** | After cache miss, before going to origin | Modify request to origin |
| **Origin response** | After origin responds, before caching | Modify response, compress, resize images |
| **Viewer response** | Before returning to viewer | Set headers, modify response |

> **Trap:** Lambda@Edge has restrictions:
> - No VPC networking (cannot access RDS, ElastiCache)
> - Max 5-second timeout for viewer events, 30s for origin events
> - Max 50 MB deployment package
> - No environment variables
> - Functions are replicated to edge locations automatically

### Geo-restriction

- **Whitelist** — only allow countries in list
- **Blacklist** — block countries in list
- Uses MaxMind GeoIP database

### WAF integration

CloudFront can associate a WebACL (WAF) for:
- Rate-based rules (block IPs exceeding threshold)
- IP reputation lists (AWS Managed Rules)
- SQL injection / XSS protection
- Geographic blocking (alternative to Geo-restriction)

---

## 6. CloudWatch

### Metrics

- **Standard resolution** — 5-minute granularity, default
- **Detailed resolution** — 1-minute granularity, enabled per service
- **Custom metrics** — publish your own (`PutMetricData`, up to 15 months retention)
- **High-resolution metrics** — sub-minute (1-second granularity, storage cost applies)

### Metric math

Combine multiple metrics into one expression:
```sql
SUM(m1, m2) / COUNT(m3)  -- average CPU across ASG
m1 / m2                   -- error rate (errors / total requests)
```

### Logs

| Component | Description |
|-----------|-------------|
| **Log group** | Collection of log streams (e.g., `/aws/lambda/my-function`) |
| **Log stream** | Sequence of log events (e.g., a single Lambda invocation) |
| **Metric filter** | Extract metrics from log text (e.g., count ERROR lines) |
| **Subscription filter** | Stream log events to Lambda, Kinesis, or S3 in real-time |
| **Logs Insights** | SQL-like query language for log analysis |
| **Export to S3** | Batch export logs to S3 (via console/CLI/API) |

**Metric filter example:**
```
Pattern: "ERROR" → metric: ErrorCount, value: 1
Alarm: ErrorCount > 10 over 5 minutes → notify SNS
```

> **Trap:** CloudWatch Logs Insights queries are charged by data scanned. Queries over large log groups can be expensive. Scope to time range and filter before running.

### Alarms

| State | Description |
|-------|-------------|
| OK | Metric is within threshold |
| ALARM | Metric breached threshold |
| INSUFFICIENT_DATA | Not enough data to determine state (e.g., just created) |

Alarm actions:
- SNS notification
- Auto Scaling action
- EC2 action (reboot, stop, terminate, recover)

> **Trap:** Missing data treatment: If instances stop sending metrics (e.g., scaled to 0), how does the alarm behave?
> - `notBreaching` — treat as OK
> - `breaching` — treat as ALARM (dangerous—could trigger false alarm)
> - `ignore` — keep last state
> - `missing` — INSUFFICIENT_DATA

### CloudWatch Agent

- Collects metrics and logs from EC2 instances and on-prem servers
- Supports: CPU, memory, disk, network, custom metrics
- Can send to CloudWatch or AWS Systems Manager
- Configuration via JSON file

### Container Insights

- Collects metrics from ECS, EKS, and K8s
- Task/pod-level CPU, memory, network, disk
- Performance dashboards for container health

### Synthetics (Canaries)

- Node.js scripts (Puppeteer/Playwright) that run every 5–60 minutes
- Monitor API endpoints and user flows
- Built-in screenshots and HAR file capture
- Integrated with CloudWatch alarms

---

## 7. Cost Management

### Pricing models

| Model | Commitment | Discount | Best for |
|-------|-----------|----------|----------|
| On-demand | None | None | Variable, unknown workloads |
| Reserved Instance | 1 or 3 years | Up to 72% | Steady-state workloads |
| Savings Plan | 1 or 3 years (compute $/hr) | Up to 72% | Flexible across EC2, Fargate, Lambda |
| Spot | None | Up to 90% | Fault-tolerant, stateless, batch jobs |

### Reserved Instance types

| Type | Change attributes | Discount |
|------|-----------------|----------|
| Standard RI | No | Highest (up to 72%) |
| Convertible RI | Yes (up to same $ value) | Moderate (up to 54%) |
| Scheduled RI | No | Variable (known time windows) |

### Savings Plans

- **Compute Savings Plans** — covers EC2, Fargate, Lambda (most flexible, lower discount)
- **EC2 Instance Savings Plans** — covers EC2 only (higher discount, specific instance family)
- Compute Savings Plans are generally recommended first due to flexibility.

### Data transfer costs

AWS data transfer costs can exceed compute costs:

| Scenario | Cost |
|----------|------|
| Inbound from internet | Free |
| Outbound to internet | $0.09/GB (first 1 GB free, then tiered) |
| Inter-AZ | $0.01/GB both ways |
| Inter-region | $0.02–$0.09/GB |
| S3 to CloudFront | Free (if origin is S3) |
| CloudFront to internet | ~$0.085/GB (tiered, varies by region) |

> **Trap:** Inter-AZ traffic costs. If your ALB in us-east-1a talks to EC2 in us-east-1b, you pay $0.01/GB × 2 (request + response). Keep traffic within same AZ in high-throughput scenarios.

### Cost allocation tags

- Tag resources with metadata (e.g., `Environment: production`, `Project: inventory-saas`, `Team: backend`)
- Enable cost allocation tags in Billing console
- Filter cost reports by tag

### AWS Budgets and Anomaly Detection

- **Budget** — set cost or usage thresholds, notify via SNS/email
- **Anomaly detection** — ML-based detection of unusual spend patterns (new services, regions, unexpected traffic)

---

## 8. Q&A

### Intermediate

**Q: What's the difference between ALB and NLB? When would you use each?**
A: ALB operates at Layer 7 (HTTP/HTTPS) with path/host-based routing, stickiness, and Lambda targets. NLB operates at Layer 4 (TCP/UDP) with static IP and ultra-low latency. Use ALB for HTTP APIs, NLB for non-HTTP protocols or when static IP is required.

**Q: How does RDS Multi-AZ differ from a Read Replica?**
A: Multi-AZ provides HA via synchronous replication to a standby (not readable). Failover is automatic. Read Replicas provide read scaling via async replication, can be promoted to standalone, and support cross-region.

**Q: What is a launch template vs a launch configuration?**
A: Launch template is the modern replacement with versioning, multiple instance type support, IMDSv2 enforcement, and T2/T3 unlimited. Launch configurations are legacy/deprecated.

**Q: How does CloudFront determine whether to serve from cache or forward to origin?**
A: CloudFront constructs a cache key from the URL + cache policy settings (selected headers, cookies, query strings). If the key exists in edge cache and TTL hasn't expired, it serves from cache. Otherwise it forwards to origin.

**Q: What routing policy in Route 53 would you use for blue-green deployment?**
A: Weighted routing. Set up two records (blue and green) with weights. Shift traffic by adjusting weights (e.g., blue: 100 → 0, green: 0 → 100).

**Q: How do you know when to use an S3 bucket policy vs IAM policy?**
A: Bucket policies are for cross-account access, public access, and service-to-service access. IAM policies are for granting permissions to users/roles within your account.

### Trap questions

**Q: You deploy an ALB in three public subnets. Target group health checks fail. What do you check?**
A: (1) Security group allows ALB traffic to target port, (2) Target instances have the application running on the correct port, (3) The health check path returns 200, (4) The target subnet has a route to the ALB (both in same VPC, so routing is usually fine), (5) NACLs don't block traffic.

**Q: An RDS instance runs out of storage. What happens?**
A: The instance becomes unavailable (cannot write). You must increase allocated storage. RDS does NOT auto-scale storage by default (but you can enable storage autoscaling in RDS settings with a max limit).

**Q: You need to give a third-party access to your S3 bucket for exactly 2 hours. How?**
A: Generate a pre-signed URL with 2-hour expiry. The third-party gets temporary access to the specific object.

**Q: Can you use Route 53 to load balance between multiple on-premises data centers?**
A: Yes. Use Route 53 with health checks and weighted routing policy pointing to on-prem IP addresses. If health check fails, traffic is routed to healthy data centers.

**Q: You receive a bill for high data transfer costs. What's the most likely cause?**
A: Inter-AZ traffic or internet egress. Check if your ALB is in different AZs from your instances, or if EC2 instances are downloading large files from the internet.

### Follow-up questions

**Q: You mentioned ALB can route to Lambda. How does the Lambda response need to be formatted?**
A: Lambda must return a JSON with `statusCode`, `headers`, `body`, and optionally `isBase64Encoded`. ALB translates this into an HTTP response.

**Q: How would you design a zero-downtime deployment using ASG + ALB?**
A: Use a rolling update with the ASG (update launch template → ASG replaces instances gradually). ALB health check ensures traffic only goes to healthy instances. Or use blue-green with two ASGs behind Route 53 weighted routing.

**Q: What's the difference between CloudFront signed URLs and S3 pre-signed URLs?**
A: CloudFront signed URLs control access to CloudFront-distributed content and can enforce path/IP/expiry. S3 pre-signed URLs grant direct S3 access. CloudFront signed URLs offload auth to the edge and don't expose S3 directly.
