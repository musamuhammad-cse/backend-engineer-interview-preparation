# MongoDB — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant SaaS (PostgreSQL primary — MongoDB as document store for flexible schemas), trading platform (high-throughput writes), Chronos (job metadata storage). MongoDB is a different paradigm from relational databases — interviewers test whether you know *when* to use NoSQL, not just how.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**The senior signal is knowing when to use MongoDB and when not to.** Relational vs document trade-offs, embedding vs referencing, and understanding MongoDB's consistency model are more important than memorizing query syntax.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | MongoDB document model, BSON, CRUD operations, query operators, indexing basics, collection & database concepts, `_id` field, comparison to relational databases, basic aggregation pipeline | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Advanced queries (text search, geospatial, array operations), aggregation pipeline stages ($match, $group, $lookup, $unwind, $bucket, $facet), indexing strategies (compound, multikey, text, geospatial, TTL, partial, sparse), explain() & query optimization, transactions (MongoDB 4.0+), schema design patterns (embedded vs referenced, one-to-many, many-to-many, polymorphic) | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Replication (replica sets, election, write concern, read preference), sharding (shard key selection, chunk splitting, balancer, hash vs range sharding, zone sharding), aggregation pipeline optimization ($match early, $sort memory), indexing strategies for production (covered queries, index intersection, index build in background), security (authentication, authorization, TLS, field-level encryption), performance tuning (WiredTiger cache, compression, opcounters), change streams, Atlas (managed MongoDB), migration from relational to document | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 140+ interview questions, code puzzles, debugging scenarios, system design prompts | Ongoing drill |

---

## Coverage map

### MongoDB fundamentals
- Document model: BSON, JSON-like documents, dynamic schema, `_id` field (ObjectId, UUID, custom)
- Databases, collections, documents — analogous to relational database, table, row
- CRUD operations: insertOne/insertMany, find/findOne, updateOne/updateMany/replaceOne, deleteOne/deleteMany
- Query operators: comparison ($eq, $ne, $gt, $gte, $lt, $lte, $in, $nin), logical ($and, $or, $not, $nor), element ($exists, $type), evaluation ($regex, $expr, $mod), geospatial ($geoWithin, $geoIntersects, $near), array ($all, $elemMatch, $size)
- Sorting, projection, limit, skip
- Cursors: iteration, batch size, timeout
- Bulk writes: ordered vs unordered

### Indexing
- Single field indexes
- Compound indexes: ESR rule (Equality, Sort, Range)
- Multikey indexes: array fields
- Text indexes: `$text`, weights, `$meta: "textScore"`
- Geospatial indexes: 2dsphere, 2d
- Hashed indexes (for sharding)
- TTL indexes: auto-expire documents
- Partial indexes: filter-based
- Sparse indexes: only index documents with the field
- Unique indexes
- Covered queries (all fields in index)
- Index intersection
- `explain()`: winning plan, rejected plans, execution stats, IXSCAN vs COLLSCAN

### Aggregation pipeline
- Pipeline stages: $match, $project, $group, $sort, $limit, $skip, $unwind, $lookup (left outer join), $bucket, $bucketAuto, $facet, $replaceRoot, $addFields, $set, $count, $out, $merge, $sample
- Expression operators: $sum, $avg, $min, $max, $push, $addToSet, $first, $last, $cond, $ifNull, $convert
- Optimization: $match + $sort early, $match before $project, $lookup performance
- Aggregation pipeline vs mapReduce (mapReduce deprecated in 5.0+)

### Schema design
- Embedding vs referencing: read/write ratio, data access patterns, document size growth, 16MB document limit
- One-to-one: embed (preferred) or reference
- One-to-many: embed (few, small) or reference array (many) or parent reference (many, large)
- Many-to-many: reference arrays on both sides or junction collection
- Polymorphic schema: discriminator field, multiple types in one collection
- Time-series: pre-aggregation, bucket pattern, timeseries collection (MongoDB 5.0+)
- Tree structures: parent references, child references, nested sets, materialized paths
- Schema anti-patterns: massive arrays (unbounded growth), unnecessary indexes, overly deep embedding

### Replication (Replica Sets)
- Primary-secondary architecture: one primary (writes), multiple secondaries (reads with read preference)
- Election: majority of voting members elect primary (Raft-based consensus)
- Write concern: w:1 (ack from primary), w:majority (ack from majority), w:"tag" (custom), wtimeout
- Read preference: primary, primaryPreferred, secondary, secondaryPreferred, nearest
- Oplog (operations log): capped collection, replication source, oplog size management
- Replica set configuration: members, priority, votes, hidden, delayed, arbiter
- Automatic failover and rollback
- Chaining: secondary replicating from another secondary
- Monitoring: replSetGetStatus, rs.status(), replication lag

### Sharding
- Shard key: must be indexed, immutable, high cardinality, monotonically growing avoids (unless using hash sharding)
- Cluster components: mongos (router), config servers (metadata), shards (data)
- Chunk splitting: default 64MB, move chunks via balancer
- Balancer: automatic redistribution of chunks, balancing window
- Zone sharding: data locality, tiered storage
- Hashed vs ranged sharding: distribution vs range query performance
- Shard key strategies: hashed shard key (even distribution), compound shard key (locality + distribution)
- Jumbo chunks: chunks that can't be split

### Operations & performance
- WiredTiger: default storage engine, document-level concurrency, compression (snappy, zlib, zstd), cache (default 50% of RAM - 1GB)
- Journal: write-ahead log, durability, journal commit interval
- Profiler: `db.setProfilingLevel()`, system.profile collection
- Current operations: `db.currentOp()`, killOp
- Index build: foreground (locks collection) vs background (slower, non-blocking after 4.2+)
- `serverStatus`, `dbStats`, `collStats`
- Maintenance: compact, repairDatabase (standalone only), reIndex
- Backups: mongodump/mongorestore, mongodb-consistent-backup, Atlas snapshots
- MongoDB Ops Manager / Cloud Manager
- Atlas: managed MongoDB, auto-scaling, backup, monitoring, BI Connector, Atlas Search (Lucene-based)

### MongoDB vs PostgreSQL comparison
- Schema: dynamic vs fixed
- Joins: $lookup (slower) vs JOIN (optimized) — MongoDB is not built for relational joins
- Transactions: multi-document ACID (4.0+), but with performance cost
- Indexing: rich index types in both, but MongoDB's multikey and text have no PG parallel
- Aggregation: MongoDB pipeline vs PG window functions/CTEs
- Consistency: eventually consistent by default (can be tuned with write concern) vs strong consistency
- When to use MongoDB: flexible schema, write-heavy, hierarchical/embedded data, rapid prototyping
- When to use PostgreSQL: relational data, complex queries, joins, strong consistency, mature tooling

---

## Study order recommendation

MongoDB represents a different paradigm. Focus on understanding *when* to use a document database vs a relational one — that's what senior interviewers care about.

```
Week 1:  01-basic.md          + Basic Q&A drill
Week 2:  02-intermediate.md   + Intermediate Q&A drill
Week 3:  03-senior.md         + Senior Q&A drill
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** Redis.
