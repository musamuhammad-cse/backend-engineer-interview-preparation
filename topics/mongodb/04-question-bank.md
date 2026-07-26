# MongoDB — Question Bank

> **Target:** Senior Backend Engineer interview preparation  
> **Format:** Rapid-fire Q&A, code puzzles, debugging scenarios, system design prompts, STAR templates  
> **Real-world anchors:** Multi-tenant SaaS (schema design, sharding), trading platform (high-throughput writes), Chronos (change streams, job state)

---

## 1. Rapid-fire Q&A (140+ questions)

### Fundamentals (25 questions)

1. **Q:** What is BSON and how does it differ from JSON?  
   **A:** BSON is binary-encoded serialization of JSON. It supports more data types (Date, ObjectId, Int32, Int64, Decimal128, Binary) and is optimized for space and scan speed over readability.

2. **Q:** What is the maximum document size in MongoDB?  
   **A:** 16 MB. For larger files, use GridFS which splits data into 255 KB chunks.

3. **Q:** What is the `_id` field and how is it generated?  
   **A:** It's the primary key. If not provided, MongoDB generates an ObjectId (12 bytes: 4-byte timestamp, 5-byte random, 3-byte increment).

4. **Q:** Can you change the `_id` after document creation?  
   **A:** No, `_id` is immutable.

5. **Q:** How does MongoDB handle schema validation?  
   **A:** MongoDB uses `$jsonSchema` validation with `validationAction: "error"` or `"warn"`. Schema is validated on insert and update (not existing docs).

6. **Q:** What is the difference between `insertOne` and `insertMany` with ordered: false?  
   **A:** `ordered: true` (default) inserts sequentially, stops at first error. `ordered: false` inserts in parallel, skips errors, continues with remaining.

7. **Q:** What operators are used for array element updates?  
   **A:** `$push`, `$pop`, `$pull`, `$pullAll`, `$addToSet`, positional `$`, `$[]`, `$[identifier]`.

8. **Q:** How does MongoDB implement upsert?  
   **A:** Using `{ upsert: true }` in `updateOne` — creates a new document if no match, otherwise updates.

9. **Q:** What is `$setOnInsert` used for?  
   **A:** Sets fields only during insert (when upsert creates a new document), does nothing on update.

10. **Q:** What is the difference between `$elemMatch` and a regular array query?  
    **A:** `$elemMatch` ensures all conditions match the same array element. Without it, conditions can match different elements.

11. **Q:** How does MongoDB handle projection?  
    **A:** `{ field: 1 }` includes; `{ field: 0 }` excludes. `_id` is always included unless explicitly set to 0.

12. **Q:** What is the difference between `find()` and `aggregate()`?  
    **A:** `find()` returns documents matching a filter with sort/skip/limit/projection. `aggregate()` is a pipeline of stages for complex transformations.

13. **Q:** What are the cursor methods in MongoDB?  
    **A:** `.sort()`, `.skip()`, `.limit()`, `.count()`, `.forEach()`, `.map()`, `.toArray()`.

14. **Q:** Do the order of `.sort()`, `.skip()`, `.limit()` affect execution?  
    **A:** No. MongoDB applies them in fixed order: sort → skip → limit regardless of method chain order.

15. **Q:** What is `getLastError` and is it still used?  
    **A:** Legacy (pre-2.6). Replaced by write concern.

16. **Q:** What is the default write concern?  
    **A:** `w: 1` — acknowledged by primary.

17. **Q:** How does MongoDB handle `null` values in queries?  
    **A:** `{ field: null }` matches documents where the field is null OR the field doesn't exist.

18. **Q:** What is the difference between `$exists: false` and `$eq: null`?  
    **A:** `$exists: false` matches documents missing the field. `$eq: null` matches documents where field is null OR missing.

19. **Q:** How do you rename a field in a document?  
    **A:** `db.collection.updateMany({}, { $rename: { "oldName": "newName" } })`.

20. **Q:** What is `$inc` and when should you use it?  
    **A:** Atomic increment — `{ $inc: { field: 5 } }`. Use for counters, scores, balances to avoid read-then-write race conditions.

21. **Q:** Can you use `$inc` with a negative value?  
    **A:** Yes — `{ $inc: { field: -5 } }` decrements.

22. **Q:** What happens to indexes when you drop a collection?  
    **A:** All indexes on that collection are also dropped.

23. **Q:** How do you delete all documents but keep the collection?  
    **A:** `db.collection.deleteMany({})`. `db.collection.remove({})` is deprecated.

24. **Q:** How do you drop a database?  
    **A:** `db.dropDatabase()`.

25. **Q:** How do you see all collections in a database?  
    **A:** `db.getCollectionNames()` or `show collections`.

### Indexes (20 questions)

26. **Q:** What is a compound index?  
    **A:** An index on multiple fields — `{ field1: 1, field2: -1 }`.

27. **Q:** Can a compound index support queries on the second field only?  
    **A:** No — the index prefix must be matched. `{ a: 1, b: 1 }` supports queries on `a` and `a + b` but not `b` alone.

28. **Q:** What is the ESR rule for compound indexes?  
    **A:** Equality fields → Sort field → Range field. Place equality-test fields first.

29. **Q:** What is a multikey index?  
    **A:** An index on an array field. Each array element gets its own index entry.

30. **Q:** How many multikey fields can a compound index have?  
    **A:** One. Indexing two array fields causes a cartesian product explosion.

31. **Q:** What is a covering query?  
    **A:** A query where all required fields are in the index — no document fetch needed (only IXSCAN, no FETCH).

32. **Q:** What is a TTL index?  
    **A:** Auto-expires documents after a specified duration. The field must be a date or array of dates.

33. **Q:** How often does TTL run?  
    **A:** Every 60 seconds. Document deletion can lag by up to 60 seconds.

34. **Q:** What is a sparse index?  
    **A:** Only indexes documents that contain the indexed field. Useful when many documents don't have the field.

35. **Q:** What is a partial index?  
    **A:** Indexes only documents matching a filter condition. More flexible than sparse.

36. **Q:** What is a unique index and its behavior with null?  
    **A:** `unique: true` treats `null` as a value. Only one document can have `{ field: null }` or missing field.

37. **Q:** How do you create a unique index that allows null?  
    **A:** Partial + unique: `partialFilterExpression: { field: { $exists: true } }`.

38. **Q:** What is `explain("executionStats")`?  
    **A:** Returns the query plan, including winning plan, rejected plans, execution time, documents scanned, and index keys examined.

39. **Q:** What does high `totalDocsExamined` relative to `nReturned` indicate?  
    **A:** Poor index selectivity — the query scans many documents to find few matches.

40. **Q:** What is an index intersection?  
    **A:** MongoDB uses multiple indexes to satisfy a query. Less efficient than a single compound index.

41. **Q:** What is a hashed index?  
    **A:** Indexes the hash of a field value. Used for hashed sharding. Does not support range queries.

42. **Q:** Can you create a text index on multiple fields?  
    **A:** Yes — `{ title: "text", body: "text" }`. You can set weights per field.

43. **Q:** How many text indexes can a collection have?  
    **A:** One. For multiple text search patterns, use Atlas Search.

44. **Q:** What is the background index build?  
    **A:** In MongoDB 4.2+, all index builds are in background by default (non-blocking).

45. **Q:** What is `$indexStats`?  
    **A:** `db.collection.aggregate([{ $indexStats: {} }])` shows access frequency for each index.

### Aggregation Pipeline (20 questions)

46. **Q:** What is the 100 MB memory limit in aggregation?  
    **A:** Each stage has a 100 MB RAM limit. Use `allowDiskUse: true` to bypass (but disk spill is slow).

47. **Q:** What is the optimal pipeline stage order?  
    **A:** `$match` → `$sort` → `$project` → `$lookup` → `$unwind` → `$group` → `$project` → `$sort` → `$limit`.

48. **Q:** Why should `$match` be the first stage in a pipeline?  
    **A:** It filters documents early and can use indexes, reducing the document count for subsequent stages.

49. **Q:** What does `$lookup` do?  
    **A:** Left outer join with another collection. Creates an array field on the source document.

50. **Q:** Is `$lookup` equivalent to SQL JOIN in performance?  
    **A:** No — `$lookup` performs one query per input document. For large datasets, it can be very slow.

51. **Q:** What is `$unwind` used for?  
    **A:** Deconstructs an array field into multiple documents (one per array element).

52. **Q:** What does `preserveNullAndEmptyArrays` do in `$unwind`?  
    **A:** If true, documents with null/missing/empty array fields are kept instead of dropped.

53. **Q:** What is `$group` used for?  
    **A:** Groups documents by a key expression and applies accumulator expressions.

54. **Q:** What accumulators does `$group` support?  
    **A:** `$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last`, `$stdDevPop`, `$stdDevSamp`.

55. **Q:** What is `$facet`?  
    **A:** Runs multiple pipelines in parallel on the same input. Limited by 16 MB output.

56. **Q:** What is `$bucket`?  
    **A:** Categorizes documents into buckets based on a value. Like histogram.

57. **Q:** What is `$bucketAuto`?  
    **A:** Automatically determines bucket boundaries to create a target number of buckets.

58. **Q:** What does `$out` do?  
    **A:** Writes the pipeline results to a collection (replaces entire collection).

59. **Q:** What does `$merge` do?  
    **A:** Merges pipeline results into a collection with configurable merge logic (insert, merge, replace, fail, keepExisting).

60. **Q:** What is `$replaceRoot`?  
    **A:** Promotes a subdocument to the top level of the document.

61. **Q:** What is `$addFields`?  
    **A:** Adds new fields or modifies existing fields in the document.

62. **Q:** What is `$set`?  
    **A:** Alias for `$addFields` (MongoDB 4.2+).

63. **Q:** What is `$graphLookup`?  
    **A:** Recursive graph traversal — queries across a collection to find connected documents (e.g., org hierarchy).

64. **Q:** What is `$setWindowFields`?  
    **A:** Window functions (MongoDB 5.0+) — running totals, moving averages, partitions.

65. **Q:** How do you debug a slow aggregation pipeline?  
    **A:** Use `.explain("executionStats")` to see stage execution times, document counts, and index usage.

### Replication (20 questions)

66. **Q:** What is a replica set?  
    **A:** A group of `mongod` instances that maintain the same data set for high availability.

67. **Q:** What is the minimum recommended replica set size?  
    **A:** 3 nodes (primary + 2 secondaries) or 2 nodes + arbiter (risk: no data redundancy on failover).

68. **Q:** What is an arbiter?  
    **A:** A voting member that holds no data. Participates in elections to maintain majority.

69. **Q:** How does a primary election work?  
    **A:** When the primary becomes unavailable, secondary detects (default 10s timeout), calls election, majority votes, winner becomes primary. Raft-based.

70. **Q:** What is the default election timeout?  
    **A:** `electionTimeoutMillis: 10000` (10 seconds).

71. **Q:** What is the oplog?  
    **A:** A capped collection in the `local` database that records all write operations. Secondaries tail this to replicate.

72. **Q:** What is the default oplog size?  
    **A:** 5% of free disk space (min 50 MB, max 237 MB for small disks).

73. **Q:** What happens when a secondary falls behind the oplog?  
    **A:** It enters RECOVERING state and must be resynchronized (restore from backup or full resync).

74. **Q:** What is write concern?  
    **A:** Controls how many members must acknowledge a write. `w: 1` (default), `w: "majority"`, `w: N`.

75. **Q:** What is read concern?  
    **A:** Controls consistency level: `"local"` (default, could be rolled back), `"majority"` (survives election), `"snapshot"` (5.0+).

76. **Q:** What is read preference?  
    **A:** Controls which member to read from: primary (default), secondary, primaryPreferred, secondaryPreferred, nearest.

77. **Q:** What is the risk of reading from secondaries?  
    **A:** Stale reads — secondary may lag behind primary. Not suitable for strong consistency requirements.

78. **Q:** What is a rollback?  
    **A:** When a former primary reconnects and has writes not on the new primary, those writes are rolled back and saved to `rollback/` directory.

79. **Q:** How do you minimize rollback?  
    **A:** Use `w: "majority"` write concern — writes must survive on majority before returning to client.

80. **Q:** What are retryable writes?  
    **A:** Auto-retry on network errors/elections (MongoDB 3.6+). Requires replica set.

81. **Q:** What is causal consistency?  
    **A:** Ensures read-your-writes and monotonic reads even with secondary reads (MongoDB 3.6+).

82. **Q:** How do you monitor replication lag?  
    **A:** `rs.status().members[].optimeDate` — compare each member's latest oplog time.

83. **Q:** What is the difference between a hidden and a delayed secondary?  
    **A:** Hidden: invisible to application, priority 0. Delayed: applies oplog with a delay (for disaster recovery).

84. **Q:** Can a hidden member become primary?  
    **A:** No — hidden members must have priority 0.

85. **Q:** How do you resize the oplog?  
    **A:** `db.adminCommand({ replSetResizeOplog: 1, size: 25600 })` (MB) — MongoDB 4.0+.

### Sharding (15 questions)

86. **Q:** What is a shard key?  
    **A:** The field(s) MongoDB uses to distribute documents across shards.

87. **Q:** What are the three criteria for a good shard key?  
    **A:** High cardinality, low frequency (even distribution), non-monotonically changing.

88. **Q:** What is the difference between hashed and ranged sharding?  
    **A:** Hashed: even distribution but range queries scatter. Ranged: good for range queries but can create hot shards with monotonic keys.

89. **Q:** What is a chunk?  
    **A:** A contiguous range of shard key values. Default size: 64 MB.

90. **Q:** What is the balancer?  
    **A:** Background process that migrates chunks across shards to maintain balance.

91. **Q:** When should you disable the balancer?  
    **A:** During peak traffic (migration adds overhead) or planned maintenance.

92. **Q:** What is a jumbo chunk?  
    **A:** A chunk where all documents share the same shard key value — cannot be split or moved.

93. **Q:** What causes jumbo chunks?  
    **A:** Low-cardinality shard key (e.g., `status` with only 3 values).

94. **Q:** How do you choose between hashed and ranged sharding?  
    **A:** Hashed for write-heavy uniform workloads. Ranged for range-query-heavy workloads.

95. **Q:** What is zone sharding?  
    **A:** Associates data ranges with specific shards for data locality (e.g., US data on US shards).

96. **Q:** What is a mongos?  
    **A:** The query router — routes client requests to appropriate shards.

97. **Q:** Are config servers a replica set?  
    **A:** Yes — config servers must run as a replica set (CSRS).

98. **Q:** Can you change the shard key after sharding?  
    **A:** No — shard key is immutable after sharding. Must re-shard by dumping and restoring.

99. **Q:** What is the recommended shard key for a new collection?  
    **A:** Hashed `_id` for general-purpose, or a compound key like `{ orgId: 1, _id: 1 }` for multi-tenant.

100. **Q:** What happens to transactions in a sharded cluster?  
     **A:** MongoDB 4.2+ supports distributed transactions across shards (2PC). Higher latency than single-shard transactions.

### Storage Engine / WiredTiger (10 questions)

101. **Q:** What is the default storage engine?  
     **A:** WiredTiger (since MongoDB 3.2).

102. **Q:** What concurrency model does WiredTiger use?  
     **A:** Document-level concurrency with MVCC — readers don't block writers.

103. **Q:** What is the default WiredTiger cache size?  
     **A:** 50% of (RAM - 1 GB). Minimum 256 MB.

104. **Q:** What compression does WiredTiger use by default?  
     **A:** Snappy compression for collection data, prefix compression for indexes.

105. **Q:** What is the journal in WiredTiger?  
     **A:** Write-ahead log for crash recovery. Commit interval: 100ms (default).

106. **Q:** How do you monitor cache pressure?  
     **A:** `db.serverStatus().wiredTiger.cache` — look at `tracked dirty bytes` and `pages written from cache`.

107. **Q:** What does high page fault rate indicate?  
     **A:** Working set doesn't fit in WiredTiger cache.

108. **Q:** What is the relationship between WiredTiger cache and transaction performance?  
     **A:** Modified documents in transactions are held in cache until commit. Low cache = poor transaction throughput.

109. **Q:** What are the supported block compressors?  
     **A:** Snappy (default), zlib, zstd (most compression, slowest).

110. **Q:** How does write-ahead log (journal) work?  
     **A:** Every write is journaled. On crash, MongoDB replays the journal to restore the last consistent state.

### Change Streams (10 questions)

111. **Q:** What are change streams?  
     **A:** Real-time event stream of data changes on a collection, database, or deployment (MongoDB 3.6+).

112. **Q:** What operations do change streams capture?  
     **A:** insert, update, replace, delete, invalidate, drop, dropDatabase, rename.

113. **Q:** Do change streams require replica sets?  
     **A:** Yes — change streams use the oplog.

114. **Q:** What is a resume token?  
     **A:** A `_data` field in change stream events that allows resuming from a specific point after a restart.

115. **Q:** What happens if the oplog rolls over past the resume token?  
     **A:** The change stream is invalidated with a `ChangeStreamHistoryLost` error. Must restart from current point.

116. **Q:** Can you filter change stream events?  
     **A:** Yes — use an aggregation pipeline with `$match` on `operationType` or `fullDocument` fields.

117. **Q:** What is `fullDocument: "updateLookup"`?  
     **A:** Returns the full document after the change (by looking it up from the collection).

118. **Q:** Are change streams ordered globally on a sharded cluster?  
     **A:** No — ordering is per-shard. No global order guarantee.

119. **Q:** What is a use case for change streams?  
     **A:** Cache invalidation, real-time sync, event-driven architecture, search indexing, audit logging.

120. **Q:** How do you use change streams with your Chronos scheduler?  
     **A:** Watch job state collection for status changes → trigger worker assignment, update dashboard, publish job completion events.

### Security (10 questions)

121. **Q:** How do you enable authentication in MongoDB?  
     **A:** `security.authorization: "enabled"` in mongod.conf, then create users via `db.createUser()`.

122. **Q:** What authentication mechanisms does MongoDB support?  
     **A:** SCRAM-SHA-256 (default), MONGODB-X509 (certificates), MONGODB-AWS (IAM), LDAP (Enterprise), KERBEROS (Enterprise).

123. **Q:** What is RBAC in MongoDB?  
     **A:** Role-Based Access Control — assign roles to users that define privileges on resources.

124. **Q:** What is the `root` role?  
     **A:** Superuser role with all privileges across all resources.

125. **Q:** How do you encrypt data at rest?  
     **A:** WiredTiger encryption at rest (Enterprise) or filesystem-level encryption (LUKS, EBS encryption).

126. **Q:** What is field-level encryption?  
     **A:** Client-side encryption of specific fields (MongoDB 4.2+). Data is encrypted before sent to server.

127. **Q:** How do you enable TLS for MongoDB?  
     **A:** `net.tls.mode: "requireTLS"` with certificate key file and CA file.

128. **Q:** What is SCRAM?  
     **A:** Salted Challenge Response Authentication Mechanism — MongoDB's default password-based auth (SHA-256).

129. **Q:** What privileges does the `readWrite` role provide?  
     **A:** Read and write on all non-system collections in the database.

130. **Q:** How do you audit database operations?  
     **A:** MongoDB Audit (Enterprise) or application-level audit using change streams.

### Operations / General (10 questions)

131. **Q:** What is the difference between `mongod`, `mongos`, and `mongo`?  
     **A:** `mongod` is the database server. `mongos` is the sharding router. `mongo` is the shell client.

132. **Q:** How do you back up a MongoDB database?  
     **A:** `mongodump` (logical), file system snapshot (block-level), Atlas backup (continuous).

133. **Q:** What is the difference between `mongodump` and file system snapshot?  
     **A:** `mongodump` exports BSON, slower but flexible. Snapshot is fast, crash-consistent.

134. **Q:** How do you monitor MongoDB?  
     **A:** `serverStatus()`, `dbStats()`, `collStats()`, `rs.status()`, `currentOp()`, profiler, MongoDB Cloud Manager/Atlas.

135. **Q:** What is the MongoDB profiler?  
     **A:** Logs operations exceeding a threshold. Enable: `db.setProfilingLevel(1, { slowms: 100 })`.

136. **Q:** How do you kill a long-running operation?  
     **A:** Find opid with `db.currentOp()`, kill with `db.killOp(opid)`.

137. **Q:** What is GridFS?  
     **A:** Specification for storing files > 16 MB. Stores file metadata in `fs.files` and chunks in `fs.chunks` (255 KB each).

138. **Q:** What is the difference between MongoDB and PostgreSQL for schema design?  
     **A:** MongoDB: embed or reference based on access patterns. PostgreSQL: normalize for relational integrity.

139. **Q:** What is a cursor timeout?  
     **A:** Cursors time out after 10 minutes of inactivity by default. Use `noCursorTimeout()` to override.

140. **Q:** What is `allowDiskUse` for?  
     **A:** Allows aggregation pipeline stages to spill to disk when exceeding 100 MB RAM limit.

141. **Q:** When should you NOT use MongoDB?  
     **A:** Complex relational queries with joins, strict schema enforcement, ACID across many documents, or mature tooling/ecosystem needed.

142. **Q:** What is the `$merge` stage alternative to `$out`?  
     **A:** `$merge` allows incremental updates (insert, merge, replace, fail, keepExisting). `$out` replaces the entire collection.

---

## 2. Code puzzles (10 puzzles)

### Puzzle 1 — Index the query

```js
// This query on a 50M-document collection takes 30 seconds:
db.orders.find({
  organization_id: "org_123",
  status: { $in: ["pending", "processing"] },
  created_at: { $gte: ISODate("2024-06-01") }
}).sort({ created_at: -1 }).limit(50)
```

Design the optimal index.

<details>
<summary>Answer</summary>

```js
db.orders.createIndex({ organization_id: 1, status: 1, created_at: -1 })
```

ESR: `organization_id` (equality) → `status` (equality, `$in` is multiple equality) → `created_at` (range + sort, -1 matches query sort direction).

Before: COLLSCAN on 50M documents.  
After: IXSCAN targeting only matching documents.
</details>

### Puzzle 2 — Aggregation for dashboard

You have a `transactions` collection. Write an aggregation that returns:
- Total revenue per month for 2024
- Running total (cumulative revenue)
- Compared to previous month (month-over-month growth)

Documents:
```json
{ "_id": 1, "amount": 100, "createdAt": ISODate("2024-01-15"), "type": "sale" }
```

<details>
<summary>Answer</summary>

```js
db.transactions.aggregate([
  { $match: {
      createdAt: { $gte: ISODate("2024-01-01"), $lt: ISODate("2025-01-01") },
      type: "sale"
    }
  },
  { $group: {
      _id: { year: { $year: "$createdAt" }, month: { $month: "$createdAt" } },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } },
  { $setWindowFields: {
      sortBy: { "_id.year": 1, "_id.month": 1 },
      output: {
        runningTotal: {
          $sum: "$revenue",
          window: { documents: ["unbounded", "current"] }
        },
        prevMonthRevenue: {
          $shift: { output: "$revenue", by: -1, default: 0 }
        }
      }
    }
  },
  { $addFields: {
      momGrowth: {
        $cond: {
          if: { $gt: ["$prevMonthRevenue", 0] },
          then: { $divide: [{ $subtract: ["$revenue", "$prevMonthRevenue"] }, "$prevMonthRevenue"] },
          else: null
        }
      }
    }
  }
])
```
</details>

### Puzzle 3 — Schema design: E-commerce order

Design a MongoDB schema for an e-commerce system with:
- Customers (name, email, addresses)
- Orders (items, total, shipping address, status, dates)
- Customers can have many orders
- Orders are queried by customer, by date range, and by status
- Items have product references, quantity, and price at time of order

<details>
<summary>Answer</summary>

```json
// Customer
{
  "_id": "c1",
  "name": "Alice",
  "email": "alice@example.com",
  "addresses": [
    { "type": "home", "street": "123 Main", "city": "NY", "zip": "10001" }
  ],
  "createdAt": ISODate("2024-01-01")
}

// Order — reference customer, embed order items
{
  "_id": "o1",
  "customerId": "c1",
  "customerEmail": "alice@example.com",  // denormalized for display
  "items": [
    { "productId": "p1", "name": "Widget", "qty": 2, "unitPrice": 29.99 },
    { "productId": "p2", "name": "Gadget", "qty": 1, "unitPrice": 49.99 }
  ],
  "subtotal": 109.97,
  "shipping": 5.99,
  "total": 115.96,
  "shippingAddress": { "street": "123 Main", "city": "NY", "zip": "10001" },
  "status": "shipped",
  "createdAt": ISODate("2024-06-15"),
  "shippedAt": ISODate("2024-06-16")
}

// Indexes
db.orders.createIndex({ customerId: 1, createdAt: -1 })           // customer order history
db.orders.createIndex({ status: 1, createdAt: -1 })                // admin order management
db.orders.createIndex({ createdAt: -1 })                            // recent orders
```
</details>

### Puzzle 4 — Find the bug: updateMany

```js
// Update all completed orders older than 90 days to "archived"
db.orders.updateMany(
  { status: "completed", created_at: { $lt: 90 } },
  { $set: { status: "archived" } }
)
```

<details>
<summary>Answer</summary>

`$lt: 90` compares `created_at` to the number 90, not "90 days ago". Use:

```js
const cutoff = new Date()
cutoff.setDate(cutoff.getDate() - 90)

db.orders.updateMany(
  { status: "completed", created_at: { $lt: cutoff } },
  { $set: { status: "archived" } }
)
```

Also consider a TTL index if archiving follows a fixed schedule.
</details>

### Puzzle 5 — Aggregation: Top customers with percentage

Return top 10 customers by total spend, including:
- Total spend
- Percentage of overall revenue
- Number of orders
- Average order value
- Last order date

<details>
<summary>Answer</summary>

```js
db.orders.aggregate([
  { $match: { status: { $ne: "cancelled" } } },
  { $group: {
      _id: "$customerId",
      totalSpend: { $sum: "$total" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$total" },
      lastOrderDate: { $max: "$createdAt" }
    }
  },
  { $sort: { totalSpend: -1 } },
  { $limit: 10 },
  { $group: {
      _id: null,
      topCustomers: { $push: "$$ROOT" },
      grandTotal: { $sum: "$totalSpend" }
    }
  },
  { $unwind: "$topCustomers" },
  { $addFields: {
      "topCustomers.percentage": {
        $round: [{ $multiply: [{ $divide: ["$topCustomers.totalSpend", "$grandTotal"] }, 100] }, 2]
      }
    }
  },
  { $replaceRoot: { newRoot: "$topCustomers" } },
  { $sort: { totalSpend: -1 } }
])
```
</details>

### Puzzle 6 — Replica set scenario

You have a 3-node replica set (P-S-S). The primary fails. What happens?

<details>
<summary>Answer</summary>

1. Within ~10 seconds (electionTimeoutMillis), secondaries detect primary is unreachable
2. Secondary with highest priority + most recent oplog calls an election
3. Majority (2 of 3 members) votes → one secondary becomes primary
4. If the old primary comes back, it steps down and becomes secondary
5. If the old primary had writes not replicated before the election, those writes are rolled back

Application impact:
- ~10-15 seconds of write unavailability
- Reads to primary fail; reads to secondaries continue (but might see stale data)
- Retryable writes on client auto-retry after election
- `w: "majority"` writes from before the election are safe
</details>

### Puzzle 7 — Schema migration pattern

You need to add a `phoneNumber` field to all existing `users` documents. Some users already have it, most don't. How do you handle this in a production system?

<details>
<summary>Answer</summary>

**Option 1 — Lazy migration (recommended):**
- Update application code to handle both schemas
- On access, if `phoneNumber` is missing, set it (with an update)
- Background job to backfill in batches

```js
// Batch backfill
let processed = 0
while (true) {
  const result = await db.users.updateMany(
    { phoneNumber: { $exists: false } },
    { $set: { phoneNumber: null } },
    { limit: 1000 }
  )
  processed += result.modifiedCount
  if (result.modifiedCount === 0) break
}
```

**Option 2 — Schema validation update:**
```js
db.runCommand({ collMod: "users", validator: { ... } })
// Validation only affects new inserts/updates, not existing docs
```

**For your multi-tenant SaaS:** Use `organization_id` filter in backfill to limit scope per tenant, monitor performance.
</details>

### Puzzle 8 — $lookup vs embedding decision

You have `posts` and `comments`. A post has up to 10,000 comments. Users view a post page showing the post + latest 20 comments. Should you embed or reference?

<details>
<summary>Answer</summary>

**Don't embed** — an unbounded array of 10K comments would:
- Blow past the 16 MB document limit (assuming average 500 bytes per comment = 5 MB, might fit but still poor)
- Make every post update load all 10K comments
- Make it hard to query comments independently

**Use referencing with pagination:**

```json
// Post
{
  "_id": "post_001",
  "title": "MongoDB vs PostgreSQL",
  "body": "...",
  "commentCount": 9876,
  "lastCommentAt": ISODate("2024-01-15")
}

// Comment (referenced)
{
  "_id": "comment_9876",
  "postId": "post_001",
  "author": "Alice",
  "body": "Great comparison!",
  "createdAt": ISODate("2024-01-15")
}

// Index: { postId: 1, createdAt: -1 }
// Query: db.comments.find({ postId: "post_001" }).sort({ createdAt: -1 }).limit(20)
```

**Alternative (hybrid):** Embed last 3-5 comments for fast preview, store full comments in separate collection.
</details>

### Puzzle 9 — Shard key selection

You have an `events` collection receiving 100K writes/second from IoT devices across the US. Queries: (1) all events for a device in a time range, (2) real-time dashboard per device. Select a shard key.

<details>
<summary>Answer</summary>

`{ deviceId: 1, timestamp: -1 }` — compound shard key:
- `deviceId` provides query isolation (all events for one device go to one shard)  
- `timestamp` within the same device ensures ordered chunks, good for range queries

But: If deviceIds are monotonically increasing (or timestamp-heavy), writes may hotspot to one shard.

**Better:** `{ deviceId: "hashed" }` — uniform distribution but range queries scatter (every query goes to every shard).

**Best for IoT:** `{ deviceId: 1, timestamp: -1 }` with zone sharding by region to keep device data in the closest shard. Since each device's own queries are per-device, scatter for range queries is expected.

Alternative: Use hashed `_id` and add a `deviceId` index. Writes distribute evenly; per-device queries use the `deviceId` index within each shard.
</details>

### Puzzle 10 — Explain output analysis

```js
db.orders.find({ status: "pending", total: { $gt: 100 } }).sort({ createdAt: -1 }).explain("executionStats")
```

Output:
```
winningPlan: COLLSCAN
executionTimeMillis: 8500
totalDocsExamined: 10000000
nReturned: 45000
totalKeysExamined: 0
```

Current indexes:
```js
db.orders.getIndexes()
// { v: 2, key: { _id: 1 } }
// { v: 2, key: { status: 1 } }
// { v: 2, key: { createdAt: -1 } }
```

What's wrong and how do you fix it?

<details>
<summary>Answer</summary>

**Problem:** No single index covers both the filter AND the sort. MongoDB chose COLLSCAN because:
- `{ status: 1 }` — can find 45K matches from 10M, but must then sort in memory (32 MB limit)
- `{ createdAt: -1 }` — can sort but then must filter (scans many docs)
- MongoDB's query planner may reject both due to poor selectivity, defaulting to COLLSCAN

**Fix:** Create a compound index:
```js
db.orders.createIndex({ status: 1, createdAt: -1, total: 1 })
```

Now:
- `status` (equality) routes to matching docs
- `createdAt` (sort, matches direction) avoids in-memory sort
- `total` (range, last due to ESR) filters within matched docs

Result: IXSCAN, targeted keys, sorted in order.
</details>

---

## 3. Debugging scenarios (6 scenarios)

### Scenario 1 — Replica set: primary stepdown during peak

**Symptom:** Every day at 2 PM (peak traffic), the primary steps down briefly. Write errors spike for 15 seconds.

**Steps to diagnose:**
1. Check `rs.status()` — look for stepdown count
2. Check logs for reason: `rs/someMember/primary?` or `stepDown`
3. Check for network latency between members (`rs.status().members[].health`)
4. Check for slow operations blocking heartbeat (oplog apply delay)
5. Check for clock skew between nodes

**Likely cause:** Network congestion or slow oplog apply on secondaries causing election timeout. The primary steps down thinking it's isolated.

**Solution:**
1. Increase `electionTimeoutMillis` to 15-20 seconds
2. Ensure replica set is in same AZ/region (or use priority settings)
3. Check network bandwidth — peak traffic may be saturating the network
4. Add monitoring on `replSetGetStatus` to alert on pre-election conditions

### Scenario 2 — Slow aggregation query for trading dashboard

**Symptom:** The OHLCV aggregation runs every minute but takes 45 seconds, causing timeouts for the trading dashboard.

```js
db.trades.aggregate([
  { $match: { timestamp: { $gte: oneHourAgo } } },
  { $sort: { timestamp: -1 } },
  { $group: {
      _id: { symbol: "$symbol", minute: { $dateTrunc: { date: "$timestamp", unit: "minute" } } },
      open: { $first: "$price" },
      high: { $max: "$price" },
      low: { $min: "$price" },
      close: { $last: "$price" },
      volume: { $sum: "$qty" }
    }
  }
], { allowDiskUse: true })
```

**Diagnosis:**
1. `$match` uses `timestamp` only — if no index, scans all recent trades
2. `$sort` without index uses 32 MB RAM (but `allowDiskUse` mitigates)
3. `$group` after `$sort` still processes all matching documents

**Solution:**
```js
// Index: { timestamp: -1, symbol: 1 }
db.trades.createIndex({ timestamp: -1, symbol: 1 })

// Optimized pipeline — $match early, $group before $sort
db.trades.aggregate([
  { $match: { timestamp: { $gte: oneHourAgo } } },
  { $group: {
      _id: { symbol: "$symbol", minute: { $dateTrunc: { date: "$timestamp", unit: "minute" } } },
      open: { $first: "$price" },
      high: { $max: "$price" },
      low: { $min: "$price" },
      close: { $last: "$price" },
      volume: { $sum: "$qty" }
    }
  },
  { $sort: { "_id.minute": -1 } }
])
```

**Alternative — pre-aggregation:** Use change streams + `$merge` to maintain pre-computed OHLCV data.

### Scenario 3 — MongoDB connection pool exhaustion

**Symptom:** Applications randomly get timeout errors. `serverStatus().connections.current` is at the maximum (500). `currentOp()` shows many idle connections.

**Diagnosis:**
1. Check application connection pool settings — likely `maxPoolSize` too high per app instance
2. Check if connections are released after use (e.g., `mongos.close()` called?)
3. Check for database health checks or monitoring that opens new connections per check

**Solution:**
1. Reduce application `maxPoolSize` — tune based on concurrent request count
2. Increase MongoDB `maxIncomingConnections` (default: 65536; but limited by OS/thread count)
3. Add connection pooling middleware or a proxy (like ProxySQL but for MongoDB)
4. Implement connection monitoring: `db.serverStatus().connections.available` alert
5. Check for `waitQueueTimeoutMS` — set to 5000ms to avoid infinite waits

### Scenario 4 — MongoDB write performance degradation

**Symptom:** Write throughput dropped 50% after adding a new index on a collection with 20M documents.

**Diagnosis:**
1. Write throughput drop is expected — each insert/update now updates the new index
2. Check `serverStatus().wiredTiger.cache` for increased page faults (working set increased)
3. Check index build completed in background (if foreground, blocked writes entirely)

**Solution:**
1. If index was for a worst-case query, consider if it's truly needed
2. Drop the index and use a partial index if possible (`partialFilterExpression`)
3. For your multi-tenant SaaS: ensure indexes are only on fields that are actually queried
4. Consider index build during low traffic window

### Scenario 5 — Secondaries falling behind

**Symptom:** Secondary members show increasing replication lag (30 minutes+). Application reads from secondaries are increasingly stale.

**Diagnosis:**
1. `rs.status().members[].optimeDate` — check lag per secondary
2. Check secondary network bandwidth — saturation?
3. Check secondary's CPU/WiredTiger cache — is it keeping up with oplog apply?
4. Check oplog window: `rs.printReplicationInfo()` — if oplog is 24 hours and lag is 30 min, the secondary will catch up. If oplog is 1 hour, it will fall into RECOVERING.

**Solution:**
1. Increase oplog size: `replSetResizeOplog(1, 102400)` — 100 GB
2. Upgrade secondary hardware (more RAM, faster disk)
3. Add more secondaries to distribute read load
4. Reduce secondary read load — use `nearest` read preference to spread reads
5. Check if a slow query on secondary is blocking oplog apply

### Scenario 6 — Multi-tenant query performance regression

**Symptom:** Dashboard queries for the largest tenant (10M orders) are fast during low traffic but slow during peak. Smaller tenants are fast always.

**Diagnosis:**
1. Query pattern: `db.orders.find({ organization_id: "big_org" }).sort({ created_at: -1 }).limit(50)`
2. Check index: `{ organization_id: 1, created_at: -1 }` exists
3. Check WiredTiger cache: the big tenant's data doesn't fit in cache
4. `explain()` shows IXSCAN but `totalDocsExamined` is high (many docs for that org)

**Solution:**
1. If the big tenant's data doesn't fit in cache, increase cache or add more RAM
2. Consider sharding by `organization_id` — large tenant gets its own shard(s)
3. Add pagination cursor (use `_id` with `$gt` instead of skip for deep pagination)
4. Pre-aggregate dashboard data with a background job
5. For your SaaS: consider tenant tier limits — larger tenants pay more, get dedicated resources

---

## 4. System design prompts (5 prompts)

### Prompt 1 — Design a real-time leaderboard

Design a MongoDB-backed real-time leaderboard for a gaming platform with:
- 1M+ daily active users
- Scores update on every game completion (100K writes/sec)
- Leaderboard sorted by score descending
- Must show: rank, player name, score, last updated
- Query: top 100 players, and a player's own rank

**Approach:**

Schema:
```json
{
  "_id": "player_001",
  "name": "Alice",
  "score": 15000,
  "updatedAt": ISODate("2024-01-15T10:00:00Z")
}
```

Index: `{ score: -1, updatedAt: -1 }`

Problems with this approach:
- Every write updates the same hot documents (top 100 players)
- Sorting by score for rank is expensive at 1M+ players
- Rank query requires counting documents with higher score

**Better approach — bucketed score ranges:**

```json
// Bucket collection
{
  "_id": ObjectId("..."),
  "scoreRange": "10000-10999",
  "players": [
    { "playerId": "p1", "name": "Alice", "score": 10500 },
    ...
  ],
  "playerCount": 150
}
```

Or use Redis for the leaderboard (sorted sets):
- Redis `ZADD leaderboard 15000 player_001` — O(log N)
- Redis `ZREVRANGE leaderboard 0 99` — top 100
- Redis `ZRANK leaderboard player_001` — player rank
- MongoDB persists player profiles and history

**MongoDB + Redis hybrid:** Use MongoDB for persistence/audit, Redis for the real-time leaderboard. Back MongoDB with change stream → sync to Redis.

### Prompt 2 — Design a time-series data store for IoT

Design a MongoDB-based time-series store for 50K IoT devices sending temperature readings every 5 seconds (10M writes/sec aggregate).

**Schema — Bucket pattern:**
```json
{
  "_id": { "deviceId": "d1", "hour": ISODate("2024-01-15T10:00:00Z") },
  "deviceId": "d1",
  "hour": ISODate("2024-01-15T10:00:00Z"),
  "readings": [
    { "t": ISODate("2024-01-15T10:00:05Z"), "v": 23.5 },
    { "t": ISODate("2024-01-15T10:00:10Z"), "v": 23.7 }
  ],
  "count": 720,
  "avg": 23.6,
  "min": 22.1,
  "max": 25.3
}
```

Index: `{ deviceId: 1, hour: -1 }`

Shard key: `{ deviceId: "hashed" }` — uniform write distribution across shards.

**Trade-offs:**
- ~720 readings/hour per device = ~720 documents/hour = ~17,280/day per device
- For 50K devices: ~864M documents/day
- Each bucket doc: ~50 KB → 43 GB/day raw, compressed ~10 GB
- Sharded cluster needed (4-8 shards)

**Alternative — MongoDB Time Series Collections (5.0+):**
```js
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "deviceId",
    granularity: "seconds"
  }
})
```
MongoDB internally manages bucketing — no application-level bucket logic needed.

### Prompt 3 — Design a distributed job scheduler (Chronos on MongoDB)

Replace Chronos's Raft-based state with MongoDB for job state management. Design the schema.

**Schema:**
```json
{
  "_id": "job_001",
  "type": "send_email",
  "payload": { "to": "user@example.com", "template": "welcome" },
  "status": "scheduled",           // scheduled, running, completed, failed, retrying
  "priority": 5,                    // 1-10, higher = more urgent
  "schedule": "*/5 * * * *",       // cron expression
  "nextRunAt": ISODate("2024-01-15T10:00:00Z"),
  "lastRunAt": ISODate("2024-01-15T09:55:00Z"),
  "retryCount": 0,
  "maxRetries": 3,
  "assignedWorker": "worker-1",
  "assignedAt": ISODate("2024-01-15T10:00:00Z"),
  "heartbeat": ISODate("2024-01-15T10:00:30Z"),
  "createdAt": ISODate("2024-01-01T00:00:00Z")
}
```

**Indexes:**
```js
db.jobs.createIndex({ status: 1, nextRunAt: 1, priority: -1 })
db.jobs.createIndex({ assignedWorker: 1, status: 1 })
db.jobs.createIndex({ heartbeat: 1 }, { sparse: true })
```

**Change streams for real-time:**
```js
const changeStream = db.jobs.watch([
  { $match: { operationType: "update", "updateDescription.updatedFields.status": { $exists: true } } }
])
// Publish job events to workers via WebSocket/pub-sub
```

**Transaction for job claim (atomic claim):**
```js
const session = client.startSession()
session.startTransaction()

const result = await db.jobs.findOneAndUpdate(
  { _id: "job_001", status: "scheduled", nextRunAt: { $lte: new Date() } },
  { $set: { status: "running", assignedWorker: "worker-1", assignedAt: new Date() } },
  { session, returnDocument: "before" }
)

if (result) {
  // Process job
  await session.commitTransaction()
} else {
  await session.abortTransaction()  // already claimed by another worker
}
session.endSession()
```

**Heartbeat-based worker health:**
```js
// Worker sends heartbeat every 10 seconds
db.jobs.updateOne(
  { _id: "job_001" },
  { $set: { heartbeat: new Date() } }
)

// Background reaper: find jobs with stale heartbeats, reschedule
db.jobs.updateMany(
  { status: "running", heartbeat: { $lt: new Date(Date.now() - 30000) } },
  { $set: { status: "rescheduled" } }
)
```

This is a simpler alternative to Chronos's Raft-based leader election, but trades strong consistency for simpler operations.

### Prompt 4 — Design a notification delivery system

Design a MongoDB-backed notification system for your SaaS:
- 10K tenants, 1M users
- Notifications: in-app, email, push
- Track delivery status, retry logic
- Real-time delivery tracking for admins

**Schema:**
```json
// Notification template
{
  "_id": "tpl_001",
  "organizationId": "org_001",
  "name": "welcome_email",
  "channels": ["email", "in_app"],
  "subject": "Welcome {{name}}!",
  "body": "Hi {{name}}, welcome to Acme!",
  "createdAt": ISODate("2024-01-01")
}

// Notification queue
{
  "_id": "notif_001",
  "organizationId": "org_001",
  "templateId": "tpl_001",
  "userId": "user_001",
  "channels": ["email", "in_app"],
  "status": "pending",       // pending, processing, sent, failed, delivered, read
  "channelStatuses": [
    { "channel": "email", "status": "sent", "sentAt": ISODate("2024-01-15T10:00:00Z") },
    { "channel": "in_app", "status": "delivered", "deliveredAt": ISODate("2024-01-15T10:00:01Z") }
  ],
  "priority": 3,
  "retryCount": 0,
  "maxRetries": 3,
  "nextRetryAt": null,
  "createdAt": ISODate("2024-01-15T10:00:00Z"),
  "readAt": null
}

// Notification preferences per user
{
  "_id": "pref_001",
  "userId": "user_001",
  "channels": {
    "email": true,
    "push": false,
    "in_app": true
  },
  "digestFrequency": "realtime"
}
```

**Indexes:**
```js
db.notifications.createIndex({ organizationId: 1, userId: 1, createdAt: -1 })
db.notifications.createIndex({ status: 1, nextRetryAt: 1 })
db.notifications.createIndex({ userId: 1, readAt: 1 }, { sparse: true })
```

**Change stream for real-time:**
When a notification is inserted, watch → push to WebSocket for in-app notification.

**Atomic status transition (single-document):**
```js
db.notifications.updateOne(
  { _id: "notif_001", status: "pending" },
  { $set: { status: "processing" } }
)
```

### Prompt 5 — Design a multi-tenant SaaS schema on MongoDB

Design a MongoDB schema for a multi-tenant inventory management SaaS (like yours) that must:
- Support 10K tenants, each with their own organization
- Products, inventory levels per warehouse, orders
- Row-level multi-tenancy (every query scoped to organization)
- Flexible product attributes per tenant
- Real-time inventory updates

**Schema:**
```json
// Organization
{
  "_id": "org_001",
  "name": "Acme Corp",
  "plan": "enterprise",
  "settings": {
    "currency": "USD",
    "timezone": "America/New_York",
    "features": ["real_time", "reports", "api"]
  },
  "createdAt": ISODate("2023-01-01")
}

// Product — flexible attributes per tenant
{
  "_id": "prod_001",
  "orgId": "org_001",
  "sku": "WDG-001",
  "name": "Widget",
  "category": "tools",
  "price": NumberDecimal("29.99"),
  "attributes": {                     // Flexible per-tenant
    "color": "red",
    "weight_kg": 1.5,
    "material": "steel"
  },
  "warehouses": [                     // Embedded inventory snapshots
    { "warehouseId": "wh_001", "qty": 150, "reserved": 10 },
    { "warehouseId": "wh_002", "qty": 42, "reserved": 5 }
  ],
  "totalQty": 192,                    // Denormalized for fast queries
  "totalReserved": 15,
  "available": 177,
  "createdAt": ISODate("2023-06-01"),
  "updatedAt": ISODate("2024-01-15")
}

// Order
{
  "_id": "ord_001",
  "orgId": "org_001",
  "orderNumber": "ORD-2024-00001",
  "customer": { "name": "Alice", "email": "alice@example.com" },
  "items": [
    { "productId": "prod_001", "sku": "WDG-001", "name": "Widget", "qty": 2, "unitPrice": 29.99 }
  ],
  "subtotal": 59.98,
  "shipping": 5.99,
  "tax": 4.80,
  "total": 70.77,
  "status": "shipped",
  "shippingAddress": { "street": "123 Main St", "city": "NY", "zip": "10001" },
  "createdAt": ISODate("2024-06-15"),
  "shippedAt": ISODate("2024-06-16")
}
```

**Indexes (all start with orgId for multi-tenancy):**
```js
db.products.createIndex({ orgId: 1, sku: 1 }, { unique: true })
db.products.createIndex({ orgId: 1, category: 1 })
db.products.createIndex({ orgId: 1, name: "text", sku: "text" })
db.products.createIndex({ orgId: 1, updatedAt: -1 })

db.orders.createIndex({ orgId: 1, createdAt: -1 })
db.orders.createIndex({ orgId: 1, status: 1, createdAt: -1 })
db.orders.createIndex({ orgId: 1, "customer.email": 1 })
```

**Dealing with inventory atomicity:**
```js
// Use atomic single-document update for reserving inventory
db.products.updateOne(
  { _id: "prod_001", "warehouses.warehouseId": "wh_001" },
  { $inc: { "warehouses.$.qty": -2, "warehouses.$.reserved": 2, "available": -2 } }
)
```

**When this doesn't work well on MongoDB:**
- Cross-warehouse inventory queries (need `$unwind`)
- Complex reporting across orgs (need `$group` + `$lookup`)
- Inventory reconciliation (needs strong consistency across products)

**MongoDB vs PostgreSQL for this workload:**
- MongoDB better: Flexible product attributes per tenant, single-document atomicity for inventory
- PostgreSQL better: Complex reporting, joins, cross-tenant analytics, strong consistency

---

## 5. STAR story templates

### Story: Choosing MongoDB document model for flexible attributes

**Situation:** Our multi-tenant SaaS needed to support per-tenant custom product attributes (some tenants had colors/sizes, others had weights/dimensions). With a fixed PostgreSQL schema, we'd need either EAV (slow queries) or JSONB (limited indexing).

**Task:** Design a schema that supports flexible per-tenant attributes without sacrificing query performance for standard fields.

**Action:**
- Used MongoDB's document model with a single `products` collection
- Standard fields (orgId, sku, name, price, category) indexed for common queries
- Flexible `attributes` subdocument for per-tenant customization
- Used `$jsonSchema` validator to ensure required fields per org are present
- Indexed commonly-queried attribute paths for tenants that need them

**Result:** Zero schema migrations needed for new tenant-custom attributes. Query performance for standard fields matched PostgreSQL. Tenant onboarding time reduced from 2 weeks (schema migration + app changes) to 2 days (configuration only).

### Story: Trading platform write throughput

**Situation:** Our 20K+ DAU trading platform required real-time trade recording at 5K+ writes/second during market hours. PostgreSQL was struggling with write throughput under contention.

**Task:** Migrate trade storage to MongoDB for higher write throughput and flexible query capabilities (OHLCV, user trade history).

**Action:**
- Evaluated shard key: selected `{ symbol: "hashed" }` for uniform write distribution
- Used write concern `w: "majority"` for critical trades, `w: 1` for non-critical analytics
- Implemented bucketed time-series schema for OHLCV pre-aggregation (change stream → $merge)
- Set up replica sets with nearest read preference for fast trade reads

**Result:** Write throughput increased 5x with no data loss. OHLCV queries dropped from 8 seconds to 50ms (pre-aggregation). System handled a 3x traffic spike during a volatile market day without degradation.

---

## 6. Key metrics to remember

| Metric | Target | Why |
|--------|--------|-----|
| WiredTiger cache hit ratio | > 95% | Low = working set doesn't fit in RAM |
| Replication lag | < 2 seconds | Prevents stale reads |  
| Election time | < 15 seconds | Minimizes write unavailability |
| Document size | < 1 MB (avg) | Large docs slow reads/writes |
| Index key size | < 1 KB | Keeps indexes efficient |
| Write concern w:majority latency | < 10ms added | Acceptable for most workloads |
| Aggregation RAM | < 100 MB | Avoids disk spill |
| Connections | < 80% of max | Prevents pool exhaustion |

---

## 7. Interview preparation checklist

- [ ] I can explain BSON vs JSON with all data types
- [ ] I can design a compound index using ESR rule from a query
- [ ] I can read `explain("executionStats")` and identify problems
- [ ] I can write an aggregation pipeline with `$match`, `$group`, `$lookup`, `$unwind`
- [ ] I can explain replica set elections, write concern, read preference, and consistency guarantees
- [ ] I can design a shard key for a given workload
- [ ] I know when to embed vs reference
- [ ] I can articulate when MongoDB is the wrong choice
- [ ] I know the change streams API and real-world use cases
- [ ] I can explain WiredTiger cache, compression, journal
- [ ] I can describe a zero-downtime migration from PostgreSQL to MongoDB
- [ ] I can discuss MongoDB Atlas, serverless, and managed MongoDB trade-offs
