# DynamoDB — Question Bank

> **Target:** Senior Backend Engineer interview preparation  
> **Format:** Rapid-fire Q&A, code puzzles, debugging scenarios, system design prompts, STAR templates  
> **Real-world anchors:** Multi-tenant SaaS (single-table design), trading platform (time-series trade data), Chronos (job state with TTL and GSIs), AWS serverless stack

---

## 1. Rapid-fire Q&A (140+ questions)

### Fundamentals (25 questions)

1. **Q:** What is DynamoDB?  
   **A:** Fully managed NoSQL key-value and document database by AWS.

2. **Q:** What is an item in DynamoDB?  
   **A:** A single record in a table. Max 400 KB (including attribute names).

3. **Q:** What are the two types of primary keys?  
   **A:** Partition key only (simple PK) and partition key + sort key (composite PK).

4. **Q:** What does the partition key determine?  
   **A:** Which physical partition the item is stored on (via hash).

5. **Q:** What does the sort key do?  
   **A:** Orders items with the same partition key. Enables range queries (BETWEEN, >, <, begins_with).

6. **Q:** What is the max item size?  
   **A:** 400 KB (including attribute names).

7. **Q:** Is DynamoDB schemaless?  
   **A:** Yes — items in the same table can have different attributes. Only the primary key is required.

8. **Q:** What is RCU?  
   **A:** Read Capacity Unit — 1 strongly consistent read of 4 KB/sec, 2 eventually consistent reads of 4 KB/sec.

9. **Q:** What is WCU?  
   **A:** Write Capacity Unit — 1 write of 1 KB/sec.

10. **Q:** How do you calculate RCU for a strongly consistent read of a 10 KB item?  
    **A:** Ceil(10/4) = 3 RCU.

11. **Q:** How do you calculate WCU for a 2.5 KB write?  
    **A:** Ceil(2.5/1) = 3 WCU.

12. **Q:** What is provisioned capacity?  
    **A:** You specify RCU/WCU. Pay per hour. Can auto-scale.

13. **Q:** What is on-demand capacity?  
    **A:** Pay per request. No capacity planning needed. More expensive per operation.

14. **Q:** What is burst capacity?  
    **A:** Unused RCU/WCU accumulates for up to 5 minutes for burst usage.

15. **Q:** What happens when you exceed provisioned capacity?  
    **A:** Throttling — `ProvisionedThroughputExceededException`.

16. **Q:** What is the difference between Query and Scan?  
    **A:** Query reads items with the same partition key (efficient). Scan reads every item (expensive).

17. **Q:** Does FilterExpression reduce RCU?  
    **A:** No — filter is applied after reading. You pay for all read RCU.

18. **Q:** What is LastEvaluatedKey?  
    **A:** Pagination token returned when a Query/Scan has more results.

19. **Q:** What is ConditionalExpression?  
    **A:** Condition checked before write. If false, operation fails (ConditionalCheckFailedException).

20. **Q:** Does a failed conditional write consume WCU?  
    **A:** No — it does consume RCU for the condition check.

21. **Q:** What is the default consistency model?  
    **A:** Eventually consistent.

22. **Q:** What is strongly consistent read?  
    **A:** Returns latest committed data. 2x RCU cost. Can fail during partition moves.

23. **Q:** What is the max read throughput per partition?  
    **A:** 3,000 RCU per partition.

24. **Q:** What is the max write throughput per partition?  
    **A:** 1,000 WCU per partition.

25. **Q:** What is the max storage per partition?  
    **A:** 10 GB per partition.

### Indexes (15 questions)

26. **Q:** What is a Local Secondary Index (LSI)?  
    **A:** Same PK as base table, different SK. Created at table creation. Max 5 per table.

27. **Q:** What is a Global Secondary Index (GSI)?  
    **A:** Different PK + SK from base table. Created anytime. Max 20 per table.

28. **Q:** Can LSIs be created after table creation?  
    **A:** No — LSIs must be created at table creation time.

29. **Q:** Can GSIs be created after table creation?  
    **A:** Yes — GSIs can be created anytime.

30. **Q:** Do GSIs support strongly consistent reads?  
    **A:** No — GSIs are eventually consistent only. Only base table and LSIs support strong consistency.

31. **Q:** What are the three projection types?  
    **A:** KEYS_ONLY, INCLUDE (specific attributes), ALL.

32. **Q:** What is the benefit of using `INCLUDE` projection?  
    **A:** Lower write costs (GSI only stores projected attributes, not the full item).

33. **Q:** Does an LSI have separate throughput?  
    **A:** No — LSI shares throughput with the base table.

34. **Q:** Does a GSI have separate throughput?  
    **A:** Yes — GSI has its own provisioned throughput.

35. **Q:** What is the 10 GB limit per partition key?  
    **A:** An LSI's partition key items (same PK as base table) can't exceed 10 GB total across base + LSIs.

36. **Q:** What is GSI write sharding?  
    **A:** Adding random suffix to the GSI PK to distribute writes across partitions (mitigates hot keys).

37. **Q:** What is a sparse index?  
    **A:** A GSI that only indexes items that have the GSI key attributes. Used to index a subset of items.

38. **Q:** Do empty attributes get indexed by GSIs?  
    **A:** No — GSIs only index items that have a non-empty value for the GSI key attributes.

39. **Q:** How many GSIs can a table have by default?  
    **A:** 20 (soft limit — can request more).

40. **Q:** What happens to GSIs when you update the base table?  
    **A:** GSI updates are asynchronous. Short propagation delay before the GSI reflects the change.

### Streams (10 questions)

41. **Q:** What is DynamoDB Streams?  
    **A:** Captures item-level changes (INSERT, MODIFY, REMOVE) in near-real-time.

42. **Q:** How long are stream records retained?  
    **A:** 24 hours.

43. **Q:** What are the four stream view types?  
    **A:** KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES.

44. **Q:** Are stream records globally ordered?  
    **A:** No — ordered per partition key. Across partitions, order is not guaranteed.

45. **Q:** What is a typical use case for DynamoDB Streams?  
    **A:** Lambda triggers for: cache invalidation, search indexing, cross-region replication, event-driven processing.

46. **Q:** Can you consume DynamoDB Streams without Lambda?  
    **A:** Yes — you can use the Kinesis Adapter to consume with KCL.

47. **Q:** What is Kinesis Data Streams for DynamoDB?  
    **A:** (2023+) Alternative stream destination with longer retention (up to 365 days) and higher throughput.

48. **Q:** Does TTL deletion appear in streams?  
    **A:** Yes — as an event with `eventName: REMOVE`.

49. **Q:** What happens to a stream when you delete a table?  
    **A:** The stream is also deleted.

50. **Q:** Can you have multiple consumers on the same stream?  
    **A:** Yes — each Lambda trigger or KCL consumer reads independently.

### DAX (8 questions)

51. **Q:** What is DAX?  
    **A:** DynamoDB Accelerator — in-memory cache for microsecond reads.

52. **Q:** What operations does DAX accelerate?  
    **A:** GetItem, BatchGetItem, Query. Scan is NOT accelerated.

53. **Q:** Is DAX strongly consistent?  
    **A:** No — DAX serves eventually consistent data (write-through from DynamoDB).

54. **Q:** Does DAX support transactions?  
    **A:** No.

55. **Q:** When should you use DAX?  
    **A:** Read-heavy workloads, eventual consistency acceptable, need < 1 ms reads.

56. **Q:** When should you NOT use DAX?  
    **A:** Write-heavy, strong consistency required, small tables that fit in DynamoDB cache.

57. **Q:** How does DAX handle writes?  
    **A:** Write-through — writes go to DynamoDB first, then the cache is updated.

58. **Q:** Does DAX require application changes?  
    **A:** Minimal — drop-in replacement for the DynamoDB client (same API).

### TTL (5 questions)

59. **Q:** What is DynamoDB TTL?  
    **A:** Automatic item deletion after a specified timestamp (Unix epoch seconds).

60. **Q:** How soon after TTL expiry is an item deleted?  
    **A:** Within 48 hours (typically much faster, but not guaranteed).

61. **Q:** Does TTL deletion consume WCU?  
    **A:** No — TTL deletions are free.

62. **Q:** Does TTL deletion appear in Streams?  
    **A:** Yes — as a REMOVE event.

63. **Q:** Can you recover a TTL-deleted item?  
    **A:** If PITR is enabled, you can restore to a point before the TTL expiry.

### Transactions (6 questions)

64. **Q:** What is TransactWriteItems?  
    **A:** Atomic write of up to 100 items across multiple tables.

65. **Q:** What is TransactGetItems?  
    **A:** Consistent read of up to 100 items across multiple tables.

66. **Q:** How much does a transactional read cost?  
    **A:** 2 RCU per item (vs 1 for strongly consistent, 0.5 for eventual).

67. **Q:** How much does a transactional write cost?  
    **A:** 2 WCU per item (vs 1 for standard).

68. **Q:** If one operation in a transaction fails, what happens?  
    **A:** The entire transaction is rejected. No partial execution.

69. **Q:** Can transactions span multiple tables?  
    **A:** Yes — up to 100 items across multiple tables in the same region.

### DynamoDB Streams + Lambda (6 questions)

70. **Q:** How do you trigger a Lambda from DynamoDB Streams?  
    **A:** Create an Event Source Mapping in Lambda, pointing to the DynamoDB Stream ARN.

71. **Q:** What is the batch size for DynamoDB Streams → Lambda?  
    **A:** Default 100 (max 10,000). Number of records per Lambda invocation.

72. **Q:** What happens if a Lambda fails to process a batch?  
    **A:** The batch is retried based on the Lambda retry policy (max 2 additional attempts, then sent to DLQ).

73. **Q:** What is the starting position for a DynamoDB Stream trigger?  
    **A:** `LATEST` (new records only), `TRIM_HORIZON` (all available), or `AT_TIMESTAMP`.

74. **Q:** Can one stream trigger multiple Lambdas?  
    **A:** Yes — each Lambda creates its own Event Source Mapping.

75. **Q:** What is `eventName` in a stream record?  
    **A:** `INSERT`, `MODIFY`, or `REMOVE`.

### Global Tables (5 questions)

76. **Q:** What are Global Tables?  
    **A:** Multi-region active-active replication.

77. **Q:** What is the conflict resolution strategy?  
    **A:** Last-writer-wins (based on timestamp).

78. **Q:** What is required to create a Global Table?  
    **A:** Streams must be enabled on the table.

79. **Q:** Is data in Global Tables strongly consistent?  
    **A:** No — eventually consistent. Last-writer-wins.

80. **Q:** What is the typical replication latency?  
    **A:** < 1 second across regions (typically).

### Backup and PITR (5 questions)

81. **Q:** What is on-demand backup?  
    **A:** Full table backup. Restores to a new table.

82. **Q:** What is PITR?  
    **A:** Point-in-Time Recovery — 35-day continuous backup, restore to any second.

83. **Q:** Does PITR restore replace the original table?  
    **A:** No — restores to a **new** table.

84. **Q:** What is the cost of PITR?  
    **A:** Additional storage cost for incremental backup data.

85. **Q:** Can you export DynamoDB data to S3?  
    **A:** Yes (2023+) — export without consuming RCU. Supports DynamoDB JSON and Amazon Ion formats.

### Performance and Operations (15 questions)

86. **Q:** How do partitions scale in DynamoDB?  
    **A:** Auto-split when throughput exceeds 3,000 RCU or 1,000 WCU or 10 GB per partition.

87. **Q:** What is the partition count formula?  
    **A:** MAX(ceil(RCU/3000), ceil(WCU/1000), ceil(SizeGB/10)).

88. **Q:** What is a hot key?  
    **A:** A partition key that receives disproportionate traffic and causes throttling.

89. **Q:** How do you mitigate hot keys?  
    **A:** Shard the key (random suffix), use DAX, cache in Redis, distribute writes across more keys.

90. **Q:** What is auto-scaling?  
    **A:** Application Auto Scaling adjusts RCU/WCU based on utilization. Target tracking with cooldown.

91. **Q:** What is the default auto-scaling cooldown?  
    **A:** Scale-out: 60 seconds. Scale-in: 60 seconds.

92. **Q:** When should you use on-demand over provisioned?  
    **A:** Unpredictable traffic, new apps, spiky workloads. Simpler but more expensive.

93. **Q:** What is DynamoDB Standard vs Standard-IA table class?  
    **A:** Standard-IA offers ~60% lower storage cost for tables accessed less than once per month.

94. **Q:** How do you monitor DynamoDB throttling?  
    **A:** CloudWatch: `ThrottledRequests`, `ReadThrottleEvents`, `WriteThrottleEvents`.

95. **Q:** What is CloudWatch Contributor Insights for DynamoDB?  
    **A:** Identifies which partition keys are causing throttling.

96. **Q:** How do you handle large items in DynamoDB?  
    **A:** Store large data in S3, reference by URL. Max item is 400 KB.

97. **Q:** What is the DynamoDB Fluent API?  
    **A:** Builder pattern for DynamoDB operations in the AWS SDK.

98. **Q:** What is PartiQL?  
    **A:** SQL-compatible query language for DynamoDB. Supports SELECT, INSERT, UPDATE, DELETE.

99. **Q:** Does PartiQL support JOINs?  
    **A:** No — PartiQL is a thin SQL layer over DynamoDB's native API.

100. **Q:** What is DynamoDB Local?  
     **A:** Local Docker container for testing without AWS. Works at localhost:8000.

### Single-table Design (15 questions)

101. **Q:** What is single-table design?  
     **A:** One DynamoDB table for an entire application. Entity types encoded in PK/SK prefixes.

102. **Q:** What is the first step in single-table design?  
     **A:** Identify all access patterns (queries) first, then design the schema.

103. **Q:** What is a composite sort key?  
     **A:** SK that encodes multiple dimensions (e.g., `STATUS#pending#DATE#2024-06-15`).

104. **Q:** What is the adjacency list pattern?  
     **A:** Store relationships as items with PK/SK encoding both directions (e.g., `FOLLOWS#user` and `FOLLOWED_BY#user`).

105. **Q:** What is a sparse GSI?  
     **A:** A GSI that only indexes items with a specific attribute. Used for filtering subsets.

106. **Q:** What is an overloaded GSI?  
     **A:** A single GSI used for multiple entity types (GSI1PK and GSI1SK have different meanings per entity type).

107. **Q:** Why shorten attribute names?  
     **A:** Attribute names count toward the 400 KB item limit. Short names save space.

108. **Q:** What is a transaction in single-table design?  
     **A:** TransactWrite across multiple items (possibly different entities) in the same table.

109. **Q:** How do you model one-to-many in single-table design?  
     **A:** User with orders: PK=USER#u1001, SK=ORDER#date#order_id. Query with `SK begins_with 'ORDER#'`.

110. **Q:** How do you model many-to-many?  
     **A:** Adjacency list: two items per relationship (forward + inverse).

111. **Q:** How do you model hierarchical data (org → user, org → product)?  
     **A:** PK=ORG#org_42, SK=USER#u1001 or SK=PRODUCT#p1001. Query begins_with for each entity type.

112. **Q:** What is the 10 GB per partition limitation's impact on single-table?  
     **A:** An LSI uses the same PK, so items sharing that PK can't exceed 10 GB.

113. **Q:** How do you handle full-text search in single-table design?  
     **A:** You don't — use DynamoDB Streams → Lambda → Elasticsearch for full-text.

114. **Q:** What is the cold start challenge in single-table design?  
     **A:** None directly — single-table is about schema, not infrastructure.

115. **Q:** What is the biggest anti-pattern in single-table design?  
     **A:** Designing an RDBMS schema first and translating to DynamoDB. Always start with access patterns.

### General / Senior (15 questions)

116. **Q:** When would you choose DynamoDB over RDS?  
     **A:** Simple key-value access, high throughput, automatic scaling, serverless, no complex queries.

117. **Q:** When would you choose RDS over DynamoDB?  
     **A:** Complex queries (JOINs), relational integrity, ad-hoc analytics, full-text search.

118. **Q:** What is DynamoDB's biggest weakness?  
     **A:** Limited query flexibility (no JOINs, no complex WHERE), single-table design requires upfront access pattern analysis.

119. **Q:** How does DynamoDB compare to Cassandra?  
     **A:** DynamoDB: fully managed, simpler API, AWS-native. Cassandra: self-managed, CQL (richer), multi-cloud.

120. **Q:** What is the difference between DynamoDB and ScyllaDB?  
     **A:** ScyllaDB is Cassandra-compatible (C++), lower latency (no JVM). DynamoDB is fully managed.

121. **Q:** How does DynamoDB compare to MongoDB?  
     **A:** DynamoDB: simpler query model, auto-scaling, less flexibility. MongoDB: richer queries, manual sharding, more features.

122. **Q:** What is the Consistency Mode in Lambda Event Source Mapping?  
     **A:** Not applicable — DynamoDB Streams delivers records at least once. Handle duplicates.

123. **Q:** How do you handle duplicate stream records?  
     **A:** Idempotent processing. Use `eventID` as a dedup key or condition expressions on writes.

124. **Q:** What is the batch window in DynamoDB Streams → Lambda?  
     **A:** Maximum time to wait before invoking Lambda with a batch (default 0, max 300 seconds).

125. **Q:** What is DynamoDB's SLA?  
     **A:** 99.99% availability for Standard tables.

126. **Q:** How many tables can you have per account?  
     **A:** 2,500 tables per region (default soft limit).

127. **Q:** What is the maximum number of GSIs per table?  
     **A:** 20 (default).

128. **Q:** What is the maximum number of LSIs per table?  
     **A:** 5.

129. **Q:** Can you delete a GSI?  
     **A:** Yes — `UpdateTable` with `Delete` action for the GSI.

130. **Q:** Can you add an LSI after table creation?  
     **A:** No — you must create a new table and migrate.

### AWS CLI and SDK (10 questions)

131. **Q:** How do you list tables via AWS CLI?  
     **A:** `aws dynamodb list-tables`.

132. **Q:** How do you describe a table?  
     **A:** `aws dynamodb describe-table --table-name Orders`.

133. **Q:** How do you get an item via CLI?  
     **A:** `aws dynamodb get-item --table-name Orders --key '{"user_id":{"S":"u1001"}}'`.

134. **Q:** How do you put an item via CLI?  
     **A:** `aws dynamodb put-item --table-name Users --item '{"user_id":{"S":"u1001"},"name":{"S":"Alice"}}'`.

135. **Q:** How do you query via CLI?  
     **A:** `aws dynamodb query --table-name Orders --key-condition-expression "user_id = :uid" --expression-attribute-values '{":uid":{"S":"u1001"}}'`.

136. **Q:** How do you update capacity?  
     **A:** `aws dynamodb update-table --table-name Orders --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=5`.

137. **Q:** How do you enable PITR?  
     **A:** `aws dynamodb update-continuous-backups --table-name Orders --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true`.

138. **Q:** How do you enable TTL?  
     **A:** `aws dynamodb update-time-to-live --table-name Sessions --time-to-live-specification Enabled=true,AttributeName=ttl`.

139. **Q:** How do you create a table with an LSI?  
     **A:** Pass `--local-secondary-indexes` with the key schema and projection.

140. **Q:** How do you create a GSI on an existing table?  
     **A:** `aws dynamodb update-table --table-name Orders --attribute-definitions ... --global-secondary-index-updates "[{\"Create\":{...}}]"`.

---

## 2. Code puzzles (10 puzzles)

### Puzzle 1 — Create a table with LSI + GSI

Create a table `Orders` with:
- PK: `user_id` (String)
- SK: `order_date` (String)
- LSI: `StatusIndex` with SK: `status`
- GSI: `OrderLookup` with PK: `order_id`

<details>
<summary>Answer</summary>

```bash
aws dynamodb create-table \
  --table-name Orders \
  --key-schema AttributeName=user_id,KeyType=HASH AttributeName=order_date,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=user_id,AttributeType=S \
    AttributeName=order_date,AttributeType=S \
    AttributeName=status,AttributeType=S \
    AttributeName=order_id,AttributeType=S \
  --local-secondary-indexes \
    "[{\"IndexName\":\"StatusIndex\",\"KeySchema\":[{\"AttributeName\":\"user_id\",\"KeyType\":\"HASH\"},{\"AttributeName\":\"status\",\"KeyType\":\"RANGE\"}],\"Projection\":{\"ProjectionType\":\"ALL\"}}]" \
  --global-secondary-indexes \
    "[{\"IndexName\":\"OrderLookup\",\"KeySchema\":[{\"AttributeName\":\"order_id\",\"KeyType\":\"HASH\"}],\"Projection\":{\"ProjectionType\":\"ALL\"},\"ProvisionedThroughput\":{\"ReadCapacityUnits\":5,\"WriteCapacityUnits\":5}}]" \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10
```
</details>

### Puzzle 2 — Query with filtering and pagination

Query the `Orders` table for user `u1001`'s orders in June 2024 with total > $50. Paginate 10 at a time.

<details>
<summary>Answer</summary>

```js
async function queryOrders(userId, startDate, endDate, minTotal, lastKey = null) {
  const result = await docClient.send(new QueryCommand({
    TableName: 'Orders',
    KeyConditionExpression: 'user_id = :uid AND order_date BETWEEN :start AND :end',
    FilterExpression: 'total > :min_total',
    ExpressionAttributeValues: {
      ':uid': userId,
      ':start': startDate,
      ':end': endDate,
      ':min_total': minTotal
    },
    Limit: 10,
    ExclusiveStartKey: lastKey
  }))

  return {
    items: result.Items,
    lastKey: result.LastEvaluatedKey
  }
}

// First page
const page1 = await queryOrders('u1001', '2024-06-01', '2024-06-30', 50)

// Second page (if hasMore)
if (page1.lastKey) {
  const page2 = await queryOrders('u1001', '2024-06-01', '2024-06-30', 50, page1.lastKey)
}
```
</details>

### Puzzle 3 — Transactional inventory update

Design a transactional operation for placing an order: decrement inventory and create order. Both must succeed atomically.

<details>
<summary>Answer</summary>

```js
await docClient.send(new TransactWriteCommand({
  TransactItems: [
    {
      Update: {
        TableName: 'Inventory',
        Key: { product_id: 'p1001', warehouse: 'WH-01' },
        UpdateExpression: 'SET stock = stock - :qty',
        ConditionExpression: 'stock >= :qty',
        ExpressionAttributeValues: { ':qty': 2 }
      }
    },
    {
      Put: {
        TableName: 'Orders',
        Item: {
          order_id: `ORD-${Date.now()}`,
          user_id: 'u1001',
          product_id: 'p1001',
          qty: 2,
          total: 59.98,
          status: 'confirmed',
          created_at: new Date().toISOString()
        },
        ConditionExpression: 'attribute_not_exists(order_id)'
      }
    }
  ]
}))
```
</details>

### Puzzle 4 — Optimistic locking

Design an optimistic locking mechanism for updating a product's price.

<details>
<summary>Answer</summary>

```js
// Read
const result = await docClient.send(new GetCommand({
  TableName: 'Products',
  Key: { product_id: 'p1001' }
}))
const currentVersion = result.Item.version

// Update with version check
try {
  await docClient.send(new UpdateCommand({
    TableName: 'Products',
    Key: { product_id: 'p1001' },
    UpdateExpression: 'SET price = :new_price, #v = #v + :one',
    ConditionExpression: '#v = :expected_version',
    ExpressionAttributeNames: { '#v': 'version' },
    ExpressionAttributeValues: {
      ':new_price': 34.99,
      ':one': 1,
      ':expected_version': currentVersion
    }
  }))
} catch (err) {
  if (err.name === 'ConditionalCheckFailedException') {
    // Retry: re-read the latest version and try again
    return await updatePriceWithRetry('p1001', 34.99)
  }
  throw err
}
```
</details>

### Puzzle 5 — Single-table design: e-commerce

Design a single DynamoDB table for an e-commerce system with these access patterns:
1. Get user by ID
2. Get user's orders sorted by date
3. Get all products for an org
4. Get orders by status across all orgs
5. Get products by category

<details>
<summary>Answer</summary>

```js
// Table: ECommerce

// Entity keys:
// PK = entity identifier
// SK = entity type + identifier

// GSI1: for cross-entity queries
// GSI1PK = org_id or status or category
// GSI1SK = entity type + timestamp

// Users
{ PK: 'USER#u1001',        SK: 'PROFILE',          name: 'Alice', email: 'alice@example.com', org_id: 'org_42', role: 'admin' }

// Orders (user's orders)
{ PK: 'USER#u1001',        SK: 'ORDER#2024-06-15#ORD-001', total: 100, status: 'pending', GSI1PK: 'STATUS#pending', GSI1SK: 'ORDER#2024-06-15#ORD-001' }

// Products by org
{ PK: 'ORG#org_42',        SK: 'PROD#p1001',       name: 'Widget', price: 29.99, category: 'tools', stock: 100, GSI1PK: 'CAT#tools', GSI1SK: 'PROD#p1001' }
```

Access patterns:
```js
// 1. Get user by ID
// Query: PK = 'USER#u1001', SK = 'PROFILE'

// 2. Get user's orders
// Query: PK = 'USER#u1001', SK begins_with 'ORDER#'
// ScanIndexForward: false (newest first)

// 3. Get org's products
// Query: PK = 'ORG#org_42', SK begins_with 'PROD#'

// 4. Orders by status (GSI1)
// Query: GSI1PK = 'STATUS#pending', GSI1SK begins_with 'ORDER#'

// 5. Products by category (GSI1)
// Query: GSI1PK = 'CAT#tools', GSI1SK begins_with 'PROD#'
```
</details>

### Puzzle 6 — Composite sort key design

Design a composite sort key that allows querying:
- Orders by status (pending, shipped, completed)
- Orders by status + date range
- Orders by status + date + user (for sorting)

<details>
<summary>Answer</summary>

```js
// Composite SK: STATUS#<status>#DATE#<date>#USER#<user_id>
SK = 'STATUS#pending#DATE#2024-06-15#USER#u1001'

// Query all orders for a user:
PK = 'USER#u1001', SK begins_with 'ORDER#'

// Query pending orders:
// Using GSI: PK = status, SK = composite
GSI PK = 'STATUS#pending', SK begins_with 'DATE#2024-06'  // all pending in June

// Query pending orders sorted by date:
GSI PK = 'STATUS#pending', ScanIndexForward: true
// Sort key is alphabetical: 'DATE#2024-01-01' < 'DATE#2024-06-15'

// This works because ISO dates sort alphabetically
```
</details>

### Puzzle 7 — Stream + Lambda for search indexing

Design a DynamoDB Stream → Lambda pipeline that indexes product data into Elasticsearch.

<details>
<summary>Answer</summary>

```js
// Lambda: indexer.js
const { Client } = require('@elastic/elasticsearch')
const esClient = new Client({ node: process.env.ES_ENDPOINT })

exports.handler = async (event) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT' || record.eventName === 'MODIFY') {
      const item = AWS.DynamoDB.Converter.unmarshall(record.dynamodb.NewImage)

      await esClient.index({
        index: 'products',
        id: item.product_id,
        body: {
          name: item.name,
          description: item.description,
          price: item.price,
          category: item.category,
          org_id: item.org_id,
          in_stock: item.stock > 0,
          updated_at: new Date().toISOString()
        }
      })
    }

    if (record.eventName === 'REMOVE') {
      const keys = AWS.DynamoDB.Converter.unmarshall(record.dynamodb.Keys)

      await esClient.delete({
        index: 'products',
        id: keys.product_id
      })
    }
  }
}
```
</details>

### Puzzle 8 — GSI hot key mitigation

You have a GSI with PK = `status`. The value `"pending"` is extremely hot (80% of items). How do you mitigate?

<details>
<summary>Answer</summary>

**Strategy: Write shard the GSI partition key**

```js
// Instead of:
GSI1PK = status  // "pending" — all goes to one partition

// Shard with random suffix:
GSI1PK = `${status}#${Math.floor(Math.random() * 10)}`
// Items distributed across "pending#0" through "pending#9"

// Query all shards and merge:
async function queryPendingOrders() {
  const shards = Array.from({ length: 10 }, (_, i) => `pending#${i}`)

  const results = await Promise.all(
    shards.map(shard =>
      docClient.send(new QueryCommand({
        TableName: 'Orders',
        IndexName: 'StatusGSI',
        KeyConditionExpression: 'GSI1PK = :pk',
        ExpressionAttributeValues: { ':pk': shard }
      }))
    )
  )

  return results.flatMap(r => r.Items)
}
```

**Alternative approaches:**
1. DAX — cache the query result (not good for write-heavy hot keys)
2. SQS queue — buffer writes to spread throughput
3. On-demand capacity — absorbs spikes without throttling
</details>

### Puzzle 9 — DynamoDB as state store for Chronos

Design a DynamoDB schema for Chronos's distributed job scheduler state.

<details>
<summary>Answer</summary>

```js
// Table: ChronosJobs

// Job definition
{ PK: 'JOB#email-welcome',  SK: 'DEF',            type: 'email', payload: '{"template":"welcome"}', schedule: 'cron(0 9 * * *)', max_retries: 3 }

// Job instance (each run)
{ PK: 'JOB#email-welcome',  SK: 'RUN#2024-06-15T09:00:00Z', status: 'running', worker: 'worker-1', started_at: '2024-06-15T09:00:00Z', heartbeat: '2024-06-15T09:00:30Z' }

// Scheduled jobs (due notifications)
{ PK: 'SCHEDULE#2024-06-15', SK: 'JOB#email-welcome', job_id: 'email-welcome', run_at: '2024-06-15T09:00:00Z' }

// Worker state
{ PK: 'WORKER#worker-1',    SK: 'STATE',           status: 'active', last_heartbeat: '2024-06-15T09:00:30Z', assigned_jobs: 5 }

// GSI: Find jobs by status
// GSI1PK = status, GSI1SK = run_at
GSI1PK: 'RUNNING', GSI1SK: '2024-06-15T09:00:00Z#JOB#email-welcome'

// TTL for completed jobs (delete after 7 days)
{ PK: 'JOB#email-welcome',  SK: 'RUN#...', status: 'completed', ttl: Math.floor(Date.now()/1000) + 604800 }
```

**Access patterns:**
```js
// Find due jobs: Query PK = 'SCHEDULE#2024-06-15', SK >= NOW
// Claim job: UpdateItem with ConditionExpression on status
// Stale workers: Query GSI for RUNNING jobs with heartbeat < NOW - 30s
```
</details>

### Puzzle 10 — Billing and capacity calculation

A table has:
- Item size: 8 KB
- Average read size: 6 KB
- 500 reads/second (70% eventual, 30% strong)
- 200 writes/second
- 1 transaction write batch per second (2 items of 4 KB each)

How many RCU and WCU to provision?

<details>
<summary>Answer</summary>

**Reads:**
- Eventual: 500 × 0.7 = 350 reads/sec
  - 6 KB → ceil(6/4) = 2 RCU per strong, 1 RCU per eventual
  - 350 × 1 = 350 RCU
- Strong: 500 × 0.3 = 150 reads/sec
  - 150 × 2 = 300 RCU
- Total reads: 350 + 300 = **650 RCU**

**Writes:**
- Standard writes: 200/sec
  - 8 KB → ceil(8/1) = 8 WCU per write
  - 200 × 8 = 1,600 WCU
- Transactional writes: 1 batch/sec with 2 items of 4 KB
  - 4 KB → ceil(4/1) = 4 WCU per item
  - Transactional cost: 2 × 4 × 2 = 16 WCU
- Total writes: 1,600 + 16 = **1,616 WCU**

**Provisioned:** 650 RCU, 1,616 WCU (round to 650 and 1,600 or 2,000 depending on how you round)
</details>

---

## 3. Debugging scenarios (5 scenarios)

### Scenario 1 — Throttled writes on a specific partition

**Symptom:** `ProvisionedThroughputExceededException` for write operations. Only affects one type of item.

**Debugging:**
1. Check CloudWatch `ThrottledRequests` metric
2. Use Contributor Insights to identify the throttled partition key
3. Check if a specific key (e.g., `org_42`, `user_viral`, `status:pending`) receives most traffic

**Likely cause:** A hot partition key receiving more than 1,000 WCU.

**Solution:**
1. **Write sharding:** Add random suffix to the hot key
2. **DAX:** Not applicable (writes still go to DynamoDB)
3. **On-demand:** Switch temporarily to handle spikes
4. **Queue:** Buffer writes with SQS to spread throughput

### Scenario 2 — GSI throttling affecting base table

**Symptom:** Base table writes fail with throttling errors, but base table RCU/WCU is below limit.

**Debugging:**
1. Check GSI throttling metrics: `WriteThrottleEvents` for the GSI
2. Check GSI provisioned throughput — is it lower than needed?
3. Check GSI projection size — `ALL` projection costs more WCU per write

**Solution:**
1. Increase GSI provisioned throughput
2. Reduce GSI projection from `ALL` to `KEYS_ONLY` or `INCLUDE`
3. Shard the GSI partition key
4. Remove unnecessary GSIs

### Scenario 3 — Slow reads even within capacity

**Symptom:** GetItem and Query latency increased from 5ms to 50ms. Table is not throttled.

**Debugging:**
1. Check item size — items growing close to 400 KB?
2. Check partition distribution — are some partitions overloaded?
3. Check for large items with many attributes
4. Check DAX (if used) — is it warmed up?

**Likely causes:**
- Items too large (> 100 KB) — multiple RCUs per read
- "Uber partition" — one partition key has many items, query returns many results
- Network latency between app and DynamoDB (cross-AZ)

**Solution:**
1. Keep items small (< 10 KB ideally)
2. Use `ProjectionExpression` to return only needed attributes
3. Consider DAX for sub-millisecond reads
4. Use Global Tables for geographic proximity

### Scenario 4 — GSI still backfilling

**Symptom:** After creating a new GSI, queries on the GSI return empty or partial results. This persists for hours.

**Debugging:**
1. Check GSI status: `aws dynamodb describe-table --table-name Orders | jq .Table.GlobalSecondaryIndexes[].IndexStatus`
2. If status is `CREATING` — GSI is still backfilling data

**Solution:**
- **Wait** — GSI backfill can take hours for large tables
- The GSI becomes available only when status changes to `ACTIVE`
- You can query during backfill, but results are partial

**Trap:** Creating a GSI on a large table (100s of GB) can take **hours to days**. Plan ahead.

### Scenario 5 — Recovery from accidental delete

**Symptom:** Someone accidentally deleted important items or a whole table.

**Solution:**

**If table is deleted:**
1. Restore from latest PITR backup: `aws dynamodb restore-table-to-point-in-time --source-table-name Orders --target-table-name Orders-restored --use-latest-restorable-time`
2. Or restore from on-demand backup: `aws dynamodb restore-table-from-backup --target-table-name Orders-restored --backup-arn ...`

**If items are deleted (TTL or explicit):**
1. If PITR enabled: restore table to a point before the deletion
2. Export the restored items and re-insert them

**Prevention:**
- Enable PITR on all production tables
- Use IAM permissions to restrict delete operations
- Use CloudFormation for infrastructure-as-code (replace on delete)

---

## 4. System design prompts (4 prompts)

### Prompt 1 — Design a DynamoDB schema for a multi-tenant SaaS

Design a DynamoDB schema for a multi-tenant inventory management SaaS (modeled on your experience).

**Access patterns:**
1. Get tenant settings
2. List tenant's products (paginated, filterable by category)
3. List tenant's users
4. Get user's profile
5. Get user's order history (sorted by date)
6. Get all pending orders for a tenant
7. Get product by SKU

<details>
<summary>Answer</summary>

```js
// Table: SaaS
// Billing: PAY_PER_REQUEST (or provisioned with auto-scaling)

// GSI1 for cross-entity queries (tenant-scoped)
// GSI1PK = org_id, GSI1SK = entity_type + ID

// 1. Tenant settings
{ PK: 'ORG#org_42',        SK: 'SETTINGS',        name: 'Acme Corp', plan: 'enterprise', features: ['real_time', 'reports'] }

// 2. Products (sorted by SKU within org)
{ PK: 'ORG#org_42',        SK: 'PROD#WDG-001',    product_id: 'p1001', name: 'Widget', price: 29.99, category: 'tools', stock: 100, GSI1PK: 'CAT-tools', GSI1SK: 'PROD#WDG-001' }

// 3. Users within org
{ PK: 'ORG#org_42',        SK: 'USER#u1001',      name: 'Alice', email: 'alice@acme.com', role: 'admin' }

// 4. User profile
{ PK: 'USER#u1001',        SK: 'PROFILE',         name: 'Alice', email: 'alice@acme.com', org_id: 'org_42', role: 'admin' }

// 5. Orders (user's orders sorted by date)
{ PK: 'USER#u1001',        SK: 'ORDER#2024-06-15#ORD-001', total: 100, status: 'pending', product_id: 'p1001', org_id: 'org_42', GSI1PK: 'STATUS#pending', GSI1SK: 'ORDER#2024-06-15#ORD-001' }

// 6. Pending orders query: GSI1PK = 'STATUS#pending'
// 7. Product lookup: PK = 'ORG#org_42', SK = 'PROD#WDG-001'

// Index: GSI1 (PK: GSI1PK, SK: GSI1SK)
// For category queries: Query GSI1PK = "CAT-tools"
// For status queries: Query GSI1PK = "STATUS#pending"
```
</details>

### Prompt 2 — Design a DynamoDB-based leaderboard

Design a real-time trading leaderboard using DynamoDB.

**Requirements:**
- Update trader P&L on every trade
- Show top 100 traders
- Show a trader's rank
- 10K+ updates/sec during peak

<details>
<summary>Answer</summary>

**Option 1: Single table with daily leaderboard**

```js
// Table: Leaderboard

// Daily leaderboard (sorted by score descending)
{ PK: 'LEADERBOARD#2024-06-15', SK: 'TRADER#u1001', score: 15000, name: 'Alice', rank: 1 }

// Get top 100: Query PK = 'LEADERBOARD#2024-06-15', ScanIndexForward: false, Limit: 100
// Get trader rank: GetItem PK = 'LEADERBOARD#2024-06-15', SK = 'TRADER#u1001'

// Update score:
await docClient.send(new UpdateCommand({
  TableName: 'Leaderboard',
  Key: { PK: 'LEADERBOARD#2024-06-15', SK: 'TRADER#u1001' },
  UpdateExpression: 'ADD score :pnl',
  ExpressionAttributeValues: { ':pnl': 100 }
}))

// To get rank, you need another approach (no native ordering)
// Option: Use a sorted set (Redis) for the real-time leaderboard
// DynamoDB persists the data, Redis provides O(log N) rank queries
```

**Option 2: Redis for real-time, DynamoDB for persistence**

```js
// Trade executes:
// 1. Redis: ZINCRBY leaderboard:daily:2024-06-15 100 trader:u1001
// 2. DynamoDB: UpdateItem (async, via Stream)

// Top 100: ZREVRANGE leaderboard:daily:2024-06-15 0 99 WITHSCORES
// Rank: ZREVRANK leaderboard:daily:2024-06-15 trader:u1001
```

**DynamoDB alone challenge:** Getting rank requires reading all items, counting those with higher scores. For a top 100, better to pre-compute top 100 in a sorted set (Redis).
</details>

### Prompt 3 — Design an event-driven order processing system

Design a DynamoDB-based order processing system using Streams and Lambda.

**Requirements:**
- Order placed → send confirmation email → update inventory → send to warehouse
- Each step is decoupled
- Reliability (no lost messages)
- Retry on failure

<details>
<summary>Answer</summary>

**Architecture:**
```
API Gateway → CreateOrder Lambda → DynamoDB Orders Table
                                          ↓
                                    DynamoDB Streams
                                          ↓
                          ┌────────────────┼────────────────┐
                          ↓                ↓                ↓
                    Email Lambda    Inventory Lambda    Warehouse Lambda
                          ↓                ↓                ↓
                      SES/SQS        UpdateItem         SQS/Warehouse API
```

**Order schema:**
```js
{ PK: 'ORDER#ORD-001', SK: 'META',
  user_id: 'u1001',
  items: [{ product_id: 'p1001', qty: 2, price: 29.99 }],
  total: 59.98,
  status: 'pending',
  processing: {
    email_sent: false,
    inventory_updated: false,
    warehouse_notified: false
  }
}
```

**Stream processor (fan-out pattern):**
```js
// Each Lambda handles its own responsibility
// Lambda checks the processing flags before acting

const record = AWS.DynamoDB.Converter.unmarshall(event.Records[0].dynamodb.NewImage)

// Email Lambda
if (!record.processing.email_sent) {
  await sendEmail(record)
  await markProcessingComplete(record, 'email_sent')
}

// Inventory Lambda
if (!record.processing.inventory_updated) {
  await updateInventory(record)
  await markProcessingComplete(record, 'inventory_updated')
}
```

**Reliability:**
- Stream records are at-least-once — handle duplicates idempotently
- Each Lambda has DLQ for failed records
- Processing flags prevent duplicate work on retry
- Dead letter queue for manual inspection

### Prompt 4 — Design data migration from RDBMS to DynamoDB

Design a strategy for migrating from PostgreSQL to DynamoDB for your SaaS.

**Requirements:**
- Zero-downtime migration
- Data consistency during cutover
- Application code changes

<details>
<summary>Answer</summary>

**Phase 1 — Analyze access patterns**
```js
// Identify how the application queries data
// Each query becomes a DynamoDB access pattern
// Design the single-table schema based on these patterns
```

**Phase 2 — Dual-write**
```js
// Application writes to both PostgreSQL and DynamoDB
// Reads still come from PostgreSQL initially

async function createOrder(orderData) {
  await db.query('INSERT INTO orders ...', orderData)    // PostgreSQL
  await dynamoClient.putItem({ TableName: 'Orders', ... })  // DynamoDB (async)
}
```

**Phase 3 — Backfill**
```js
// Batch export from PostgreSQL, import to DynamoDB
// Use parallel scan with DynamoDB's BatchWriteItem

const { Pool } = require('pg')
const pool = new Pool({ ... })

async function backfill() {
  const result = await pool.query('SELECT * FROM orders')
  const batches = chunk(result.rows, 25)  // Max 25 items per BatchWrite

  for (const batch of batches) {
    await docClient.send(new BatchWriteCommand({
      RequestItems: {
        Orders: batch.map(item => ({
          PutRequest: { Item: transformItem(item) }
        }))
      }
    }))
  }
}
```

**Phase 4 — Validation**
```js
// Compare counts and sample records
const pgCount = await pool.query('SELECT COUNT(*) FROM orders')
const ddbCount = await docClient.send(new DescribeTableCommand({ TableName: 'Orders' }))

console.log('PostgreSQL:', pgCount.rows[0].count)
console.log('DynamoDB:', ddbCount.Table.ItemCount)
```

**Phase 5 — Cutover**
1. Stop writes to PostgreSQL (maintenance window if needed)
2. Final sync of any missed data
3. Switch application reads to DynamoDB
4. Drop PostgreSQL tables after verification

**Phase 6 — Cleanup**
- Remove dual-write code
- Retire PostgreSQL instances
- Monitor DynamoDB performance and adjust capacity
</details>

---

## 5. STAR story templates

### Story: Migrating from PostgreSQL to DynamoDB for trading platform

**Situation:** The trading platform's order book was stored in PostgreSQL. As volume grew to 10K trades/hour, write contention caused deadlocks and latency. PostgreSQL CPU was consistently > 80%.

**Task:** Move to a write-scalable database with sub-10ms latency for both writes and reads.

**Action:**
- Analyzed all access patterns: write trade, read trade by ID, read recent trades by symbol, aggregate daily volume
- Designed single-table DynamoDB schema with composite sort keys for time-series access
- Used DynamoDB Streams → Lambda for real-time aggregation into OHLCV pre-computed items
- Set up auto-scaling with on-demand for peak trading hours
- Implemented dual-write phase for 1 week, then cut over

**Result:** Write latency dropped from 50ms to 5ms. No throttling during 3x volume spikes. P99 read latency under 10ms. Database ops cost reduced by 40%.

### Story: Stream-based search indexing

**Situation:** The SaaS product search was slow, using PostgreSQL `LIKE` queries. Users complained about search quality.

**Task:** Implement full-text search without changing the existing DynamoDB-backed API.

**Action:**
- Set up DynamoDB Streams on the Products table
- Created a Lambda function that reads stream records and indexes into Elasticsearch
- Used NEW_AND_OLD_IMAGES to handle updates and deletes
- Configured batch size of 500 and parallelization factor of 10
- Added error handling with DLQ for failed index operations

**Result:** Search latency dropped from 3s to 50ms. Product discovery improved (search-as-you-type, faceting, relevance scoring). Dual-write cost was zero (Streams are built-in).

---

## 6. Key metrics to remember

| Metric | Target | Why |
|--------|--------|-----|
| RCU utilization | < 80% | Over > 80% = risk of throttling on burst |
| WCU utilization | < 80% | Over > 80% = risk of throttling on burst |
| Throttled requests | 0 | Throttling = poor user experience |
| GSI throttling | 0 | GSI throttling back-impacts base table |
| Item size | < 10 KB | Large items consume more RCU/WCU |
| Read latency (DDB) | < 10ms | Higher = need DAX or query optimization |
| Write latency (DDB) | < 10ms | Higher = partition split or hot key |
| Stream lag | < 1 second | Higher = consumer can't keep up |
| Partitions per table | Balanced | Skewed = hot partitions |
| PITR backup window | < 24 hours | Recovery point objective |

---

## 7. Interview preparation checklist

- [ ] I can explain DynamoDB's partition model and how it scales
- [ ] I can calculate RCU/WCU for any read/write pattern
- [ ] I can design an LSI and a GSI given access patterns
- [ ] I understand the difference between LSI and GSI (created, throughput, consistency, limit)
- [ ] I can write CRUD operations with the AWS SDK
- [ ] I can write a Query with sort key conditions, filter, and pagination
- [ ] I understand when to use eventual vs strong consistency
- [ ] I can design a single-table schema from access patterns
- [ ] I can implement DynamoDB Streams + Lambda processing
- [ ] I know how to mitigate hot keys (write sharding)
- [ ] I understand DAX, TTL, transactions, and optimistic locking
- [ ] I can compare DynamoDB with Cassandra, MongoDB, and RDS
- [ ] I can design a Global Table for multi-region
- [ ] I understand PITR and backup strategies
- [ ] I can design a serverless application with DynamoDB + Lambda + API Gateway
