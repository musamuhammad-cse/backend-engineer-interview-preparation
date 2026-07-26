# DynamoDB — Senior

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Single-table design, hot keys, partition splitting, auto-scaling, global tables, backup/restore, serverless stack, DynamoDB vs alternatives  
> **Real-world anchor:** Multi-tenant SaaS (single-table design, access patterns), trading platform (time-series trades), Chronos (job state with TTL and GSIs)

---

## 1. Single-table design — the FAANG interview topic

Single-table design is the #1 DynamoDB topic in senior interviews. The key insight: **you design your schema based on access patterns, not on entity relationships.**

### The problem with multi-table design

In a traditional RDBMS approach you'd model your SaaS like this:

```sql
CREATE TABLE users (id PK, org_id FK, name, email, role);
CREATE TABLE orders (id PK, user_id FK, org_id FK, total, status, created_at);
CREATE TABLE products (id PK, org_id FK, name, price, stock);
```

In DynamoDB, this would mean 3 tables with multiple GSIs each — complex, expensive, and not scalable.

### Single-table design approach

One table to rule them all:

```js
// One table for the entire application
// PK = partition key, SK = sort key

// Users
{ PK: 'USER#u1001', SK: 'PROFILE', name: 'Alice', email: 'alice@example.com', org_id: 'org_42', role: 'engineer', data: '2024-01-01' }

// Orders (user's orders sorted by date)
{ PK: 'USER#u1001', SK: 'ORDER#2024-06-15#ORD-001', total: 100, status: 'shipped', product_id: 'p1001' }

// Orders by product (for product analytics)
{ PK: 'PROD#p1001', SK: 'ORDER#2024-06-15#ORD-001', user_id: 'u1001', total: 100, status: 'shipped' }

// Products by org
{ PK: 'ORG#org_42', SK: 'PROD#p1001', name: 'Widget', price: 29.99, stock: 100 }

// Users by org (for admin dashboard)
{ PK: 'ORG#org_42', SK: 'USER#u1001', name: 'Alice', role: 'engineer' }
```

### Access pattern → GSI mapping

| Access Pattern | Base Table Query | GSI Needed |
|----------------|-----------------|------------|
| Get user by ID | `PK=USER#u1001, SK=PROFILE` | — |
| Get user's orders sorted by date | `PK=USER#u1001, SK begins_with ORDER#` | — |
| Get all products for an org | `PK=ORG#org_42, SK begins_with PROD#` | — |
| Get all users for an org | `PK=ORG#org_42, SK begins_with USER#` | — |
| Find order by ID | — | GSI on `order_id` |
| Find all orders by status | — | GSI on `status` |
| Find products by category | — | GSI on `category` |

### Composite sort key patterns

```js
// Hierarchical sorting — SK encodes hierarchy
SK = 'STATUS#pending#DATE#2024-06-15'

// Query all pending orders:
PK = 'USER#u1001', SK begins_with 'STATUS#pending'

// Query all orders for a specific status + date range:
PK = 'USER#u1001', SK BETWEEN 'STATUS#pending#DATE#2024-06-01' AND 'STATUS#pending#DATE#2024-06-30'
```

### Adjacency list pattern (graphs)

For modeling relationships (e.g., user follows user):

```js
// Alice follows Bob
{ PK: 'USER#u1001', SK: 'FOLLOWS#u1002', followed_at: '2024-06-01' }
{ PK: 'USER#u1002', SK: 'FOLLOWED_BY#u1001', followed_at: '2024-06-01' }

// Query: who does Alice follow?
Query: PK = 'USER#u1001', SK begins_with 'FOLLOWS#'

// Query: who follows Alice?
Query: PK = 'USER#u1002', SK begins_with 'FOLLOWED_BY#'
```

---

## 2. Hot key mitigation

### What is a hot key?

A **hot key** is a partition key that receives disproportionate traffic, causing throttling:

```
Table: Orders (PK = status, SK = order_date)
GSI PK: status = "pending"  (millions of items, high write throughput)

Result: GSI partition for "pending" is throttled.
```

### Mitigation strategies

#### 1. Shard the hot key

Add a random suffix to the hot key:

```js
// Instead of:
status = 'pending'

// Use:
status = 'pending#0'   // write approximately 1/10 of items here
status = 'pending#1'
...
status = 'pending#9'

// Query: read from all shards and merge
const shards = ['pending#0', 'pending#1', ..., 'pending#9']
const results = await Promise.all(
  shards.map(shard => queryByStatus(shard))
)
const merged = results.flat()
```

#### 2. Use write sharding with a GSI

```js
// Base table PK is already distributed (e.g., user_id)
// GSI has the hot key — shard the GSI PK
// Add a random suffix to the GSI PK attribute
GSI1PK: `${status}#${Math.floor(Math.random() * 10)}`
```

#### 3. DAX for read-heavy hot keys

If the hot key is read-heavy (e.g., a popular product page), use DAX to cache:

```js
// DAX caches the GetItem/Query results
// Reads hit DAX (microseconds), writes go to DynamoDB (cache is updated asynchronously)
```

#### 4. Cache in Redis (your experience)

For extreme hot keys, cache the hot data in Redis:

```js
// Check Redis first
const cached = await redis.get('hot_product:p1001')
if (cached) return JSON.parse(cached)

// Cache miss — read from DynamoDB
const result = await docClient.send(new GetCommand({
  TableName: 'Products',
  Key: { product_id: 'p1001' }
}))
await redis.setex('hot_product:p1001', 60, JSON.stringify(result.Item))
return result.Item
```

---

## 3. Partition splitting — how DynamoDB scales

### How partitions work

DynamoDB **automatically** manages partitions. A single partition can hold:
- Up to **10 GB** of data
- Up to **3,000 RCU** or **1,000 WCU**

When a table exceeds these limits, DynamoDB splits partitions:

```
Initial: 1 partition (up to 3,000 RCU / 1,000 WCU / 10 GB)

If you provision > 3,000 RCU → split into 2 partitions
If you provision > 1,000 WCU → split into 2 partitions
If storage > 10 GB → split into 2 partitions

Formula for partition count:
  partitions = MAX(ceil(RCU / 3000), ceil(WCU / 1000), ceil(SizeGB / 10))
```

Example: Table with 10,000 RCU, 5,000 WCU, 50 GB data:
```
Partitions = MAX(ceil(10000/3000), ceil(5000/1000), ceil(50/10))
           = MAX(4, 5, 5)
           = 5 partitions
```

### Implication for hot keys

Each partition handles its share of the RCU/WCU:

```
Table: 5,000 RCU → 5 partitions → ~1,000 RCU per partition
```

If one partition key (e.g., `user_id = "viral_user"`) generates > 1,000 RCU, it will **throttle** even if the table as a whole has capacity.

**Trap:** Provisioned throughput is evenly distributed across all partitions. If you have 10 partitions and 10,000 RCU, each partition gets 1,000 RCU — regardless of which partition keys are actually accessed.

---

## 4. Auto-scaling vs On-demand

### Auto-scaling (Application Auto Scaling)

```bash
# Create auto-scaling target
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 1000

# Create scaling policy
aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --policy-name "ReadScalingPolicy" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-scaling-policy-configuration \
    '{"TargetValue": 70.0, "PredefinedMetricSpecification": {"PredefinedMetricType": "DynamoDBReadCapacityUtilization"}, "ScaleOutCooldown": 60, "ScaleInCooldown": 60}'
```

**Trap:** Auto-scaling has **cooldown periods** (default 60s scale-out, 60s scale-in). If traffic spikes faster than the cooldown, you'll get throttled. Use on-demand for unpredictable spikes.

### On-demand

```bash
aws dynamodb update-table \
  --table-name Orders \
  --billing-mode PAY_PER_REQUEST
```

**On-demand scaling:**
- Scales up to **2x previous peak** within minutes
- No throttle (within soft limits)
- Costs **~3-5x more** per RCU/WCU than provisioned

### When to use which

| Workload | Mode | Reason |
|----------|------|--------|
| Predictable steady state | Provisioned + auto-scaling | Lower cost |
| Spiky unknown traffic | On-demand | No throttling |
| New application | On-demand initially | No capacity planning |
| High-volume known traffic | Provisioned with reserved capacity | Best pricing |
| POC / small apps | On-demand | Simple |

---

## 5. Global Tables (multi-region)

Global Tables provide active-active replication across AWS regions:

```bash
# Create Global Table (from existing table)
# Requires DynamoDB Streams to be enabled
aws dynamodb update-table \
  --table-name Orders \
  --replica-updates '[{"Create": {"RegionName": "eu-west-1"}}]'
```

### Global Table semantics

| Property | Behavior |
|----------|----------|
| **Replication** | Active-active — writes in any region propagate to all |
| **Consistency** | Eventually consistent (last-writer-wins for conflicts) |
| **Latency** | Typically < 1 second cross-region |
| **Conflict resolution** | Last writer wins (based on timestamp) |
| **Cost** | Additional RCU/WCU in each replica |

### Use cases

- **Disaster recovery:** Replicate to another region, failover with Route53
- **Geo-proximity:** Users in EU read from eu-west-1, users in US read from us-east-1
- **Global applications:** Real-time multiplayer, global leaderboards

**Trap:** Global Tables use **last-writer-wins** for conflict resolution. If two regions write to the same item simultaneously, one overwrites the other. For your trading platform's accounting, this may not be acceptable — consider region-sharding instead.

---

## 6. Backup and restore

### On-demand backup

```bash
# Create full backup
aws dynamodb create-backup \
  --table-name Orders \
  --backup-name "Orders-2024-01-15"

# Restore from backup (to a new table)
aws dynamodb restore-table-from-backup \
  --target-table-name Orders-restored \
  --backup-arn arn:aws:dynamodb:...:backup/...
```

### Point-in-Time Recovery (PITR)

```bash
# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name Orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

- **35-day retention** of continuous backups
- Restore to **any second** within 35 days
- Restores to a **new table** (not in place)
- Costs: additional storage (incremental)

```bash
# Restore to point in time
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-restored \
  --restore-date-time "2024-01-15T10:30:00Z"
```

### Backup strategy

| Backup Type | RPO | RTO | Cost |
|-------------|-----|-----|------|
| PITR | < 5 min (continuous) | Minutes (restore to new table) | Medium (incremental storage) |
| On-demand | Manual | Minutes | Low (per-backup) |
| Export to S3 (DynamoDB 2023+) | Manual | Hours | Low (no RCU usage) |

---

## 7. Serverless stack — DynamoDB + Lambda + API Gateway

The classic AWS serverless triad:

```
Client → API Gateway → Lambda → DynamoDB
                            ↕
                    DynamoDB Streams → Lambda → SQS/SNS/Elasticsearch
```

### Full example: serverless order API

```yaml
# SAM template.yaml
Resources:
  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: Orders
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: user_id
          AttributeType: S
        - AttributeName: order_date
          AttributeType: S
      KeySchema:
        - AttributeName: user_id
          KeyType: HASH
        - AttributeName: order_date
          KeyType: RANGE
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES

  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: create-order.handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
      Environment:
        Variables:
          TABLE_NAME: !Ref OrdersTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /orders
            Method: POST

  OrderStreamProcessor:
    Type: AWS::Serverless::Function
    Properties:
      Handler: stream-processor.handler
      Policies:
        - DynamoDBStreamReadPolicy:
            TableName: !Ref OrdersTable
            StreamName: !GetAtt OrdersTable.StreamArn
      Events:
        Stream:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt OrdersTable.StreamArn
            StartingPosition: LATEST
            BatchSize: 100
```

### Lambda handler for CRUD

```js
// create-order.handler
exports.handler = async (event) => {
  const body = JSON.parse(event.body)
  const { user_id, product_id, qty, total } = body

  await docClient.send(new PutCommand({
    TableName: process.env.TABLE_NAME,
    Item: {
      user_id: user_id,
      order_date: new Date().toISOString().split('T')[0],
      order_id: `ORD-${Date.now()}`,
      product_id,
      qty,
      total,
      status: 'pending'
    },
    ConditionExpression: 'attribute_not_exists(order_id)'
  }))

  return {
    statusCode: 201,
    body: JSON.stringify({ message: 'Order created' })
  }
}
```

### Best practices for serverless + DynamoDB

1. **Use single-table design** — minimize Lambda cold start impact by reducing number of tables
2. **Use DAX for read-heavy paths** — cache frequently accessed data
3. **Batch writes** — Lambda has 15-minute max execution, batch for throughput
4. **Use pessimistic concurrency for critical writes** — DynamoDB transactions
5. **Use `ConditionExpression` for idempotency** — avoid duplicate orders on retry

---

## 8. DynamoDB vs alternatives

### DynamoDB vs Cassandra

| Factor | DynamoDB | Cassandra |
|--------|----------|-----------|
| Management | Fully managed | Self-managed or managed (DataStax) |
| Consistency | Tunable (eventual/strong) | Tunable (eventual/quorum/all) |
| Query model | Key-value + document (limited) | CQL (SQL-like, richer queries) |
| Single-table | Strong culture | Possible but less common |
| Scaling | Automatic | Manual (add nodes) |
| Cost | Pay per RCU/WCU/storage | Pay per instance/storage |
| Best for | AWS-native, serverless | Multi-cloud, on-prem, complex queries |

### DynamoDB vs ScyllaDB

| Factor | DynamoDB | ScyllaDB |
|--------|----------|----------|
| Engine | DynamoDB (proprietary) | Rewrite of Cassandra in C++ |
| Performance | Single-digit ms | Sub-ms (C++, no JVM) |
| Compatibility | DynamoDB API | CQL + DynamoDB API (alternator) |
| Management | Fully managed | Self-managed |
| Pricing | Per-request | Per-instance |

### DynamoDB vs MongoDB

| Factor | DynamoDB | MongoDB |
|--------|----------|---------|
| Query flexibility | Limited (PK + SK + GSI) | Rich (MQL, aggregation pipeline) |
| Consistency | Strong (w/ ConsistentRead) | Tunable |
| Indexes | LSI + GSI (limit 20) | Multiple index types |
| Transactions | Yes (since 2018) | Yes (since 4.0) |
| Scaling | Automatic partitions | Manual sharding |
| Management | Fully managed | Self-managed or Atlas |
| Schema flexibility | Yes (DynamoDB JSON) | Yes (BSON) |

### DynamoDB vs PostgreSQL (RDS)

| Factor | DynamoDB | RDS PostgreSQL |
|--------|----------|----------------|
| Data model | Key-value + document | Relational |
| Query | Limited | Full SQL (JOINs, subqueries, CTEs) |
| Scaling | Sharding-only (no joins) | Read replicas, read/write splitting |
| Consistency | Eventual + strong (limited) | ACID |
| Schema | No schema enforcement | Fixed schema |
| Best for | Simple access patterns | Complex queries |

### When to use DynamoDB vs RDS

| Scenario | Choose |
|----------|--------|
| Simple key-value lookups | DynamoDB |
| Complex queries with JOINs | RDS |
| High throughput writes | DynamoDB |
| Relational data integrity | RDS |
| Serverless application | DynamoDB |
| Ad-hoc analytics | RDS |
| Time-series data | DynamoDB (with single-table) |
| Multi-row transactions | Both (DynamoDB since 2018) |

---

## 9. DynamoDB for multi-tenant SaaS

### Access patterns for your SaaS

```js
// 1. Get org's products (paginated, filtered by category)
PK: 'ORG#org_42', SK: begins_with('PRODUCT#')
GSI: { PK: 'PRODUCT#org_42', SK: 'category#tools#name#Widget' }

// 2. Get org's users
PK: 'ORG#org_42', SK: begins_with('USER#')

// 3. Get user's profile
PK: 'USER#u1001', SK: 'PROFILE'

// 4. Get user's orders (sorted by date, newest first)
PK: 'USER#u1001', SK: begins_with('ORDER#'), ScanIndexForward: false

// 5. Get all pending orders across org
GSI PK: 'STATUS#org_42', SK: begins_with('ORDER#pending')

// 6. Search products by name
// Use OpenSearch/Elasticsearch for full-text search
// Stream → Lambda → ES
```

### Single-table schema

```js
// One table: SaaSApp

// Users
{ PK: 'USER#u1001',        SK: 'PROFILE',        name: 'Alice', email: 'alice@example.com', org_id: 'org_42', role: 'admin', GSI1PK: 'ORG#org_42', GSI1SK: 'USER#u1001' }

// Orders
{ PK: 'USER#u1001',        SK: 'ORDER#2024-06-15#ORD-001', total: 100, status: 'pending', GSI1PK: 'STATUS#pending', GSI1SK: 'ORDER#2024-06-15#ORD-001' }

// Products
{ PK: 'PROD#p1001',        SK: 'META',           name: 'Widget', price: 29.99, org_id: 'org_42', category: 'tools', GSI1PK: 'ORG#org_42', GSI1SK: 'PROD#p1001', GSI2PK: 'CAT#tools', GSI2SK: 'PROD#p1001' }

// GSI1: Org index (organization-scoped queries)
// GSI1PK = org_id, GSI1SK = entity prefix + ID

// GSI2: Category index (products by category)
// GSI2PK = category, GSI2SK = product_id
```

---

## 10. Common production issues

### Issue 1: Throttled writes on hot key

**Symptom:** `ProvisionedThroughputExceededException` for a specific partition key.

**Solution:**
1. Identify the hot key via CloudWatch `ConsumedWriteCapacityPerPartition`
2. Apply write sharding — add random suffix to the hot key
3. Use DAX for reads to reduce load on hot partitions
4. Switch to on-demand (temporary spike)

### Issue 2: GSI throttling affects base table

**Symptom:** Base table operations are slow/throttled even though base table has capacity.

**Cause:** GSI is throttled → GSI writes back up → affects base table's ability to write.

**Solution:**
1. Provision adequate RCU/WCU on the GSI (separate from base table)
2. Shard the GSI partition key for hot attributes
3. Reduce GSI projection from `ALL` to `KEYS_ONLY` or `INCLUDE`
4. Use `ConditionExpression` to skip GSI writes when attribute is unchanged

### Issue 3: Large items causing high latency

**Symptom:** GetItem/Query latency increases as items grow.

**Cause:** Items near 400 KB take multiple RCUs (ceil(size/4KB) for eventual read).

**Solution:**
1. Keep items small (< 10 KB ideally)
2. Store large attributes in S3, reference by URL in DynamoDB
3. Use `ProjectionExpression` to read only needed attributes
4. Compress large attributes

### Issue 4: Expensive Scan operations

**Symptom:** High RCU consumption, slow application performance.

**Solution:**
1. Replace Scan with Query + GSI
2. Use parallel Scan with `TotalSegments`
3. Batch scan during off-peak hours
4. Use Export to S3 (2023+) instead of Scan for bulk exports

---

## 11. Time-series data in DynamoDB

For your trading platform's time-series trade data:

```js
// Single-table design for trades
// Table: TradingData

// Trade record
{ PK: 'TRADE#2024-06-15',       SK: 'TS#1718467200000#TRD-001', symbol: 'AAPL', price: 150.50, qty: 100, side: 'buy', trader: 'u1001' }

// OHLCV pre-aggregation (1-min candles)
{ PK: 'OHLCV#AAPL#2024-06-15',  SK: 'MIN#1718467200', open: 150.00, high: 151.00, low: 149.50, close: 150.50, volume: 5000 }

// Daily aggregation
{ PK: 'OHLCV#AAPL',             SK: 'DAY#2024-06-15', open: 148.00, high: 152.00, low: 147.00, close: 150.50, volume: 150000 }
```

**Query:** Get OHLCV for AAPL on June 15, 2024:
```
Query PK = 'OHLCV#AAPL#2024-06-15', SK begins_with 'MIN#'
```

**Issues with hot partitions:** If you have too many trades per second, a single `PK = 'TRADE#2024-06-15'` can become a hot partition. Shard by hour:
```
PK = 'TRADE#2024-06-15#10'  // trades in 10:00-10:59
PK = 'TRADE#2024-06-15#11'  // trades in 11:00-11:59
```

---

## 12. DynamoDB vs RDS decision framework

For a senior interview, be prepared to justify your database choice:

```
Q: Why not just use RDS PostgreSQL for everything?
A: DynamoDB is better for:
  - Simple access patterns (key-value, hierarchical)
  - Infinite scale (auto-partitioning)
  - Serverless (no ops, auto-scaling to zero)
  - Microsecond reads (with DAX)
  
RDS PostgreSQL is better for:
  - Complex queries (JOINs, aggregations)
  - Relational data integrity (FKs, constraints)
  - Ad-hoc analytics
  - Full-text search (use Elasticsearch with either)
```

---

## Interview traps cheatsheet — Senior

| Trap | The truth |
|------|-----------|
| "Single-table design means one physical table" | It means one DynamoDB table per application, with entity types encoded in PK/SK prefixes. |
| "GSIs replicate instantly" | GSI propagation is asynchronous — slight delay. Eventually consistent only. |
| "Provisioned throughput is per table" | Throughput is split evenly across all partitions. A hot partition can throttle even if the table has spare capacity. |
| "Global Tables are strongly consistent" | Eventually consistent, last-writer-wins. Not suitable for strong consistency across regions. |
| "Auto-scaling handles all traffic spikes" | Has cooldown periods. Sudden spikes > 2x capacity cause throttling. |
| "On-demand never throttles" | Has soft limits. Extreme (>5x previous peak) volumes can still throttle. |
| "TTL deletions are free" | Free in terms of WCU, but TTL items count toward storage until deleted. |
| "DynamoDB Stream records are globally ordered" | Ordered within a partition key, not globally. For global ordering, use Kinesis. |
| "PITR restores are instant" | Restores to a new table — data must be copied. Takes minutes to hours for large tables. |
| "DynamoDB is always cheaper than RDS" | At low volume, DynamoDB on-demand is expensive. At high volume, provisioned DynamoDB is often cheaper than RDS for simple access patterns. |
</details>
