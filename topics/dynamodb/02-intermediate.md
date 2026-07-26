# DynamoDB — Intermediate

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** LSI vs GSI deep dive, consistency models, DynamoDB Streams, DAX, TTL, transactions, optimistic locking, PartiQL  
> **Real-world anchor:** Multi-tenant SaaS, trading platform (event-driven with streams), Chronos (job state with TTL)

---

## 1. LSI vs GSI — deep dive

### When to use LSI

```js
// Table: Orders (PK=user_id, SK=order_date)
// Use case: get all orders for a user, sorted by status
// LSI: StatusIndex (PK=user_id, SK=status)

// Query: all orders for user, sorted by status
await docClient.send(new QueryCommand({
  TableName: 'Orders',
  IndexName: 'StatusIndex',
  KeyConditionExpression: 'user_id = :uid',
  ExpressionAttributeValues: { ':uid': 'u1001' }
}))
```

**Use LSI when:**
- You need alternative sort order **within the same partition key**
- You need **strongly consistent reads** on the index
- The index is known at table creation time
- You're within the 5 LSI limit

### When to use GSI

```js
// Table: Orders (PK=user_id, SK=order_date)
// Use case: find all orders by status across ALL users
// GSI: StatusGlobalIndex (PK=status, SK=order_date)

// Query: all shipped orders globally
await docClient.send(new QueryCommand({
  TableName: 'Orders',
  IndexName: 'StatusGlobalIndex',
  KeyConditionExpression: 'status = :status',
  ExpressionAttributeValues: { ':status': 'shipped' }
}))
```

**Use GSI when:**
- You need a **different partition key** (query across all partitions)
- You need the index created after table creation
- Eventually consistent reads are acceptable
- You need separate throughput control

### LSI restrictions

1. **Must be created at table creation time** — cannot add LSIs after
2. **Same partition key** as the base table
3. **Max 5 LSIs** per table
4. **10 GB limit** on items per partition key (across base table + all LSIs)
5. **Shares throughput** with base table — throttling on base affects LSI and vice versa

### GSI restrictions

1. **Eventually consistent only** — no strongly consistent reads on GSIs
2. **Max 20 GSIs** per table (soft limit, can be increased)
3. **Separate throughput** — must provision RCU/WCU for each GSI
4. **GSI writes are asynchronous** — slight propagation delay
5. **GSI throttling does not affect base table** (but base table throttling can delay GSI updates)

### GSI write sharding for hot keys

If a GSI partition key is a "hot key" (e.g., `status: pending` has millions of items), the GSI partition throttles. Mitigation: add a shard suffix:

```js
// Add random suffix to GSI PK for write distribution
// Original: status = "pending"
// Sharded:  status = "pending#0", "pending#1", ..., "pending#9"

// In application, query all shards and merge:
const shards = ['pending#0', 'pending#1', ..., 'pending#9']
const promises = shards.map(shard =>
  docClient.send(new QueryCommand({
    TableName: 'Orders',
    IndexName: 'StatusGlobalIndex',
    KeyConditionExpression: 'status = :status',
    ExpressionAttributeValues: { ':status': shard }
  }))
)
const results = await Promise.all(promises)
```

---

## 2. Consistency models

### Eventually consistent reads (default)

- **Cost:** 0.5 RCU per 4 KB
- **Latency:** Low (~1-5 ms)
- **Staleness:** ~1 second maximum (usually much less)
- **Use when:** stale data is acceptable (social feeds, product catalogs)

### Strongly consistent reads

- **Cost:** 1 RCU per 4 KB
- **Latency:** Higher (requires consensus)
- **Behavior:** Always returns the latest committed data
- **Use when:** stale data is unacceptable (account balances, inventory counts)
- **Limitation:** Can fail with 500 error during partition moves or maintenance

```js
// Strongly consistent read
const result = await docClient.send(new GetCommand({
  TableName: 'Accounts',
  Key: { account_id: 'acc_1001' },
  ConsistentRead: true
}))
```

**Trap:** Strongly consistent reads are not supported on GSIs. If you need strong consistency on a GSI query, model your data so that the base table's primary key covers the access pattern.

### Transactional reads and writes

Since DynamoDB **transactions** (2018):

- **transactGet:** 2 RCU per item per transaction
- **transactWrite:** 2 WCU per item per transaction
- Support up to 100 items or 4 MB per transaction
- **No rollback on error** — if a condition fails, the entire transaction is rejected (doesn't partially apply)

```js
// Transactional write — transfer funds
await docClient.send(new TransactWriteCommand({
  TransactItems: [
    {
      Update: {
        TableName: 'Accounts',
        Key: { account_id: 'acc_1001' },
        UpdateExpression: 'SET balance = balance - :amount',
        ConditionExpression: 'balance >= :amount',
        ExpressionAttributeValues: { ':amount': 100 }
      }
    },
    {
      Update: {
        TableName: 'Accounts',
        Key: { account_id: 'acc_1002' },
        UpdateExpression: 'SET balance = balance + :amount',
        ExpressionAttributeValues: { ':amount': 100 }
      }
    }
  ]
}))
```

**Trap:** Transactions cost **2x RCU/WCU** per item. Only use them when you truly need ACID across multiple items.

---

## 3. DynamoDB Streams

### Overview

DynamoDB Streams captures item-level changes in near-real-time:

- **24-hour retention** of stream records
- **Ordered per partition** (not globally)
- **Record format:** Kinesis-compatible
- **View types:** KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES

### Stream records

```json
{
  "eventID": "1",
  "eventName": "INSERT",  // INSERT, MODIFY, REMOVE
  "eventVersion": "1.0",
  "eventSource": "aws:dynamodb",
  "awsRegion": "us-east-1",
  "dynamodb": {
    "Keys": { "user_id": { "S": "u1001" } },
    "NewImage": { "user_id": { "S": "u1001" }, "name": { "S": "Alice" }, "balance": { "N": "100" } },
    "OldImage": { "user_id": { "S": "u1001" }, "name": { "S": "Alice" }, "balance": { "N": "200" } },
    "SequenceNumber": "111",
    "SizeBytes": 26,
    "StreamViewType": "NEW_AND_OLD_IMAGES"
  },
  "eventSourceARN": "arn:aws:dynamodb:us-east-1:123456789012:table/Accounts/stream/2024-01-15"
}
```

### View types

| View Type | Captures | Use Case |
|-----------|----------|----------|
| `KEYS_ONLY` | Only the primary key attributes | Minimal processing, dedup |
| `NEW_IMAGE` | Entire item after the modification | Triggers, caches, search indexing |
| `OLD_IMAGE` | Entire item before the modification | Audit logs, undo |
| `NEW_AND_OLD_IMAGES` | Both before and after | Full change tracking |

### Lambda trigger

```js
// Lambda handler for DynamoDB Streams
exports.handler = async (event) => {
  for (const record of event.Records) {
    console.log('Event ID:', record.eventID)
    console.log('Event Name:', record.eventName)

    if (record.eventName === 'INSERT') {
      const newItem = record.dynamodb.NewImage
      // Process new item — update search index, send notification, etc.
      await updateSearchIndex(newItem)
    }

    if (record.eventName === 'MODIFY') {
      const oldItem = record.dynamodb.OldImage
      const newItem = record.dynamodb.NewImage
      // Detect specific field changes
      if (oldItem.balance.N !== newItem.balance.N) {
        await publishBalanceChange(newItem)
      }
    }

    if (record.eventName === 'REMOVE') {
      const oldItem = record.dynamodb.OldImage
      // Cleanup associated resources
      await deleteFromCache(oldItem)
    }
  }
}
```

### Stream + Lambda use cases

| Use case | Pattern |
|----------|---------|
| **Materialized view** | Stream → Lambda → Elasticsearch (search index) |
| **Cache invalidation** | Stream → Lambda → Redis (remove cached item) |
| **Event-driven** | Stream → Lambda → SQS/SNS (notifications) |
| **Cross-region replication** | Stream → Lambda → DynamoDB in other region (custom) |
| **Audit log** | Stream → Lambda → S3 / CloudWatch Logs |

### Kinesis Data Streams for DynamoDB (2023+)

Newer alternative to DynamoDB Streams:
- **Longer retention** (up to 365 days vs 24 hours)
- **Higher throughput** (no shard limits)
- **Kinesis Client Library integration** (standard KCL consumers)
- **Cost:** Pay for Kinesis stream + DynamoDB stream records

---

## 4. DAX (DynamoDB Accelerator)

DAX is an in-memory cache for DynamoDB that provides **microsecond latency**:

```
Application → DAX cluster (cache) → DynamoDB (fallback)
              ↓
         Microsecond reads
         Write-through (updates DynamoDB, then caches)
```

### DAX key features

| Feature | Detail |
|---------|--------|
| **Latency** | Microseconds (vs single-digit ms for DynamoDB) |
| **Consistency** | Eventually consistent (writes go to DynamoDB first, then cache) |
| **Write mode** | Write-through (writes to DynamoDB + DAX) |
| **TTL** | Items in DAX expire based on TTL |
| **Cluster** | Primary + replicas (multi-AZ) |
| **Node types** | cache.r5.large to cache.r5.24xlarge |

```js
// DAX client (replaces regular DynamoDB client)
const AmazonDaxClient = require('amazon-dax-client')
const daxClient = new AmazonDaxClient({
  endpoints: ['my-cluster.dax-clusters.us-east-1.amazonaws.com:8111'],
  region: 'us-east-1'
})
const docClient = DynamoDBDocumentClient.from(daxClient)

// Same API as regular DynamoDB — drop-in replacement
const result = await docClient.send(new GetCommand({
  TableName: 'Sessions',
  Key: { session_id: token }
}))
```

**Trap:** DAX only accelerates **GetItem**, **BatchGetItem**, and **Query**. **Scan** and **PutItem/UpdateItem/DeleteItem** still go to DynamoDB (but writes are cached). DAX does **not** support strongly consistent reads or transactions.

**When to use DAX:**
- Read-heavy workloads with eventual consistency tolerance
- Session stores, product catalogs, leaderboards
- When DynamoDB single-digit ms latency is not enough

**When NOT to use DAX:**
- Strong consistency required
- Write-heavy workloads (DAX adds cost with minimal benefit)
- Small tables that already fit in DynamoDB's internal cache

---

## 5. TTL (Time to Live)

DynamoDB automatically deletes items after a specified timestamp:

```bash
# Enable TTL on a table (point to an attribute)
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification Enabled=true,AttributeName=ttl

# The ttl attribute must be a Unix epoch timestamp in seconds
```

```js
// Insert item with TTL (expire in 24 hours)
const ttl = Math.floor(Date.now() / 1000) + 86400
await docClient.send(new PutCommand({
  TableName: 'Sessions',
  Item: {
    session_id: token,
    user_id: 'u1001',
    ttl: ttl  // DynamoDB will delete this item after ttl
  }
}))
```

### TTL behavior

- Items are **marked for deletion** within ~48 hours of TTL expiry
- TTL deletions do **not** consume WCU
- TTL deletions **do** appear in DynamoDB Streams (with `eventName: REMOVE`)
- TTL items are **not returned** in Query/Scan results (filtered out)
- TTL items still count toward table storage until deleted

**Trap:** TTL is not exact. Items may persist up to 48 hours after TTL expiry. If you need exact expiry (e.g., legal compliance), use a scheduled job to delete items.

**Use cases:** Session expiry, temp credentials, event data, OTP codes, password reset tokens.

---

## 6. Optimistic locking

DynamoDB doesn't have native pessimistic locking. Instead, use optimistic locking with versions:

```js
// 1. Read item with version
const result = await docClient.send(new GetCommand({
  TableName: 'Products',
  Key: { product_id: 'p1001' }
}))
const item = result.Item  // { product_id: 'p1001', stock: 50, version: 3 }

// 2. Update with condition (only if version hasn't changed)
try {
  await docClient.send(new UpdateCommand({
    TableName: 'Products',
    Key: { product_id: 'p1001' },
    UpdateExpression: 'SET stock = stock - :dec, #v = #v + :one',
    ConditionExpression: '#v = :expected_version AND stock >= :dec',
    ExpressionAttributeNames: { '#v': 'version' },
    ExpressionAttributeValues: {
      ':dec': 1,
      ':one': 1,
      ':expected_version': 3
    }
  }))
} catch (err) {
  if (err.name === 'ConditionalCheckFailedException') {
    // Version conflict — retry or reject
    console.log('Concurrent update detected, retrying...')
  }
}
```

---

## 7. PartiQL (SQL-compatible queries)

PartiQL provides SQL-compatible syntax for DynamoDB operations:

```js
// SELECT
const result = await docClient.send(new ExecuteStatementCommand({
  Statement: 'SELECT * FROM Orders WHERE user_id = ? AND order_date BETWEEN ? AND ?',
  Parameters: ['u1001', '2024-01-01', '2024-12-31']
}))

// INSERT (PutItem)
await docClient.send(new ExecuteStatementCommand({
  Statement: 'INSERT INTO Orders VALUE {\'user_id\': ?, \'order_date\': ?, \'total\': ?}',
  Parameters: ['u1001', '2024-06-15', 100]
}))

// UPDATE
await docClient.send(new ExecuteStatementCommand({
  Statement: 'UPDATE Orders SET total = ? WHERE user_id = ? AND order_date = ?',
  Parameters: [150, 'u1001', '2024-06-15']
}))

// DELETE
await docClient.send(new ExecuteStatementCommand({
  Statement: 'DELETE FROM Orders WHERE user_id = ? AND order_date = ?',
  Parameters: ['u1001', '2024-06-15']
}))
```

**Trap:** PartiQL does not support JOINs, subqueries, or complex SQL constructs. It's a thin SQL layer over DynamoDB's native API. For complex queries, stick with the native API.

---

## 8. DynamoDB Local

Test DynamoDB locally without connecting to AWS:

```bash
# Install via Docker
docker run -p 8000:8000 amazon/dynamodb-local

# Access at http://localhost:8000
# Use with --endpoint-url flag in CLI
aws dynamodb list-tables --endpoint-url http://localhost:8000
```

```js
// Local DynamoDB client
const client = new DynamoDBClient({
  region: 'us-east-1',
  endpoint: 'http://localhost:8000',
  credentials: {
    accessKeyId: 'fake',        // not validated locally
    secretAccessKey: 'fake'
  }
})
```

---

## 9. Practical drills

### Drill 1 — Design a GSI for status queries

Given a table `Orders` (PK: `user_id`, SK: `order_date`), you need to:
- Query all orders by status (across all users)
- Query orders by status sorted by creation date
- Get recent pending orders (last 7 days)

Design the GSI.

<details>
<summary>Answer</summary>

```bash
# GSI: StatusByDateIndex
# PK: status (String)
# SK: order_date (String) — ISO format for alphabetical sorting

aws dynamodb update-table \
  --table-name Orders \
  --attribute-definitions AttributeName=status,AttributeType=S AttributeName=order_date,AttributeType=S \
  --global-secondary-index-updates \
    "[{\"Create\":{\"IndexName\":\"StatusByDateIndex\",\"KeySchema\":[{\"AttributeName\":\"status\",\"KeyType\":\"HASH\"},{\"AttributeName\":\"order_date\",\"KeyType\":\"RANGE\"}],\"Projection\":{\"ProjectionType\":\"ALL\"},\"ProvisionedThroughput\":{\"ReadCapacityUnits\":10,\"WriteCapacityUnits\":5}}}]"
```

Query:
```js
// Recent pending orders (last 7 days)
const sevenDaysAgo = new Date(Date.now() - 7 * 86400000).toISOString().split('T')[0]

await docClient.send(new QueryCommand({
  TableName: 'Orders',
  IndexName: 'StatusByDateIndex',
  KeyConditionExpression: 'status = :status AND order_date >= :since',
  ExpressionAttributeValues: {
    ':status': 'pending',
    ':since': sevenDaysAgo
  },
  ScanIndexForward: false  // newest first
}))
```
</details>

### Drill 2 — Stream + Lambda for real-time processing

Design a DynamoDB Stream → Lambda pipeline for your trading platform. When a trade is inserted, the system should:
1. Update a real-time leaderboard in Redis
2. Publish trade event to SNS for WebSocket broadcasting
3. Archive the trade to S3

<details>
<summary>Answer</summary>

**Table:** `Trades` (PK: `trade_id`)  
**Stream:** `NEW_IMAGE` (we only need the new item)

```js
// Lambda handler
exports.handler = async (event) => {
  const redis = require('redis')
  const SNS = new AWS.SNS()
  const S3 = new AWS.S3()

  for (const record of event.Records) {
    if (record.eventName !== 'INSERT') continue

    const trade = AWS.DynamoDB.Converter.unmarshall(record.dynamodb.NewImage)

    // 1. Update Redis leaderboard (trader ranking by P&L)
    await redis.zincrby('leaderboard:daily', trade.pnl, `trader:${trade.trader_id}`)

    // 2. Publish to SNS for WebSocket broadcast
    await SNS.publish({
      TopicArn: process.env.TRADE_TOPIC_ARN,
      Message: JSON.stringify({
        type: 'NEW_TRADE',
        trade: trade
      })
    }).promise()

    // 3. Archive to S3
    await S3.putObject({
      Bucket: process.env.TRADE_ARCHIVE_BUCKET,
      Key: `trades/${trade.trade_id}.json`,
      Body: JSON.stringify(trade)
    }).promise()
  }
}
```
</details>

### Drill 3 — Transactional order placement

Design a transactional order placement system:
- Check if product has sufficient inventory
- Decrement inventory
- Create the order
- All three must succeed atomically

<details>
<summary>Answer</summary>

```js
await docClient.send(new TransactWriteCommand({
  TransactItems: [
    {
      Update: {
        TableName: 'Inventory',
        Key: { product_id: 'p1001', warehouse: 'WH-01' },
        UpdateExpression: 'SET stock = stock - :qty, reserved = reserved + :qty',
        ConditionExpression: 'stock >= :qty',
        ExpressionAttributeValues: { ':qty': 2, ':reserved': 2 }
      }
    },
    {
      Put: {
        TableName: 'Orders',
        Item: {
          order_id: 'ORD-001',
          user_id: 'u1001',
          product_id: 'p1001',
          qty: 2,
          status: 'confirmed',
          created_at: new Date().toISOString()
        },
        ConditionExpression: 'attribute_not_exists(order_id)'
      }
    }
  ]
}))
```

Both operations succeed or fail together. If inventory is insufficient, no order is created.
</details>

---

## Interview traps cheatsheet — Intermediate

| Trap | The truth |
|------|-----------|
| "GSIs are strongly consistent" | GSIs are **eventually consistent only**. Only the base table and LSIs support strong consistency. |
| "DynamoDB transactions are like SQL transactions" | Similar ACID guarantees but 2x RCU/WCU cost, max 100 items, no nested transactions. |
| "DAX makes everything faster" | DAX only accelerates GetItem, BatchGetItem, and Query. Scan is not cached. |
| "TTL deletion is instant" | TTL items can persist up to 48 hours. Not suitable for exact-time deletion requirements. |
| "Stream records are ordered globally" | Ordered **per partition key** only. Across partitions, order is not guaranteed. |
| "PartiQL replaces the native API" | PartiQL is a thin wrapper. No JOINs, no complex SQL. Most use cases still need native API. |
| "Optimistic locking prevents all conflicts" | Reduces conflicts but at high concurrency, retries balloon. Consider queueing writes. |
| "Stream retention is configurable" | Fixed at 24 hours. No extensions. Use Kinesis Data Streams for DynamoDB for longer. |
| "Batch operations are transactional" | BatchGetItem and BatchWriteItem are not atomic. Use TransactGetItems/TransactWriteItems for atomic batches. |
| "DynamoDB Local is identical to production" | Close but not identical. Provisioned throughput throttling, auto-scaling, and Global Tables don't work locally. |
</details>
