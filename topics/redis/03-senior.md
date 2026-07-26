# Redis — Senior

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Redis Cluster, caching strategies, distributed locks (Redlock, Redisson), rate limiting algorithms, performance tuning, production operations, Redis as queue, Redis 7.x features, comparisons with Valkey/KeyDB  
> **Real-world anchors:** Multi-tenant SaaS (caching strategy, session management, rate limiting), trading platform (real-time leaderboard, stream processing), Chronos (distributed locks, leader election), 88% query reduction (caching layer design)

---

## 1. Redis Cluster

### Architecture

Redis Cluster provides **automatic sharding** and **high availability** without a separate Sentinel:

```
Client → Redis Cluster
         ├── Node A (master, slots 0-5460)
         │    └── Replica A1
         ├── Node B (master, slots 5461-10922)
         │    └── Replica B1
         └── Node C (master, slots 10923-16383)
              └── Replica C1
```

### Key concepts

| Concept | Description |
|---------|-------------|
| **Hash slots** | 16384 slots. Key → CRC16(key) % 16384 → slot → node |
| **Gossip protocol** | Nodes exchange information via cluster bus (port +10000) |
| **Configuration epoch** | Version number for conflict resolution during failover |
| **MOVED redirect** | Client sent to wrong node — must update routing table |
| **ASK redirect** | Slot migration in progress — client retries on target node |
| **Replica migration** | Orphan replicas auto-migrate to under-replicated masters |

### Hash slot calculation

```bash
# How a key maps to a slot
HASH SLOT mykey          # shows slot number
CLUSTER KEYSLOT mykey    # same

# Hash tags — force multiple keys to same slot
# {user:1000}:orders and {user:1000}:cart
# The hash is computed on the tag only (text within {})
# Use for multi-key operations in cluster mode
```

### Cluster configuration

```bash
# redis.conf for cluster node
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-replica-validity-factor 10
cluster-migration-barrier 1
cluster-require-full-coverage yes  # no writes if any slot is uncovered (default)
```

### Cluster management commands

```bash
# Create cluster (redis-cli)
redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
  --cluster-replicas 1

# Cluster info
CLUSTER INFO               # cluster state, slots assigned
CLUSTER NODES              # all nodes, their roles, slots

# Check cluster health
redis-cli --cluster check 127.0.0.1:6379

# Resharding (move slots between nodes)
redis-cli --cluster reshard 127.0.0.1:6379 \
  --cluster-from node-id \
  --cluster-to node-id \
  --cluster-slots 1000

# Rebalance (evenly distribute slots)
redis-cli --cluster rebalance 127.0.0.1:6379

# Add node
redis-cli --cluster add-node 127.0.0.1:6390 127.0.0.1:6379

# Add replica
redis-cli --cluster add-node 127.0.0.1:6391 127.0.0.1:6379 --cluster-slave

# Remove node (migrate slots first, then forget)
redis-cli --cluster del-node 127.0.0.1:6390 node-id

# Failover (force replica promotion)
CLUSTER FAILOVER             # coordinated (replica syncs then takes over)
CLUSTER FAILOVER FORCE       # immediate (may lose data)
CLUSTER FAILOVER TAKEOVER    # forced without consensus (for manual recovery)
```

### Client-side routing

```bash
# Client receives MOVED on wrong node
GET key
-MOVED 12345 127.0.0.1:6381   # slot 12345 is on node 6381

# Client must update routing table and retry
GET key
-ASK 12345 127.0.0.1:6381    # slot is being migrated, ask target node

# Smart clients (redis-cli -c, ioredis Cluster, go-redis ClusterClient)
# automatically track slot → node mapping and handle redirects
```

### Cluster limitations

1. **Multi-key operations restricted** — keys must be in same slot (use hash tags)
2. **No multi-key transactions** across slots
3. **No Lua scripts with keys in multiple slots** (Redis 7.0+ allows cross-slot when using hash tags)
4. **Increased latency** — client must route to the correct node
5. **Resharding is complex** — affects performance during slot migration
6. **No SELECT database** — cluster only supports database 0

```bash
# Works: same slot via hash tag
MGET {user:1000}:orders {user:1000}:cart

# Fails (CROSSSLOT error):
MGET user:1000:orders user:2000:cart
```

### Cluster vs Sentinel decision

| Factor | Choose Sentinel | Choose Cluster |
|--------|----------------|----------------|
| Data size | < 10-20 GB (fits single node) | > 20 GB (need sharding) |
| Write throughput | < 100K ops/sec | > 100K ops/sec |
| Complexity tolerance | Lower | Higher |
| Multi-key operations | No restrictions | Restricted |
| Number of nodes | 3-5 | 6+ (3 masters + replicas) |

**For your SaaS:** Sentinel is likely sufficient (single node with replicas). Cluster is needed if you're caching 50+ GB of data or need > 100K writes/sec.

---

## 2. Caching strategies

### Cache-aside (Lazy Loading)

The most common pattern — application checks cache first, falls back to database on miss.

```
Read:  GET key → miss → SELECT DB → SET cache → return
Write: UPDATE DB → DEL cache (or SET)
```

```php
function getUser($userId) {
    $cacheKey = "user:{$userId}";

    $user = Redis::get($cacheKey);
    if ($user) {
        return json_decode($user);
    }

    $user = DB::table('users')->find($userId);
    if ($user) {
        Redis::setex($cacheKey, 3600, json_encode($user));
    }

    return $user;
}

function updateUser($userId, $data) {
    DB::table('users')->where('id', $userId)->update($data);
    Redis::del("user:{$userId}");  // Invalidate cache
}
```

**Pros:** Simple, handles cache misses gracefully, only caches what's requested.  
**Cons:** Cache stampede on first request after expiry, stale data until cache expiry.

### Write-through

Application writes to database and cache atomically (in the same transaction).

```
Write: WRITE DB → SET cache (both succeed or neither)
Read:  GET key → hit → return
```

**Pros:** Cache always consistent with database (within transaction boundaries).  
**Cons:** Slower writes, cache stores data that may never be read.

### Write-behind (Write-back)

Application writes to cache, async process flushes to database.

```
Write: SET cache → return (async: batch flush to DB)
Read:  GET key → hit → return (or miss → DB)
```

**Pros:** Very fast writes, can batch updates, reduces DB write load.  
**Cons:** Data loss risk if cache fails before DB flush, eventual consistency.

### Refresh-ahead

Proactively refresh cache before TTL expires (using probabilistic early recomputation).

```php
function getWithRefreshAhead($key, $ttl, $recomputeFn) {
    $data = Redis::get($key);
    if ($data) {
        $remainingTtl = Redis::ttl($key);
        $chance = (1.0 * log(mt_rand() / mt_getrandmax())) / $remainingTtl;
        if ($chance < -1) {
            // Trigger async refresh
            dispatch(fn() => recomputeCache($key, $ttl, $recomputeFn));
        }
        return json_decode($data);
    }

    return recomputeCache($key, $ttl, $recomputeFn);
}

function recomputeCache($key, $ttl, $recomputeFn) {
    $data = $recomputeFn();
    Redis::setex($key, $ttl, json_encode($data));
    return $data;
}
```

**Pros:** No cache stampede, always fresh data for active keys.  
**Cons:** Complex implementation, background CPU cost for recomputation.

### Cache invalidation strategies

| Strategy | Description | Best for |
|----------|-------------|----------|
| **TTL** | Automatic expiry | Simple caching |
| **Explicit DELETE** | Delete cache key on write | Strong consistency |
| **Pattern invalidation** | Delete by pattern (SCAN + DEL) | Batch updates (e.g., all user caches) |
| **Versioned keys** | `user:v2:1000`, increment version on schema change | Schema migrations |
| **Change Data Capture** | Invalidate via DB triggers or WAL | Complex multi-service caching |

### Cache stampede prevention

When a popular cache key expires simultaneously, many requests hit the database:

1. **Locking**: Only one request recomputes, others wait (or serve stale)
   ```php
   // Mutex lock for cache recomputation
   $lock = Redis::set("lock:{$key}", "1", "EX", 5, "NX");
   if ($lock) {
       $data = recomputeExpensiveData();
       Redis::setex($key, 300, json_encode($data));
   } else {
       usleep(50000); // 50ms
       return json_decode(Redis::get($key)); // likely available now
   }
   ```

2. **Probabilistic early recomputation**: See refresh-ahead above

3. **Stale while revalidate**: Serve stale data + async refresh in background
   ```php
   $staleData = Redis::get("stale:{$key}"); // Extended TTL stale copy
   if ($staleData && mt_rand(1, 10) === 1) {
       dispatch(fn() => recomputeCache($key)); // 10% chance of refresh
   }
   return $staleData ?: recomputeCache($key);
   ```

4. **Exponential backoff**: Retry with increasing delay

### Cache warming

Pre-load cache on application startup:

```php
// On deploy / cache flush
function warmCache() {
    $tenants = DB::table('organizations')->pluck('id');
    foreach ($tenants as $orgId) {
        $dashboard = DashboardService::calculateForOrg($orgId);
        Redis::setex("org:{$orgId}:dashboard", 300, json_encode($dashboard));
    }
}
```

### Multi-tier caching (L1 + L2)

```
App Server (local memory cache L1 — small, fast)
  ↓ miss
Redis (distributed cache L2 — larger, shared)
  ↓ miss
Database
```

```php
class MultiTierCache {
    private $local = []; // In-process array (or APCu/Symfony Cache)
    private $ttl = 60;

    public function get($key) {
        // L1 — local memory (fastest)
        if (isset($this->local[$key])) {
            return $this->local[$key];
        }

        // L2 — Redis (distributed)
        $value = Redis::get($key);
        if ($value !== null) {
            $this->local[$key] = json_decode($value);
            return $this->local[$key];
        }

        return null;
    }

    public function set($key, $value) {
        $this->local[$key] = $value;
        Redis::setex($key, $this->ttl, json_encode($value));
    }

    public function invalidate($key) {
        unset($this->local[$key]);
        Redis::del($key);
    }
}
```

**Trap:** L1 cache invalidation across app servers is hard. Use Redis Pub/Sub or client tracking to invalidate L1 caches on other servers.

### Cache patterns for your SaaS

```php
// Organization dashboard (expensive query, cached 5 min)
$dashboard = Cache::remember("org:{$orgId}:dashboard", 300, function () use ($orgId) {
    return DB::table('orders')
        ->where('org_id', $orgId)
        ->selectRaw('COUNT(*) as total_orders, SUM(total) as revenue')
        ->first();
});

// Session (user-specific, TTL matches session lifetime)
Cache::put("session:{$token}", $userData, 86400);

// Rate limit counters (short-lived, high write)
$key = "ratelimit:api:{$userId}:{$window}";
$attempts = Redis::incr($key);
if ($attempts === 1) Redis::expire($key, 60);

// Product catalog (frequently read, rarely changed)
// Cache-aside with tag-based invalidation
Cache::tags(["products", "org:{$orgId}"])
    ->remember("product:{$productId}", 3600, fn() => Product::find($productId));
```

---

## 3. Distributed locks

### Simple lock (naive)

```bash
SET lock:resource "worker_1" NX EX 30
# ... do work ...
DEL lock:resource
```

**Problems:**
- No safe release — if worker crashes before `DEL`, lock stuck until TTL
- No safe detection — worker A acquires lock, task takes > 30s, lock expires, worker B acquires lock, worker A finishes and DEL worker B's lock
- No fencing — no way for resource to verify the lock holder

### Safe lock with Lua

```lua
-- Lock.lua
local key = KEYS[1]
local owner = ARGV[1]
local ttl = tonumber(ARGV[2])

return redis.call("SET", key, owner, "NX", "PX", ttl)

-- Unlock.lua
local key = KEYS[1]
local owner = ARGV[1]
local value = redis.call("GET", key)

if value == owner then
    redis.call("DEL", key)
    return 1
end

return 0
```

### Redlock algorithm

Proposed by Redis author (antirez). Works with **N independent Redis masters** (typically 5):

```
1. Get current time (T1)
2. Try to acquire lock on all N nodes sequentially with short timeout (~5-50ms)
3. If acquired on majority (N/2 + 1) nodes AND total elapsed < lock TTL → lock acquired
4. Lock validity = TTL - elapsed_time
5. To release: delete lock key on ALL nodes
```

```python
import redis
import time

class RedLock:
    def __init__(self, nodes, retry_count=3, retry_delay=200):
        self.nodes = [redis.Redis(host=n['host'], port=n['port']) for n in nodes]
        self.quorum = len(nodes) // 2 + 1
        self.retry_count = retry_count
        self.retry_delay = retry_delay / 1000  # seconds

    def acquire(self, resource, ttl, owner):
        for _ in range(self.retry_count):
            start = time.time() * 1000  # ms
            acquired = 0

            for node in self.nodes:
                try:
                    if node.set(resource, owner, nx=True, px=ttl):
                        acquired += 1
                except redis.RedisError:
                    pass  # node unavailable, skip

            elapsed = (time.time() * 1000) - start
            validity = ttl - elapsed

            if acquired >= self.quorum and validity > 0:
                return True, validity

            # Release partial locks
            for node in self.nodes:
                try:
                    if node.get(resource) == owner:
                        node.delete(resource)
                except redis.RedisError:
                    pass

            time.sleep(self.retry_delay)

        return False, 0
```

### Redlock criticism (Martin Kleppmann)

Martin Kleppmann's 2016 analysis highlighted issues:

1. **Clock drift**: If a Redis node's clock jumps forward, TTL expires early → two processes hold the lock
2. **GC pause**: Process holds lock, GC pauses for 30s, lock expires, another process acquires lock, first process resumes thinking it still holds lock
3. **Network delay**: Similar to GC pause — message delayed enough for lock to expire

**Mitigations:**
- Use **fencing tokens** — monotonically increasing tokens checked by the protected resource:
  ```python
  # Redis automatically increments the lock value
  token = INCR lock_token_counter
  result = SET lock:resource token NX EX 30
  # On unlock: compare token
  if GET lock:resource == token:
      DEL lock:resource
  ```
- **Resource-level fencing**: The protected resource (database, file) checks the token. If an older token arrives, reject it.

### Redlock vs simpler alternatives

| Approach | Complexity | Safety | Best for |
|----------|-----------|--------|----------|
| Single Redis `SET NX EX` | Low | Good enough for most | Simple mutual exclusion |
| `SET NX EX` + Lua unlock | Low | Good (with owner check) | Production use |
| Redlock | High | Theoretical issues | Multi-node environments |
| ZooKeeper/etcd locks | Medium | Stronger guarantees | Critical infrastructure |

**For Chronos or your SaaS:** Single Redis lock with proper owner check (Lua) is sufficient for most use cases. Redlock is overkill unless you have multiple Redis nodes in different datacenters.

### Redisson (Java) lock implementation

Redisson is a popular Redis client with built-in distributed lock support:

```java
RLock lock = redisson.getLock("lock:resource");
lock.lock(30, TimeUnit.SECONDS);
try {
    // do work
} finally {
    lock.unlock();
}
```

Redisson implements **automatic lock extension** (watchdog) — if the lock holder is still working, it automatically extends the TTL (default: every 10s, extend by 30s). This solves the "lock expires before work completes" problem.

---

## 4. Rate limiting algorithms

### Fixed window counter

Simplest approach — count requests in a fixed time window.

```python
import time

def is_rate_limited(user_id, limit=100, window=60):
    key = f"ratelimit:{user_id}:{int(time.time() / window)}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window)
    return count > limit
```

**Problem:** Burst at window boundary — 100 requests allowed at 10:59:59, another 100 at 11:00:00 (200 requests in 2 seconds).

### Sliding window log

Track each request timestamp and count within a rolling window.

```lua
-- sliding_window.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2]) -- seconds
local now = tonumber(ARGV[3]) -- current timestamp

local window_start = now - window

-- Remove old entries
redis.call("ZREMRANGEBYSCORE", key, 0, window_start)

local count = redis.call("ZCARD", key)

if count >= limit then
    return 0  -- rate limited, return ttl of oldest entry
end

redis.call("ZADD", key, now, now .. ":" .. math.random(0, 1000000))
redis.call("EXPIRE", key, window)

return 1  -- allowed
```

**Pros:** Accurate sliding window, no burst at boundaries.  
**Cons:** Memory O(limit) per user, O(log N) per request for ZADD/ZREMRANGE.

### Sliding window counter

Approximation using two counters: current window + previous window.

```python
def is_rate_limited_sliding(user_id, limit=100, window=60):
    now = time.time()
    current_window = int(now / window)
    previous_window = current_window - 1

    current_key = f"ratelimit:{user_id}:{current_window}"
    previous_key = f"ratelimit:{user_id}:{previous_window}"

    current_count = int(redis.get(current_key) or 0)
    previous_count = int(redis.get(previous_key) or 0)

    # Percentage of current window elapsed
    elapsed = (now % window) / window

    # Estimate: previous * (1 - elapsed) + current
    estimated = previous_count * (1 - elapsed) + current_count

    if estimated >= limit:
        return True

    # Increment current counter
    pipe = redis.pipeline()
    pipe.incr(current_key)
    expire_at = (current_window + 1) * window + 1
    pipe.expireat(current_key, expire_at)
    pipe.execute()

    return False
```

**Pros:** Low memory, no cleanup overhead, fast.  
**Cons:** Approximate (error up to ~1 request per window).

### Token bucket

A bucket holds N tokens. Each request consumes a token. Tokens are added at rate R per second.

```lua
-- token_bucket.lua
local key = KEYS[1]
local max_tokens = tonumber(ARGV[1])   -- burst size
local rate = tonumber(ARGV[2])         -- tokens per second
local cost = tonumber(ARGV[3])         -- typically 1

local bucket = redis.call("HMGET", key, "tokens", "last_refill")
local tokens = tonumber(bucket[1] or max_tokens)
local last_refill = tonumber(bucket[2] or 0)

local now = redis.call("TIME")[1]
local elapsed = math.max(0, now - last_refill)
tokens = math.min(max_tokens, tokens + elapsed * rate)

if tokens < cost then
    return 0  -- rate limited
end

tokens = tokens - cost
redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
redis.call("EXPIRE", key, math.ceil(max_tokens / rate) * 2)

return 1  -- allowed
```

**Pros:** Allows bursts up to bucket size, smooths traffic over time.  
**Cons:** More complex implementation.

### GCRA (Generic Cell Rate Algorithm)

Used by Redis module `redis-cell` and Kong API Gateway:

```lua
-- GCRA: rate = 2/second, burst = 5
-- Key formula: T = 1/rate (emission interval), tau = burst * T (max burst time)

local key = KEYS[1]
local rate = tonumber(ARGV[1])   -- requests per second
local burst = tonumber(ARGV[2])  -- max burst

local T = 1 / rate               -- time between emissions
local tau = burst * T            -- max burst duration

local now = redis.call("TIME")[1] * 1000000  -- microseconds
local tat = tonumber(redis.call("GET", key) or now)

local new_tat = math.max(now, tat) + T * 1000000

if new_tat - tau * 1000000 > now then
    return 0  -- rate limited, retry after = (new_tat - now) / 1000000
end

redis.call("SETEX", key, 2 * tau, new_tat)
return 1  -- allowed, remaining = (tau - (new_tat - now) / 1000000) / T
```

### Rate limiting comparison

| Algorithm | Burst handling | Memory cost | Accuracy |
|-----------|---------------|-------------|----------|
| Fixed window | Poor (boundary burst) | O(1) per user | Low |
| Sliding log | Good | O(N) per user | High |
| Sliding counter | Good | O(1) per user | Medium |
| Token bucket | Excellent (burst + throttle) | O(1) per user | High |
| GCRA | Excellent | O(1) per user | High |

**For your SaaS:** Token bucket or sliding counter is the best balance. Fixed window is fine for simple API rate limiting.

---

## 5. Performance tuning

### Slow log

```bash
# Configure
CONFIG SET slowlog-log-slower-than 10000  # log commands > 10ms (microseconds)
CONFIG SET slowlog-max-len 1000            # keep last 1000 slow commands

# View
SLOWLOG GET 10                             # last 10 slow commands
SLOWLOG LEN                                # number of slow commands
SLOWLOG RESET                              # clear slow log

# Output format:
# 1) 1) integer ID
#    2) timestamp
#    3) execution time (microseconds)
#    4) command + args
#    5) client IP:port
```

### Latency monitoring

```bash
# Built-in latency monitor
CONFIG SET latency-monitor-threshold 100   # monitor events > 100ms

LATENCY LATEST                             # latest latency events
LATENCY HISTORY command                    # history for a specific event
LATENCY GRAPH command                      # ASCII graph
LATENCY RESET                              # reset all data
LATENCY DOCTOR                             # latency analysis & recommendations
```

Common latency sources:
- **Fork**: `BGSAVE`/`BGREWRITEAOF` forks the process (can pause for seconds on large instances)
- **Command**: Slow commands (`KEYS *`, `SMEMBERS` on large set, `LRANGE` on large list)
- **Expiry**: Active expiry cycle (Redis checks 20 random keys, repeats if >25% expired)
- **Eviction**: Key eviction under memory pressure (especially `allkeys-lru`)
- **AOF fsync**: AOF with `appendfsync always` blocks on every write
- **Transparent Huge Pages**: THP causes latency spikes. Always disable: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
- **Swap**: If Redis swaps to disk (memory overcommit), latency spikes to seconds

### Memory management

```bash
# Eviction policies
CONFIG SET maxmemory-policy allkeys-lru    # DEFAULT: noeviction
```

| Policy | Description |
|--------|-------------|
| `noeviction` | Return errors on writes when memory limit reached (default) |
| `allkeys-lru` | Evict least recently used keys from all keys |
| `allkeys-lfu` | Evict least frequently used keys (Redis 4.0+) |
| `volatile-lru` | Evict LRU from keys with TTL set |
| `volatile-lfu` | Evict LFU from keys with TTL set |
| `allkeys-random` | Evict random keys |
| `volatile-random` | Evict random keys with TTL |
| `volatile-ttl` | Evict keys with shortest TTL |

**For your SaaS (cache use):** `allkeys-lru` is the best choice — Redis evicts the least recently used keys when memory fills.

```bash
# Monitor evictions
INFO stats
# evicted_keys: 15000 — keys evicted since startup
# If evicted_keys increases rapidly, maxmemory is too low

# Monitor memory fragmentation
INFO memory
# mem_fragmentation_ratio: 1.5 — > 1.5 indicates fragmentation
# Restart Redis to defragment, or use ACTIVE DEFRAG (Redis 4.0+)
CONFIG SET activedefrag yes
```

### Big keys analysis

```bash
# Scan for big keys (built-in)
redis-cli --bigkeys

# Output example:
# Biggest string found 'session:data' has 1048576 bytes
# Biggest list found 'job:queue' has 50000 items
# Biggest set found 'unique:v:2024-01' has 2000000 members
# Biggest hash found 'user:1000' has 1000 fields
# Biggest zset found 'leaderboard' has 500000 members

# Manual check
MEMORY USAGE user:1000
DEBUG OBJECT user:1000  # serialized length, lru, refcount
```

**Trap:** `redis-cli --bigkeys` uses `SCAN` and memory estimation — it's safe for production but gives approximate results.

### Command statistics

```bash
INFO commandstats
# cmdstat_get:calls=100000,usec=50000,usec_per_call=0.50
# cmdstat_set:calls=50000,usec=30000,usec_per_call=0.60
# cmdstat_zrangebyscore:calls=2000,usec=400000,usec_per_call=200.00
# -> ZRANGEBYSCORE is slow! (200us per call)
```

### Network optimization

```bash
# Disable TCP keepalive
CONFIG SET tcp-keepalive 300

# Set timeout for idle connections
CONFIG SET timeout 300                   # close idle connections after 300s

# Increase backlog for connection bursts
CONFIG SET tcp-backlog 511

# I/O threads (Redis 6+)
CONFIG SET io-threads 4                  # I/O handling threads
CONFIG SET io-threads-do-reads yes       # also use threads for reads
```

**Trap:** I/O threads only handle network I/O, not command execution. Setting `io-threads` above CPU core count degrades performance.

### Production configuration

```bash
# /etc/redis/redis.conf — production settings
# Memory
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 10                     # LRU accuracy vs CPU (default 5)

# Persistence
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# Networking
bind 0.0.0.0
port 6379
timeout 300
tcp-keepalive 300

# Security
requirepass your-strong-password
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command KEYS ""

# Performance
hz 10                                    # Redis cron frequency (default 10)
lfu-log-factor 10                        # LFU counter log factor (default 10)
lfu-decay-time 1                         # LFU counter decay per minute
activedefrag yes
jemalloc-bg-thread yes                   # background jemalloc threads

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 1000
```

---

## 6. Security

### ACL (Redis 6+)

Redis 6+ supports fine-grained ACL:

```bash
# Create user with specific permissions
ACL SETUSER app_user on >password123 ~cached:* +get +set +exists

# Read-only user
ACL SETUSER readonly on >password123 ~* +@read

# User for pub/sub only
ACL SETUSER pubsub on >password123 ~* +publish +subscribe +psubscribe

# Reset user
ACL DELUSER app_user

# List all users
ACL LIST

# Who am I?
ACL WHOAMI

# ACL log
ACL LOG
```

### ACL categories

```bash
+@all          # all commands
+@read         # read commands
+@write        # write commands
+@admin        # admin commands
+@dangerous    # dangerous commands (FLUSHALL, DEBUG, etc.)
+@fast         # O(1) commands
+@slow         # O(N) or O(log N) commands
+@connection   # connection commands
+@keyspace     # key-related commands
```

### Other security measures

```bash
# TLS configuration (Redis 6+)
tls-port 6380
tls-cert-file /etc/redis/redis.crt
tls-key-file /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt

# Rename dangerous commands
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_CMD"
rename-command DEBUG ""

# Bind to specific interface
bind 127.0.0.1 192.168.1.100

# Protected mode (default: yes — reject external connections if no password)
protected-mode yes
```

---

## 7. Redis as message queue

### List-based queue (simple)

```python
# Producer
redis.lpush("queue:email", json.dumps(email_data))

# Consumer (blocking)
while True:
    data = redis.brpop("queue:email", timeout=0)  # block forever
    email = json.loads(data[1])
    send_email(email)
```

**Pros:** Simple, fast.  
**Cons:** No ack (if consumer crashes after `brpop` but before processing → message lost), no fan-out, no re-queue.

### List-based reliable queue (with backup)

```python
# Producer
redis.lpush("queue:email", json.dumps(email_data))

# Consumer
while True:
    data = redis.brpoplpush("queue:email", "queue:email:processing", timeout=0)
    email = json.loads(data)
    try:
        send_email(email)
        redis.lrem("queue:email:processing", 1, data)
    except Exception:
        # After retries, move to dead letter
        redis.lpush("queue:email:dead", data)
        redis.lrem("queue:email:processing", 1, data)
```

### Stream-based queue (recommended)

```lua
-- Producer
XADD orders:stream * order_id 1001 user_id 42 amount 99.99

-- Consumer group
XGROUP CREATE orders:stream email-workers $ MKSTREAM

-- Consumer
XREADGROUP GROUP email-workers worker-1 COUNT 1 BLOCK 2000 STREAMS orders:stream >
```

```python
class StreamConsumer:
    def __init__(self, stream, group, consumer, redis):
        self.stream = stream
        self.group = group
        self.consumer = consumer
        self.redis = redis

    def process_messages(self, handler):
        while True:
            messages = self.redis.xreadgroup(
                self.group, self.consumer,
                {self.stream: '>'},
                count=10,
                block=2000
            )

            if not messages:
                continue

            for stream_name, entries in messages:
                for msg_id, data in entries:
                    try:
                        handler(data)
                        self.redis.xack(self.stream, self.group, msg_id)
                    except Exception as e:
                        # Move to dead letter after 3 retries
                        pending = self.redis.xpending(self.stream, self.group)
                        # ... retry logic
```

### Comparison: List vs Pub/Sub vs Stream

| Feature | List | Pub/Sub | Stream |
|---------|------|---------|--------|
| Persistence | Yes (RDB/AOF) | No | Yes (RDB/AOF) |
| Fan-out | No (each msg once) | Yes (all subscribers) | Yes (consumer groups) |
| ACK | No | No | Yes (XACK) |
| Re-queue | RPOPLPUSH | No | XCLAIM |
| Pending list | No | No | XPENDING |
| Dead letter | Manual | No | Manual |
| Throughput | ~200K/s | ~1M/s | ~100K/s |
| Consumer groups | No | No | Yes |

**For your SaaS:** Use Streams for reliable job processing (email, notifications, async tasks). Use Lists for simple FIFO where message loss is acceptable.

---

## 8. Redis 7.x features

| Feature | Description |
|---------|-------------|
| **Redis Functions** | Replace Lua with functions that are managed, persisted, and replicated — better than `SCRIPT LOAD` |
| **ACL V2** | Improved ACL with key-based permissions, selectors, and more granular control |
| **Sharded Pub/Sub** | Pub/Sub within Redis Cluster (messages only sent to nodes with subscribers) |
| **Auto failover** | Improved replica failover in Cluster |
| **Listpack for Hashes** | More memory-efficient than ziplist for small hashes |
| **CLUSTER SHARDS** | Simpler cluster topology discovery for clients |
| **RESP3 support** | New Redis serialization protocol (push-based, types) |

### Redis Functions example

```lua
# Register a function (managed, persists across restarts)
FUNCTION LOAD "#!lua name=mylib\n
local function my_hset(keys, args)\n
    redis.call('HSET', keys[1], args[1], args[2])\n
    return redis.call('HGETALL', keys[1])\n
end\n
redis.register_function('my_hset', my_hset)"

# Call function
FCALL mylib my_hset 1 user:1000 name "Alice"
```

---

## 9. Redis alternatives (Valkey, KeyDB, Dragonfly)

### Valkey

Valkey is the Linux Foundation fork of Redis (after Redis changed to SSPL license in 2024). It is fully compatible with Redis (forked from Redis 7.2.4).

**Key differences:**
- License: BSD-3 (vs Redis' SSPL)
- Maintained by Linux Foundation
- Same API — drop-in replacement
- No new Redis features after the fork (separate roadmap)

### KeyDB

KeyDB is a multi-threaded Redis-compatible server:

| Feature | Redis | KeyDB |
|---------|-------|-------|
| Threading | Single-threaded (I/O threads in 6+) | Multi-threaded (full command parallelism) |
| Throughput | ~200K ops/sec | ~2M ops/sec (on multi-core) |
| Compatibility | Reference | Redis-compatible API |
| Active Replication | No | Yes (multi-master) |
| Memory | Same | Same (forked from Redis) |

**For your use case:** Redis is sufficient. KeyDB is useful when you need > 200K ops/sec on a single node (but Redis Cluster or more replicas can also achieve this).

### Dragonfly

Dragonfly is a multi-threaded, high-throughput in-memory data store compatible with Redis/Memcached:

| Feature | Redis | Dragonfly |
|---------|-------|-----------|
| Threading | Single | Multi (shared-nothing architecture) |
| Max throughput | ~3M QPS (10-node cluster) | ~4M QPS (single node) |
| Memory efficiency | Good | Better (snapshot compression) |
| Compatibility | Reference | Redis API (95%+ compatible) |
| Maturity | 15+ years | Relatively new |

---

## 10. Production operations

### Backup and restore

```bash
# Trigger RDB snapshot
redis-cli BGSAVE

# Copy RDB to backup location
cp /var/lib/redis/dump.rdb /backups/redis/dump-$(date +%Y%m%d).rdb

# AOF backup (copy AOF file)
cp /var/lib/redis/appendonly.aof /backups/redis/

# Restore from RDB
# 1. Stop Redis
# 2. Copy dump.rdb to Redis data directory
# 3. Start Redis

# Restore specific key from backup
# redis-rdb-tools (rdb --command json dump.rdb) → extract → import
```

### Monitoring checklist

```bash
# Every 5 minutes, check:
# 1. Redis is reachable
redis-cli PING

# 2. Memory usage
redis-cli INFO memory | grep -E 'used_memory_human|maxmemory|mem_fragmentation_ratio'

# 3. Replication health
redis-cli INFO replication | grep -E 'role|connected_slaves|master_link_status'

# 4. Slow commands
redis-cli SLOWLOG GET 3

# 5. Connected clients
redis-cli INFO clients

# 6. Key count
redis-cli DBSIZE
redis-cli INFO keyspace

# 7. Hit rate (cache hit ratio = keyspace_hits / (keyspace_hits + keyspace_misses))
redis-cli INFO stats | grep -E 'keyspace_(hits|misses)'
```

### Upgrade strategy

```bash
# Zero-downtime upgrade with replicas:
# 1. Upgrade a replica node
# 2. Failover to upgraded replica (Sentinel or CLUSTER FAILOVER)
# 3. Upgrade old master (now replica)
# 4. Repeat for remaining nodes

# Sentinel:
redis-sentinel /etc/redis/sentinel.conf  # upgraded first
# Then upgrade replicas one by one
```

### Troubleshooting checklist

| Symptom | Likely cause | Check |
|---------|-------------|-------|
| Latency spikes | Fork (BGSAVE/BGREWRITEAOF), THP, swap | `LATENCY DOCTOR`, /proc/meminfo |
| High CPU | High throughput, expensive commands | `INFO commandstats`, `SLOWLOG` |
| Memory full | Evictions, fragmentation | `INFO stats` (evicted_keys), `INFO memory` |
| Connection refused | Max clients reached | `INFO clients`, `CONFIG GET maxclients` |
| Replication lag | Slow replica, network, backlog too small | `INFO replication`, `CONFIG GET repl-backlog-size` |
| AOF corruption | Hardware issue, power loss | `redis-check-aof --fix appendonly.aof` |
| Key loss | Eviction policy, expired keys | `INFO stats` (expired_keys, evicted_keys) |

---

## 11. Caching for the 88% query reduction story

Your 88% query reduction likely involved Redis caching. Here's how to articulate the strategy:

### Problem

Database queries for the dashboard were slow (multiple JOINs, aggregations). Each page load triggered complex queries.

### Solution

1. **Identify hot queries**: `EXPLAIN ANALYZE` on slowest queries → dashboard data
2. **Cache-aside with 5-minute TTL**: 
   ```php
   $dashboard = Cache::remember("org:{$orgId}:dashboard", 300, function () {
       return DashboardService::calculate($orgId);
   });
   ```
3. **Hierarchical caching**: Cache parts independently (order count, revenue, user count) and compose on read
4. **Invalidation on write**: When a new order arrives, delete the dashboard cache key
   ```php
   // OrderObserver::created()
   Cache::forget("org:{$org->org_id}:dashboard");
   ```
5. **Rate limiting on expensive endpoints**: Token bucket rate limiter for dashboard API

### Result

88% fewer database queries (from ~50K queries/day to ~6K queries/day) — cache hit rate of ~95%.

---

## Interview traps cheatsheet — Senior

| Trap | The truth |
|------|-----------|
| "Redlock is the gold standard for distributed locks" | Martin Kleppmann's analysis shows theoretical flaws. Single Redis + Lua unlock is safer for most use cases. |
| "Redis Cluster means I never lose data" | Cluster uses async replication. Writes acknowledged by master may not be replicated before crash. |
| "Cache-aside is always the right pattern" | Write-through is better for consistency-critical data. Refresh-ahead prevents stampedes. |
| "Fixed window rate limiting is fine" | Burst at window boundaries can allow 2x traffic in 1-second span. Use sliding window or token bucket. |
| "Streams can replace Kafka" | Redis Streams are limited to ~100K msg/sec per node. Kafka handles millions per partition. |
| "allkeys-lru means I don't need to worry about memory" | If working set exceeds maxmemory, frequently accessed data gets evicted too. Monitor evicted_keys. |
| "Redis Functions replace all Lua scripts" | Functions are managed/persisted but have same sandbox limitations. Use EVALSHA for simple scripts. |
| "CONFIG SET is safe in production" | Changing `maxmemory`, `appendfsync`, or save intervals can cause unexpected behavior. Restart to apply. |
| "I should use SELECT database for multi-tenant isolation" | SELECT (db 0-15) is not recommended. Use separate Redis instances per tenant or key prefix (org_id:...) |
| "redis-cli --bigkeys is accurate" | Estimates based on sampling. Use `MEMORY USAGE` for accurate per-key measurement. |
</details>
