# AWS Services — Basic Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — this is ground-level AWS concepts  
> **Estimated time:** 4–6 hours

---

## Table of Contents

1. AWS Global Infrastructure
2. Shared Responsibility Model
3. IAM (Identity and Access Management)
4. VPC (Virtual Private Cloud)
5. EC2 (Elastic Compute Cloud)
6. S3 (Simple Storage Service)
7. Q&A

---

## 1. AWS Global Infrastructure

### Regions

An AWS Region is a physical geographic area with multiple isolated AZs. Choosing a region is driven by:

- **Latency** — deploy close to users
- **Compliance** — data residency laws (GDPR requires eu-central-1 or eu-west-1)
- **Service availability** — not all services exist in all regions
- **Cost** — prices vary by region (us-east-1 is cheapest, ap-southeast-1 is ~20% more)

> **Trap:** `us-east-1` has the most services but also the most outages due to being the "first" region. Always design for multi-region if you need five-9s.

### Availability Zones (AZs)

Each region has 3–6 AZs (typically 3). Each AZ is:

- One or more discrete data centers with redundant power, network, and cooling
- Connected via high-bandwidth, low-latency fibre (single-digit ms)
- Isolated from other AZ failures (independent power, cooling, physical facility)

You should always deploy across at least **2 AZs** for high availability.

> **Trap:** A "single-AZ" deployment is a common interview weakness. If you deploy one EC2 in one AZ and the AZ goes down, you're down. Always design for multi-AZ.

### Edge Locations / Points of Presence

Used by CloudFront and Route 53. Edge locations cache content for low-latency delivery. There are 600+ edge locations, far more than regions. They are not AZs — they run caching services only (no EC2/RDS).

### AWS Global vs Regional Services

| Global | Regional |
|--------|----------|
| IAM (users, roles, policies) | EC2 |
| Route 53 (DNS) | S3 |
| CloudFront | RDS |
| WAF (can be regional in some configs) | Lambda |
| Organizations | ECS/EKS |

> **Trap:** IAM is global — but IAM policies reference regional ARNs. A policy allowing access to "arn:aws:s3:::*" works globally, but S3 itself is regional.

---

## 2. Shared Responsibility Model

This is the single most important AWS concept. **Know this cold.**

```
                                 ┌──────────────────────┐
                                 │   Customer (you)      │
                                 │ ┌──────────────────┐  │
                                 │ │ DATA              │  │
                                 │ │ Platform/App/IAM  │  │
                                 │ │ OS/Network/FW     │  │ (S3: AWS handles this)
                                 │ └──────────────────┘  │
┌────────────────────────────────┼──────────────────────┘
│ AWS                            │
│ ┌──────────────────────────────┐│
│ │ Virtualization layer         │ │
│ │ Physical hardware (Compute)  │ │
│ │ Physical hardware (Storage)  │ │
│ │ Physical hardware (Network)  │ │
│ │ Data center facilities       │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
```

**AWS responsibility** — "Security OF the cloud"
- Physical security of data centers
- Hardware, software, networking (power, cooling, cabling)
- Hypervisor and virtualization layer

**Customer responsibility** — "Security IN the cloud"
- Data encryption (at rest and in transit)
- IAM configuration (users, roles, policies)
- OS patching (EC2) vs managed services (RDS handles this)
- Network ACLs and security groups
- Application security

The line shifts depending on service type:

| Service type | Examples | Customer manages | AWS manages |
|---|---|---|---|
| IaaS (Infrastructure as a Service) | EC2, EBS, VPC | OS, patches, network config, app | Physical hardware, virtualization |
| PaaS (Platform as a Service) | RDS, ElastiCache | Data, IAM, schema | OS, patches, failover, backups |
| SaaS (Software as a Service) | S3, DynamoDB, SQS | Data, IAM | Everything below data layer |

> **Trap:** Interviewers love: "Who is responsible for patching the OS of an RDS instance?" Answer: **AWS** (it's managed). But if it's EC2 with RDS MySQL installed manually, **you** are responsible.

---

## 3. IAM (Identity and Access Management)

### Concepts

| Term | Definition |
|------|------------|
| **User** | Long-term identity for a person or application (access keys, password) |
| **Group** | Collection of users; attach policies to groups, not individual users |
| **Role** | Temporary identity — assumed by users, apps, or services (short-term credentials via STS) |
| **Policy** | JSON document specifying permissions; attached to users, groups, or roles |
| **Trust policy** | JSON document specifying WHO can assume a role |

### Policy structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

Key elements:

- **Effect** — `Allow` or `Deny` (Deny always wins, even if Allow matches)
- **Action** — `service:Operation` (wildcards: `s3:*`, `ec2:Describe*`)
- **Resource** — ARN of the resource (`arn:partition:service:region:account:resource`)
- **Condition** — Optional restrictions (IP, time, MFA, SSL)

> **Trap:** `Resource: "*"` in an IAM policy grants access to ALL resources of matching actions. Always scope to specific ARNs where possible.

### Policy types

| Type | Managed by | Features |
|------|-----------|----------|
| AWS managed | AWS | Read-only, full-access, etc. Cannot modify. |
| Customer managed | You | You create and maintain. Reusable across accounts. |
| Inline | You | Embedded directly in user/group/role. Not reusable. Best for exceptions. |

### Trust policy example (EC2 role)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This trust policy is attached to an IAM role. It says "EC2 instances can assume this role." The role then has a permissions policy (e.g., `s3:GetObject`).

### IAM Roles vs Users

| | IAM User | IAM Role |
|---|---|---|
| Credentials | Long-term (access key + secret key, password) | Short-term (STS: temporary creds, expire 15 min–12 h) |
| Use case | Humans, long-running apps, CLI, SDK | EC2, Lambda, ECS, cross-account, federated users |
| Rotation | Manual | Automatic (STS expiration) |

> **Trap:** Never put IAM access keys in code, env files, or config files. Use IAM roles for EC2/Lambda/ECS, and use Secrets Manager or Parameter Store for third-party API keys.

### Evaluating permissions (policy evaluation logic)

1. By default, all requests are **Denied**
2. An explicit `Allow` overrides the default deny
3. An explicit `Deny` overrides any `Allow`
4. Policies are evaluated: AWS managed → customer managed → inline → resource-based → SCP → permissions boundary

**SCP (Service Control Policy)** — used in AWS Organizations to set permission guardrails across accounts. SCPs alone do not grant permissions; they filter what IAM policies can grant.

> **Trap:** SCPs affect all users including root. However, SCPs do NOT affect service-linked roles.

### CLI credentials chain

When you run AWS CLI commands, it looks for credentials in this order:

1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)
2. CLI `--profile` flag
3. AWS credentials file (`~/.aws/credentials`)
4. AWS config file (`~/.aws/config`)
5. EC2 instance profile (IMDS)
6. ECS task role
7. Lambda execution role

> **Trap:** If you set environment variables and instance profile credentials, the environment variables win. This can cause unexpected behavior when you SSH into an EC2 instance and the CLI uses env vars from a previous session.

### Common IAM patterns

**EC2 accessing S3:**
```
EC2 instance → IAM role (instance profile) → policy allowing s3:GetObject
```

**Lambda accessing DynamoDB:**
```
Lambda function → execution IAM role → policy allowing dynamodb:Query
```

**Cross-account access:**
```
Account A IAM role (trust policy allows Account B)
Account B IAM user/role (assume Role in Account A)
```

---

## 4. VPC (Virtual Private Cloud)

### Core components

A VPC is a logically isolated network in AWS. It's your own private data center in the cloud.

| Component | Description |
|-----------|------------|
| **VPC** | Virtual network, defined by CIDR block (e.g., `10.0.0.0/16`) |
| **Subnet** | Subdivision of VPC CIDR, in a single AZ (e.g., `10.0.1.0/24` in us-east-1a) |
| **Route table** | Controls traffic leaving a subnet |
| **Internet Gateway (IGW)** | Allows public internet access (attached to VPC) |
| **NAT Gateway** | Allows private subnets to reach internet (not inbound) |
| **Security Group** | Stateful firewall — instance-level, allow rules only |
| **NACL** | Stateless firewall — subnet-level, allow/deny rules |
| **VPC Peering** | Connect two VPCs via private IP |
| **VPC Endpoint** | Private access to AWS services without internet |

### Subnet types

| Type | Route to IGW | Route to NAT | Use case |
|------|-------------|-------------|----------|
| Public | Yes | — | ALB, NAT Gateway, bastion host |
| Private | No | Yes | EC2, ECS tasks, RDS |
| Isolated | No | No | RDS, Redis (backend services with no outbound) |

> **Trap:** A "public subnet" is just a subnet with a route to an IGW. There's no special flag. If you remove the IGW route, it becomes private.

### Security Groups vs NACLs

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| Level | Instance (ENI) | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (return traffic must be explicitly allowed) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated together | Rules evaluated in number order (lowest first) |
| Default | Deny all inbound, allow all outbound | Allow all inbound, allow all outbound |
| Use case | Application-level (web server: allow 443 from 0.0.0.0/0) | Subnet-level guardrails (block specific IP ranges) |

**Security group is stateful:**
```
Inbound rule: allow 443 from 0.0.0.0/0
→ Response traffic from the instance on ephemeral ports is automatically allowed
→ No need for explicit outbound rule
```

**NACL is stateless:**
```
Inbound rule 100: allow 443 from 0.0.0.0/0
→ Response traffic on ephemeral ports (1024–65535) needs an explicit outbound rule
→ Outbound rule 100: allow ephemeral ports to 0.0.0.0/0
```

> **Trap:** "I couldn't SSH into my EC2. I added an inbound security group rule for port 22." But if you also have a NACL blocking port 22, it won't work. Security groups and NACLs are **independent** — both must allow the traffic.

### VPC Peering

- Connects two VPCs via private IP (no IGW, no VPN)
- Not transitive — VPC-A peered with VPC-B, VPC-B peered with VPC-C does NOT mean A can reach C
- CIDR blocks of peered VPCs cannot overlap
- Can peer cross-account and cross-region

> **Trap:** "Transitive peering" is not supported. Use Transit Gateway for hub-and-spoke.

### VPC Endpoints

- **Gateway Endpoint** — S3 and DynamoDB only. Uses prefix lists in route tables. Free.
- **Interface Endpoint** (AWS PrivateLink) — All other AWS services. Uses ENI in subnets. Paid by the hour + data processing.

Use VPC endpoints to keep traffic within AWS network — never traverses the internet.

> **Trap:** S3 Gateway Endpoints work only from the same region. For cross-region S3 access, use Interface Endpoint or public internet.

### Flow Logs

Captures IP traffic metadata (not payload). Useful for:
- Security audits (who is connecting to what)
- Troubleshooting connectivity
- Analyzing netflow patterns

Stored in CloudWatch Logs or S3.

---

## 5. EC2 (Elastic Compute Cloud)

### Instance types naming convention

```
m5.xlarge
│ │ └──── size (CPU, memory)
│ └────── generation (5th gen)
└──────── instance family (general purpose)
```

Common families:

| Family | Use case |
|--------|----------|
| `t3`, `t4g` | Burstable (baseline + CPU credits) — dev/test, small apps |
| `m5`, `m6i`, `m7g` | General purpose — balanced compute/memory |
| `c5`, `c6i`, `c7g` | Compute optimized — batch processing, encoding, compute-heavy apps |
| `r5`, `r6i`, `r7g` | Memory optimized — in-memory caches, databases |
| `i3`, `i4i` | Storage optimized — high I/O, NVMe SSDs |
| `p3`, `p4`, `p5` | GPU — ML training, rendering |

> **Trap:** T3 instances have CPU credits. If credits run out, performance is throttled to baseline. Not suitable for sustained high-CPU workloads. Use M5/C5 instead.

### EBS (Elastic Block Store)

Volume types:

| Type | Max IOPS | Max throughput | Use case |
|------|----------|---------------|----------|
| gp3 (SSD) | 16,000 (base 3,000) | 1,000 MB/s | General purpose — boot volumes, most apps |
| gp2 (SSD) | 16,000 (depends on size) | 250 MB/s | Older gen, being replaced by gp3 |
| io1/io2 (SSD) | 64,000 (io1), 256,000 (io2) | 1,000 MB/s | High-performance — databases |
| st1 (HDD) | 500 | 500 MB/s | Throughput-intensive — big data, logs |
| sc1 (HDD) | 250 | 250 MB/s | Cold storage — infrequent access |

**gp3 vs gp2:** gp3 has a baseline of 3,000 IOPS regardless of size. gp2's IOPS scales with size (3 IOPS/GB). gp3 is cheaper for most workloads.

> **Trap:** EBS volumes are in a single AZ. You cannot attach an EBS volume to an instance in a different AZ. You can snapshot it and recreate in another AZ.

### EBS Snapshots

- Incremental (only changed blocks — first snapshot is full, subsequent are deltas)
- Stored in S3, but not directly accessible as files
- Can copy cross-region and cross-account
- Can restore as new volume (must be at least same size)
- **Snapshot performance:** First read from a new EBS volume created from a snapshot is slower (fetch from S3). Use `fio` to pre-warm (initialize).

> **Trap:** Deleting a snapshot does not affect subsequent snapshots. Each snapshot only references blocks that changed since the previous snapshot. The previous snapshot still has the full data.

### Instance metadata (IMDS)

IMDS is how an EC2 instance gets information about itself (IP, hostname, IAM role credentials).

- **IMDSv1** — no protection. Any process can call `http://169.254.169.254/latest/meta-data/`
- **IMDSv2** — session-oriented. Need `PUT /latest/api/token` first, then use token for subsequent calls

```bash
# IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

> **Trap:** SSRF attacks can steal IMDSv1 credentials (notorious Capital One breach). Always enforce IMDSv2 on your instances. This is a classic interview topic.

### Key Pairs

- Public key stored by AWS
- Private key is your responsibility — lose it, and you lose SSH access
- Only for EC2 — not used for RDS, Lambda, etc.
- Windows instances use key pair to decrypt administrator password

### Placement Groups

| Strategy | Description | Use case |
|----------|-------------|----------|
| **Cluster** | Instances in a single AZ, low latency (10 Gbps), high risk (if rack fails, all instances fail) | HPC, tightly-coupled apps |
| **Spread** | Instances across distinct hardware (max 7 per AZ per group) | Small HA app, each instance on separate rack |
| **Partition** | Instances divided into partitions (max 7 per AZ). Each partition is isolated from others. | Distributed systems (Hadoop, Kafka, Cassandra) |

### User Data

Script passed at instance launch that runs on boot:
- Runs as root on first boot only
- Can be used for installation, configuration, OS updates
- Logs at `/var/log/cloud-init-output.log`

---

## 6. S3 (Simple Storage Service)

### Core concepts

| Term | Description |
|------|-------------|
| **Bucket** | Container for objects. Globally unique name. Regional resource. |
| **Object** | File + metadata + key (path). Max 5 TB. |
| **Key** | Full path: `folder/subfolder/file.txt` (max 1,024 bytes UTF-8) |
| **Version ID** | Unique ID for each version of an object |
| **ARN** | `arn:aws:s3:::bucket-name/key` |

> **Trap:** S3 bucket names must be globally unique across ALL AWS accounts. Not just your account — every AWS account. Common failure: `my-app-data` is likely taken.

### Storage classes

| Class | Durability | Availability | Min storage | Retrieval | Use case |
|-------|-----------|-------------|-------------|-----------|----------|
| S3 Standard | 99.999999999% (11 9's) | 99.99% | None | Instant | Frequently accessed |
| Intelligent-Tiering | 11 9's | 99.9% | 30 days | Instant | Unknown patterns |
| Standard-IA | 11 9's | 99.9% | 30 days | Instant | Infrequent access |
| One Zone-IA | 11 9's | 99.5% | 30 days | Instant | Non-critical, re-creatable data |
| Glacier | 11 9's | 99.99% | 90 days | 1–5 min (expedited), 3–5 h (standard), 5–12 h (bulk) | Archives |
| Glacier Deep Archive | 11 9's | 99.99% | 180 days | 12 hours | Long-term archives (compliance, 7+ year retention) |

> **Trap:** One Zone-IA only stores data in one AZ. If that AZ goes down, you lose the data. Don't use for anything important.

### S3 Lifecycle policies

Automatically transition objects between storage classes or delete them:

```
Creation → Standard (30 days) → Standard-IA (60 days) → Glacier (180 days) → Glacier Deep Archive (365 days) → Delete (730 days)
```

Rules can be applied to:
- Current versions, previous versions, incomplete multipart uploads
- Prefix or tags

> **Trap:** Minimum 30 days before transition from Standard to Standard-IA. Minimum 30 days before transition from Standard-IA to Glacier. You cannot create a rule that transitions a 1-day-old object to Glacier — it won't execute.

### Versioning

- Once enabled, cannot be turned off (only suspended)
- Stores all versions of an object (including deleted markers)
- Protect against accidental deletes and overwrites
- Includes delete markers — "deleting" a versioned object adds a delete marker, doesn't remove the object
- Permanently delete by specifying version ID

```bash
# List versions
aws s3api list-object-versions --bucket my-bucket --prefix path/to/file

# Permanently delete (requires version ID)
aws s3api delete-object --bucket my-bucket --key path/to/file --version-id abc123
```

> **Trap:** MFA Delete — requires MFA to change versioning state or permanently delete versions. Enforce this for critical buckets.

### S3 bucket policies vs IAM policies

| Aspect | Bucket Policy | IAM Policy |
|--------|--------------|------------|
| Scope | Specific bucket | User/role (allows access to any matching ARN) |
| Who can access | Cross-account (principal specifies account) | Only attached identity |
| Size limit | 20 KB | No limit per user |
| Complexity | 1 policy per bucket | Multiple policies per identity |
| Use case | Public access, cross-account, service-to-service | User permissions, least-privilege |

**Bucket policy granting cross-account access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/AppRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### Pre-signed URLs

- Generate temporary URLs that grant time-limited access to specific objects
- URL includes credentials as query params (signature)
- Can be generated for `GetObject` (download) or `PutObject` (upload by third party)
- Uses your IAM credentials at generation time

```php
// Laravel example — generating a pre-signed URL via S3 client
use Aws\S3\S3Client;

$client = new S3Client([
    'region'  => 'us-east-1',
    'version' => 'latest',
]);

$command = $client->getCommand('GetObject', [
    'Bucket' => 'my-bucket',
    'Key'    => 'path/to/file.pdf',
]);

$request = $client->createPresignedRequest($command, '+5 minutes');
$presignedUrl = (string) $request->getUri();
```

> **Trap:** The pre-signed URL is generated with YOUR permissions. If you have admin access, the URL grants admin-level access to that object. Always use a restricted IAM user/role for pre-signed URLs.

### Multipart Upload

Required for objects > 5 GB, recommended for > 100 MB:

1. `CreateMultipartUpload` → returns upload ID
2. `UploadPart` (1–10,000 parts, each 5 MB–5 GB except last)
3. `CompleteMultipartUpload` → assembles parts

Benefits:
- Parallel uploads (faster)
- Retry individual parts (not entire upload)
- Pause/resume

### S3 event notifications

S3 can send events (ObjectCreated, ObjectRemoved, RestoreObject, etc.) to:
- SNS Topic
- SQS Queue
- Lambda Function
- EventBridge (newer, more flexible)

> **Trap:** S3 event notifications are **at least once**. Your Lambda/SQS must be idempotent.

### Consistency model

S3 provides **strong consistency** for all operations since December 2020:

- `PUT` of a new object → immediately readable
- `PUT` of an existing object (overwrite) → immediately readable
- `DELETE` → immediately reflected

Before December 2020, S3 had eventual consistency for overwrite/deletes. Older interview answers may still say "eventual consistency" — **you must know the current model.**

### Object Lock (WORM)

- **Governance mode** — Overridable by users with specific IAM permissions (good for testing)
- **Compliance mode** — No one can override, including root (used for SEC Rule 17a-4)
- Can set retention period (days/years) or legal hold (no expiry)

---

## 7. Q&A

### Basic

**Q: What's the difference between a region, an AZ, and an edge location?**
A: Region = geographic area with multiple AZs. AZ = one or more data centers, isolated failure domain. Edge location = CloudFront/Route 53 POP for content caching, not general compute.

**Q: What's the shared responsibility model?**
A: AWS secures the cloud (physical facilities, hardware, virtualization). You secure what's in the cloud (data, IAM, OS on EC2, network config). The line shifts based on service type (IaaS vs PaaS vs SaaS).

**Q: What's the difference between a security group and a NACL?**
A: Security group is stateful, instance-level, allow-only rules. NACL is stateless, subnet-level, allow+deny rules, evaluated in number order.

**Q: What's the difference between an IAM role and an IAM user?**
A: User = long-term credentials (access key + secret key or password). Role = temporary credentials via STS (auto-expire). Roles are for services (EC2, Lambda) and cross-account access.

**Q: How do you enforce IMDSv2 on EC2?**
A: Set `MetadataOptions` `HttpTokens` to `required` on the instance or launch template.

**Q: What storage class should you use for logs that must be queryable for 30 days, then archived for 7 years?**
A: S3 Standard for 30 days, lifecycle rule transitions to Glacier after 30 days, then to Glacier Deep Archive after 1 year. After 7 years, delete.

**Q: How do you grant a Lambda function access to S3?**
A: Create an IAM role with a policy allowing `s3:GetObject` (etc.), attach that role as the Lambda execution role.

### Trap questions

**Q: An EC2 instance cannot reach the internet. Both security group and NACL allow outbound traffic. What could be wrong?**
A: The subnet's route table doesn't have a route to an IGW (for public subnet) or NAT Gateway (for private subnet). Security groups/NACLs are not the only factor — routing matters.

**Q: Can a public subnet directly host an RDS instance?**
A: Yes, technically. But you should never put RDS in a public subnet — it has a public endpoint option but best practice is private subnet with no IGW route.

**Q: Are S3 bucket names globally unique or regional?**
A: Globally unique across all AWS accounts, but the bucket itself is stored in a specific region.

**Q: Can you attach an EBS volume to an EC2 instance in a different AZ?**
A: No. EBS volumes are AZ-specific. You must snapshot and recreate in the target AZ.

**Q: Who is responsible for patching the OS on an ECS EC2 instance?**
A: You are (if using EC2 launch type). AWS is (if using Fargate — they manage the underlying instances).

### Follow-up questions

**Q: You said IAM roles are better than access keys for EC2. How does the EC2 instance get the role credentials?**
A: Through the instance metadata service (IMDS). The instance profile is attached to the EC2 at launch. The AWS SDK automatically calls IMDS to get temporary credentials from STS.

**Q: How do you prevent accidental S3 bucket deletion?**
A: Enable versioning (MFA delete), enable S3 Object Lock, use IAM policies that prevent `s3:DeleteBucket`, use SCPs in Organizations.

**Q: What happens if an EBS io1 volume reaches its max IOPS?**
A: It will throttle (queue I/O requests). Latency increases. Use CloudWatch `VolumeQueueLength` to monitor and scale up.
