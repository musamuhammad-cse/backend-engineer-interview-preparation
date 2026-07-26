# Redis — Question Bank

> **Target:** Senior Backend Engineer interview preparation  
> **Format:** Rapid-fire Q&A, code puzzles, debugging scenarios, system design prompts, STAR templates  
> **Real-world anchors:** Multi-tenant SaaS (caching, sessions, rate limiting), trading platform (real-time leaderboard, stream processing), Chronos (distributed locks, leader election), 88% query reduction

---

## 1. Rapid-fire Q&A (150+ questions)

### Fundamentals (25 questions)

1. **Q:** What is Redis and what makes it fast?  
   **A:** In-memory data structure store. Fast due to: in-memory storage, single-threaded event loop (no lock overhead), non-blocking I/O (epoll/kqueue), O(1) commands.

2. **Q:** Is Redis single-threaded?  
   **A:** Command processing is single-threaded. I/O threads (Redis 6+) handle network. BGSAVE/BGREWRITEAOF fork child processes.

3. **Q:** What is the max key size?  
   **A:** 512 MB for both keys and values (String type).

4. **Q:** What data structures does Redis support?  
   **A:** String, List, Set, Sorted Set, Hash, Bitmap, HyperLogLog, GeoSpatial, Stream.

5. **Q:** What is the max number of keys?  
   **A:** 2^32 (about 4.2 billion) keys per instance.

6. **Q:** How do you set a key with TTL in one command?  
   **A:** `SET key value EX 3600` or `SETEX key 3600 value` or `PSETEX key 5000 value`.

7. **Q:** What is the difference between `KEYS` and `SCAN`?  
   **A:** `KEYS` blocks Redis (returns all at once). `SCAN` is cursor-based, returns results incrementally, safe for production.

8. **Q:** What is the default number of databases?  
   **A:** 16 (0-15). `SELECT n` to switch. Only database 0 in Cluster mode.

9. **Q:** What command removes all keys in all databases?  
   **A:** `FLUSHALL`. Use `FLUSHALL ASYNC` (Redis 4.0+) to avoid blocking.

10. **Q:** How does key expiry work?  
    **A:** Passive (checked on access) + active (every 100ms, test 20 random keys with TTL, repeat if >25% expired).

11. **Q:** What is `MEMORY USAGE`?  
    **A:** Returns approximate memory used by a key and its value in bytes.

12. **Q:** What is `OBJECT ENCODING`?  
    **A:** Shows internal encoding of a key (int, embstr, raw, ziplist, linkedlist, hashtable, intset, skiplist, quicklist).

13. **Q:** Can you rename a key?  
    **A:** `RENAME oldkey newkey`. If newkey exists, it's replaced.

14. **Q:** What is the difference between `DEL` and `UNLINK`?  
    **A:** `DEL` is synchronous (blocks). `UNLINK` (Redis 4.0+) is async — key deletion happens in background thread.

15. **Q:** What does `TYPE key` return?  
    **A:** The data type: string, list, set, zset, hash, stream, none.

16. **Q:** What is the max length of a list?  
    **A:** 2^32 - 1 elements (about 4.2 billion).

17. **Q:** What is a sorted set and how is it ordered?  
    **A:** Set of unique members with double scores. Ordered by score ascending (then lexicographically on tie).

18. **Q:** What is `ZREVRANK` used for?  
    **A:** Returns rank from highest to lowest. Used for leaderboards (rank 0 = highest score).

19. **Q:** What does `BITCOUNT` do?  
    **A:** Counts bits set to 1 in a string value.

20. **Q:** What is HyperLogLog and what error rate does it have?  
    **A:** Probabilistic unique counter. ~12 KB per key. ~0.81% error rate.

21. **Q:** What is the difference between `PFADD` and `SADD`?  
    **A:** `PFADD` adds to HyperLogLog (probabilistic, no duplicate tracking). `SADD` adds to Set (exact, stores all values).

22. **Q:** What is `GEOADD`?  
    **A:** Adds geospatial coordinates (longitude, latitude, member) to a geospatial key.

23. **Q:** What does `GEODIST` return?  
    **A:** Distance between two geospatial members, optionally in unit (m, km, mi, ft).

24. **Q:** What is a Stream in Redis?  
    **A:** Append-only log of entries with IDs, supporting consumer groups, ACK, and blocking reads.

25. **Q:** What does `XACK` do?  
    **A:** Acknowledges a message in a consumer group — removes from pending entries list.

### Persistence (15 questions)

26. **Q:** What are the two persistence mechanisms?  
    **A:** RDB (snapshots) and AOF (append-only file).

27. **Q:** What is the difference between `SAVE` and `BGSAVE`?  
    **A:** `SAVE` blocks Redis. `BGSAVE` forks child process — non-blocking.

28. **Q:** What is `BGREWRITEAOF`?  
    **A:** Compacts the AOF file by removing redundant operations.

29. **Q:** What is the default AOF fsync policy?  
    **A:** `everysec` — fsync every second. Good balance of safety and performance.

30. **Q:** What is hybrid persistence?  
    **A:** `aof-use-rdb-preamble yes` — AOF starts with RDB format (fast load) then appends AOF commands.

31. **Q:** Can you lose data with AOF `everysec`?  
    **A:** Yes — up to 1 second of data on crash.

32. **Q:** What happens if AOF file is corrupted?  
    **A:** Redis exits on start. Fix with `redis-check-aof --fix appendonly.aof`.

33. **Q:** What is the default RDB save configuration?  
    **A:** `save 900 1`, `save 300 10`, `save 60 10000`.

34. **Q:** When would you use no persistence?  
    **A:** Cache-only use case where data loss is acceptable.

35. **Q:** How does Redis handle persistence during BGSAVE?  
    **A:** Forks a child process. Child writes RDB to disk. Parent continues serving requests.

36. **Q:** Does BGSAUSE use copy-on-write?  
    **A:** Yes — parent forks, child gets snapshot of memory. Parent pages are marked COW. Writes by parent create copies.

37. **Q:** What is the performance impact of BGSAVE?  
    **A:** Fork can pause Redis (depends on memory size). COW overhead if parent does many writes during BGSAVE.

38. **Q:** Which persistence is faster to restart?  
    **A:** RDB (single file, load into memory). AOF needs to replay all commands.

39. **Q:** Can you disable both RDB and AOF?  
    **A:** Yes — `save ""` and `appendonly no`. Data only in memory.

40. **Q:** Can you switch persistence modes without restart?  
    **A:** Yes — use `CONFIG SET`:
    ```bash
    CONFIG SET save "900 1 300 10"
    CONFIG SET appendonly yes
    ```

### Transactions and Lua (15 questions)

41. **Q:** How do Redis transactions work?  
    **A:** `MULTI` queues commands, `EXEC` executes atomically. No rollback. Commands execute in order.

42. **Q:** What is `WATCH` used for?  
    **A:** Optimistic locking — if watched key changes before `EXEC`, transaction aborts.

43. **Q:** What does `DISCARD` do?  
    **A:** Aborts a transaction, clears the command queue, unwatches keys.

44. **Q:** Can a Redis transaction be partially executed?  
    **A:** Yes — if a command fails at runtime, others still execute. No rollback.

45. **Q:** What is the difference between `MULTI`/`EXEC` and Lua scripting?  
    **A:** Lua: server-side logic (if/else, loops), returns values mid-script. MULTI/EXEC: just queued commands, can't read own writes.

46. **Q:** How do you load and execute a Lua script?  
    **A:** `SCRIPT LOAD "script"` returns SHA. `EVALSHA sha 2 key1 key2 arg1 arg2`. Or `EVAL "script" 2 key1 key2 arg1 arg2`.

47. **Q:** What is the difference between `redis.call()` and `redis.pcall()`?  
    **A:** `call()` raises error (halts script). `pcall()` returns error table (continues).

48. **Q:** Can Lua scripts access global variables?  
    **A:** No — strict mode. Must use `local`. Can't access file system or network.

49. **Q:** How are Lua scripts replicated?  
    **A:** Replicated as `EVAL` (not `EVALSHA`) to ensure replicas have the script. All writes replicated atomically.

50. **Q:** What happens if a Lua script takes too long?  
    **A:** Default max execution time: 5 seconds. After that, logs warning. After `lua-time-limit` (default 5000ms), other commands start getting BUSY errors.

51. **Q:** What is `SCRIPT KILL`?  
    **A:** Kills a running script (only if it hasn't executed any write commands).

52. **Q:** What is `SCRIPT FLUSH`?  
    **A:** Clears the Lua script cache.

53. **Q:** Can Lua scripts use the Redis TIME command?  
    **A:** Yes — `redis.call("TIME")` returns [seconds, microseconds].

54. **Q:** What is `redis.log()` in Lua?  
    **A:** Logs to Redis log file: `redis.log(redis.LOG_WARNING, "message")`.

55. **Q:** Can you use hash tags in Lua scripts for cluster?  
    **A:** Yes — but all keys must be in the same slot (or use tag). Redis 7.0+ allows cross-slot with hash tags.

### Pipelining (10 questions)

56. **Q:** What is pipelining?  
    **A:** Sending multiple commands without waiting for responses — reduces round-trip time.

57. **Q:** Is a pipeline a transaction?  
    **A:** No — commands from different clients can interleave. Use MULTI/EXEC for atomicity.

58. **Q:** When should you use pipelining?  
    **A:** Batch operations where atomicity isn't required — bulk inserts, cache warming.

59. **Q:** What memory considerations does pipelining have?  
    **A:** Server queues responses. Sending millions of commands in one pipeline exhausts memory.

60. **Q:** Can you pipeline with Lua scripts?  
    **A:** Send EVAL/EVALSHA in a pipeline, but Lua itself is single command execution.

61. **Q:** What's the typical speedup from pipelining?  
    **A:** 5-10x depending on network latency. RTT eliminated.

62. **Q:** Does pipelining change command semantics?  
    **A:** No — commands execute the same way, just batched.

63. **Q:** Can you pipeline MULTI/EXEC?  
    **A:** Yes — pipeline MULTI + commands + EXEC for atomic batch with reduced round trips.

64. **Q:** What happens if a pipeline command fails?  
    **A:** Each command returns success/error independently. One failure doesn't affect others.

65. **Q:** How do you handle pipeline results?  
    **A:** Results returned in order of commands. Each result corresponds to one command.

### Replication (15 questions)

66. **Q:** What is master-replica replication?  
    **A:** Replica connects to master, receives full RDB on initial sync, then continuous stream of writes.

67. **Q:** What is PSYNC2?  
    **A:** Partial resynchronization (Redis 4.0+). Replica can resume with replication ID + offset instead of full sync.

68. **Q:** What is the replication backlog?  
    **A:** Circular buffer on master (default 1 MB). Keeps recent write commands for partial resync.

69. **Q:** When does a full resync happen?  
    **A:** First connection, or when replica's offset falls outside the backlog window.

70. **Q:** Are replicas read-only?  
    **A:** By default yes (`replica-read-only yes`). Can be set to no, but not recommended.

71. **Q:** Can replicas have replicas?  
    **A:** Yes — replica-of-replica (cascading replication).

72. **Q:** How do you check replication lag?  
    **A:** `INFO replication` — `master_repl_offset - slave_repl_offset` or `lag` field.

73. **Q:** What is `min-replicas-to-write`?  
    **A:** Master rejects writes if fewer than N replicas are connected with lag < max-lag.

74. **Q:** What happens if master fails without replicas?  
    **A:** Data loss. Writes after last sync to any replica are lost.

75. **Q:** Is replication synchronous or asynchronous?  
    **A:** Asynchronous by default. `WAIT` command can wait for N replicas to acknowledge (but not true sync replication).

76. **Q:** What is `WAIT`?  
    **A:** `WAIT numreplicas timeout` — blocks until N replicas have acknowledged writes.

77. **Q:** Can you read from replicas?  
    **A:** Yes — but reads are eventually consistent (replication lag).

78. **Q:** What is the `replica-serve-stale-data` setting?  
    **A:** If yes (default), replica serves data during sync. If no, returns error until sync complete.

79. **Q:** How do you promote a replica to master?  
    **A:** `REPLICAOF NO ONE` on the replica.

80. **Q:** What is replica priority?  
    **A:** Lower priority = more likely to be promoted by Sentinel. Priority 0 = never promoted.

### Sentinel (10 questions)

81. **Q:** What is Redis Sentinel?  
    **A:** High availability system: monitoring, notification, automatic failover, configuration provider.

82. **Q:** What is the Sentinel quorum?  
    **A:** Number of Sentinels that must agree master is down before failover.

83. **Q:** What is the difference between SDOWN and ODOWN?  
    **A:** SDOWN (subjective) — one Sentinel thinks node is down. ODOWN (objective) — quorum of Sentinels agree.

84. **Q:** How does Sentinel choose which replica to promote?  
    **A:** Remove priority 0, disconnected, outdated replicas. Pick lowest priority, highest offset, lexicographically sorted run ID.

85. **Q:** What port does Sentinel use?  
    **A:** 26379 by default.

86. **Q:** Can Sentinel be used with Redis Cluster?  
    **A:** No — Cluster has its own failover mechanism.

87. **Q:** What is Sentinel split-brain?  
    **A:** Master isolated from Sentinels but still serves clients. Sentinel promotes new master. Both accept writes until partition heals.

88. **Q:** How do you mitigate Sentinel split-brain?  
    **A:** `min-replicas-to-write` — old master rejects writes when replicas disconnect.

89. **Q:** How do clients discover the current master via Sentinel?  
    **A:** `SENTINEL get-master-addr-by-name mymaster` returns [host, port].

90. **Q:** How many Sentinels do you recommend?  
    **A:** 3 (minimum for quorum of 2). 5 for higher fault tolerance.

### Cluster (15 questions)

91. **Q:** What is Redis Cluster?  
    **A:** Automatic sharding across multiple nodes with built-in failover. No separate Sentinel needed.

92. **Q:** How many hash slots are there?  
    **A:** 16384.

93. **Q:** How is a key mapped to a slot?  
    **A:** `CRC16(key) % 16384`.

94. **Q:** What is a hash tag?  
    **A:** `{...}` in key — CRC16 is computed on the tag only. Forces keys to same slot for multi-key ops.

95. **Q:** What is the difference between MOVED and ASK redirects?  
    **A:** MOVED — key permanently on another node (update routing table). ASK — slot migrating, retry on target node.

96. **Q:** What port does the cluster bus use?  
    **A:** Client port + 10000 (e.g., 6379 → 16379).

97. **Q:** How does Cluster handle failover?  
    **A:** Replicas detect master failure via gossip (cluster-node-timeout). Replica with highest offset promotes itself.

98. **Q:** What is replica migration?  
    **A:** Orphan replicas (no master to replicate) auto-migrate to under-replicated masters.

99. **Q:** What is the minimum Cluster size?  
    **A:** 3 masters (for quorum). Plus replicas for HA.

100. **Q:** What is `cluster-require-full-coverage`?  
     **A:** Default yes — reject writes if any slot is uncovered (node down without replica). Set to no for partial availability.

101. **Q:** Can you do multi-key operations in Cluster?  
     **A:** Only if keys are in the same slot (hash tag). Cross-slot MULTI/EXEC and Lua scripts are rejected.

102. **Q:** How do you reshard in Cluster?  
     **A:** `redis-cli --cluster reshard host:port --cluster-from node-id --cluster-to node-id --cluster-slots 1000`.

103. **Q:** What is the gossip protocol in Cluster?  
     **A:** Nodes exchange PING/PONG messages with information about other nodes. Cluster bus. ~100 nodes max recommended.

104. **Q:** How does Cluster differ from Sentinel?  
     **A:** Cluster = sharding + HA. Sentinel = HA only (no sharding).

105. **Q:** Can you add/remove nodes without downtime?  
     **A:** Yes — slots migrate via resharding. Client handles MOVED/ASK redirects.

### Caching (15 questions)

106. **Q:** What is cache-aside?  
     **A:** Read: check cache → miss → DB → set cache. Write: update DB → invalidate cache.

107. **Q:** What is write-through caching?  
     **A:** Write to DB + cache atomically. Slower writes, always consistent.

108. **Q:** What is write-behind (write-back) caching?  
     **A:** Write to cache first, async flush to DB. Very fast writes, risk of data loss.

109. **Q:** What is refresh-ahead caching?  
     **A:** Proactively refresh cache before TTL expires (probabilistic early recomputation).

110. **Q:** What is cache stampede?  
     **A:** Multiple requests simultaneously recompute the same expired cache key → DB overwhelmed.

111. **Q:** How do you prevent cache stampede?  
     **A:** Locking (SET NX), probabilistic early recomputation, stale-while-revalidate, exponential backoff.

112. **Q:** What is the best eviction policy for a cache?  
     **A:** `allkeys-lru` — evicts least recently used keys when memory is full.

113. **Q:** What is `allkeys-lfu`?  
     **A:** Evicts least frequently used keys (Redis 4.0+). Better for steady-state workloads with stable access patterns.

114. **Q:** What is `volatile-ttl` eviction?  
     **A:** Evicts keys with the shortest remaining TTL. Good when keys have meaningful TTLs.

115. **Q:** What is `noeviction`?  
     **A:** Default (if not set). Returns error on writes when maxmemory reached.

116. **Q:** How do you measure cache hit rate?  
     **A:** `keyspace_hits / (keyspace_hits + keyspace_misses)` from `INFO stats`.

117. **Q:** What is a good cache hit rate?  
     **A:** > 90% for well-designed caches. Your 88% query reduction implies ~95% hit rate.

118. **Q:** What is cache warming?  
     **A:** Pre-loading cache on startup/deploy to avoid cold-start misses.

119. **Q:** What is multi-tier caching?  
     **A:** L1 (local memory) + L2 (Redis). L1 is faster, L2 is shared.

120. **Q:** How do you invalidate an L1 cache across servers?  
     **A:** Redis Pub/Sub or client-side tracking (Redis 6+) sends invalidation messages.

### Distributed Locks (10 questions)

121. **Q:** How do you create a simple distributed lock?  
     **A:** `SET lock:resource owner_id NX EX 30`. Release with Lua: `if GET == owner then DEL`.

122. **Q:** What is the problem with `DEL` to release a lock?  
     **A:** Without owner check, you can delete another client's lock (especially if lock expired). Use Lua for safe release.

123. **Q:** What is Redlock?  
     **A:** Distributed lock algorithm using N independent Redis nodes (typically 5). Acquire on majority + within TTL.

124. **Q:** What are the criticisms of Redlock?  
     **A:** Clock drift, GC pauses, network delays can cause two clients to hold the lock simultaneously.

125. **Q:** What is a fencing token?  
     **A:** Monotonically increasing token that the protected resource checks. Prevents stale lock holders.

126. **Q:** What is the Redisson watchdog?  
     **A:** Automatically extends lock TTL while client is still working (default: extend every 10s by 30s).

127. **Q:** What happens if a lock holder crashes?  
     **A:** Lock auto-releases after TTL. Ensure TTL is long enough for the critical section.

128. **Q:** How long should a lock TTL be?  
     **A:** Short enough to auto-release quickly on crash, long enough for the critical section. Use watchdog for variable-length work.

129. **Q:** What is the difference between optimistic and pessimistic locking in Redis?  
     **A:** Optimistic: `WATCH` + `MULTI`/`EXEC` (retry on conflict). Pessimistic: `SET NX` lock (block/retry).

130. **Q:** For Chronos, would you use Redlock or simple lock?  
     **A:** Simple lock with Lua release is sufficient for scheduler leader election. Redlock is overkill unless multi-datacenter.

### Rate Limiting (10 questions)

131. **Q:** What is fixed window rate limiting?  
     **A:** Count requests in time window (e.g., 100 requests per minute). Simple but allows boundary bursts.

132. **Q:** What is sliding window log?  
     **A:** Track each request timestamp in a sorted set. Count in rolling window. Memory O(N).

133. **Q:** What is sliding window counter?  
     **A:** Two counters (current + previous window). Approximate sliding window. O(1) memory.

134. **Q:** What is token bucket rate limiting?  
     **A:** Bucket of N tokens. Each request consumes 1. Tokens refill at rate R per second. Allows bursts.

135. **Q:** What is GCRA?  
     **A:** Generic Cell Rate Algorithm. Used by `redis-cell` module. Precise rate limiting with burst support.

136. **Q:** How do you implement a rate limiter in Lua?  
     **A:** Sliding window sorted set: `ZREMRANGEBYSCORE` old entries, `ZCARD` to check limit, `ZADD` new entry.

137. **Q:** How does Redis handle atomicity in rate limiting?  
     **A:** Use `INCR` + `EXPIRE` (best-effort atomic) or Lua script (truly atomic).

138. **Q:** How do you rate limit by user + endpoint?  
     **A:** Key pattern: `ratelimit:{userId}:{endpoint}:{window}`. Different limits per endpoint.

139. **Q:** How do you handle rate limit headers for API responses?  
     **A:** Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` from Redis state.

140. **Q:** What rate limiting strategy is best for your SaaS?  
     **A:** Sliding window counter (memory efficient, reasonable accuracy) or token bucket (burst-friendly).

### Performance and Operations (10 questions)

141. **Q:** What is the SLOWLOG?  
     **A:** Logs commands exceeding `slowlog-log-slower-than` microseconds. `SLOWLOG GET 10` to view.

142. **Q:** What is LATENCY DOCTOR?  
     **A:** Analyzes latency sources and provides recommendations. Addresses fork, THP, swap, etc.

143. **Q:** What is mem_fragmentation_ratio?  
     **A:** `used_memory_rss / used_memory`. > 1.5 indicates fragmentation. Use `activedefrag yes`.

144. **Q:** What is the default maxmemory?  
     **A:** No limit by default (uses all available RAM). Must be configured for production.

145. **Q:** What happens when maxmemory is reached?  
     **A:** Depends on eviction policy. Default: `noeviction` — writes return error.

146. **Q:** How does THP affect Redis?  
     **A:** Transparent Huge Pages causes latency spikes during fork/COW. Disable: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`.

147. **Q:** What is `redis-cli --bigkeys`?  
     **A:** Scans for largest keys by type. Safe for production (uses SCAN). Approximate.

148. **Q:** What is jemalloc?  
     **A:** Redis's default memory allocator. Better for multi-threaded allocation, less fragmentation.

149. **Q:** How do you monitor Redis health?  
     **A:** `INFO` sections: memory, stats, replication, clients. Plus SLOWLOG, LATENCY, MONITOR (dev only).

150. **Q:** What is the `MONITOR` command?  
     **A:** Streams all commands to client. Only for debugging — can double CPU usage in production.

### Redis 7.x / Advanced (10 questions)

151. **Q:** What are Redis Functions?  
     **A:** Managed Lua functions that persist across restarts and are replicated. Better than `SCRIPT LOAD`.

152. **Q:** How do you register a Redis Function?  
     **A:** `FUNCTION LOAD "#!lua name=mylib\n function my_func(keys, args) ... end"`.

153. **Q:** What is sharded Pub/Sub?  
     **A:** Pub/Sub in Cluster — messages only sent to nodes with subscribers (no global broadcast).

154. **Q:** What is RESP3?  
     **A:** New Redis serialization protocol (6+). Supports push-based messages, typed responses.

155. **Q:** What is client-side caching?  
     **A:** Server tracks client cache, sends invalidation on key changes. CLIENT TRACKING ON.

156. **Q:** How does client-side caching work?  
     **A:** Client requests `CLIENT TRACKING ON`. Server remembers keys client accessed. On key change, sends INVALIDATE message.

157. **Q:** What is BCAST mode in client tracking?  
     **A:** Client specifies key prefixes. Server broadcasts invalidations for all matching keys. No per-client tracking.

158. **Q:** What is the difference between Valkey and Redis?  
     **A:** Valkey is the Linux Foundation fork (after Redis license change to SSPL). BSD-3 license. Started from Redis 7.2.4.

159. **Q:** What is KeyDB?  
     **A:** Multi-threaded Redis-compatible server. Higher throughput on multi-core CPUs. Redis API compatible.

160. **Q:** What is Dragonfly?  
     **A:** Multi-threaded in-memory store with Redis/Memcached compatibility. Up to 4M QPS on single node.

---

## 2. Code puzzles (10 puzzles)

### Puzzle 1 — Safe distributed lock

Write a Lua script for atomic lock acquisition and another for safe release.

<details>
<summary>Answer</summary>

```lua
-- Lock.lua
local key = KEYS[1]         -- lock:resource
local owner = ARGV[1]       -- unique owner ID (UUID + thread ID)
local ttl = tonumber(ARGV[2]) -- TTL in ms

return redis.call("SET", key, owner, "NX", "PX", ttl)

-- Unlock.lua
local key = KEYS[1]
local owner = ARGV[1]

if redis.call("GET", key) == owner then
    return redis.call("DEL", key)
end

return 0
```

Usage:
```bash
# Acquire
EVAL "<lock_script>" 1 lock:my_resource "worker-1:uuid-123" 30000

# Release
EVAL "<unlock_script>" 1 lock:my_resource "worker-1:uuid-123"
```
</details>

### Puzzle 2 — Sliding window rate limiter

Write a Lua script that implements a sliding window rate limiter using a sorted set. Limit 100 requests per 60 seconds per user.

<details>
<summary>Answer</summary>

```lua
local key = KEYS[1]            -- ratelimit:user:1000
local limit = tonumber(ARGV[1]) -- 100
local window = tonumber(ARGV[2]) -- 60 (seconds)

local now = redis.call("TIME")[1]
local window_start = now - window

-- Remove expired entries
redis.call("ZREMRANGEBYSCORE", key, 0, window_start)

local count = redis.call("ZCARD", key)

if count >= limit then
    -- Get TTL of oldest entry for retry-after header
    local oldest = redis.call("ZRANGE", key, 0, 0, "WITHSCORES")
    local retry_after = tonumber(oldest[2]) - window_start
    return {-1, retry_after}
end

-- Add current request
local member = now .. ":" .. math.random(1000000, 9999999)
redis.call("ZADD", key, now, member)
redis.call("EXPIRE", key, window)

return {1, limit - count - 1}  -- {allowed, remaining}
```
</details>

### Puzzle 3 — Cache-aside with stampede protection

Write a Lua script that implements cache-aside with a mutex lock to prevent cache stampede.

<details>
<summary>Answer</summary>

```lua
-- Attempt to get cached value; if missing, try to acquire recompute lock
local key = KEYS[1]               -- cache:expensive:org:42
local lock_key = KEYS[2]          -- lock:cache:expensive:org:42
local ttl = tonumber(ARGV[1])     -- 300 (seconds)
local lock_ttl = tonumber(ARGV[2]) -- 10 (seconds)
local owner = ARGV[3]             -- unique ID

-- Try cache first
local cached = redis.call("GET", key)
if cached then
    return {0, cached}  -- 0 = cache hit
end

-- Cache miss — try to acquire lock
local locked = redis.call("SET", lock_key, owner, "NX", "EX", lock_ttl)
if not locked then
    return {1, nil}  -- 1 = someone else is recomputing, wait
end

-- We got the lock — recompute needed
return {2, nil}  -- 2 = lock acquired, must recompute
```

```php
// Application code
$result = Redis::eval($script, 2, "cache:data", "lock:cache:data", 300, 10, $uuid);
if ($result[0] == 0) {
    return json_decode($result[1]); // cache hit
}
if ($result[0] == 1) {
    usleep(50000); // 50ms wait
    return getFromCacheOrCompute(); // retry
}
// $result[0] == 2: we hold the lock
$data = recomputeExpensiveData();
Redis::setex("cache:data", 300, json_encode($data));
Redis::eval($unlockScript, 1, "lock:cache:data", $uuid);
return $data;
```
</details>

### Puzzle 4 — Leaderboard update

A game leaderboard has 10M players. Each game completion updates a player's score. Show the leaderboard update script that handles concurrency.

<details>
<summary>Answer</summary>

```lua
local leaderboard = KEYS[1]    -- leaderboard:global
local player = ARGV[1]        -- player_12345
local score_to_add = tonumber(ARGV[2])  -- 100

-- Atomic score update
local new_score = redis.call("ZINCRBY", leaderboard, score_to_add, player)

-- Optional: set TTL for activity-based leaderboards
redis.call("EXPIRE", leaderboard, 86400)

-- Return new score and rank
local rank = redis.call("ZREVRANK", leaderboard, player)

return {new_score, rank + 1}  -- rank is 0-indexed, +1 for display
```

For your trading platform's competition leaderboard — same approach with `ZINCRBY` for atomic score updates.
</details>

### Puzzle 5 — Redis Stream consumer group with retry

Write a consumer that reads from a Stream, processes messages, and moves failed messages to a dead-letter queue after 3 retries.

<details>
<summary>Answer</summary>

```lua
-- Process a batch from the stream
local stream = KEYS[1]
local group = KEYS[2]
local consumer = KEYS[3]
local dead_letter_stream = KEYS[4]
local count = tonumber(ARGV[1])
local block = tonumber(ARGV[2])  -- ms

-- Read messages
local messages = redis.call("XREADGROUP", "GROUP", group, consumer,
    "COUNT", count, "BLOCK", block, "STREAMS", stream, ">")

if not messages or #messages == 0 then
    return {}
end

local results = {}
for _, entry in ipairs(messages[1][2]) do
    local msg_id = entry[1]
    local data = entry[2]

    -- Check retry count from pending
    local pending_info = redis.call("XPENDING", stream, group, "-", "+", 10, consumer)
    -- ... (simplified — in practice, track retries in message data)

    -- Acknowledge (after app processes)
    -- redis.call("XACK", stream, group, msg_id)

    table.insert(results, {msg_id, data})
end

return results
```

**Application-level retry pattern:**
```python
def process_messages():
    while True:
        messages = redis.xreadgroup('mygroup', 'worker-1',
                                    {'orders:stream': '>'}, count=10, block=2000)
        for stream, entries in messages:
            for msg_id, data in entries:
                retries = int(data.get(b'_retries', b'0'))
                try:
                    process(data)
                    redis.xack('orders:stream', 'mygroup', msg_id)
                except Exception:
                    if retries >= 2:
                        redis.xadd('orders:dead', data)
                        redis.xack('orders:stream', 'mygroup', msg_id)
                    else:
                        data[b'_retries'] = str(retries + 1).encode()
                        redis.xadd('orders:stream', data)
                        redis.xack('orders:stream', 'mygroup', msg_id)
```
</details>

### Puzzle 6 — Distributed rate limiter with burst

Design a token bucket rate limiter in Lua.

<details>
<summary>Answer</summary>

```lua
local key = KEYS[1]            -- token_bucket:user:1000
local max_tokens = tonumber(ARGV[1])  -- 100
local refill_rate = tonumber(ARGV[2]) -- 10 tokens/sec
local cost = tonumber(ARGV[3])        -- 1 per request

local bucket = redis.call("HMGET", key, "tokens", "last_refill")
local tokens = tonumber(bucket[1] or max_tokens)
local last_refill = tonumber(bucket[2] or 0)

local now = redis.call("TIME")[1]
local elapsed = math.max(0, now - last_refill)
tokens = math.min(max_tokens, tokens + elapsed * refill_rate)

if tokens < cost then
    local retry_after = math.ceil((cost - tokens) / refill_rate)
    return {0, retry_after, tokens}
end

tokens = tokens - cost
redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
redis.call("EXPIRE", key, math.ceil(max_tokens / refill_rate) * 2)

return {1, 0, tokens}
```
</details>

### Puzzle 7 — Leader election for Chronos

Design a Lua-based leader election mechanism for Chronos.

<details>
<summary>Answer</summary>

```lua
-- Lease-based leader election
local leader_key = KEYS[1]     -- scheduler:leader
local node_id = ARGV[1]        -- "chronos-node-1"
local lease_ttl = tonumber(ARGV[2])  -- 30 seconds

-- Try to become leader
local result = redis.call("SET", leader_key, node_id, "NX", "EX", lease_ttl)
if result then
    return {1, "became_leader", node_id}
end

-- Check if we're already the leader (extend lease)
local current_leader = redis.call("GET", leader_key)
if current_leader == node_id then
    redis.call("EXPIRE", leader_key, lease_ttl)
    return {1, "extended_lease", node_id}
end

-- Not leader
return {0, "not_leader", current_leader}
```

```
// Chronos application loop:
while (true) {
    result = redis.eval(leaderElectionScript, 1, "scheduler:leader", nodeId, 30)
    if (result[0] == 1) {
        // I'm the leader — schedule jobs, heartbeat
        redis.setex("scheduler:heartbeat:" + nodeId, 10, "alive")
    }
    sleep(5) // check every 5 seconds
}
```
</details>

### Puzzle 8 — Bulk cache warming

Write a script to pre-load cache for all active tenants on application deploy.

<details>
<summary>Answer</summary>

```python
import redis
r = redis.Redis()

def warm_tenant_cache(org_id):
    pipe = r.pipeline()
    
    # Cache dashboard data
    dashboard = calculate_dashboard(org_id)
    pipe.setex(f"org:{org_id}:dashboard", 300, json.dumps(dashboard))
    
    # Cache active users count
    user_count = get_active_user_count(org_id)
    pipe.setex(f"org:{org_id}:active_users", 300, user_count)
    
    # Cache recent orders summary
    recent_orders = get_recent_orders(org_id)
    pipe.setex(f"org:{org_id}:recent_orders", 300, json.dumps(recent_orders))
    
    pipe.execute()

# Warm in parallel with rate limiting
from concurrent.futures import ThreadPoolExecutor

active_orgs = DB::table('organizations')->where('active', true)->pluck('id')
with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(warm_tenant_cache, active_orgs)
```

**Production considerations:**
- Throttle to avoid overwhelming database during warm-up
- Warm only frequently accessed data (cache-aside will handle the rest)
- Use sequential pipeline to batch Redis commands per tenant
</details>

### Puzzle 9 — Cache invalidation with change tracking

Design a mechanism to invalidate org-level caches when any data changes for that org.

<details>
<summary>Answer</summary>

```php
class CacheInvalidator {
    // On any data change for an org (order created, user updated, product changed)
    public function invalidateOrg($orgId) {
        // Use pattern to delete all org-related caches
        $keys = Redis::keys("org:{$orgId}:*");  // Don't use KEYS in production!
        
        // Instead, use SCAN or store org keys in a set
        $cacheKeys = Redis::smembers("org:{$orgId}:cache_keys");
        
        $pipe = Redis::pipeline();
        foreach ($cacheKeys as $key) {
            $pipe->del($key);
        }
        $pipe->execute();
    }

    // When caching, track the key for this org
    public function rememberWithInvalidation($orgId, $key, $ttl, $callback) {
        // Store cache
        $value = $callback();
        Redis::setex($key, $ttl, json_encode($value));
        
        // Track key for invalidation
        Redis::sadd("org:{$orgId}:cache_keys", $key);
        Redis::expire("org:{$orgId}:cache_keys", $ttl + 3600);
        
        return $value;
    }
}

// Usage
$cacheKey = "org:{$orgId}:dashboard";
$dashboard = $invalidator->rememberWithInvalidation(
    $orgId, $cacheKey, 300,
    fn() => DashboardService::calculateForOrg($orgId)
);

// On new order:
DB::table('orders')->insert($orderData);
$invalidator->invalidateOrg($orgId);
```
</details>

### Puzzle 10 — Find the bug

```python
import redis
r = redis.Redis()

def transfer(sender, receiver, amount):
    balance = int(r.get(sender) or 0)
    if balance < amount:
        return False
    
    r.decrby(sender, amount)
    r.incrby(receiver, amount)
    return True
```

<details>
<summary>Answer</summary>

Race condition: `GET` and `DECRBY` are not atomic. Between the `GET` and `DECRBY`, another process could read the same balance and both succeed.

**Fix 1 — Lua script:**
```lua
local sender = KEYS[1]
local receiver = KEYS[2]
local amount = tonumber(ARGV[1])

local balance = tonumber(redis.call("GET", sender) or 0)
if balance < amount then
    return {0, "Insufficient funds"}
end

redis.call("DECRBY", sender, amount)
redis.call("INCRBY", receiver, amount)
return {1, "Success"}
```

**Fix 2 — WATCH + MULTI/EXEC:**
```python
def transfer_safe(sender, receiver, amount):
    while True:
        r.watch(sender)
        balance = int(r.get(sender) or 0)
        if balance < amount:
            r.unwatch()
            return False
        pipe = r.pipeline(transaction=True)
        pipe.decrby(sender, amount)
        pipe.incrby(receiver, amount)
        try:
            pipe.execute()
            return True
        except redis.WatchError:
            continue  # retry
```
</details>

---

## 3. Debugging scenarios (6 scenarios)

### Scenario 1 — High Redis CPU

**Symptom:** Redis CPU at 100%. Applications timing out.

**Debugging:**
1. `INFO commandstats` — which command consumes the most CPU?
2. `SLOWLOG GET 20` — slow commands?
3. `INFO clients` — too many connections?
4. `redis-cli --bigkeys` — large keys causing slow operations?

**Likely cause:** `KEYS *` or `SMEMBERS` on a large set, or a runaway Lua script.

**Solution:**
- Replace `KEYS` with `SCAN`
- Replace `SMEMBERS` with `SSCAN` (or maintain a separate data structure)
- Kill runaway script: `SCRIPT KILL`
- Add rate limiting for expensive commands
- Increase `lua-time-limit` and optimize Lua scripts

### Scenario 2 — Memory full, evictions increasing

**Symptom:** `INFO stats` shows `evicted_keys` increasing rapidly. Responses slow down.

**Debugging:**
1. `INFO memory` — `used_memory`, `maxmemory`, `mem_fragmentation_ratio`
2. `INFO stats` — `evicted_keys`
3. Check eviction policy: `CONFIG GET maxmemory-policy`
4. `redis-cli --bigkeys` — identify large keys
5. `MEMORY STATS` — breakdown by type

**Likely cause:** `maxmemory` too low for working set, or keys without TTL accumulating.

**Solution:**
- Increase `maxmemory` (if RAM available)
- Ensure all cache keys have TTL
- Switch to more aggressive eviction policy (`allkeys-lru`)
- Enable `activedefrag yes` if fragmentation > 1.5
- Consider Cluster if memory > 50 GB

### Scenario 3 — Replication lag increasing

**Symptom:** Replication lag grows during peak traffic. Reads from replicas return increasingly stale data.

**Debugging:**
1. `INFO replication` — check `lag` per replica
2. Check replica CPU/memory — can it keep up with oplog apply?
3. Check `repl-backlog-size` — is the backlog small enough to cause full resyncs?
4. Check network bandwidth between master and replica
5. `INFO stats` on replica — `sync_partial_err` (partial resync failures)?

**Likely cause:** Network bandwidth saturation, or replica too slow to apply write stream.

**Solution:**
- Increase `repl-backlog-size` (e.g., 256 MB for write-heavy workload)
- Add more replicas to distribute read load
- Reduce write load on master (cache more aggressively)
- Upgrade replica hardware (faster CPU, more RAM)
- Disable AOF on replicas for faster apply

### Scenario 4 — Connection pool exhaustion

**Symptom:** Applications get "connection refused" or "max number of clients reached" errors.

**Debugging:**
1. `INFO clients` — `connected_clients`, `max_clients`
2. `CLIENT LIST` — see all connections, their ages, idle time
3. Check application connection pooling — each app server opening N connections?

**Likely cause:** Connection leak (connections not returned to pool) or max_clients too low.

**Solution:**
- Fix connection pool settings (`maxPoolSize` in app)
- Implement proper connection management (close after use)
- Increase `maxclients` (default 10000)
- Add a connection proxy (e.g., Envoy, HAProxy) to multiplex connections
- Set `timeout` to close idle connections

### Scenario 5 — Latency spikes every 2 minutes

**Symptom:** Regular latency spikes every ~2 minutes on the dot.

**Debugging:**
1. `LATENCY DOCTOR` — check for fork, swap, THP
2. `INFO persistence` — check `rdb_last_bgsave_time_sec` and `aof_last_rewrite_time_sec`
3. Check cron jobs on server — any scripts or backups running?
4. `LATENCY HISTORY fork` — fork duration

**Likely cause:** `BGSAVE` or `BGREWRITEAOF` running every 2 minutes (save config too aggressive).

**Solution:**
- Adjust save intervals: less frequent RDB snapshots
- Schedule `BGREWRITEAOF` during low traffic
- Disable THP: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
- Increase `repl-backlog-size` to reduce need for full RDB sync
- Consider using replicas for persistence (let replica handle BGSAVE)

### Scenario 6 — Cache stampede after deploy

**Symptom:** After a new deploy (which flushes Redis), the database is overwhelmed and the application is slow for 5-10 minutes.

**Debugging:**
1. Check cache hit rate on startup — should be 0% initially
2. Check database query throughput — high?
3. Check application error logs — timeouts?

**Likely cause:** All cache keys expired simultaneously on deploy, causing a stampede.

**Solution:**
- **Cache warming**: pre-load critical keys on startup
- **Staggered cache flush**: don't flush all keys at once; expire gradually
- **Lazy warming**: cache-aside with stampede protection (mutex lock)
- **Gradual deploy**: roll out to 10% of instances first, let them warm the cache
- **Pre-computation**: compute cache values before deploy, store in a file, bulk load

```php
// Deployment script example
Artisan::command('cache:warm', function () {
    $tenants = Tenant::active()->pluck('id');
    $bar = $this->output->createProgressBar(count($tenants));

    foreach ($tenants as $orgId) {
        Cache::remember("org:{$orgId}:dashboard", 300, function () use ($orgId) {
            return DashboardService::calculate($orgId);
        });
        $bar->advance();
    }
    $bar->finish();
});
```

---

## 4. System design prompts (5 prompts)

### Prompt 1 — Design a real-time leaderboard

Design a real-time leaderboard for your trading platform (20K+ DAU) using Redis.

**Requirements:**
- Scores update on every trade
- Show top 100 traders by P&L
- Show a trader's rank
- Real-time updates

**Schema:**
```bash
# Leaderboard (Sorted Set)
ZADD leaderboard:daily:2024-01-15 15000 "trader:1001"
ZADD leaderboard:daily:2024-01-15 12000 "trader:1002"

# Top 100
ZREVRANGE leaderboard:daily:2024-01-15 0 99 WITHSCORES

# Trader rank
ZREVRANK leaderboard:daily:2024-01-15 "trader:1001"

# Trader info (Hash)
HSET trader:1001 name "Alice" avatar "url" total_trades 500
```

**Updates:**
```lua
-- On trade completion
local trader = ARGV[1]
local pnl_change = tonumber(ARGV[2])
local daily_key = KEYS[1]  -- leaderboard:daily:2024-01-15

redis.call("ZINCRBY", daily_key, pnl_change, trader)
redis.call("EXPIRE", daily_key, 86400)
return redis.call("ZREVRANK", daily_key, trader)
```

**Real-time push:** Use Pub/Sub to broadcast leaderboard changes to connected clients.

**Scaling:**
- For 20K DAU — single Redis instance handles this easily
- For 1M+ DAU — shard by market/symbol, pre-aggregate top 100 per shard, then merge

### Prompt 2 — Design a rate limiting layer for SaaS API

Design a rate limiting system for your multi-tenant SaaS API.

**Requirements:**
- Per-tenant rate limits (different limits per plan)
- Per-user rate limits
- Per-endpoint rate limits (expensive endpoints get lower limits)
- Burst support (short spikes allowed)
- Rate limit headers in API responses

**Design:**
```
Rate limit types:
  - Plan-wide: 1000 requests/min (free), 10000/min (enterprise)
  - Endpoint: POST /api/orders 50/min, GET /api/products 200/min
  - User: 100/min per user
```

**Token bucket per user:**
```lua
local key = KEYS[1]  -- ratelimit:user:{userId}:api
local max_tokens = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local cost = tonumber(ARGV[3])

local info = redis.call("HMGET", key, "tokens", "last_refill")
local tokens = tonumber(info[1] or max_tokens)
local last_refill = tonumber(info[2] or 0)
local now = redis.call("TIME")[1]
local elapsed = math.max(0, now - last_refill)
tokens = math.min(max_tokens, tokens + elapsed * refill_rate)

if tokens < cost then
    local retry_after = math.ceil((cost - tokens) / refill_rate)
    return {0, retry_after}
end

tokens = tokens - cost
redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
redis.call("EXPIRE", key, math.ceil(max_tokens / refill_rate) * 2)

-- Return headers info
return {1, 0, tokens, max_tokens}
```

**API response:**
```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1705000000
```

**Middleware pattern (Laravel):**
```php
class RateLimitMiddleware {
    public function handle($request, $next) {
        $key = "ratelimit:user:{$request->user()->id}:api";
        $limit = $request->user()->plan->api_rate_limit;
        
        $result = Redis::eval($script, 1, $key, $limit, 2, 1);
        
        if ($result[0] == 0) {
            return response()->json([
                'error' => 'Too many requests',
                'retry_after' => $result[1]
            ], 429);
        }
        
        $response = $next($request);
        $response->header('X-RateLimit-Limit', $limit);
        $response->header('X-RateLimit-Remaining', $result[3]);
        
        return $response;
    }
}
```

### Prompt 3 — Design a distributed job queue (Chronos)

Design a job queue for Chronos using Redis Streams.

**Requirements:**
- Schedule jobs at specific times (cron-like)
- Multiple worker types (email, report, cleanup)
- Reliable delivery (ACK-based)
- Retry with backoff
- Dead letter queue for failed jobs
- Real-time job status updates

**Schema:**
```php
// Job definition (stored separately)
HSET job:email:welcome payload '{"to":"user@example.com"}' schedule "cron(0 9 * * *)" next_run 1705000000

// Stream for scheduled jobs
XADD scheduler:stream * job_id "email:welcome" type "email" run_at 1705000000

// Consumer group per worker type
XGROUP CREATE scheduler:stream email-workers $ MKSTREAM
XGROUP CREATE scheduler:stream report-workers $ MKSTREAM

// Worker reads
XREADGROUP GROUP email-workers worker-1 COUNT 1 BLOCK 5000 STREAMS scheduler:stream >

// Job status
HSET job:email:welcome status running started_at 1705000100 worker worker-1
```

**Scheduler component (Lua):**
```lua
-- Check for due jobs every 60 seconds
local jobs_to_schedule = redis.call("ZRANGEBYSCORE", "scheduler:jobs:due",
    0, redis.call("TIME")[1], "LIMIT", 0, 100)

for _, job_id in ipairs(jobs_to_schedule) do
    -- Push to stream
    redis.call("XADD", "scheduler:stream", "*", "job_id", job_id,
        "type", redis.call("HGET", "job:" .. job_id, "type"),
        "run_at", redis.call("TIME")[1])

    -- Remove from due set (or update next run)
    redis.call("ZREM", "scheduler:jobs:due", job_id)
end
```

**Retry with backoff:**
```lua
-- On failure
local job_id = KEYS[1]
local retry_count = redis.call("HINCRBY", "job:" .. job_id, "retry_count", 1)
local max_retries = tonumber(redis.call("HGET", "job:" .. job_id, "max_retries") or 3)

if retry_count >= max_retries then
    -- Move to dead letter
    redis.call("XADD", "scheduler:dead", "*", "job_id", job_id, "reason", "max_retries_exceeded")
else
    -- Exponential backoff: 2^retry * 60 seconds
    local delay = math.pow(2, retry_count) * 60
    local next_run = redis.call("TIME")[1] + delay
    redis.call("ZADD", "scheduler:jobs:due", next_run, job_id)
end
```

### Prompt 4 — Design a session management system

Design a session management system for your SaaS using Redis.

**Requirements:**
- User sessions (remember me, token-based)
- Session invalidation (logout, admin force logout)
- Device tracking (multiple concurrent sessions)
- Rate limiting per session
- Session expiry

**Schema:**
```php
// Session token → user data
HSET session:abc123 user_id 1000 org_id org_42
      created_at 1705000000
      last_activity 1705000500
      device "Chrome/Mac"
      ip "192.168.1.1"
EXPIRE session:abc123 86400  // 24 hours

// User → active sessions (for invalidation)
SADD user:1000:sessions session:abc123 session:def456

// Remember me (30-day token)
HSET session:remember:xyz789 user_id 1000 token_hash "..."
EXPIRE session:remember:xyz789 2592000  // 30 days
```

**Login flow:**
```php
function createSession($userId, $orgId, $device, $ip, $remember = false) {
    $token = bin2hex(random_bytes(32));
    $ttl = $remember ? 2592000 : 86400;

    Redis::multi();
    Redis::hmset("session:{$token}", [
        'user_id' => $userId,
        'org_id' => $orgId,
        'created_at' => time(),
        'last_activity' => time(),
        'device' => $device,
        'ip' => $ip,
    ]);
    Redis::expire("session:{$token}", $ttl);
    Redis::sadd("user:{$userId}:sessions", $token);
    Redis::exec();

    return $token;
}
```

**Logout / force-invalidate:**
```php
function invalidateUserSessions($userId) {
    $sessions = Redis::smembers("user:{$userId}:sessions");
    foreach ($sessions as $token) {
        Redis::del("session:{$token}");
    }
    Redis::del("user:{$userId}:sessions");
}
```

**Session check (middleware):**
```php
function getSessionUser($token) {
    $session = Redis::hgetall("session:{$token}");
    if (!$session) {
        return null;
    }
    // Touch session (update last_activity + extend TTL)
    Redis::multi();
    Redis::hset("session:{$token}", 'last_activity', time());
    Redis::expire("session:{$token}", 86400);
    Redis::exec();

    return $session;
}
```

### Prompt 5 — Design a caching layer for the SaaS dashboard

Design a comprehensive caching strategy for your multi-tenant inventory SaaS targeting the 88% query reduction story.

**Requirements:**
- Dashboard data (order counts, revenue, user stats)
- Product catalog pages
- Search results
- Real-time updates when data changes
- Tenant isolation (one tenant's heavy load shouldn't affect others)

**Strategy:**

```
Layer 1: Application memory (L1) — per-tenant dashboard data, 60s TTL
Layer 2: Redis (L2) — shared across app servers, 300s TTL
Layer 3: Database — source of truth
```

**Cache granularity:**
```php
// 1. Tenant-level dashboard (aggregated)
Cache::tags(["org:{$orgId}", "dashboard"])
    ->remember("org:{$orgId}:dashboard:summary", 300, fn() =>
        DashboardService::summary($orgId));

// 2. Recent orders list
Cache::tags(["org:{$orgId}", "orders"])
    ->remember("org:{$orgId}:orders:recent:{$page}", 120, fn() =>
        Order::where('org_id', $orgId)->latest()->paginate(50));

// 3. Product count
Cache::remember("org:{$orgId}:product_count", 600, fn() =>
    Product::where('org_id', $orgId)->count());

// 4. API rate limit counters (in Redis — no L1)
Redis::incr("ratelimit:org:{$orgId}:api:" . intdiv(time(), 60));
```

**Invalidation strategy:**
```php
// When order created:
Cache::tags(["org:{$orgId}", "orders", "dashboard"])->flush();
Redis::incr("org:{$orgId}:version");  // Version key for data freshness check

// When product updated:
Cache::tags(["org:{$orgId}", "products"])->flush();
Cache::forget("product:{$productId}");
```

**Data freshness (version-based):**
```php
function getDashboardData($orgId) {
    $versionKey = "org:{$orgId}:version";
    $dataKey = "org:{$orgId}:dashboard";

    $cachedVersion = Redis::get("{$dataKey}:version");
    $currentVersion = Redis::get($versionKey);

    if ($cachedVersion === $currentVersion) {
        $data = Redis::get($dataKey);
        if ($data) return json_decode($data);
    }

    $data = DashboardService::calculate($orgId);
    Redis::multi();
    Redis::setex($dataKey, 300, json_encode($data));
    Redis::set("{$dataKey}:version", $currentVersion);
    Redis::exec();

    return $data;
}
```

**Results (88% reduction):**
- Before: 50K queries/day to PostgreSQL
- After: 6K queries/day (cache hit rate: 95%)
- Dashboard load time: 3.2s → 150ms
- API response time p95: 800ms → 45ms

---

## 5. STAR story templates

### Story: 88% query reduction via Redis caching

**Situation:** The SaaS dashboard was loading slowly (3-5 seconds) due to complex aggregate queries on every page load. As the system grew to more tenants, PostgreSQL CPU hit 70% during peak hours.

**Task:** Reduce database query load by 80%+ while keeping data fresh enough for business decisions.

**Action:**
- Profiled slowest queries using `pg_stat_statements` — identified dashboard aggregates, recent orders, and user counts as top candidates
- Implemented multi-tier cache-aside with Redis:
  - Dashboard aggregates cached for 5 minutes with per-tenant keys (`org:{id}:dashboard`)
  - Used hash tags for tenant isolation in Redis keyspace
  - Added version-based cache invalidation — bumped version on any data mutation
  - Wrote Lua script for atomic cache read + version check
- Added stampede protection — mutex lock on cache miss so only one request recomputes
- Configured `allkeys-lru` eviction policy (4 GB maxmemory)

**Result:** Database queries dropped from 50K/day to 6K/day (88% reduction). Dashboard load time 150ms (from 3.2s). PostgreSQL CPU dropped to 15%. Team scaled to 3× more tenants without database upgrade.

### Story: Distributed lock for Chronos leader election

**Situation:** Chronos (Go Raft-based scheduler) needed a simpler leader election mechanism for the MVP to avoid Raft complexity.

**Task:** Implement reliable leader election with automatic failover for the distributed scheduler.

**Action:**
- Used Redis `SET NX EX` with unique owner ID for lease-based leader election
- Implemented Lua script for safe lock acquisition + automatic heartbeat/lease extension
- Used `min-replicas-to-write` on Redis Sentinels to prevent split-brain
- Added Redisson-style watchdog: worker extends lease every 10 seconds while processing
- On connectivity loss, lease expires in 30 seconds → another worker takes over

**Result:** Simpler than Raft (single Lua script vs multi-round RPC). No leader election-related incidents in production. Mean time to elect new leader after crash: ~15 seconds.

---

## 6. Key metrics to remember

| Metric | Target | Why |
|--------|--------|-----|
| Cache hit rate | > 90% | Lower = DB getting hammered |
| Evicted keys | 0/hour in steady state | Increasing = maxmemory too low |
| mem_fragmentation_ratio | 1.0 - 1.5 | > 1.5 = fragmentation |
| Replication lag | < 1 second | Stale reads if higher |
| SLOWLOG entries | < 10/hour | Slow commands = bad |
| Connected clients | < 80% of maxclients | Exhaustion risk |
| Fork duration | < 300ms | Fork pauses Redis |
| Hit ratio per eviction policy | Stable | Frequent eviction = wrong policy |
| Average latency (p99) | < 10ms | Redis should be sub-ms |
| keyspace_hits / (keyspace_hits + keyspace_misses) | > 90% | Cache effectiveness |

---

## 7. Interview preparation checklist

- [ ] I can explain Redis's single-threaded event loop and its implications
- [ ] I can choose the right data structure for any use case
- [ ] I can write Lua scripts for atomic operations
- [ ] I can design a caching strategy with invalidation
- [ ] I can implement rate limiting (sliding window, token bucket)
- [ ] I can design a distributed lock with safe release
- [ ] I can explain Redis replication, Sentinel, and Cluster trade-offs
- [ ] I can read `INFO` output and diagnose issues
- [ ] I can prevent cache stampede (locking, early recomputation)
- [ ] I can explain when to use List vs Pub/Sub vs Streams
- [ ] I can describe my 88% query reduction story with Redis caching
- [ ] I can compare Redis with Memcached, Valkey, KeyDB, Dragonfly
- [ ] I know ACL, TLS, rename-command for Redis security
- [ ] I understand eviction policies and when to use each
- [ ] I can design session management with Redis
