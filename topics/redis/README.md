# Redis — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant SaaS (Laravel + Redis for caching, sessions, queues), trading platform (real-time data, rate limiting), Chronos (distributed locks, leader election), 88% query reduction (caching layer)  
> **Note:** Redis is one of your strongest real-world tools. Interviewers will expect deep fluency in caching strategies, data structures, and production operations — not just `SET`/`GET`.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what kills senior loops | 5 min |

**The senior signal is caching strategy — cache invalidation, cache-aside vs write-through, distributed locking, and understanding Redis's single-threaded event loop implications.** Memorizing commands is table stakes; knowing when to use which data structure and why is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Redis core concepts (in-memory, key-value, single-threaded), data structures (String, List, Set, Sorted Set, Hash), common commands, TTL/expiry, basic persistence (RDB, AOF), basic pub/sub | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Advanced data structures (Bitmap, HyperLogLog, GeoSpatial, Streams), transactions (MULTI/EXEC, optimistic locking with WATCH), Lua scripting, pipelining, replication (master-replica), Redis Sentinel for HA, client-side caching | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Redis Cluster (architecture, hash slots, resharding, failover), caching patterns (cache-aside, write-through, write-behind, cache invalidation strategies), distributed locks (Redlock, Redisson), rate limiting (fixed window, sliding window, token bucket, GCRA), Redis as message queue (List, Pub/Sub, Streams comparison), performance tuning (slow log, latency, memory optimization, eviction policies), security (ACL, TLS, rename commands), production operations (backup, monitoring, sentinel vs cluster decision), Redis 7.x features, comparison with Valkey/KeyDB | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 150+ interview questions, code puzzles (Lua scripting, distributed lock, rate limiter, cache stampede), debugging scenarios, system design prompts | Ongoing drill |

---

## Coverage map

### Core concepts
- Redis as an in-memory data structure store
- Single-threaded event loop (IO multiplexing)
- Key-value model: keyspace, key naming conventions
- Data structures: String, List, Set, Sorted Set, Hash, Bitmap, HyperLogLog, GeoSpatial, Streams
- TTL, expiry, key eviction policies (6 policies)
- Persistence: RDB (snapshots), AOF (append-only file), hybrid
- Transactions: MULTI/EXEC/DISCARD, WATCH for optimistic locking
- Lua scripting (EVAL, EVALSHA, SCRIPT LOAD)
- Pipelining (network optimization)
- Pub/Sub (fire-and-forget messaging)
- Streams (persistent log-based messaging)

### Replication and high availability
- Master-replica replication (leader-follower)
- Replication ID, offset, partial resynchronization (psync2)
- Sentinel: monitoring, notification, automatic failover
- Sentinel quorum, consensus, split-brain handling
- Client-side Sentinel discovery

### Redis Cluster
- Hash slots (16384), consistent hashing
- Cluster bus, gossip protocol
- Resharding, rebalancing, slot migration
- Cluster failover (replica promotion)
- Client-side routing (MOVED, ASK redirects)
- Cluster limitations: multi-key operations, transactions across slots, Lua across slots

### Caching strategies
- Cache-aside (lazy loading): read from cache → miss → read DB → set cache
- Write-through: write to DB + cache atomically
- Write-behind (write-back): write to cache, async flush to DB
- Refresh-ahead: proactively refresh before expiry
- Cache invalidation: TTL, explicit delete, pattern-based
- Cache stampede prevention: locking, probabilistic early recomputation, exponential backoff
- Cache warming: pre-loading on startup
- Local + distributed caching (multi-tier)
- HTTP caching with Redis (session storage, rate limiting)

### Distributed locks
- Simple lock: `SETNX` + TTL
- Redlock algorithm (Redis-based distributed lock)
- Redlock criticism (Martin Kleppmann's analysis)
- Safe distributed locking patterns, fencing tokens
- Redisson lock implementation

### Rate limiting
- Fixed window counter: `INCR` + TTL
- Sliding window log: sorted set timestamp per user
- Sliding window counter: two counters (current + previous window)
- Token bucket: `Lua` script for atomic token consumption
- GCRA (Generic Cell Rate Algorithm)

### Performance and operations
- `INFO` command, memory stats, command stats
- Slow log (SLOWLOG GET, slowlog-log-slower-than)
- Memory optimization: hash-max-ziplist-entries, ziplist encoding, intset
- Latency monitoring (LATENCY LATEST, LATENCY HISTORY)
- Big keys analysis (redis-cli --bigkeys)
- Monitor command (dangerous in production)
- `RENAME` dangerous commands
- Backup: BGSAVE, AOF rewrite, redis-check-aof, redis-check-rdb
- Security: requirepass, ACL (Redis 6+), TLS, rename-command
- Eviction policies: allkeys-lru, volatile-lru, allkeys-lfu, volatile-lfu, volatile-ttl, allkeys-random, volatile-random, noeviction

### Redis as message queue
- List-based: LPUSH + BRPOP (simple queue)
- Pub/Sub: fire-and-forget, no persistence
- Streams: persistent, consumer groups, ACK, pending entries, consumer group rebalancing
- Comparison with Kafka/RabbitMQ

### Redis vs alternatives
- Redis vs Memcached: data structures, persistence, replication, clustering
- Redis vs Valkey (Redis fork after license change)
- Redis vs KeyDB (multithreaded Redis-compatible)
- Redis vs Dragonfly (multithreaded, higher throughput)

---

## Study order recommendation

Redis is a strong topic for you — tie every concept back to your real experience.

```
Week 1:  01-basic.md          + Basic Q&A drill (commands from memory)
Week 2:  02-intermediate.md   + Intermediate Q&A drill (Lua scripts from scratch)
Week 3:  03-senior.md         + Senior Q&A drill (caching strategy design)
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** Elasticsearch.
