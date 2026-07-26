# Redis — Intermediate

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Advanced data structures, transactions, Lua scripting, pipelining, replication, Sentinel, client-side caching  
> **Real-world anchor:** Multi-tenant SaaS (session management, cache invalidation), trading platform (real-time aggregation), Chronos (distributed locks, leader election)

---

## 1. Advanced data structures

### Bitmaps

Bitmaps treat a String as an array of bits. Operations are O(1) per bit.

```bash
# Set bit at offset
SETBIT user:1000:features 0 1    # feature 0 enabled
SETBIT user:1000:features 1 0    # feature 1 disabled

# Get bit
GETBIT user:1000:features 0      # 1

# Count bits set
BITCOUNT user:1000:features      # number of enabled features

# Bit operations across keys
BITOP AND result key1 key2       # AND bits across keys
BITOP OR result key1 key2        # OR
BITOP XOR result key1 key2       # XOR
BITOP NOT result key1            # NOT

# Find first set/clear bit
BITPOS user:1000:features 1      # first bit set to 1
```

**Use cases:**
- Feature flags per user (100M users = 12.5 MB)
- Daily active users: `SETBIT daily:2024-01-15 user_id 1`, `BITCOUNT daily:2024-01-15`
- Bloom filter alternative (without false positives)

### HyperLogLog

Probabilistic data structure for **approximate unique count**. ~12 KB per key, 0.81% error rate.

```bash
# Add elements
PFADD unique:visitors:2024-01-15 "user:1000" "user:1001"
PFADD unique:visitors:2024-01-15 "user:1000"  # duplicate ignored

# Count unique
PFCOUNT unique:visitors:2024-01-15  # 2 (not 3)

# Merge multiple HLLs
PFMERGE unique:visitors:january unique:visitors:2024-01-*  # merge multiple keys
PFCOUNT unique:visitors:january
```

**Trap:** HyperLogLog is approximate. 0.81% error is acceptable for most analytics, but not for billing/exact counts. For exact counts, use Set (but memory is O(N)).

**Trap:** `PFADD` with the same element multiple times does not increase count. HLL is idempotent.

### GeoSpatial

Store and query geo-coordinates (longitude, latitude). Uses Geohash encoding.

```bash
# Add locations
GEOADD places 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"

# Get coordinates
GEOPOS places "Palermo"               # [13.361389, 38.115556]

# Distance between two members
GEODIST places "Palermo" "Catania" km  # 166.2742

# Radius query
GEORADIUS places 15 37 100 km           # places within 100 km of (15, 37)
GEORADIUS places 15 37 100 km WITHDIST WITHCOORD COUNT 5

# Radius by member
GEORADIUSBYMEMBER places "Palermo" 200 km

# Geohash
GEOHASH places "Palermo"               # sqc8b49rny0
```

**Use cases:**
- Nearby stores/places
- Geofencing (customer within delivery area)
- Location-based search

### Streams

**Streams** are Redis's answer to persistent, scalable messaging. Think Kafka but in Redis.

```bash
# Add entry to stream
XADD mystream * sensor-id 1234 temperature 19.8
# * = auto-generate ID (timestamp-sequence format)
# Returns: "1518951480106-0"

XADD mystream MAXLEN ~ 1000 * sensor-id 1234 temperature 20.1
# MAXLEN ~ 1000: keep approximately 1000 entries, ~ for efficient trimming

# Read entries
XRANGE mystream - +                 # all entries
XRANGE mystream 1518951480106-0 +   # from specific ID
XREVRANGE mystream + - COUNT 10     # last 10 entries in reverse

# Count entries
XLEN mystream

# Read new entries (blocking)
XREAD COUNT 100 BLOCK 60000 STREAMS mystream 0
# 0 = start from beginning; $ = only new; or specific ID

# Consumer groups (like Kafka consumer groups)
XGROUP CREATE mystream mygroup $    # $ = start from new messages only
XGROUP CREATE mystream mygroup 0 MKSTREAM  # 0 = from beginning, create stream if missing

# Read from consumer group
XREADGROUP GROUP mygroup consumer1 COUNT 10 BLOCK 2000 STREAMS mystream >

# Acknowledge processing
XACK mystream mygroup "1518951480106-0"

# Pending entries (unacknowledged)
XPENDING mystream mygroup

# Claim pending entries (if consumer failed)
XCLAIM mystream mygroup consumer2 60000 "1518951480106-0"
# 60000 = min idle time in ms before reclaiming

# List groups, consumers
XINFO GROUPS mystream
XINFO CONSUMERS mystream mygroup

# Delete entry from stream
XDEL mystream "1518951480106-0"

# Trim stream
XTRIM mystream MAXLEN ~ 1000
```

**Stream use cases:**
- Event sourcing (append-only log of events)
- Message queue with persistence and ACK
- Job processing with consumer groups
- Activity feeds (fan-out with multiple consumer groups)

**Streams vs Kafka:**
| Feature | Redis Streams | Apache Kafka |
|---------|---------------|--------------|
| Persistence | RDB/AOF | Disk-based, configurable retention |
| Throughput | ~100K msg/sec per instance | ~1M msg/sec per partition |
| Consumer groups | Yes | Yes |
| Rebalancing | Manual (XCLAIM) | Automatic |
| Retention | MAXLEN (cap) | Configurable (time/size) |
| Ordering | Per shard | Per partition |
| Exactly-once | Not built-in | Via idempotent producer + transactional |

---

## 2. Transactions (MULTI/EXEC/WATCH)

Redis transactions are **not like SQL transactions**. They are a **queue of commands** executed sequentially and atomically.

```bash
# MULTI starts a transaction
MULTI                           # QUEUED
SET account:1000 balance 100
INCRBY account:1000 balance -20
SET account:2000 balance 200
INCRBY account:2000 balance 20
EXEC                            # executes all commands atomically

# Discard (cancel transaction)
DISCARD

# WATCH — optimistic locking
WATCH account:1000              # watch key for changes
balance = GET account:1000      # read value outside transaction
MULTI
SET account:1000 balance (balance - 20)
SET account:2000 balance (balance + 20)  # BUG: can't reference local var
EXEC                            # fails if account:1000 changed between WATCH and EXEC
```

**Trap:** Redis transactions **do not support rollback**. If a command has a syntax error, the transaction is discarded. If a command fails at runtime (e.g., wrong type), the error is returned but other commands still execute.

**Trap:** Lua scripting is often preferred over `MULTI`/`EXEC` because scripts let you use logic (if/else, loops) and return values:

```bash
# MULTI/EXEC can't read its own writes mid-transaction
MULTI
SET counter 10
GET counter          # Returns QUEUED, not "10"
EXEC                 # Returns [OK, "10"] — GET ran AFTER SET
```

### WATCH for optimistic locking

```bash
WATCH balance:1000
balance = GET balance:1000      # e.g., 100
MULTI
SET balance:1000 (balance - 20) # Oops — this doesn't work in Redis CLI
EXEC                            # Need Lua for this logic
```

**Use WATCH + MULTI/EXEC for CAS (Compare-and-Swap):**

```python
import redis
r = redis.Redis()

def transfer(from_acct, to_acct, amount):
    while True:
        r.watch(from_acct)
        balance = int(r.get(from_acct) or 0)
        if balance < amount:
            r.unwatch()
            raise Exception("Insufficient funds")
        pipe = r.pipeline(transaction=True)
        pipe.decrby(from_acct, amount)
        pipe.incrby(to_acct, amount)
        try:
            pipe.execute()
            break
        except redis.WatchError:
            continue  # retry if watched key changed
```

---

## 3. Lua scripting

Lua scripting lets you execute complex logic **atomically** on the server side. Scripts are deterministic (no random, no system calls by default, but `REDIS` library has `math.random`).

### Basic Lua script

```lua
-- transfer.lua
local from = KEYS[1]
local to = KEYS[2]
local amount = tonumber(ARGV[1])

local balance = redis.call("GET", from)
if not balance then
    return { err = "Account not found" }
end

balance = tonumber(balance)
if balance < amount then
    return { err = "Insufficient funds" }
end

redis.call("DECRBY", from, amount)
redis.call("INCRBY", to, amount)

return { ok = "Success", from_balance = balance - amount }
```

```bash
# Load script (returns SHA)
SCRIPT LOAD "local from = KEYS[1]..."

# Run script by SHA (cached)
EVALSHA <sha> 2 account:1000 account:2000 50

# Run script directly
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey "hello"

# Check if script exists
SCRIPT EXISTS <sha>

# Flush script cache
SCRIPT FLUSH

# Kill running script (if it's blocking)
SCRIPT KILL
```

### Lua sandbox

- Scripts cannot access global variables (strict mode)
- No file I/O or network calls
- `redis.call()` — raises error on failure (halts script)
- `redis.pcall()` — returns error table on failure (continues script)
- `redis.log(redis.LOG_WARNING, "message")` — logging
- `redis.setresp()` — change response type (3 = RESP3)

### Common Lua patterns

```lua
-- Rate limiter: sliding window
local key = KEYS[1]           -- "ratelimit:user:1000"
local limit = tonumber(ARGV[1]) -- 100
local window = tonumber(ARGV[2]) -- 60 (seconds)

local now = redis.call("TIME")[1]  -- current timestamp
local window_start = now - window

-- Remove old entries
redis.call("ZREMRANGEBYSCORE", key, 0, window_start)

local count = redis.call("ZCARD", key)

if count >= limit then
    return 0  -- rate limited
end

redis.call("ZADD", key, now, now .. ":" .. math.random())
redis.call("EXPIRE", key, window)

return 1  -- allowed
```

```lua
-- Distributed lock with safety
local key = KEYS[1]
local ttl = tonumber(ARGV[1])
local owner = ARGV[2]

if redis.call("SET", key, owner, "NX", "PX", ttl) then
    return 1  -- lock acquired
end

-- Optional: check if lock is expired and reacquire
local current = redis.call("GET", key)
if current == owner then
    redis.call("PEXPIRE", key, ttl)  -- extend lock
    return 1
end

return 0  -- lock not acquired
```

```lua
-- Cache stampede prevention (probabilistic early recomputation)
local key = KEYS[1]
local ttl = tonumber(ARGV[1])
local beta = tonumber(ARGV[2])  -- 1.0 by default

local value = redis.call("GET", key)
if not value then
    return nil  -- cache miss, recompute
end

local remaining_ttl = redis.call("TTL", key)
-- Probability of early recomputation = (beta * log(random())) / remaining_ttl
-- Negative or > 1 = don't recompute
local chance = (beta * math.log(math.random())) / remaining_ttl

if chance < -1 or chance > 1 then
    return value  -- serve cached value
end

return nil  -- trigger early recomputation
```

### EVAL vs EVALSHA

- `EVAL` — sends entire script each call. Use for one-off scripts.
- `EVALSHA` — sends SHA of cached script. Use for repeated scripts (reduces bandwidth).
- If script not cached, `EVALSHA` returns `NOSCRIPT` error — fallback to `EVAL`.

```python
import redis
r = redis.Redis()

sha = r.script_load("return redis.call('GET', KEYS[1])")
try:
    result = r.evalsha(sha, 1, "mykey")
except redis.ResponseError as e:
    if "NOSCRIPT" in str(e):
        result = r.eval("return redis.call('GET', KEYS[1])", 1, "mykey")
```

**Trap:** Lua scripts are **replicated to replicas**. In Redis replication, scripts are replicated as `EVAL` (not `EVALSHA`) to ensure replicas have the script. All writes in a script are replicated atomically.

---

## 4. Pipelining

Pipelining sends multiple commands without waiting for individual responses. **Not a transaction** — commands may interleave with other connections.

```bash
# Without pipeline: 5 round trips
SET a 1
GET a
INCR b
GET b
SET c 3

# With pipeline: 1 round trip
# Client sends: SET a 1, GET a, INCR b, GET b, SET c 3
# Server processes all, returns all responses
```

```python
# Python example
pipe = r.pipeline()
pipe.set('a', 1)
pipe.get('a')
pipe.incr('b')
pipe.get('b')
pipe.set('c', 3)
results = pipe.execute()  # [True, b'1', 2, b'1', True] — one round trip
```

**Pipelining vs Transactions:**
- Pipeline: faster, non-atomic, commands may interleave
- Transaction (MULTI/EXEC): atomic, slightly slower, guaranteed sequential

**Pipelining vs Lua:**
- Pipeline: multiple commands, no logic, no atomicity guarantee
- Lua: single command with logic, atomic

**Pipelining use case:** Batch inserts, batch cache loading, bulk clearing expired data.

**Trap:** Pipelining uses memory on both client and server to queue commands. Sending 1M commands in one pipeline can exhaust memory.

---

## 5. Replication (Master-Replica)

### How it works

1. Master sends `FULLRESYNC` to replica (RDB + replication stream)
2. Replica loads RDB, then applies continuous replication stream
3. On disconnect, replica uses `PSYNC2` (Redis 4.0+) for partial resynchronization
4. Partial resync uses **replication ID** + **offset** to reconnect without full sync

```
Master (port 6379)
  │
  ├── Replica 1 (port 6380) — read-only, replicates all data
  ├── Replica 2 (port 6381) — read-only
  └── Replica 3 (port 6382) — read-only, optional
```

```bash
# Replica configuration (redis.conf on replica)
replicaof 127.0.0.1 6379
replica-read-only yes
replica-priority 100           # lower = more likely to be promoted by Sentinel
replica-lazy-flush no          # async flush on full sync (less blocking)

# On master
replica-serve-stale-data yes   # replica serves data during sync (or no)
min-replicas-to-write 2        # only accept writes if N replicas connected
min-replicas-max-lag 10        # max lag in seconds
```

### Replication ID and offset

```
PSYNC2 handles three scenarios:
1. Clean disconnect → reuse old replication ID + offset → partial resync
2. Restart after crash → new replication ID (but remembers old) → partial resync if offset within backlog
3. Master failover → replica becomes new master, inherits replication ID
```

### Replication backlog

The replication backlog is a circular buffer in memory on the master:

```bash
# Configuration
repl-backlog-size 1mb          # default 1 MB, increase for large writes
repl-backlog-ttl 3600           # seconds to keep backlog when no replicas connected
```

If the replica's offset falls outside the backlog window, a **full resync** is required. For your trading platform (high writes), increase `repl-backlog-size` to 256 MB+.

### Read scaling with replicas

```python
# Go-redis: read from replica
rdb := redis.NewFailoverClusterClient(&redis.FailoverOptions{
    MasterName:    "mymaster",
    SentinelAddrs: []string{":26379"},
    ReadOnly:      true,
    RouteByLatency: true,  // read from lowest-latency node
})
```

**Trap:** Replicas are eventually consistent. A write to master may not be visible on a replica for milliseconds (or longer under load). Use **read from master** for consistency-critical reads.

---

## 6. Redis Sentinel

Sentinel provides **high availability** without Redis Cluster:

```
Sentinel 1 (port 26379) ──┐
Sentinel 2 (port 26379) ──┼── monitors master + replicas
Sentinel 3 (port 26379) ──┘
         │
         ├── Master (port 6379)
         ├── Replica 1 (port 6380)
         └── Replica 2 (port 6381)
```

### Sentinel responsibilities

1. **Monitoring**: Checks if master/replicas are up
2. **Notification**: Alerts via scripts when issues detected
3. **Automatic failover**: Promotes replica to master when master fails
4. **Configuration provider**: Clients discover current master via Sentinel

### Sentinel configuration

```bash
# sentinel.conf
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel auth-pass mymaster password123
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

| Parameter | Meaning |
|-----------|---------|
| `mymaster` | Name of the master (cluster name) |
| `127.0.0.1 6379` | Master address (Sentinel uses this to discover replicas) |
| `2` | **Quorum** — number of Sentinels that must agree master is down |
| `down-after-milliseconds` | Time without ping to consider node down |
| `failover-timeout` | Max time to complete a failover |
| `parallel-syncs` | How many replicas sync with new master simultaneously |

### Sentinel failover process

1. Sentinel detects master is down (SDOWN — subjective)
2. Sentinels communicate: if ≥ quorum agree → ODOWN (objective down)
3. Sentinel leader elected (Raft-like)
4. Leader picks a replica for promotion:
   - Remove replicas with priority 0
   - Remove replicas disconnected > down-after-milliseconds * 10
   - Remove replicas with outdated replication
   - Pick by priority (lowest), then replication offset (highest), then run ID
5. Sentinel sends `SLAVEOF NO ONE` to chosen replica → it becomes master
6. Sentinels reconfigure other replicas to replicate from new master

### Client-side Sentinel discovery

```python
# Python (redis-py)
from redis.sentinel import Sentinel

sentinel = Sentinel([('localhost', 26379)], socket_timeout=0.1)
master = sentinel.master_for('mymaster', socket_timeout=0.1)
replica = sentinel.slave_for('mymaster', socket_timeout=0.1)

master.set('key', 'value')
print(replica.get('key'))  # eventually consistent read
```

```go
// Go (go-redis)
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "mymaster",
    SentinelAddrs: []string{":26379"},
    SentinelPassword: "password123",
})
```

### Sentinel split-brain

**Scenario:** Network partition isolates master from sentinels but not from clients.

- Sentinel promotes replica → new master exists
- Old master is still accepting writes (because clients on the other side of partition connect to it)
- When partition heals, old master becomes replica of new master → data loss

**Mitigation:**
- Use `min-replicas-to-write` + `min-replicas-max-lag` on master:
  ```bash
  min-replicas-to-write 1
  min-replicas-max-lag 10
  ```
  Master rejects writes if fewer than 1 replica is connected with < 10s lag. During partition, old master has no replicas → writes rejected.

---

## 7. Client-side caching (Redis 6+)

Redis 6+ supports **server-assisted client-side caching** (also called tracking). The server notifies the client when a key changes, so the client can invalidate its local cache.

```bash
# Client connects and enables tracking
CLIENT TRACKING ON

# Client caches key locally
GET mykey          # returns value, server remembers this client cached mykey

# When mykey is updated by another client:
SET mykey newvalue # server sends INVALIDATE "mykey" to the first client
```

### Tracking modes

| Mode | How it works |
|------|--------------|
| Default | Server tracks which keys each client has accessed. On key change, sends invalidation message. Memory on server O(keys × clients). |
| BCAST | Client specifies key prefixes. Server broadcasts invalidations for all matching keys. No per-client tracking. |
| OPTIN | Client must explicitly `CLIENT CACHING YES` before `GET`. Selective caching. |
| OPTIN with NOLOOP | Don't send invalidation to the same client that made the change. |

```bash
# BCAST mode
CLIENT TRACKING ON BCAST PREFIX user: PREFIX org:

# Invalidation messages stream to client asynchronously
# Client must have a RESP3 connection (pushing mode)
```

**Use case:** Your SaaS could use client-side caching for frequently-read organization settings. When an admin updates settings, all connected app servers get invalidation → refresh from Redis.

---

## 8. Memory optimization

### Internal encoding

Redis uses different internal encodings based on data size:

```bash
OBJECT ENCODING key
```

| Type | Encodings | Conditions |
|------|-----------|------------|
| String | `int` (integer), `embstr` (≤ 44 bytes), `raw` (> 44 bytes) | Automatic |
| List | `ziplist` (small), `quicklist` (default) | `list-max-ziplist-size -2` |
| Hash | `ziplist` (small), `listpack` (7.0+/small), `hashtable` (large) | `hash-max-ziplist-entries 512`, `hash-max-ziplist-value 64` |
| Set | `intset` (all integers, small), `hashtable` (large) | `set-max-intset-entries 512` |
| Sorted Set | `ziplist` (small), `skiplist` | `zset-max-ziplist-entries 128`, `zset-max-ziplist-value 64` |

### Memory optimization strategies

```bash
# 1. Use Hashes instead of Strings for objects
# Instead of 3 keys:
SET user:1000:name "Alice"
SET user:1000:email "alice@example.com"
SET user:1000:age 30
# Use 1 hash (saves key overhead):
HSET user:1000 name "Alice" email "alice@example.com" age 30

# 2. Short key names
SET org:42:usr:1000:nm "Alice"  # 11 bytes key
# vs
SET organization:42:user:1000:name "Alice"  # 32 bytes key

# 3. Shorter keys in Hashes
HSET user:1000 nm "Alice" em "alice@example.com"

# 4. Use integer IDs (Redis stores integer strings more efficiently)
# "int" encoding for strings under 21 digits that are valid integers

# 5. Configure ziplist thresholds
hash-max-ziplist-entries 1024
hash-max-ziplist-value 128
```

### Memory stats

```bash
# Memory usage
INFO memory
# used_memory: 2GB — total allocated
# used_memory_rss: 2.5GB — RSS (includes fragmentation)
# used_memory_peak: 3GB — peak usage
# used_memory_lua: 37KB — Lua state
# maxmemory: 4GB — configured limit
# mem_fragmentation_ratio: 1.25 — > 1.5 indicates fragmentation

# Memory per key
MEMORY USAGE user:1000           # approximate bytes for key + value
MEMORY STATS                      # detailed memory breakdown
MEMORY PURGE                      # reclaim memory (rarely needed)
```

---

## 9. Practical drills

### Drill 1 — Lua script: Atomic counter reset

Write a Lua script that increments a counter but resets it to 0 if it's been more than 60 seconds since the last increment. Return the count.

<details>
<summary>Answer</summary>

```lua
local key = KEYS[1]
local ttl = tonumber(ARGV[1])  -- 60 seconds

local last_reset = redis.call("GET", key .. ":last_reset")
local now = redis.call("TIME")[1]

if not last_reset or (now - tonumber(last_reset)) >= ttl then
    redis.call("SET", key, 1)
    redis.call("SET", key .. ":last_reset", now)
    return 1
end

return redis.call("INCR", key)
```

Usage: `EVAL "<script>" 1 counter:page_views 60`
</details>

### Drill 2 — Redis Stream vs Pub/Sub

You need to notify 3 microservices when a new order is created. Each service must process every order. The notification must survive a service restart. Which Redis feature do you use?

<details>
<summary>Answer</summary>

**Streams with consumer groups.** Each microservice gets its own consumer group:

```bash
# Each microservice creates its own group
XGROUP CREATE orders:stream email-service $ MKSTREAM
XGROUP CREATE orders:stream inventory-service $ MKSTREAM
XGROUP CREATE orders:stream analytics-service $ MKSTREAM

# Each service reads independently
# Service A:
XREADGROUP GROUP email-service consumer-1 COUNT 1 BLOCK 5000 STREAMS orders:stream >
# Process → XACK

# Service B:
XREADGROUP GROUP inventory-service consumer-1 COUNT 1 BLOCK 5000 STREAMS orders:stream >
# Process → XACK
```

Pub/Sub would NOT work because:
- No persistence — messages lost if subscriber is down
- No ACK — no way to track processing
- Fire-and-forget — subscriber must be online
</details>

### Drill 3 — Replication lag monitoring

Write a command sequence to check replication lag.

<details>
<summary>Answer</summary>

```bash
# On master:
INFO replication
# role:master
# connected_slaves:2
# slave0:ip=...,port=...,state=online,offset=12345,lag=0
# slave1:ip=...,port=...,state=online,offset=12300,lag=2

# On replica:
INFO replication
# role:slave
# master_last_io_seconds_ago: 0
# master_sync_in_progress: 0
# slave_read_repl_offset: 12345
# slave_repl_offset: 12345

# In production, monitor:
# - master_repl_offset - slave_repl_offset (lag in bytes)
# - lag field in MASTER output (seconds)
# - master_last_io_seconds_ago (seconds since last communication)
```
</details>

---

## Interview traps cheatsheet — Intermediate

| Trap | The truth |
|------|-----------|
| "Redis transactions are like SQL transactions" | No rollback, no read-your-writes mid-transaction, no isolation. Lua is better for complex logic. |
| "Pub/Sub is suitable for reliable messaging" | No persistence. Subscriber must be online. Use Streams for reliable delivery. |
| "Pipelining is the same as transactions" | Pipeline = batch, not atomic. Commands can interleave with other connections. |
| "Lua scripts are always safe" | `redis.call()` halts script on error. Use `redis.pcall()` for error handling. |
| "Sentinel guarantees no data loss" | Split-brain can cause data loss. Use min-replicas-to-write mitigation. |
| "HyperLogLog is exact" | 0.81% error rate. Fine for dashboards, not for billing. |
| "Replicas always have the same data as master" | Eventually consistent. Replication lag exists. Read from master for consistency. |
| "Streams are always better than Pub/Sub" | Pub/Sub is lower latency (no persistence overhead). Use Pub/Sub for real-time notifications where message loss is acceptable. |
| "OBJECT ENCODING is just for debugging" | Important for understanding memory usage. A hash with 513 entries uses 10x more memory than one with 512 (threshold). |
| "Sentinel is a cluster" | Sentinel provides HA, not sharding. Redis Cluster provides both. |
</details>
