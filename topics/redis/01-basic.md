# Redis — Basic

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Redis core concepts, data structures, commands, TTL, persistence overview  
> **Trap:** Most candidates know `SET`/`GET` but can't explain *when* to use each data structure. At senior level, you must articulate the trade-offs.

---

## 1. Redis core concepts

### What is Redis?

Redis is an **in-memory data structure store** used as database, cache, message broker, and queue engine. Key characteristics:

- **In-memory**: Primary storage is RAM (persistence is optional)
- **Key-value**: Data stored by key in a flat keyspace
- **Single-threaded**: All commands are processed sequentially on the main thread (since Redis 6, some I/O threads)
- **Data structures**: Not just strings — Lists, Sets, Sorted Sets, Hashes, Bitmaps, HyperLogLogs, Geospatial, Streams
- **Atomic operations**: Commands execute atomically (no race conditions within a single command)
- **Sub-millisecond latency**: Due to in-memory design

### Single-threaded event loop

Redis uses an event loop with IO multiplexing (epoll/kqueue/select):
- One thread processes all commands
- No locks needed for data access
- Blocking operations (e.g., `KEYS`, `SORT`, `LRANGE` on large lists) block the entire server
- **Trap:** `KEYS *` blocks Redis for the duration. Use `SCAN` for production.

```
Client 1: SET key value  ─┐
Client 2: GET other_key  ─┼──→ Redis Event Loop → Process sequentially
Client 3: INGR counter   ─┘
```

### Keys and naming conventions

```bash
# Keys are binary-safe strings (up to 512 MB)
SET user:1000:name "Alice"
SET user:1000:email "alice@example.com"

# Convention: object:type:id:field
SET org:42:settings:theme "dark"
SET org:42:user:1000:role "admin"
```

**Best practices:**
- Use colons for namespacing (`org:42:user:1000`)
- Keys should be meaningful but not excessive length
- Shorter keys save memory (but not at readability's expense)
- For your multi-tenant SaaS: `{tenant_id}:{entity}:{id}` pattern

### Data type overview

| Type | Description | Max Size | Use Case |
|------|-------------|----------|----------|
| String | Binary-safe string (text, JSON, serialized objects, numbers) | 512 MB | Cache values, counters, session tokens |
| List | Linked list of strings | 2^32 - 1 elements | Queue, message buffer, timeline |
| Set | Unordered set of unique strings | 2^32 - 1 members | Tags, unique visitors, social graph |
| Sorted Set | Set with score for ordering | 2^32 - 1 members | Leaderboards, rate limiting, priority queue |
| Hash | Key-value map within a key | 2^32 - 1 fields | Object representation, user profiles |
| Bitmap | Bit-level operations on a string | 512 MB | Feature flags, analytics, bloom filter |
| HyperLogLog | Probabilistic data structure | ~12 KB per key | Unique count (UV analytics) |
| GeoSpatial | Geohash-based location | 2^32 - 1 elements | Nearby places, geofencing |
| Stream | Append-only log with consumer groups | Unlimited | Event sourcing, message queue |

---

## 2. String operations (most common)

```bash
# Set and Get
SET name "Alice"
GET name                # "Alice"

# Set with options
SET name "Alice" EX 3600 NX    # expire in 1 hour, only if not exists
SET login:token "abc123" PX 5000  # expire in 5000 ms

# Get old value and set new
GETSET name "Bob"       # returns "Alice", sets to "Bob"

# Multi-get/set
MSET key1 "val1" key2 "val2"
MGET key1 key2          # ["val1", "val2"]

# Counter operations
SET counter 100
INCR counter             # 101
INCRBY counter 50        # 151
DECR counter             # 150
DECRBY counter 20        # 130
GET counter              # "130" (still string, interpreted as number)

# Append
APPEND name " Smith"     # "Bob Smith" (length returned)

# Get substring
GETRANGE name 0 2        # "Bob"

# Set substring
SETRANGE name 4 "y"      # "Boby Smith"

# String length
STRLEN name              # 10

# Get/set multiple fields (bit operations)
SETBIT flags 0 1         # set bit 0 to 1
GETBIT flags 0           # 1
BITCOUNT flags           # count of 1 bits

# SET with TTL in one command
SETEX session:123 3600 "data"    # set with TTL (seconds)
PSETEX session:123 5000 "data"   # set with TTL (milliseconds)
```

**Trap:** `SET` with `NX` is commonly used for distributed locks, but it's naive without TTL handling. If the lock holder crashes before releasing, the lock persists forever. Always set a TTL:

```bash
SET lock:resource_id "worker_1" EX 30 NX  # atomic lock + TTL
```

---

## 3. List operations

Lists are **linked lists** — fast insert/delete at ends, slow random access (O(N) for index access).

```bash
# Push to left (head) or right (tail)
LPUSH queue "job3"       # ["job3"]
LPUSH queue "job2"       # ["job2", "job3"]
RPUSH queue "job4"       # ["job2", "job3", "job4"]

# Pop from left or right
LPOP queue               # "job2" (FIFO)
RPOP queue               # "job4" (LIFO)

# Blocking pop — wait for element, with timeout
BRPOP queue 5            # Block for 5 seconds waiting for element (0 = forever)

# Peek at elements
LRANGE queue 0 -1        # all elements
LRANGE queue 0 4         # first 5 elements
LINDEX queue 0           # first element without removing

# Length
LLEN queue               # 1

# Trim — keep only a range
LTRIM queue 0 99         # keep first 100 elements

# Insert before/after a pivot
LINSERT queue BEFORE "job3" "job_new"

# Remove elements
LREM queue 2 "job3"      # remove 2 occurrences of "job3"
```

**Use cases:**
- FIFO queue: `RPUSH` + `LPOP` (or `BLPOP` for blocking)
- LIFO stack: `LPUSH` + `LPOP`
- Timeline: `LPUSH` + `LTRIM` (keep latest N items)

**Trap:** Lists are linked lists — `LINDEX` in the middle is O(N). Don't use lists as arrays with random access. Use sorted sets or hashes instead.

---

## 4. Set operations

Sets are **unordered collections of unique strings** (implemented as hash tables).

```bash
# Add/remove members
SADD tags "redis" "database" "cache"
SREM tags "cache"
SPOP tags                 # remove and return random member

# Check membership
SISMEMBER tags "redis"    # 1 (true) or 0 (false)

# All members
SMEMBERS tags              # ["redis", "database"] (unsorted)

# Random members
SRANDMEMBER tags 2         # 2 random members without removing

# Cardinality
SCARD tags                 # 2

# Move member between sets
SMOVE source dest "member"

# Set operations
SADD set1 "a" "b" "c"
SADD set2 "c" "d" "e"

SINTER set1 set2           # ["c"] — intersection
SUNION set1 set2           # ["a", "b", "c", "d", "e"] — union
SDIFF set1 set2            # ["a", "b"] — difference (in set1 not in set2)

# Store results
SINTERSTORE dest set1 set2  # store intersection in dest
SUNIONSTORE dest set1 set2
SDIFFSTORE dest set1 set2
```

**Use cases:**
- Tags: `SADD article:42:tags "redis" "database"`
- Unique visitors: `SADD visitors:2024-01-15 user:1000`
- Social graph: followers, following (union for mutual)
- Random sampling: `SRANDMEMBER` for A/B testing

---

## 5. Sorted Set operations

Sorted Sets are like Sets but each member has a **score** (double). Elements are ordered by score (then lexicographically on tie).

```bash
# Add members with scores
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2" 50 "player3"

# Get rank
ZRANK leaderboard "player2"    # 2 (0-indexed, lowest to highest)
ZREVRANK leaderboard "player2" # 0 (highest to lowest)

# Get by rank range
ZRANGE leaderboard 0 2         # ["player3", "player1", "player2"] (lowest 3)
ZREVRANGE leaderboard 0 2      # ["player2", "player1", "player3"] (highest 3)

# Get by score range
ZRANGEBYSCORE leaderboard 50 150  # players between score 50-150
ZRANGEBYSCORE leaderboard -inf +inf WITHSCORES  # all with scores

# Get score
ZSCORE leaderboard "player1"   # 100

# Increment score
ZINCRBY leaderboard 50 "player1"  # 150

# Count members by score
ZCOUNT leaderboard 0 100

# Cardinality
ZCARD leaderboard               # 3

# Remove
ZREM leaderboard "player3"
ZREMRANGEBYRANK leaderboard 0 0  # remove lowest
ZREMRANGEBYSCORE leaderboard -inf 50  # remove scores <= 50

# Lexicographic operations (same score)
ZADD dict 0 "apple" 0 "banana" 0 "cherry"
ZRANGEBYLEX dict "[apple" "(cherry"  # ["apple", "banana"] — range by lex
ZLEXCOUNT dict "-" "+"             # 3
ZREMRANGEBYLEX dict "[b" "[c"      # remove lex range
```

**Use cases:**
- Leaderboard: `ZADD` with score, `ZREVRANGE` for top N
- Rate limiting: sliding window with `ZREMRANGEBYSCORE` + `ZADD` + `ZCARD`
- Priority queue: timestamp as score for ordering by time
- Autocomplete: lexicographic range with same score
- Delayed job queue: score = next run timestamp, `ZRANGEBYSCORE` for due jobs

---

## 6. Hash operations

Hashes are **field-value pairs** within a key — like a row in Pandas or a mini-document.

```bash
# Set/Get field
HSET user:1000 name "Alice" email "alice@example.com" age 30
HGET user:1000 name           # "Alice"
HGETALL user:1000             # all fields and values

# Multiple fields
HMGET user:1000 name email    # ["Alice", "alice@example.com"]
HMSET user:1000 role "admin" active 1  # (HMSET deprecated, use HSET)

# Check field existence
HEXISTS user:1000 phone       # 0 (false)

# Delete field
HDEL user:1000 age

# Increment field
HINCRBY user:1000 loginCount 1

# All fields, all values, length
HKEYS user:1000               # ["name", "email", ...]
HVALS user:1000               # ["Alice", "alice@example.com", ...]
HLEN user:1000                # number of fields

# Get value length
HSTRLEN user:1000 name        # 5

# Set if not exists
HSETNX user:1000 phone "555-1234"  # only sets if field doesn't exist

# Scan fields
HSCAN user:1000 0 MATCH "name*"
```

**Use cases:**
- Object store: user profiles, product data, session data
- Counter groups: multiple counters per key
- Cache of DB records: `HSET org:123:settings theme "dark" api_rate 100`

**Trap:** When using Redis as a cache for DB records (your SaaS), use Hashes to store objects rather than serialized JSON in a String. This lets you read/update individual fields without deserializing the entire object.

```bash
# Instead of:
SET user:1000 '{"name":"Alice","email":"alice@example.com"}'

# Do:
HSET user:1000 name "Alice" email "alice@example.com"
# Now you can HGET user:1000 email without parsing JSON
```

---

## 7. Key management and TTL

```bash
# Set TTL on existing key
EXPIRE session:123 3600          # seconds
PEXPIRE session:123 5000         # milliseconds
EXPIREAT session:123 1700000000  # absolute Unix timestamp

# Get TTL
TTL key               # -1 = no expiry, -2 = key doesn't exist, else seconds remaining
PTTL key              # milliseconds

# Remove TTL (make persistent)
PERSIST key

# Set TTL on SET
SETEX key 3600 "value"          # SET + EXPIRE atomic
PSETEX key 5000 "value"         # milliseconds

# Check if key exists
EXISTS key

# Delete key
DEL key1 key2 key3

# Rename key
RENAME oldkey newkey
RENAMENX oldkey newkey  # only if newkey doesn't exist

# Get key type
TYPE key               # string, list, set, zset, hash, stream, none

# Object info
OBJECT ENCODING key    # Internal encoding (embstr, int, raw, ziplist, linkedlist, etc.)
OBJECT IDLETIME key    # Seconds since last access
OBJECT REFCOUNT key
```

**Trap:** `RENAME` replaces the destination key atomically. If `newkey` exists, it is deleted. This can cause data loss.

**Trap:** `EXPIRE` on a key that already has a TTL replaces it. `EXPIRE` with 0 removes the key immediately (same as `DEL`).

### Key expiry behavior

- Redis uses **passive** and **active** expiry:
  - **Passive**: When a key is accessed, check if expired → delete
  - **Active**: Every 100ms, Redis tests 20 random keys with expiry → deletes expired ones. If >25% expired, repeat.
- This means expired keys may persist briefly between active expiry cycles
- **Trap:** Don't rely on exactly-timed key expiry for critical logic

---

## 8. Persistence overview

### RDB (Redis Database file) — snapshots

```bash
# Trigger manually
SAVE                              # synchronous — blocks Redis
BGSAVE                            # background fork — does NOT block
LASTSAVE                          # last successful save timestamp

# Configuration (redis.conf)
save 900 1                        # save if at least 1 key changed in 900s
save 300 10                       # save if at least 10 keys changed in 300s
save 60 10000                     # save if at least 10000 keys changed in 60s
dbfilename dump.rdb
dir /var/lib/redis/
```

**RDB pros:** Compact, fast recovery, good for backups.  
**RDB cons:** Data loss between snapshots possible (configurable), `BGSAVE` forks (memory overhead on write-heavy workload).

### AOF (Append Only File) — write-ahead log

```bash
# Configuration
appendonly yes
appendfsync always                # every write — safest, slowest
appendfsync everysec              # every second — good balance (default)
appendfsync no                    # let OS flush — fastest, least safe

# AOF rewrite (compacts the log)
BGREWRITEAOF
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**AOF pros:** Minimal data loss (1 second with `everysec`), append-only format.  
**AOF cons:** Larger than RDB, slower recovery.

### Hybrid persistence (Redis 4.0+)

```bash
aof-use-rdb-preamble yes
```
AOF file starts with RDB format (fast load), then appends AOF commands. Best of both.

### Which to choose?

| Use case | Persistence |
|----------|-------------|
| Cache only (can lose data) | None |
| Valuable data, minimal loss | AOF (everysec) + RDB snapshots |
| Fast restart, disaster recovery | RDB |
| Best compromise | Hybrid (RDB + AOF) |

**Trap:** If you use Redis as a primary database (not just cache), you must configure persistence carefully. Even with AOF `everysec`, you can lose up to 1 second of data on crash. For your use case (cache + sessions), less persistence is fine.

---

## 9. Pub/Sub basics

```bash
# Subscriber (usually in a separate connection/client)
SUBSCRIBE channel:alerts          # subscribe to one channel
SUBSCRIBE channel:alerts channel:logs  # subscribe to multiple
PSUBSCRIBE channel:*              # pattern subscribe (glob style)

# Publisher
PUBLISH channel:alerts "Server CPU > 90%"

# Unsubscribe
UNSUBSCRIBE channel:alerts
PUNSUBSCRIBE channel:*

# List active channels
PUBSUB CHANNELS                   # list active channels
PUBSUB CHANNELS channel:*         # list matching pattern
PUBSUB NUMSUB channel:alerts       # subscriber count for channel
PUBSUB NUMPAT                      # pattern subscription count
```

**Trap:** Pub/Sub is **fire-and-forget**. Messages are NOT persisted. If a subscriber disconnects between publish and subscribe, messages are lost. Use **Streams** for persistent messaging.

**Trap:** Pub/Sub subscribers block the connection while listening. Each subscriber needs its own connection.

---

## 10. Common Redis commands

### Server management

```bash
# Info
INFO                              # server info (sections: server, clients, memory, persistence, stats, replication, cpu, cluster, keyspace)
INFO memory                       # memory-specific stats
INFO stats                        # command stats

# Monitor (dangerous — use only in dev)
MONITOR                           # streams all commands (can crash production under load)

# Ping
PING                              # PONG

# Echo
ECHO "hello"

# Select database
SELECT 0                          # Redis has 16 databases by default (0-15)

# Flush
FLUSHDB                           # delete all keys in current DB
FLUSHALL                          # delete all keys in all DBs

# Client management
CLIENT LIST                       # all connected clients
CLIENT SETNAME my-app-client
CLIENT GETNAME
CLIENT KILL addr:port

# Config
CONFIG GET *                      # all config values
CONFIG SET maxmemory 1gb
CONFIG REWRITE                    # write config to redis.conf

# Debug
DEBUG SET-ACTIVE-EXPIRE 0
```

**Trap:** `FLUSHDB`/`FLUSHALL` are synchronous and block Redis until complete. On a large database, this can take seconds. Use `FLUSHALL ASYNC` (Redis 4.0+) for asynchronous flush.

### Scanning (never use KEYS in production)

```bash
# SCAN — cursor-based iteration
SCAN 0 MATCH user:* COUNT 100    # return matching keys in batches
SSCAN set_key 0 MATCH a*          # scan set members
HSCAN hash_key 0 MATCH name*      # scan hash fields
ZSCAN zset_key 0 MATCH player*    # scan sorted set members
```

**Trap:** `KEYS *` blocks Redis. Always use `SCAN` in production. `SCAN` can return duplicate keys under certain conditions (resizing during scan) — handle idempotently.

---

## 11. Client interaction

### Node.js (ioredis)

```js
const Redis = require('ioredis')
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'optional',
  db: 0,
  retryStrategy(times) {
    return Math.min(times * 50, 2000)
  }
})

async function main() {
  await redis.set('key', 'value', 'EX', 3600)
  const value = await redis.get('key')
  console.log(value)

  // Pipeline
  const pipeline = redis.pipeline()
  pipeline.set('key1', 'val1')
  pipeline.set('key2', 'val2')
  pipeline.incr('counter')
  const results = await pipeline.exec()
}
```

### Go (go-redis)

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "",
    DB:       0,
    PoolSize: 10,
})

ctx := context.Background()

err := rdb.Set(ctx, "key", "value", time.Hour).Err()
val, err := rdb.Get(ctx, "key").Result()

// Pipeline
pipe := rdb.Pipeline()
pipe.Set(ctx, "key1", "val1", 0)
pipe.Incr(ctx, "counter")
cmds, err := pipe.Exec(ctx)
```

### Laravel (your stack)

```php
// Facade
use Illuminate\Support\Facades\Redis;

Redis::set('key', 'value', 'EX', 3600);
$value = Redis::get('key');

// Pipeline
Redis::pipeline(function ($pipe) {
    $pipe->set('key1', 'val1');
    $pipe->incr('counter');
});

// Cache facade
Cache::put('key', 'value', 3600);
Cache::remember('expensive_query', 3600, function () {
    return DB::table('orders')->where('org_id', $orgId)->get();
});
```

---

## 12. Important gotchas for senior interviews

### 1. "Redis is single-threaded" — mostly

The main command processing is single-threaded (no locks needed). However:
- Redis 6+ has **I/O threads** for handling network I/O (still single-threaded for commands)
- `BGSAVE`, `BGREWRITEAOF`, replication fork create **child processes**
- `UNLINK` (Redis 4.0+) deletes keys in a background thread
- Blocking operations (`BLPOP`, `BRPOP`) block the entire event loop

### 2. "Redis is fast because it's in-memory" — not just that

The single-threaded event loop avoids locking overhead. IO multiplexing (epoll) handles tens of thousands of connections efficiently. Most commands are O(1) or O(log N).

### 3. Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Multiple (strings, lists, sets, sorted sets, hashes, streams, etc.) | Only strings |
| Persistence | RDB, AOF, hybrid | None |
| Replication | Master-replica, Sentinel, Cluster | None |
| Transactions | MULTI/EXEC + WATCH | None |
| Lua scripting | Yes | No |
| Pub/Sub | Yes | No |
| Memory management | Configurable eviction policies | LRU only |
| Multithreaded | I/O threads only (commands single-threaded) | Multithreaded |

### 4. "Redis can replace a message queue" — carefully

- Lists + `BRPOP`: Simple queue, no persistence, no ack
- Pub/Sub: Fire-and-forget, no persistence, no ack
- Streams: Persistent, consumer groups, ACK, pending list — closest to Kafka but less throughput

Choose the right tool for your use case.

---

## 13. Practical Drills

### Drill 1 — Data structure selection

For each use case, choose the right Redis data structure:

1. Cache of user profile (name, email, role, settings)
2. FIFO job queue with blocking read
3. Top 10 leaderboard
4. Unique page visitors counter
5. Tag system for blog posts (find posts by tag)
6. Rate limiter (max 100 requests per minute per user)
7. Autocomplete for search

<details>
<summary>Answers</summary>

1. **Hash** — store each field separately for partial reads/updates
2. **List** — `RPUSH` + `BLPOP`
3. **Sorted Set** — `ZINCRBY` score, `ZREVRANGE 0 9`
4. **Set** — `SADD` + `SCARD` (or **HyperLogLog** for approximate count)
5. **Set** — `SADD post:{id}:tags "redis"`, then `SINTER` by tag
6. **Sorted Set** — `ZREMRANGEBYSCORE` old entries, `ZADD` with timestamp, `ZCARD`
7. **Sorted Set** — all scores 0, `ZRANGEBYLEX` for lexicographic range
</details>

### Drill 2 — Cache design

Your SaaS dashboard loads slowly because every visit queries the database. Design a caching strategy.

<details>
<summary>Answer</summary>

**Strategy:** Cache-aside with TTL

```php
function getOrganizationDashboard($orgId) {
    $cacheKey = "org:{$orgId}:dashboard_data";

    // Check cache
    $data = Redis::get($cacheKey);
    if ($data) {
        return json_decode($data, true);
    }

    // Cache miss — query database
    $data = DB::table('orders')
        ->where('org_id', $orgId)
        ->select(DB::raw('COUNT(*) as total, SUM(total) as revenue'))
        ->first();

    // Set cache with TTL (5 minutes)
    Redis::setex($cacheKey, 300, json_encode($data));

    return $data;
}
```

**Improvements:**
- Use Hash instead of serialized JSON for field-level access
- Pre-warm cache for top tenants
- Use `Cache::remember()` in Laravel
- For real-time data, invalidate cache when new order arrives
</details>

### Drill 3 — Connection management

```php
// Problem: Redis connection pooling in Laravel
// Each request creates a new connection → connection overhead + limits

// Solution: Use persistent connections
// config/database.php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'), // phpredis or predis
    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_DB', 0),
        'persistent' => true,  // Keep connections open
        'read_write_timeout' => 60,
    ],
],
```

**Trap:** Persistent connections can cause issues with some Redis configurations (maxclients, timeouts). Monitor connection count with `CLIENT LIST` / `INFO clients`.
</details>

---

## Interview traps cheatsheet

| Trap | The truth |
|------|-----------|
| "KEYS * is fine for development" | `KEYS *` blocks Redis — use `SCAN` even in dev if there are many keys |
| "Redis is just a cache" | Redis is a data structure server — used for queues, real-time data, leaderboards, sessions, distributed locks |
| "SET with NX is a safe distributed lock" | No TTL handling → deadlock if holder crashes. Always set TTL with NX |
| "Lists are great for random access" | Lists are linked lists — `LINDEX` is O(N). Use sorted sets or Scala/Go arrays |
| "Pub/Sub is like a message queue" | No persistence, no ACK, no consumer groups. Use Streams |
| "Redis is single-threaded = slow" | Single-threaded + in-memory + epoll = sub-millisecond. But avoid O(N) operations on large keys |
| "Always set a TTL" | True for cache. False for permanent data (queues, leaderboards, counters). Know when to use which |
| "Redis can replace the database" | Redis persists, but not as reliably. Data loss window exists. Use Redis as cache + PostgreSQL as source of truth |
| "SELECT database number" | Multiple databases (0-15) exist but using distinct Redis instances for isolation is cleaner |
| "Redis connections are cheap" | Each connection uses ~10-15 KB. Thousands of connections = GB of memory. Use connection pooling |
</details>
