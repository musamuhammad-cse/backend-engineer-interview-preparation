# MongoDB — Senior

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Replication, sharding, production operations, performance tuning, security, change streams, Atlas, migration from relational, troubleshooting  
> **Real-world anchors:** Multi-tenant SaaS (schema design decisions), trading platform (high-throughput writes, sharding strategy), Chronos (distributed job state with change streams)

---

## 1. Replication — Replica Sets

### Architecture

A replica set is a group of `mongod` instances that maintain the same data set:
- **Primary**: Accepts all write operations. One primary per replica set.
- **Secondaries**: Apply operations from the primary's oplog to maintain identical data sets.
- **Arbiters**: Participate in elections but do not hold data (only for voting, odd number of voters).

```
Client → mongos/router → Primary (write)
                          ↓ (oplog)
                    Secondary 1 (read with readPreference: secondary)
                    Secondary 2 (read with readPreference: secondary)
                    Arbiter (vote only)
```

### Oplog (Operations Log)

The oplog is a **capped collection** in the `local` database:
- Records every write operation on the primary
- Secondaries replicate by tailing the oplog
- Default size: 5% of free disk space (or 50 MB min, 237 MB max for small disks)
- Each operation includes: `ts` (timestamp), `h` (hash), `op` (operation type), `ns` (namespace), `o` (operation data), `o2` (update criteria)

**Trap:** If a secondary falls behind the oplog window (cannot keep up), it goes into `RECOVERING` state and must be resynchronized from scratch. Monitor `replSetGetStatus` for `optimeDate` lag.

```js
// Check replication lag
rs.status().members.forEach(m => {
  print(m.name + ": " + m.optimeDate)
})

// Resize oplog (MongoDB 4.0+)
db.adminCommand({ replSetResizeOplog: 1, size: 25600 })  // MB
```

### Election process

When the primary becomes unavailable:
1. A secondary detects loss of connectivity (no heartbeat for 10 seconds default)
2. The secondary calls an election
3. A majority of voting members must vote for the candidate
4. The winning secondary transitions to primary

**Raft-based consensus (MongoDB 3.2+):**
- A candidate needs votes from a majority of voting members
- In a 3-node set: needs 2 votes
- In a 5-node set: needs 3 votes
- Ties are broken by priority; if tied, the member with the most recent oplog entry wins

**Election timing:**
- Detection: `electionTimeoutMillis` (default: 10,000 ms)
- Election: typically < 5 seconds
- Total failover time: ~10-15 seconds (with default settings)
- Client-side: retry logic needed during this window

### Write concern

Controls how many members must acknowledge a write:

```js
// Fire-and-forget — fastest, no durability
{ w: 0 }

// Acknowledged by primary — default
{ w: 1 }

// Acknowledged by majority of voting members
{ w: "majority" }

// Acknowledged by a specific set of members
{ w: "tagSetName" }

// Acknowledged by N members
{ w: 3 }

// Timeout
{ w: "majority", wtimeout: 5000 }
```

**Trap:** `w: "majority"` does not mean "all members" — it means majority of **voting** members. In a 3-node set (2 data + 1 arbiter), majority is 2. The arbiter acknowledges the write but doesn't hold data.

### Read concern

Controls the consistency and isolation properties of reads:

```js
// Read the most recent data visible to this node (might be rolled back)
{ readConcern: "local" }

// Read data acknowledged by majority (survives election)
{ readConcern: "majority" }

// Read from a consistent snapshot (MongoDB 5.0+)
{ readConcern: "snapshot" }

// Read the most recent data available, even if it might roll back
{ readConcern: "available" }  // deprecated in 5.0+
```

**Trap:** `readConcern: "majority"` requires `enableMajorityReadConcern: true` (default in 4.0+). If storage engine cache pressure is high, majority read concern can cause issues because MongoDB must retain the majority-committed snapshot.

### Read preference

Controls which replica set member to read from:

```js
// Read from primary — default
{ readPreference: "primary" }

// Read from primary if available, else secondary
{ readPreference: "primaryPreferred" }

// Read only from secondary
{ readPreference: "secondary" }

// Read from secondary if available, else primary
{ readPreference: "secondaryPreferred" }

// Read from nearest member (lowest network latency)
{ readPreference: "nearest" }
```

**Trap:** Reading from secondaries with `readPreference: "secondary"` means you can read **stale data**. The secondary may be behind the primary by milliseconds or minutes (replication lag). For use cases requiring strong consistency, always read from primary.

### Write concern + read preference matrix

| Write Concern | Read Preference | Consistency Guarantee |
|--------------|-----------------|----------------------|
| w:1 | primary | Strong consistency |
| w:majority | primary | Strong consistency |
| w:1 | secondary | **Possible stale read** — secondary may not have the write yet |
| w:majority | secondary | **Causal consistency** if using causally consistent sessions |

### Causal consistency

MongoDB 3.6+ supports causally consistent sessions:

```js
const session = client.startSession({ causalConsistency: true })

// Even with secondary reads, causal consistency ensures:
// 1. Read your writes
// 2. Monotonic reads
// 3. Monotonic writes
// 4. Writes follow reads
```

### Replica set configuration

```js
// Current config
rs.conf()

// Reconfigure
cfg = rs.conf()
cfg.members[1].priority = 0     // member 1 cannot become primary
cfg.members[2].hidden = true     // member 2 is hidden from application
cfg.members[2].priority = 0     // hidden members must have priority 0
cfg.members[3].slaveDelay = 3600 // member 3 delays replication by 1 hour
rs.reconfig(cfg)
```

**Trap:** `rs.reconfig()` can cause a primary election if it changes member priorities. Use `rs.reconfig(cfg, { force: true })` only when the majority of the set is unavailable (disaster recovery). Forcing reconfiguration has data rollback risk.

### Rollback

When a former primary reconnects after a network partition and finds it has writes that the new primary does not:
1. MongoDB rolls back those writes
2. Rolled-back data is written to `rollback/` directory as BSON files
3. Must be manually replayed by an admin

**Minimizing rollback:**
- Use `w: "majority"` write concern — writes must survive on a majority before returning
- Keep elections fast — `electionTimeoutMillis: 10000`
- Use retryable writes on the client side

### Retryable writes

MongoDB 3.6+ supports automatic retry of write operations after network errors or replica set elections:

```js
const client = new MongoClient(uri, { retryWrites: true })
```

**Constraints:**
- Only single-document writes and multi-document transactions
- Requires replica sets or sharded clusters
- Not for unacknowledged writes (w:0)

---

## 2. Sharding

### Architecture

MongoDB distributes data across shards (each shard is a replica set):

```
Client → mongos (router) → Config Servers (metadata)
                            ↓
                     Shard A (replica set)
                     Shard B (replica set)
                     Shard C (replica set)
```

### Components

| Component | Purpose |
|-----------|---------|
| **mongos** | Query router — routes reads/writes to appropriate shards |
| **Config Servers** | Store cluster metadata (shard locations, chunk ranges), must be a replica set |
| **Shards** | Hold data, each shard is a replica set (for high availability) |

### Shard key

The shard key determines how data is distributed across shards:

```js
// Enable sharding on database
sh.enableSharding("mydb")

// Shard a collection with a hashed shard key
sh.shardCollection("mydb.orders", { _id: "hashed" })

// Shard with a ranged shard key
sh.shardCollection("mydb.users", { organizationId: 1, _id: 1 })
```

### Shard key selection criteria

| Criterion | Description |
|-----------|-------------|
| **High cardinality** | Many unique values — avoids jumbo chunks |
| **Low frequency** | Values are evenly distributed — avoids hot shards |
| **Non-monotonically changing** | Avoids all writes going to one shard (range sharding on auto-increment key is bad) |

**Good shard keys:**
- Hashed `_id` — perfectly distributed but range queries scatter
- Compound key: `{ organizationId: 1, _id: 1 }` — good locality for org queries + _id for distribution
- Hashed shard key on a high-cardinality field: `{ email: "hashed" }`

**Bad shard keys:**
- `{ status: 1 }` — low cardinality (only a few values), creates jumbo chunks
- `{ createdAt: 1 }` — monotonically increasing, all new writes go to one shard
- `{ _id: 1 }` (range) — same as createdAt (ObjectId is monotonic)

### Hashed vs Ranged sharding

| | Hashed | Ranged |
|---|---|---|
| Distribution | Even (hash distributes uniformly) | Clustered by key range |
| Range queries | Scatter to all shards (slow) | Targeted to specific shards (fast) |
| Point lookups | Targeted (hash → shard) | Targeted |
| Write distribution | Even | Skewed if monotonic key |
| Best for | Write-heavy, uniform distribution | Range-query-heavy, geo-locality |

```js
// Hashed sharding — even distribution
sh.shardCollection("db.events", { eventId: "hashed" })

// Ranged sharding — range query friendly
sh.shardCollection("db.users", { orgId: 1, _id: 1 })
```

### Chunk splitting

- Default chunk size: 64 MB
- When a chunk exceeds the configured size, MongoDB splits it into two chunks
- Splits are logged but do not cause data movement

```js
// Configure chunk size (in MB, default 64)
db.adminCommand({ configureCollectionBalancing: "mydb.orders", chunkSize: 128 })
```

### Balancer

The balancer runs in the background on mongos and redistributes chunks across shards:

```js
// Check balancer state
sh.getBalancerState()

// Enable/disable balancer
sh.startBalancer()
sh.stopBalancer()

// Set balancing window
db.adminCommand({
  setParameter: 1,
  balancerWindowStart: "02:00",
  balancerWindowEnd: "06:00"
})
```

**Migration process:**
1. Balancer sends `moveChunk` command to source shard
2. Source shard copies chunk data to destination shard
3. Destination shard indexes the chunk
4. Source shard sends a metadata update to config servers
5. Source shard deletes the migrated data

**Trap:** Chunk migration blocks read/write for the migrating chunk briefly. During peak traffic, the balancer can cause latency spikes. Schedule balancing during maintenance windows.

### Jumbo chunks

A chunk that cannot be split because all documents have the same shard key value:
- Cannot be split or moved
- Causes uneven data distribution
- Usually caused by low-cardinality shard key

```js
// Identify jumbo chunks
db.adminCommand({ splitChunk: "mydb.orders", ... })

// Force balancing of jumbo chunks (MongoDB 5.0+)
db.adminCommand({ cleanupOrphaned: "mydb.orders" })
```

### Zone sharding

Control data placement for locality or tiered storage:

```js
// Create zones
sh.addShardToZone("shard01", "US")
sh.addShardToZone("shard02", "EU")

// Associate range with zone
sh.updateZoneKeyRange("mydb.users", { region: "US" }, { region: "US\uffff" }, "US")
sh.updateZoneKeyRange("mydb.users", { region: "EU" }, { region: "EU\uffff" }, "EU")
```

### Choosing when to shard

**Sharding is not always the answer.** Consider:

| Problem | Alternative before sharding |
|---------|----------------------------|
| Read throughput | Read replicas (secondary reads) |
| Write throughput | Faster storage (SSD), more RAM, index optimization |
| Data size > single node | Archive old data, vertical scaling first |
| Query latency | Index optimization, schema redesign |

**Good reasons to shard:**
- Write throughput exceeds single node's capacity
- Data set exceeds single node's storage (and vertical scaling is too expensive)
- Need geographic data locality

**Trap:** Many teams shard prematurely. Vertical scaling is much simpler and more reliable. MongoDB can handle 10-20 TB on a single shard with good indexes and RAM.

### Sharding + transactions

Multi-document ACID transactions work on sharded clusters (MongoDB 4.2+):
- Transactions can span multiple shards (distributed transactions)
- Performance overhead is higher — 2-phase commit across shards
- Consider single-shard transactions when possible

---

## 3. Performance tuning

### WiredTiger cache tuning

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8   # Default: 50% of (RAM - 1GB)
```

**Cache sizing guidelines:**
- Working set should fit in cache for optimal performance
- Monitor `wiredTiger.cache.tracked dirty bytes in the cache`
- If dirty bytes approach cache size, you have write pressure
- If page faults increase, working set doesn't fit in cache

### Checking cache hit ratio

```js
db.serverStatus().wiredTiger.cache
```

Key metrics:
- `bytes currently in cache` — current cache usage
- `maximum bytes configured` — configured cache size
- `pages read into cache` — disk reads (high = working set too large)
- `pages written from cache` — evictions (high = cache too small)
- `percentage overhead` — WiredTiger internal overhead (target: < 8%)

### Profiling

```js
// Enable profiling — 0=off, 1=slow ops, 2=all
db.setProfilingLevel(1, { slowms: 100 })

// View slow queries
db.system.profile.find({ millis: { $gt: 1000 } }).sort({ ts: -1 }).limit(20)
```

### currentOp() and killOp()

```js
// Find long-running operations
db.currentOp({
  active: true,
  secs_running: { $gt: 5 },
  op: "query"
})

// Kill an operation
db.killOp(opid)
```

**Trap:** You cannot kill operations that are in index build or compact. For index builds, use `dropIndexes`.

### Index build strategies

In MongoDB 4.2+:
- Foreground index builds (default in 4.0-) block reads/writes
- Background index builds (4.2+): non-blocking by default

```js
// In MongoDB 4.2+, index builds are in background by default
db.orders.createIndex({ createdAt: 1 })

// On MongoDB < 4.2:
db.orders.createIndex({ createdAt: 1 }, { background: true })
```

**Trap:** Even in background mode (pre-4.2), index builds can impact performance. For large collections, build indexes on a secondary first, then step it up as primary.

### Query patterns for production

#### Covered queries

A query where all required fields are in the index:

```js
// Index: { email: 1, name: 1, role: 1 }
db.users.find(
  { email: "alice@example.com" },
  { name: 1, role: 1, _id: 0 }
)
// Explain: IXSCAN only, no FETCH
```

#### Limit the result set

```js
// Always add limit() unless you truly need all results
db.orders.find({ customerId: "C1" }).sort({ createdAt: -1 }).limit(50)
```

#### Use projections

```js
// Don't retrieve 16 fields if you only need 2
db.users.find({ email: "alice@example.com" }, { name: 1, _id: 0 })
```

### Slow query analysis workflow

```
1. Identify slow query → db.system.profile
2. Explain → .explain("executionStats")
3. Check nReturned vs totalDocsExamined — ratio should be near 1
4. Check winningPlan — is it using an index?
5. Check rejectedPlans — were better plans rejected?
6. If no index, create one (ESR rule)
7. If index exists but not used, check:
   a. Selectivity: Is the index selective enough?
   b. Sort order: Does the index match the sort order?
   c. Partial filter: Does the query match the partial filter expression?
```

---

## 4. Security

### Authentication

```yaml
security:
  authorization: "enabled"           # Enable role-based access control
  authenticationMechanisms:
    - SCRAM-SHA-256                   # Default, salted challenge-response
    - MONGODB-X509                   # Certificate authentication
    - MONGODB-AWS                    # AWS IAM authentication
    - LDAP                           # LDAP (Enterprise only)
    - KERBEROS                       # Kerberos (Enterprise only)
```

### Authorization — RBAC

```js
// Create user with roles
db.createUser({
  user: "app_user",
  pwd: "secure_password",
  roles: [
    { role: "readWrite", db: "myapp" }
  ],
  authenticationRestrictions: [
    { clientSource: ["192.168.1.0/24"] }
  ]
})

// Create custom role
db.createRole({
  role: "queryOnly",
  privileges: [
    { resource: { db: "myapp", collection: "" }, actions: ["find", "aggregate"] }
  ],
  roles: []
})
```

### Built-in roles

| Role | Scope | Capabilities |
|------|-------|-------------|
| `read` | Database | Read any collection |
| `readWrite` | Database | Read + write any collection |
| `dbAdmin` | Database | Schema management, index management |
| `dbOwner` | Database | All database ops (readWrite + dbAdmin + userAdmin) |
| `clusterMonitor` | Cluster | Monitoring commands |
| `clusterManager` | Cluster | Replica set and sharding management |
| `backup` | Cluster | Backup data |
| `restore` | Cluster | Restore data |
| `root` | Cluster | All privileges |

### TLS/SSL

```yaml
net:
  tls:
    mode: "requireTLS"
    certificateKeyFile: "/etc/mongodb/server.pem"
    CAFile: "/etc/mongodb/ca.pem"
    allowConnectionsWithoutCertificates: false
```

### Field-level encryption (FLE)

MongoDB 4.2+ supports client-side field-level encryption:

```js
const client = new MongoClient(uri, {
  autoEncryption: {
    keyVaultNamespace: "encryption.__keyVault",
    kmsProviders: {
      aws: {
        accessKeyId: "...",
        secretAccessKey: "..."
      }
    },
    schemaMap: {
      "mydb.users": {
        bsonType: "object",
        properties: {
          ssn: {
            encrypt: {
              keyId: [UUID("...")],
              bsonType: "string",
              algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
            }
          }
        }
      }
    }
  }
})
```

---

## 5. Change streams

Change streams allow applications to **subscribe to real-time data changes** (MongoDB 3.6+):

```js
const pipeline = [
  { $match: { "fullDocument.organization_id": "org_123" } },
  { $match: { operationType: { $in: ["insert", "update", "replace"] } } }
]

const changeStream = db.collection("orders").watch(pipeline, {
  fullDocument: "updateLookup", // Include full document after change
  resumeAfter: { _data: resumeToken } // Resume from a specific point
})

changeStream.on("change", (change) => {
  console.log(change.fullDocument)
  // Process: update cache, publish to SQS, trigger webhook, etc.
})
```

### Change stream events

```json
{
  "_id": { "_data": "8264..." },
  "operationType": "insert",
  "fullDocument": { "_id": "...", "status": "new" },
  "ns": { "db": "mydb", "coll": "orders" },
  "documentKey": { "_id": "..." },
  "clusterTime": Timestamp(1705000000, 1),
  "wallTime": ISODate("2024-01-15T10:00:00Z")
}
```

### Change stream use cases

| Use case | Pattern |
|----------|---------|
| Cache invalidation | Watch updates on products collection → invalidate Redis cache |
| Real-time sync | Watch orders → publish to WebSocket for dashboard |
| Event-driven architecture | Watch user creation → trigger email service via SQS/SNS |
| Search indexing | Watch product changes → update Elasticsearch index |
| Audit logging | Watch all collections → write to audit logs collection |

### Change streams for multi-tenant SaaS

```js
// Watch changes per organization
const pipeline = [
  { $match: { "fullDocument.organization_id": "org_123" } }
]
const changeStream = db.collection("inventory").watch(pipeline)
```

**Trap:** Change streams require a replica set (not standalone). They use the oplog — if a secondary falls behind and the oplog rolls over, the change stream is invalidated. Use `resumeAfter` with a saved resume token to recover.

### Resume tokens

```js
let resumeToken

const changeStream = db.collection("orders").watch()
changeStream.on("change", (change) => {
  resumeToken = change._id
  // Store resumeToken in MongoDB or Redis
  // On restart:
  // const changeStream = db.collection("orders").watch({ resumeAfter: resumeToken })
})
```

---

## 6. Atlas (Managed MongoDB)

### Atlas key features

| Feature | Detail |
|---------|--------|
| **Auto-scaling** | Cluster tier, storage, and IOPS auto-scale |
| **Backup** | Continuous cloud backup with point-in-time recovery (PITR) |
| **Monitoring** | Built-in monitoring, alerts, and performance advisor |
| **Atlas Search** | Lucene-based full-text search (separate from text indexes) |
| **Atlas Data Lake** | Query data stored in S3/S3-compatible storage using MongoDB query language |
| **Global Clusters** | Multi-region writes for global applications (geo-aware sharding) |
| **Serverless** | Pay-per-use, auto-scale from 0, no cluster management |
| **Atlas Stream Processing** | Process real-time data streams |

### Atlas Search

Atlas Search uses Apache Lucene for advanced full-text search:

```js
// Create search index via UI or API
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": { "type": "string", "analyzer": "lucene.english" },
      "body": { "type": "string" },
      "price": { "type": "number" },
      "tags": { "type": "string" },
      "location": { "type": "geo" }
    }
  }
}

// Query
db.articles.aggregate([
  { $search: {
      compound: {
        should: [
          { text: { query: "mongodb", path: "title", score: { boost: { value: 5 } } } },
          { text: { query: "mongodb", path: "body" } }
        ],
        filter: [
          { range: { path: "price", gte: 10, lte: 100 } }
        ]
      }
    }
  },
  { $project: { title: 1, score: { $meta: "searchScore" } } },
  { $limit: 20 }
])
```

### Atlas Performance Advisor

Atlas's Performance Advisor analyzes slow queries and recommends indexes:

```js
// Equivalent manual approach:
// 1. Enable profiler — db.setProfilingLevel(1, { slowms: 100 })
// 2. Review slow queries — db.system.profile.find({ millis: { $gt: 100 } })
// 3. Check index usage — db.collection.aggregate([{ $indexStats: {} }])
```

### Atlas backup

| Backup Type | Retention | Use Case |
|-------------|-----------|----------|
| Continuous Backup (PITR) | Up to 35 days | Minimize data loss (restore to any second) |
| Snapshots | Configurable (1-365 days) | Periodic backups for compliance |

---

## 7. Monitoring and observability

### Essential commands

```js
// Server status (comprehensive)
db.serverStatus()

// Database stats
db.stats()

// Collection stats
db.collection("orders").stats()

// Index usage
db.collection("orders").aggregate([{ $indexStats: {} }])

// Replica set status
rs.status()

// sharding status
sh.status()

// Current operations
db.currentOp()

// Connection stats
db.serverStatus().connections

// Network stats
db.serverStatus().network
```

### Key metrics to monitor

| Metric | Command | Target |
|--------|---------|--------|
| Cache hit ratio | `serverStatus().wiredTiger.cache` | > 95% |
| Page faults | `serverStatus().extra_info.page_faults` | Low / stable |
| Connections | `serverStatus().connections.current` | < 80% of max |
| Replication lag | `rs.status().members[].optimeDate` | < 2 seconds |
| Queued operations | `serverStatus().globalLock.currentQueue.total` | Close to 0 |
| Opcounters | `serverStatus().opcounters` | Correlate with traffic |
| Flushes (journal) | `serverStatus().wiredTiger.log` | Steady rate |
| Assertions | `serverStatus().asserts.regular` | Low / no warnings |

### Alerts

Alert on:
- Replication lag > 10 seconds
- Number of connections > 80% of max
- Page faults > 10x baseline
- Oplog window < 2 hours
- Disk usage > 80%
- Cache dirty ratio > 20%
- Primary stepdown rate > 0

---

## 8. Migration from relational to MongoDB

### When to migrate

**Good reasons:**
- Flexible/evolving schema — rapid product iteration
- Embedded data access pattern — always read user + profile + settings together
- Write-heavy workload — need horizontal write scaling
- Avoiding complex joins — pre-join via embedding

**Bad reasons:**
- "NoSQL is faster" — it depends on access patterns
- "Schema design is easier" — it's different, not easier
- "We need to scale infinitely" — 90% of applications don't

### Migration strategies

#### Strategy 1: Embed what you join most

```sql
-- Relational:
SELECT u.*, p.* FROM users u JOIN profiles p ON u.id = p.user_id

-- MongoDB: Embed profile into user
{
  "_id": "u1",
  "name": "Alice",
  "profile": {
    "bio": "Engineer",
    "avatar": "url.jpg"
  }
}
```

#### Strategy 2: Denormalize selectively

```sql
-- Relational:
SELECT o.*, c.name FROM orders o JOIN customers c ON o.customer_id = c.id

-- MongoDB: Embed customer name into order
{
  "_id": "o1",
  "customerId": "c1",
  "customerName": "Alice",
  "total": 100
}
```

**Trap:** Denormalization means data duplication. When the customer changes their name, all orders referencing that name are stale. Use a background job or change stream to update denormalized data.

#### Strategy 3: Use references for large independent data

```sql
-- Relational: orders and audit_logs
-- MongoDB: reference orderId in audit_log
{
  "_id": "a1",
  "orderId": "o1",
  "action": "shipped",
  "timestamp": ISODate("2024-01-15")
}
```

#### Strategy 4: Polymorphic collections

```sql
-- Relational: separate tables for different product types
-- MongoDB: single collection with type discriminator
{
  "_id": "p1",
  "type": "physical",
  "name": "Widget",
  "weight": 2.5,
  "dimensions": { "width": 10, "height": 5, "depth": 3 }
}

{
  "_id": "p2",
  "type": "digital",
  "name": "eBook",
  "downloadUrl": "https://...",
  "fileSizeMB": 5.2
}
```

### Migration execution

**Zero-downtime approach (similar to your 15M migration):**

1. **Dual-write phase**: Write to both MongoDB and PostgreSQL
2. **Backfill phase**: Batch migrate historical data
3. **Validation phase**: Compare counts, checksums, and sample records
4. **Cutover phase**: Switch reads to MongoDB
5. **Cleanup phase**: Remove PostgreSQL writes, drop old tables

---

## 9. Common production issues and debugging

### Issue 1: Oplog too small

**Symptom:** Secondaries falling into RECOVERING state, unable to keep up with oplog.

**Solution:** Increase oplog size:

```js
db.adminCommand({ replSetResizeOplog: 1, size: 51200 })  // 50 GB
```

### Issue 2: Write concern timeout

**Symptom:** Write operations timing out during replica set elections.

**Solution:**
- Use `w: "majority"` with `wtimeout` for critical writes
- Implement client-side retry logic
- Consider `retryWrites: true`

### Issue 3: Hot shard

**Symptom:** One shard handles disproportionate write traffic.

**Solution:**
- Review shard key — monotonic key (e.g., `_id` range, `createdAt`)?
- Switch to hashed shard key
- Add zone sharding for data locality if needed

### Issue 4: Slow aggregation pipeline

**Symptom:** Aggregation pipeline takes > 5 seconds.

**Solution:**
1. Run `explain()` on the aggregation
2. Check if `$match` is the first stage and uses indexes
3. Check for `$lookup` (one query per input document)
4. Check for `$sort` without index (32 MB RAM limit)
5. Check for `$unwind` on large arrays
6. Add `allowDiskUse: true` if needed
7. Pre-aggregate data using `$merge` or `$out`

### Issue 5: Connection pool exhaustion

**Symptom:** New connection attempts fail with timeout errors.

**Solution:**
- Increase `maxPoolSize`
- Add application-level connection retry with backoff
- Monitor connection count in `serverStatus().connections`
- Use connection pooling proxy (e.g., ProxySQL for MySQL-style pooling, or a PgBouncer-like proxy for MongoDB)

### Issue 6: Data distribution imbalance (sharded)

**Symptom:** Some shards have much more data than others.

**Solution:**
- Check chunk distribution: `sh.status()`
- Identify jumbo chunks (same shard key value for many documents)
- Consider splitting large chunks by using a more granular shard key
- Rebalance: `sh.startBalancer()` (or wait for next window)

### Issue 7: Migrating too many documents (blow past 16 MB)

**Symptom:** Document exceeds 16 MB due to unbounded embedded array growth.

**Solution:**
- Move array to separate collection (reference)
- Implement a document size limit at application layer
- Use GridFS for large files (> 16 MB)

---

## 10. Migration guide: Relational → MongoDB for multi-tenant SaaS

Your multi-tenant inventory SaaS (PostgreSQL) is a good candidate for discussing when and why you'd move to MongoDB:

### Schema comparison

**PostgreSQL (relational):**
```sql
CREATE TABLE organizations (id SERIAL PK, name TEXT, settings JSONB);
CREATE TABLE users (id SERIAL PK, org_id FK, name TEXT, email TEXT);
CREATE TABLE products (id SERIAL PK, org_id FK, name TEXT, sku TEXT UNIQUE, price DECIMAL);
CREATE TABLE inventory (id SERIAL PK, product_id FK, quantity INT, warehouse TEXT);
```

**MongoDB (document):**
```json
// Single collection — embedded settings and users
{
  "_id": "org_001",
  "name": "Acme Corp",
  "settings": { "currency": "USD", "timezone": "UTC" },
  "users": [
    { "userId": "u1", "name": "Alice", "email": "alice@acme.com", "role": "admin" },
    { "userId": "u2", "name": "Bob", "email": "bob@acme.com", "role": "viewer" }
  ],
  "createdAt": ISODate("2023-01-01"),
  "plan": "enterprise"
}

// Products — embedded inventory
{
  "_id": "prod_001",
  "orgId": "org_001",
  "name": "Widget",
  "sku": "WDG-001",
  "price": NumberDecimal("29.99"),
  "inventory": [
    { "warehouse": "WH-NY", "qty": 150 },
    { "warehouse": "WH-SF", "qty": 42 }
  ],
  "createdAt": ISODate("2023-06-01"),
  "lastUpdated": ISODate("2024-01-15")
}
```

**Trade-offs:**
- Reads: faster (no JOINs)
- Writes: slower if updating many embedded arrays
- Query flexibility: limited (e.g., "find products across ALL orgs where inventory < 10" requires $unwind)

### When to use which for the SaaS

| Query Pattern | MongoDB | PostgreSQL |
|---------------|---------|------------|
| Dashboard: show org with all users | ✅ Embed users | JOIN |
| Report: total inventory across orgs | ❌ $unwind + $group | ✅ GROUP BY |
| Audit log: who changed what? | ✅ Single document | ✅ Separate table |
| Search products by org + status | ✅ Compound index | ✅ Index |
| Complex reports (multiple JOINs) | ❌ Aggregation pipeline | ✅ SQL |

---

## 11. Disaster recovery and backups

### mongodump / mongorestore

```bash
# Dump specific database
mongodump --uri="mongodb://..." --db=mydb --out=/backups/2024-01-15/

# Dump specific collection
mongodump --uri="mongodb://..." --collection=orders --out=/backups/

# Restore
mongorestore --uri="mongodb://..." --drop /backups/2024-01-15/
```

**Trap:** `mongodump` with `oplog` flag creates a point-in-time backup only for replica sets. Without `--oplog`, the backup may not be consistent across collections.

### MongoDB tools for backup

| Tool | Method | Consistency |
|------|--------|-------------|
| mongodump | Logical (BSON) | Consistent with --oplog |
| mongorestore | Logical (BSON) | N/A |
| File system snapshot | Block-level | Crash-consistent |
| Atlas backup | Continuous | PITR |
| Ops Manager backup | Continuous | PITR |

### File system snapshot backup

```bash
# 1. Flush writes
use admin
db.fsyncLock()

# 2. Take LVM/EBS snapshot
lvcreate -L 10G -s -n mongo-snap /dev/mongodb/mongodb-data

# 3. Unlock
db.fsyncUnlock()

# 4. Mount snapshot and copy files
```

---

## 12. Interview traps cheatsheet — Senior

| Trap | The truth |
|------|-----------|
| "Replica sets with 2 nodes + arbiter are high-availability" | The arbiter provides the majority vote but holds no data. If the primary fails, the secondary promotes but the set has only one data copy. |
| "w:majority means all members acknowledge" | Majority of voting members, not all. In PSS (3 data nodes), 2 nodes, not all 3. |
| "Sharding is automatic — just enable it" | Shard key selection is the hardest decision in MongoDB. Bad shard key = non-functional cluster. |
| "Change streams are exactly like Kafka" | Change streams are ordered within a single shard, not globally (for sharded clusters). Kafka has stronger ordering guarantees. |
| "Hashed shard keys are always best" | Hashed keys prevent range queries. For geographic or time-series queries, ranged is better. |
| "Secondary reads are consistent" | Secondaries can lag. You can read data that doesn't exist on primary anymore (rollback). |
| "I don't need indexes on both sides of $lookup" | Both localField and foreignField need indexes for $lookup performance. |
| "GridFS is a file system" | GridFS stores files in two collections (fs.files + fs.chunks). It's still limited by the 16 MB per chunk. |
| "MongoDB Atlas Search replaces Elasticsearch" | Atlas Search is new and rapidly evolving, but Elasticsearch is far more mature for complex search use cases. |
| "Sharded clusters are always more performant" | Sharded clusters add latency for simple queries (mongos router overhead). Optimize single-shard first. |
</details>
