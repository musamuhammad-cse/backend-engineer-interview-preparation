# PostgreSQL — Question Bank & Drill Material

> **Target:** Senior Backend Engineer (8+ years)  
> **Your anchor:** PostgreSQL is your PRIMARY database — multi-tenant SaaS, 15M+ row zero-downtime migrations, 88% query reduction, trading platform concurrency  
> **How to drill:** Cover 1–2 sections per day. For each question: answer verbally → check answer → mark confidence. Re-drill weak areas next session.

---

## Table of Contents

1. [Rapid-Fire: PostgreSQL Architecture (25 questions)](#1-rapid-fire-postgresql-architecture-25-questions)
2. [Rapid-Fire: MVCC & Vacuum (25 questions)](#2-rapid-fire-mvcc--vacuum-25-questions)
3. [Rapid-Fire: Data Types & SQL (25 questions)](#3-rapid-fire-data-types--sql-25-questions)
4. [Rapid-Fire: Indexing (25 questions)](#4-rapid-fire-indexing-25-questions)
5. [Rapid-Fire: EXPLAIN & Query Optimization (20 questions)](#5-rapid-fire-explain--query-optimization-20-questions)
6. [Rapid-Fire: Locking & Concurrency (20 questions)](#6-rapid-fire-locking--concurrency-20-questions)
7. [Rapid-Fire: Replication & HA (15 questions)](#7-rapid-fire-replication--ha-15-questions)
8. [Rapid-Fire: Performance & Operations (15 questions)](#8-rapid-fire-performance--operations-15-questions)
9. [Code Puzzles (8 puzzles)](#9-code-puzzles-8-puzzles)
10. [Debugging Scenarios (6 scenarios)](#10-debugging-scenarios-6-scenarios)
11. [System Design Prompts (6 prompts)](#11-system-design-prompts-6-prompts)
12. [STAR Stories (4 templates)](#12-star-stories-4-templates)
13. [Questions to Ask the Interviewer (10 questions)](#13-questions-to-ask-the-interviewer-10-questions)
14. [Red Flags to Avoid (10 red flags)](#14-red-flags-to-avoid-10-red-flags)

---

## 1. Rapid-Fire: PostgreSQL Architecture (25 questions)

1. **Q:** What are the main background processes in a PostgreSQL server, and what does each do?

<details><summary>Answer</summary>

- **postmaster:** Parent process; listens for connections, forks backends
- **checkpointer:** Writes all dirty shared buffers to disk at checkpoints
- **bgwriter:** Periodically writes dirty shared buffers to disk *between* checkpoints to spread I/O
- **WAL writer:** Flushes WAL buffer to WAL segments
- **autovacuum launcher:** Schedules autovacuum workers based on dead tuple thresholds
- **stats collector:** Gathers table/index/function access statistics (pg_stat_*)
- **archiver:** Copies WAL segments to archive location (if archive_mode = on)
- **WAL sender:** Sends WAL to replicas (streaming replication)
- **logical replication launcher:** Manages logical replication slots

</details>

2. **Q:** What is the relationship between checkpointer and bgwriter?

<details><summary>Answer</summary>

bgwriter periodically writes some dirty buffers to disk to keep checkpointer from having to write everything at once. checkpointer ensures all dirty buffers are flushed at a checkpoint, guaranteeing crash recovery can start from that LSN. Without bgwriter, checkpoints would cause massive I/O spikes.

**Trap:** "They do the same thing" — no, bgwriter spreads I/O; checkpointer guarantees consistency.

</details>

3. **Q:** What is the WAL (Write-Ahead Log) and why is it fundamental?

<details><summary>Answer</summary>

WAL records every change to data files *before* the data files themselves are updated (write-ahead). It enables:
- **Crash recovery:** Replay WAL from last checkpoint to restore consistency
- **Replication:** Standbys replay WAL from primary
- **PITR:** Archive WAL segments + base backup to restore to any point in time

**Trap:** "WAL is just for replication" — crash recovery is the *primary* purpose; replication is a derived benefit.

</details>

4. **Q:** What data lives in PostgreSQL shared memory?

<details><summary>Answer</summary>

- **Shared buffers** (shared_buffers): Cached data pages
- **WAL buffer** (wal_buffers): Pending WAL records before flush
- **CLOG (Commit Log):** Visibility status of transactions
- **Lock space:** Row-level and table-level locks
- **Proc array:** Active backend process info
- **Buffer descriptors:** Metadata about shared buffer slots

</details>

5. **Q:** What happens when a CHECKPOINT occurs?

<details><summary>Answer</summary>

1. WAL is flushed up to the checkpoint's REDO point
2. All dirty buffers in shared buffers are written to the OS
3. Data files are fsync'd
4. pg_control is updated with the checkpoint LSN
5. Any WAL segments before the REDO point can be recycled/removed

**Follow-up:** What's the difference between a full checkpoint and a restartpoint? A restartpoint is the checkpoint on a standby.

</details>

6. **Q:** What is pg_hba.conf and how does authentication flow work?

<details><summary>Answer</summary>

pg_hba.conf (Host-Based Authentication) controls which hosts, users, databases, and authentication methods are allowed. Rules are evaluated **top-down** — first match wins. Common methods: scram-sha-256, md5, trust, peer, cert.

**Trap:** "It only controls host connections" — local socket connections are also controlled via local entries.

</details>

7. **Q:** What are system catalogs, and name the most important ones?

<details><summary>Answer</summary>

System catalogs (in pg_catalog schema) store database metadata:
- pg_class: Tables, indexes, sequences, views
- pg_attribute: Column definitions
- pg_index: Index metadata
- pg_namespace: Schemas
- pg_proc: Functions/procedures
- pg_stat_all_tables: Table-level statistics
- pg_stat_statements (extension): Query execution statistics

</details>

8. **Q:** Is PostgreSQL thread-based or process-based?

<details><summary>Answer</summary>

**Process-based** (not threaded). Each client connection gets a dedicated backend process forked from postmaster. This provides strong isolation (one backend crash doesn't take down others) but more memory overhead per connection vs thread-based databases like MySQL.

</details>

9. **Q:** What is max_connections and what happens when it's exceeded?

<details><summary>Answer</summary>

max_connections limits concurrent backend processes. When exceeded: "FATAL: sorry, too many clients already". Each connection consumes ~5–10 MB. With PgBouncer in transaction pooling, you can handle thousands of app connections with just 50–100 actual PostgreSQL connections.

</details>

10. **Q:** What does pg_stat_activity show, and how do you detect a long-running query?

<details><summary>Answer</summary>

pg_stat_activity shows: pid, usename, application_name, state (active/idle/in transaction), query, query_start, wait_event, wait_event_type.

```sql
SELECT pid, now() - query_start AS duration, query, state, wait_event
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

</details>

11. **Q:** What is the difference between checkpointer and bgwriter in dirty buffer writing?

<details><summary>Answer</summary>

| Process | When | What |
|---------|------|------|
| bgwriter | Continuously between checkpoints | Writes some older buffers to spread I/O |
| checkpointer | At checkpoint time | Writes all dirty buffers and fsyncs |

**Follow-up:** Why have both? Without bgwriter, all dirty buffer writes happen at checkpoint, causing massive I/O spikes.

</details>

12. **Q:** What is the purpose of the stats collector? Does it impact query performance?

<details><summary>Answer</summary>

Collects table/index/function access statistics via shared memory and writes to disk. Used by:
- Autovacuum to decide when to vacuum
- Query planner for row estimates (pg_class.reltuples, pg_statistic)

**Impact:** Minimal — stats collector receives message-based updates.

</details>

13. **Q:** What are wait_event and wait_event_type in pg_stat_activity?

<details><summary>Answer</summary>

wait_event_type categories: LWLock, Lock, BufferPin, Activity, Client, Extension, IPC, Timeout, IO. wait_event is the specific event name (e.g., WALWriteLock, relation for table-level lock waits).

**Trap:** Confusing LWLock (lightweight lock — internal, short-duration) with Lock (heavyweight lock — table/row level, held for transaction duration).

</details>

14. **Q:** What components comprise the WAL?

<details><summary>Answer</summary>

WAL consists of 16 MB segments stored in pg_wal/. Each segment has a 24-byte header with timeline ID, LSN, and checksum. WAL records contain page data (full page image or modified tuple data) and XLOG record header (LSN, transaction ID, previous LSN, record type).

**Follow-up:** What is full_page_writes? When enabled (default), the first page modification after a checkpoint writes the entire page to WAL for crash safety.

</details>

15. **Q:** How does max_worker_processes relate to max_parallel_workers?

<details><summary>Answer</summary>

- max_worker_processes: Total system-wide background worker limit
- max_parallel_workers: Subset allocated to parallel query execution
- max_parallel_workers_per_gather: Workers per single parallel operation

**Trap:** Setting max_parallel_workers_per_gather high while max_parallel_workers is low won't help.

</details>

16. **Q:** What is the purpose of the pg_control file?

<details><summary>Answer</summary>

pg_control (in PGDATA/global/) stores critical state: latest checkpoint LSN, database system identifier, PostgreSQL version, WAL position, system state. If corrupted, PostgreSQL cannot start.

</details>

17. **Q:** What happens when a backend crashes?

<details><summary>Answer</summary>

1. Postmaster receives SIGCHLD for the crashed backend
2. Postmaster rolls back the crashed backend's transaction
3. Postmaster releases all locks held by the backend
4. Postmaster cleans up shared memory state for that PID
5. If restart_after_crash = on (default), postmaster restarts all other backends and performs crash recovery

**Trap:** A crashed backend does not take down other backends (process isolation), but postmaster may restart all backends as a safety measure.

</details>

18. **Q:** What is the difference between pg_ctl stop -m smart, -m fast, and -m immediate?

<details><summary>Answer</summary>

| Mode | Behavior | Crash recovery needed? |
|------|----------|----------------------|
| smart | Waits for all clients to disconnect | No |
| fast (default) | Rolls back open transactions, disconnects clients | No |
| immediate | Kills all processes immediately, aborting in-flight transactions | Yes (WAL replay on restart) |

</details>

19. **Q:** What directory structure does PGDATA contain?

<details><summary>Answer</summary>

- PG_VERSION, postgresql.conf, pg_hba.conf, pg_ident.conf
- base/: Per-database subdirectories (OID-named)
- global/: Cluster-wide tables + pg_control
- pg_wal/: WAL segments
- pg_stat/: Persistent statistics
- pg_xact/: Transaction commit state
- pg_multixact/: Multi-transaction data
- pg_logical/: Logical replication state
- pg_replslot/: Replication slots

</details>

20. **Q:** What is the difference between shared_buffers and OS page cache?

<details><summary>Answer</summary>

PostgreSQL does not manage its own I/O. shared_buffers caches data pages inside PostgreSQL shared memory. OS page cache caches pages in kernel space. Reads go: shared_buffers → OS page cache → disk. Recommendation: shared_buffers at 25% of RAM.

**Trap:** "Double caching" — yes, some pages exist in both, but OS cache is still valuable because PostgreSQL doesn't use direct I/O.

</details>

21. **Q:** What is archive_mode and archive_command used for?

<details><summary>Answer</summary>

archive_mode = on enables WAL archiving. archive_command is the shell command that copies each completed WAL segment to an archive location:

```
archive_command = 'pgbackrest --stanza=prod archive-push %p'
```

This enables Point-In-Time Recovery (PITR) and can feed replicas.

</details>

22. **Q:** What is the purpose of WAL segment recycling vs removal?

<details><summary>Answer</summary>

Instead of deleting old WAL segments, PostgreSQL renames them for reuse. This reduces filesystem I/O. Segments are recycled once they're no longer needed for crash recovery (past the latest checkpoint) and archiving.

**Follow-up:** How does max_wal_size affect recycling? It sets a target for total WAL size; segments beyond this are recycled more aggressively.

</details>

23. **Q:** What is the function of the syslogger process?

<details><summary>Answer</summary>

The syslogger writes PostgreSQL log output to files. It handles log rotation when logging_collector = on. It's optional.

</details>

24. **Q:** How does track_activities impact monitoring?

<details><summary>Answer</summary>

When track_activities = on (default), PostgreSQL tracks per-backend query and state info in pg_stat_activity. Disabling it saves a small amount of CPU but breaks all monitoring tools. Almost always left on in production.

</details>

25. **Q:** What is the impact of many idle-in-transaction connections?

<details><summary>Answer</summary>

Idle-in-transaction connections hold locks, prevent vacuum from cleaning dead tuples, and block DDL. They're a common source of bloat and replication lag. Mitigate with idle_in_transaction_session_timeout.

**Trap:** "Idle connections are harmless" — idle-in-transaction is not harmless.

</details>

---

## 2. Rapid-Fire: MVCC & Vacuum (25 questions)

1. **Q:** What are xmin and xmax in a tuple header?

<details><summary>Answer</summary>

- xmin: The transaction ID (XID) that created this tuple version
- xmax: The XID that deleted/updated this tuple version (0 if not deleted)

Together they determine tuple visibility to a snapshot.

</details>

2. **Q:** How does PostgreSQL determine if a tuple is visible to a transaction?

<details><summary>Answer</summary>

Using the snapshot's xmin/xmax ranges:
- A tuple is visible if xmin is committed and xmax is not committed (or xmax is current transaction's own delete that rolled back)
- xmin > snapshot xmax → invisible (future transaction)
- xmax committed and xmax < snapshot xmin → dead

**Follow-up:** Subtransactions complicate visibility — tracked in pg_subtrans.

</details>

3. **Q:** What is a dead tuple?

<details><summary>Answer</summary>

A tuple version no longer visible to any running transaction. Created by:
- UPDATE (old version becomes dead, new version created — PostgreSQL does not update in place)
- DELETE (the deleted version is dead)

Dead tuples are cleaned by VACUUM.

</details>

4. **Q:** What is the difference between VACUUM and VACUUM FULL?

<details><summary>Answer</summary>

| Operation | Bloat removal | Locks | Table size |
|-----------|--------------|-------|------------|
| VACUUM | Space marked reusable, not returned to OS | ShareUpdateExclusiveLock (allows reads) | Does not shrink |
| VACUUM FULL | Rewrites entire table, returns space to OS | AccessExclusiveLock (blocks everything) | Shrinks |

**Trap:** "VACUUM shrinks tables" — it does not. Only VACUUM FULL or CLUSTER does.

</details>

5. **Q:** What is a HOT (Heap-Only Tuple) update?

<details><summary>Answer</summary>

When an UPDATE changes no indexed columns, the new tuple can be placed in the same page as the old one, linked via a heap-only chain. This avoids adding an index entry. Conditions: No indexed columns change AND enough free space on the page.

**Follow-up:** How does fillfactor affect HOT updates? A lower fillfactor (default 100, recommend 70–90 for frequently updated tables) reserves free space.

</details>

6. **Q:** What is XID wraparound and how is it prevented?

<details><summary>Answer</summary>

Transaction IDs are 32-bit (4 billion). When XID wraps past 0, old tuples appear to be in the future and become invisible. Prevention: **freezing** — VACUUM marks old tuples with FrozenTransactionId (2), which is always visible.

**Trap:** XID wraparound is cluster-down critical. If hit, PostgreSQL shuts down to prevent data loss.

</details>

7. **Q:** What are the autovacuum triggering thresholds?

<details><summary>Answer</summary>

autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor x reltuples). Default: 50 + 0.2 x row count.

```sql
ALTER TABLE t SET (autovacuum_vacuum_scale_factor = 0.01);
```

**Trap:** Scale factor is dangerous for large tables — on 100M rows, 0.2 means 20M dead tuples before vacuum kicks in. Use 0.01 or fixed thresholds.

</details>

8. **Q:** What does pg_stat_user_tables.n_dead_tup tell you?

<details><summary>Answer</summary>

Estimated number of dead tuples. Used by autovacuum to decide if vacuum is needed. **It's an estimate** — updated lazily. If it stays high, autovacuum is not keeping up.

**Follow-up:** How do you detect autovacuum falling behind? n_dead_tup keeps growing, autovacuum_count doesn't increase, or VACUUM runs continuously.

</details>

9. **Q:** What is the difference between SnapBuild and a normal MVCC snapshot?

<details><summary>Answer</summary>

- **MVCC snapshot:** Used by transactions for visibility (xmin/xmax ranges)
- **SnapBuild:** Used by logical replication to build initial snapshot of all visible tuples for the subscriber

</details>

10. **Q:** What is the process of freezing?

<details><summary>Answer</summary>

1. VACUUM scans heap pages
2. For tuples with xmin older than freeze_min_age (default: 50M transactions), set a hint bit or replace xmin with FrozenTransactionId (2)
3. The tuple is now always visible

**Trap:** Freezing does not remove the tuple — it makes it permanently visible so wraparound doesn't affect it.

</details>

11. **Q:** What is age() of a transaction, and how do you monitor wraparound risk?

<details><summary>Answer</summary>

age() returns the number of XIDs since the given XID was used.

```sql
SELECT datname, age(datfrozenxid) FROM pg_database;
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relkind = 'r';
```

When age(datfrozenxid) approaches 2 billion, PostgreSQL auto-vacuums. If it hits 2 billion — shutdown and PANIC.

</details>

12. **Q:** What does autovacuum_freeze_max_age control?

<details><summary>Answer</summary>

Default: 200 million. When a table's pg_class.relfrozenxid is older than this, autovacuum forces a vacuum just to freeze old tuples.

**Trap:** Setting too high increases wraparound risk; too low causes frequent freeze vacuums.

</details>

13. **Q:** What happens during a freeze storm?

<details><summary>Answer</summary>

When many tables reach autovacuum_freeze_max_age simultaneously, autovacuum aggressively freezes all of them. Causes massive I/O, CPU spikes, replication lag. Prevented by: per-table freeze_max_age tuning, proactive VACUUM FREEZE during low traffic, monitoring age(relfrozenxid).

</details>

14. **Q:** What is tuple pruning?

<details><summary>Answer</summary>

Inline dead tuple removal performed during queries (not just VACUUM). When PostgreSQL scans a heap page and encounters dead tuples, it can reclaim space within the page immediately.

**Trap:** Pruning is opportunistic during query execution — not the same as vacuum.

</details>

15. **Q:** What are hint bits?

<details><summary>Answer</summary>

Bits in the tuple header (HEAP_XMIN_COMMITTED, HEAP_XMIN_INVALID, HEAP_XMAX_COMMITTED, HEAP_XMAX_INVALID) that cache visibility decisions. Setting hint bits avoids checking the CLOG for every tuple access.

**Trap:** Hint bits are not WAL-logged. If the server crashes, hint bits are lost. Setting wal_log_hints = on enables WAL logging for tools like pg_rewind.

</details>

16. **Q:** What is the effect of a long-running transaction on vacuum?

<details><summary>Answer</summary>

A long-running transaction holds an open snapshot. Vacuum cannot remove dead tuples visible to that snapshot. This causes table bloat and stalls XID wraparound prevention.

```sql
SELECT pid, now() - xact_start AS xact_duration
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
ORDER BY xact_start;
```

</details>

17. **Q:** What are pg_stat_all_tables.n_live_tup and n_dead_tup — how accurate are they?

<details><summary>Answer</summary>

Estimates updated by VACUUM, ANALYZE, and DDL. Not updated by every INSERT/UPDATE. Autovacuum uses them. For accuracy, run manual VACUUM/ANALYZE.

</details>

18. **Q:** What mechanism distinguishes an active transaction ID from a committed one?

<details><summary>Answer</summary>

**CLOG (Commit Log)** — stored in pg_xact/. It's a bit array indexed by XID storing status: in-progress, committed, aborted, sub-committed. Cached in shared memory.

</details>

19. **Q:** What is pg_multixact and when is it used?

<details><summary>Answer</summary>

Multiple transaction IDs. Used when more than one transaction holds a shared lock on a row (FOR SHARE / FOR KEY SHARE). A single XMAX slot can't represent multiple locker XIDs, so a MultiXactId is stored.

</details>

20. **Q:** What does vacuum_defer_cleanup_age do?

<details><summary>Answer</summary>

Defers cleanup by N XIDs after a transaction completes. Used in streaming replication to prevent standby conflicts. Rarely used today — hot_standby_feedback is preferred.

</details>

21. **Q:** How does ANALYZE differ from VACUUM?

<details><summary>Answer</summary>

| Command | Purpose | Impact |
|---------|---------|--------|
| VACUUM | Removes dead tuples, freezes XIDs | Frees space, prevents wraparound |
| ANALYZE | Updates statistics for query planner | Better query plans |

ANALYZE does not remove dead tuples. VACUUM does not update stats (unless VACUUM ANALYZE).

</details>

22. **Q:** What is the difference between autovacuum_vacuum_cost_delay and autovacuum_vacuum_cost_limit?

<details><summary>Answer</summary>

Vacuum uses cost-based I/O throttle:
- vacuum_cost_limit: Total cost units before sleeping (default: 200)
- vacuum_cost_delay: Milliseconds to sleep when limit reached (default: 20 for autovacuum, 0 for manual)

**Trap:** Setting autovacuum_vacuum_cost_delay = 0 means no throttling — aggressive but risky.

</details>

23. **Q:** When would you manually run VACUUM instead of relying on autovacuum?

<details><summary>Answer</summary>

- After bulk loads
- Before a planned maintenance window for aggressive freeze
- When autovacuum is falling behind (n_dead_tup keeps growing)
- After large DELETE batches
- When wraparound age is approaching alert threshold

</details>

24. **Q:** What are all-visible pages and the visibility map?

<details><summary>Answer</summary>

The visibility map is a bitmap alongside the table, marking pages where all tuples are visible. Used by:
- **Index-only scans:** Skip heap fetches for fully-visible pages
- **Vacuum:** Skip fully-visible pages

If corrupted, index-only scans fall back to heap fetches.

</details>

25. **Q:** What is the freeze map introduced in PG 14?

<details><summary>Answer</summary>

PG 14 added a freeze map tracking which pages have all tuples frozen. Makes VACUUM FREEZE faster — skips pages already fully frozen, avoiding full-table scans on anti-wraparound vacuums.

**Follow-up:** Before PG 14, anti-wraparound vacuums scanned the entire table every time. The freeze map is a significant improvement.

</details>

---

## 3. Rapid-Fire: Data Types & SQL (25 questions)

1. **Q:** What is the difference between TIMESTAMP and TIMESTAMPTZ?

<details><summary>Answer</summary>

TIMESTAMP stores date/time with no timezone info. TIMESTAMPTZ stores internally as UTC and converts to session timezone on display.

```sql
SET timezone = 'America/New_York';
SELECT '2025-01-15 10:00:00+00'::timestamptz;
```

**Trap:** Always use TIMESTAMPTZ for multi-timezone applications.

</details>

2. **Q:** How does NUMERIC differ from FLOAT/DOUBLE PRECISION?

<details><summary>Answer</summary>

| Type | Storage | Precision | Use case |
|------|---------|-----------|----------|
| NUMERIC(p,s) | Variable | Exact | Money, accounting |
| REAL/FLOAT4 | 4 bytes | ~6 digits | Scientific |
| DOUBLE PRECISION/FLOAT8 | 8 bytes | ~15 digits | High-precision |

**Trap:** Using FLOAT for monetary values leads to rounding errors. Use NUMERIC.

</details>

3. **Q:** What are JSONB operators — @>, ?, ?|, ?&, ->, ->>?

<details><summary>Answer</summary>

- @>: Contains (left contains right)
- ?: Does key exist
- ?|: Any of these keys exist
- ?&: All of these keys exist
- ->: Get JSON field (returns JSON)
- ->>: Get JSON field as text

```sql
SELECT '{"a": {"b": [1,2]}}'::jsonb @> '{"a": {"b": [1]}}'::jsonb;
```

</details>

4. **Q:** When does JSONB indexing with GIN help?

<details><summary>Answer</summary>

GIN indexes on JSONB columns accelerate @>, ?, ?|, ?& operators.

```sql
CREATE INDEX idx ON t USING GIN (data jsonb_path_ops);
```

jsonb_path_ops creates smaller, faster indexes for @> but doesn't support ? operators.

</details>

5. **Q:** What are tsvector and tsquery?

<details><summary>Answer</summary>

- tsvector: Document representation — stores lexemes (normalized words) with positions
- tsquery: Query representation — lexemes combined with & (AND), | (OR), ! (NOT)

```sql
SELECT to_tsvector('english', 'The quick brown fox')
       @@ to_tsquery('english', 'quick & fox');
```

</details>

6. **Q:** How do you build a full-text search index?

<details><summary>Answer</summary>

```sql
ALTER TABLE articles ADD COLUMN fts tsvector
  GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;
CREATE INDEX articles_fts_idx ON articles USING GIN (fts);
```

Or functional index: CREATE INDEX ON articles USING GIN (to_tsvector('english', title || ' ' || body));

</details>

7. **Q:** What is a recursive CTE and what is the UNION ALL requirement?

<details><summary>Answer</summary>

Recursive CTEs reference themselves. They require UNION ALL between anchor and recursive member.

```sql
WITH RECURSIVE org_tree AS (
  SELECT id, name, parent_id, 1 AS depth
  FROM employees WHERE parent_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.parent_id, t.depth + 1
  FROM employees e JOIN org_tree t ON e.parent_id = t.id
)
SELECT * FROM org_tree;
```

**Trap:** Without UNION ALL, PostgreSQL tries to deduplicate, breaking recursion.

</details>

8. **Q:** How do window functions differ from GROUP BY?

<details><summary>Answer</summary>

GROUP BY collapses rows — one output row per group. Window functions compute across rows related to current row without collapsing.

```sql
SELECT department_id, employee_name, salary,
       RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank
FROM employees;
```

</details>

9. **Q:** What does DISTINCT ON do?

<details><summary>Answer</summary>

Returns the first row per unique value of the expression(s), ordered by ORDER BY:

```sql
SELECT DISTINCT ON (department_id) id, name, salary
FROM employees
ORDER BY department_id, salary DESC;
```

Without proper ORDER BY, which row is returned is non-deterministic.

</details>

10. **Q:** What does LATERAL do in a join?

<details><summary>Answer</summary>

LATERAL allows a subquery in FROM to reference columns from preceding FROM items:

```sql
SELECT d.name, top_emp.name, top_emp.salary
FROM departments d
LEFT JOIN LATERAL (
  SELECT * FROM employees
  WHERE department_id = d.id
  ORDER BY salary DESC
  LIMIT 3
) top_emp ON true;
```

**Trap:** Without LATERAL, the subquery cannot reference d.id.

</details>

11. **Q:** What are GROUPING SETS, CUBE, and ROLLUP?

<details><summary>Answer</summary>

Extensions of GROUP BY for multiple grouping levels:

```sql
SELECT year, month, day, SUM(sales)
FROM orders
GROUP BY ROLLUP (year, month, day);

SELECT year, month, SUM(sales)
FROM orders
GROUP BY CUBE (year, month);

SELECT year, month, SUM(sales)
FROM orders
GROUP BY GROUPING SETS ((year), (month), ());
```

</details>

12. **Q:** What is the RETURNING clause?

<details><summary>Answer</summary>

Returns data from modified rows after INSERT, UPDATE, DELETE, or MERGE:

```sql
DELETE FROM orders WHERE id = 42 RETURNING *;
INSERT INTO users (name) VALUES ('Alice') RETURNING id, created_at;
```

Essential for getting auto-generated values without a second query.

</details>

13. **Q:** What is the difference between ARRAY_AGG and UNNEST?

<details><summary>Answer</summary>

- ARRAY_AGG: Collects values into an array (aggregate)
- UNNEST: Expands an array into rows (set-returning function)

```sql
SELECT ARRAY_AGG(name) FROM employees WHERE department_id = 1;
SELECT UNNEST(ARRAY['a','b','c']);
```

</details>

14. **Q:** What are range types and when would you use them?

<details><summary>Answer</summary>

Types like INT4RANGE, TSRANGE, DATERANGE represent ranges:

```sql
CREATE TABLE bookings (
  room_id INT,
  during TSRANGE,
  EXCLUDE USING GIST (room_id WITH =, during WITH &&)
);
```

**Follow-up:** Exclusion constraint prevents overlapping bookings. GiST supports the && operator.

</details>

15. **Q:** What does pg_typeof() do?

<details><summary>Answer</summary>

Returns the data type of any expression:

```sql
SELECT pg_typeof(ARRAY[1,2,3]); -- integer[]
```

</details>

16. **Q:** What is the difference between VARCHAR(n) and TEXT?

<details><summary>Answer</summary>

No performance difference. Both stored identically. VARCHAR(n) adds a length check constraint. TEXT has no limit. Convention: TEXT for unbounded, VARCHAR(n) only when DB-level length validation is required.

</details>

17. **Q:** What is the difference between CHAR, VARCHAR, and TEXT?

<details><summary>Answer</summary>

- CHAR(n): Fixed-length, blank-padded. Wasteful.
- VARCHAR(n): Variable-length with limit.
- TEXT: Variable-length, unlimited.

Performance: identical. Storage: variable-length in all cases.

**Trap:** CHAR(n) is not faster than VARCHAR(n) for fixed-length data — they perform the same.

</details>

18. **Q:** What are the SERIAL types and why is IDENTITY preferred in PG 10+?

<details><summary>Answer</summary>

SERIAL, BIGSERIAL create auto-incrementing columns using sequences. GENERATED AS IDENTITY (PG 10+) is preferred:

```sql
CREATE TABLE t (id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY);
```

IDENTITY is SQL-standard, cannot be overridden (unless OVERRIDING SYSTEM VALUE), and doesn't increase the sequence on manual inserts.

</details>

19. **Q:** What are the network data types in PostgreSQL?

<details><summary>Answer</summary>

- INET: IPv4/IPv6 addresses (includes netmask)
- CIDR: Network blocks (requires non-zero host bits)
- MACADDR: MAC addresses

```sql
SELECT '192.168.1.0/24'::inet;
SELECT '08:00:2b:01:02:03'::macaddr;
```

</details>

20. **Q:** What is the hstore extension?

<details><summary>Answer</summary>

hstore stores key-value pairs in a single column (pre-JSONB). Supports GIN/GiST indexing. Largely superseded by JSONB, but more compact for simple key-value data.

</details>

21. **Q:** What is the difference between ANY and ALL with arrays?

<details><summary>Answer</summary>

```sql
-- ANY: true if comparison is true for ANY element
SELECT * FROM t WHERE id = ANY(ARRAY[1,2,3]);

-- ALL: true if comparison is true for ALL elements
SELECT * FROM t WHERE id > ALL(ARRAY[1,2,3]);
```

</details>

22. **Q:** What is MERGE (UPSERT) in PostgreSQL?

<details><summary>Answer</summary>

MERGE (PG 15+) performs INSERT, UPDATE, or DELETE based on a source:

```sql
MERGE INTO products p
USING staging s ON p.id = s.id
WHEN MATCHED THEN UPDATE SET name = s.name, price = s.price
WHEN NOT MATCHED THEN INSERT (id, name, price) VALUES (s.id, s.name, s.price);
```

Before PG 15, use INSERT ... ON CONFLICT DO UPDATE.

</details>

23. **Q:** What is the difference between EXCEPT and NOT IN?

<details><summary>Answer</summary>

- EXCEPT: Set operation that returns distinct rows from first query not in second
- NOT IN: Can return unexpected results with NULLs (NULL = NULL is NULL, not TRUE, so no rows returned)

```sql
-- Dangerous with NULLs:
SELECT * FROM a WHERE id NOT IN (SELECT id FROM b); -- returns 0 rows if b.id has NULL

-- Safe:
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE a.id = b.id);
```

**Trap:** NOT IN with NULL subquery returns zero rows — use NOT EXISTS.

</details>

24. **Q:** What does the OVERLAPS operator do?

<details><summary>Answer</summary>

Tests if two date/time ranges overlap:

```sql
SELECT (DATE '2025-01-01', DATE '2025-01-10') OVERLAPS
       (DATE '2025-01-05', DATE '2025-01-15'); -- true
```

</details>

25. **Q:** What are generated columns?

<details><summary>Answer</summary>

STORED generated columns compute values from other columns:

```sql
CREATE TABLE t (
  price NUMERIC,
  quantity INT,
  total NUMERIC GENERATED ALWAYS AS (price * quantity) STORED
);
```

VIRTUAL generated columns are not yet implemented in PostgreSQL.

</details>

---

## 4. Rapid-Fire: Indexing (25 questions)

1. **Q:** What index types does PostgreSQL support?

<details><summary>Answer</summary>

B+Tree (default), GiST, GIN, BRIN, SP-GiST, Hash. Each is optimized for different operators and data types.

</details>

2. **Q:** How does a B+Tree index work internally?

<details><summary>Answer</summary>

A balanced tree: internal nodes guide search, leaf nodes contain index entries (key + TID pointer). Leaf pages are linked in a doubly-linked list for range scans. Fan-out is high (hundreds of entries per page).

</details>

3. **Q:** What is a composite index and what is the left-prefix rule?

<details><summary>Answer</summary>

An index on multiple columns:

```sql
CREATE INDEX idx ON t (a, b, c);
```

Left-prefix rule: the index can be used for queries filtering on `a`, `(a, b)`, or `(a, b, c)`. Cannot be used for queries filtering only on `b` or `c`.

</details>

4. **Q:** What column order should you choose for a composite index?

<details><summary>Answer</summary>

Order by: **equality conditions first, then range columns, then sort columns, then INCLUDE**.

```sql
-- Query: WHERE org_id = 42 AND status = 'active' ORDER BY created_at DESC
CREATE INDEX idx ON t (org_id, status, created_at DESC);
```

Equality columns first allow the index to narrow to a specific subtree. Range/sort columns last.

</details>

5. **Q:** What is a partial index?

<details><summary>Answer</summary>

Indexes only rows matching a WHERE condition:

```sql
CREATE INDEX idx_active_orders ON orders (created_at) WHERE status = 'active';
```

Smaller index, faster writes, faster reads for queries that match the condition.

**Trap:** The query planner only uses the partial index if the query's WHERE clause matches the index's WHERE condition.

</details>

6. **Q:** What is a covering index (INCLUDE)?

<details><summary>Answer</summary>

Includes non-key columns for index-only scans:

```sql
CREATE INDEX idx ON t (a, b) INCLUDE (c, d);
```

The index can serve queries selecting c, d without touching the heap. INCLUDE columns are stored only in leaf pages (not in internal nodes).

**Follow-up:** Why not just put c, d in the key columns? INCLUDE columns don't affect ordering, don't count toward index key size limits, and keep the index narrower.

</details>

7. **Q:** What is an expression index?

<details><summary>Answer</summary>

Index on the result of a function or expression:

```sql
CREATE INDEX idx_lower_name ON users (LOWER(name));
```

Used for queries like: SELECT * FROM users WHERE LOWER(name) = 'alice';

</details>

8. **Q:** How do you create an index without blocking writes?

<details><summary>Answer</summary>

CREATE INDEX CONCURRENTLY:

```sql
CREATE INDEX CONCURRENTLY idx ON t (col);
```

**Trap:** It takes longer, consumes more resources, and can fail (leaving an invalid index). Requires two table scans. Must clean up failed indexes manually.

</details>

9. **Q:** What is a GiST index and when would you use it?

<details><summary>Answer</summary>

Generalized Search Tree. Supports: full-text search, geometric data, range types, fuzzy matching (pg_trgm), exclusion constraints. Uses: overlap (&&), contains (@>), nearest-neighbor (<->).

</details>

10. **Q:** What is a GIN index and when would you use it?

<details><summary>Answer</summary>

Generalized Inverted Index. Stores multiple entries per row. Uses: JSONB, arrays, full-text search (tsvector). Supports @>, ?, ?|, ?&, @@ operators.

**Follow-up:** GIN is slower to build/update than B+Tree but has fast reads. Consider fastupdate = on for write-heavy tables.

</details>

11. **Q:** What is a BRIN index and when is it appropriate?

<details><summary>Answer</summary>

Block Range INdex. Stores min/max values per page range. Best for append-only, physically ordered data (time-series, logs). Extremely small (100-1000x smaller than B+Tree).

```sql
CREATE INDEX idx_brin ON events USING BRIN (created_at) WITH (pages_per_range = 32);
```

**Trap:** BRIN is useless if data is randomly ordered in the heap. It relies on physical correlation.

</details>

12. **Q:** What is an SP-GiST index?

<details><summary>Answer</summary>

Space-Partitioned GiST. Best for non-overlapping structures: k-d trees (multidimensional), quad-trees (points), radix trees (strings, IP addresses). Supports queries on network types (<<, >>) and text (LIKE with prefix).

</details>

13. **Q:** What is SKIP SCAN (PG 15+)?

<details><summary>Answer</summary>

PostgreSQL 15+ can skip non-leading columns in a composite index to find distinct values efficiently:

```sql
-- Index on (org_id, status)
-- Query: SELECT DISTINCT status FROM orders;
-- PG15+ can use SKIP SCAN: scan org_id groups, pick first status per group
```

Before PG 15, this required a full index scan or a separate query per group.

</details>

14. **Q:** What are operator classes?

<details><summary>Answer</summary>

Define which operators an index family supports. For B+Tree:

```sql
CREATE INDEX idx ON t (col text_pattern_ops);
```

text_pattern_ops enables LIKE 'prefix%' queries without a separate index. Default operator class uses database collation for equality and ordering.

</details>

15. **Q:** How does CREATE INDEX CONCURRENTLY differ from a regular CREATE INDEX?

<details><summary>Answer</summary>

| Aspect | CREATE INDEX | CREATE INDEX CONCURRENTLY |
|--------|-------------|--------------------------|
| Lock | ACCESS EXCLUSIVE (blocks all) | SHARE UPDATE EXCLUSIVE (allows reads/writes) |
| Phases | Single scan | Two scans + wait |
| Failure | Rolled back | Leaves invalid index |
| Duration | Faster | 2-3x slower |

</details>

16. **Q:** What is an index-only scan and when can it be used?

<details><summary>Answer</summary>

When all required columns are in the index (either key or INCLUDE columns), PostgreSQL can read directly from the index without visiting the heap. Requires the visibility map to confirm tuples are all-visible.

**Trap:** If the visibility map shows pages that aren't fully visible, PostgreSQL still fetches from heap, negating the benefit. Frequent VACUUM helps maintain the visibility map.

</details>

17. **Q:** What is index bloat and how do you detect it?

<details><summary>Answer</summary>

Index bloat occurs when index pages contain many dead tuples or free space. Detection:

```sql
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexname::regclass)) as size,
       avg_leaf_density
FROM pg_stat_user_indexes
JOIN ...; -- or use pgstattuple extension
```

Remediation: REINDEX INDEX, REINDEX TABLE, or pg_repack.

</details>

18. **Q:** What is the difference between REINDEX and CLUSTER?

<details><summary>Answer</summary>

- REINDEX: Rebuilds index, doesn't change heap order
- CLUSTER: Rewrites table in index order (requires ACCESS EXCLUSIVE) and rebuilds all indexes

CLUSTER physically reorders the heap. REINDEX only rebuilds index structure.

</details>

19. **Q:** What is pg_repack and when would you use it?

<details><summary>Answer</summary>

Extension that rebuilds tables and indexes online (no ACCESS EXCLUSIVE). Uses a trigger-based approach. Preferred over VACUUM FULL for zero-downtime bloat removal.

</details>

20. **Q:** How do unused indexes hurt performance?

<details><summary>Answer</summary>

- Slows writes (every INSERT/UPDATE/DELETE must update the index)
- Wastes disk space
- Wastes shared buffer memory
- Slows vacuum (must scan all indexes)

Detection:

```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

</details>

21. **Q:** What is a B+Tree index on a boolean column good for?

<details><summary>Answer</summary>

Generally not useful (only 2 values → poor selectivity). Exceptions:
- Partial index filtering on status
- Combined with other columns in a composite index
- Highly skewed data (5% active, 95% inactive — index on active=true is useful)

**Trap:** A standalone index on a boolean column is almost always wasteful.

</details>

22. **Q:** How does NULL behavior work in B+Tree indexes?

<details><summary>Answer</summary>

By default, NULLs are sorted after all values (NULLS LAST). In a UNIQUE index, multiple NULLs are allowed (NULL != NULL). For queries with IS NULL, a B+Tree index can be used if the index is created with NULLS FIRST or if the query uses IS NULL.

**Follow-up:** Partial indexes can help with IS NULL queries: CREATE INDEX idx_null ON t (col) WHERE col IS NULL;

</details>

23. **Q:** What is a bloom filter index?

<details><summary>Answer</summary>

PostgreSQL supports bloom indexes via the bloom extension. A space-efficient probabilistic index for multi-column equality queries. Not commonly used — suitable for queries with many columns in WHERE but any subset.

</details>

24. **Q:** How do you decide between GiST and GIN for full-text search?

<details><summary>Answer</summary>

| Aspect | GiST | GIN |
|--------|------|-----|
| Build time | Faster | Slower |
| Index size | Larger | Smaller |
| Search speed | Slower | Faster |
| Update overhead | Lower | Higher |

GIN is generally preferred for full-text search on read-heavy workloads. GiST for write-heavy or when you need smaller build time.

</details>

25. **Q:** What is the P_BRIN (parallel BRIN) feature?

<details><summary>Answer</summary>

PostgreSQL 15+ can build BRIN indexes in parallel. For very large append-only tables, this significantly speeds up initial index creation.

</details>

---

## 5. Rapid-Fire: EXPLAIN & Query Optimization (20 questions)

1. **Q:** What is the difference between Seq Scan, Index Scan, and Index Only Scan?

<details><summary>Answer</summary>

- **Seq Scan:** Full sequential read of the entire table (O(n)). Used when retrieving large portions or no suitable index.
- **Index Scan:** Reads index to find matching TIDs, then fetches heap pages (O(log n) + random I/O per row).
- **Index Only Scan:** Reads only the index — no heap fetch needed. Requires all columns in the index and visibility map to show all-visible pages.

</details>

2. **Q:** What is a Bitmap Scan?

<details><summary>Answer</summary>

Two-phase scan: 1) Index scan builds a bitmap of matching page locations 2) Sorts pages by disk location and fetches in order. Efficient when returning a moderate number of rows (many matching TIDs but scattered across pages).

- **Bitmap Heap Scan:** Reads from heap using the bitmap
- **Bitmap Index Scan:** Builds the bitmap from index

**Trap:** Bitmap scans are often more efficient than Index Scans for queries returning >2-5% of rows because they convert random I/O to sequential I/O.

</details>

3. **Q:** What are the three join types and when is each used?

<details><summary>Answer</summary>

- **Nested Loop:** For each row in outer, scan inner. Best when outer is small and inner has an index (O(outer * log(inner))).
- **Hash Join:** Build hash table on one side, probe with other. Best when joining large, unsorted, or non-indexed datasets.
- **Merge Join:** Sort both sides, then merge. Best when both inputs are already sorted (e.g., from indexes) and join is on equality.

</details>

4. **Q:** How do you read an EXPLAIN plan top to bottom?

<details><summary>Answer</summary>

Read from the **innermost** node outward. Each node shows:
- **cost=startup..total:** Startup cost before first row, total cost to completion
- **rows:** Estimated row count
- **width:** Average row width in bytes
- **actual time=... (with ANALYZE):** Actual timing

**Trap:** Costs are in arbitrary units — focus on ratios and row estimate accuracy, not absolute cost values.

</details>

5. **Q:** What does rows estimate mismatch indicate?

<details><summary>Answer</summary>

If actual rows >> estimated rows (e.g., estimate=10, actual=1,000,000):
- Stale statistics → run ANALYZE
- Correlated columns → need extended statistics
- Complex predicates → planner misestimates selectivity

This is the #1 cause of bad query plans.

</details>

6. **Q:** What does "loops=N" mean in EXPLAIN ANALYZE?

<details><summary>Answer</summary>

How many times the plan node was executed. If loops > 1 with high actual rows, it may indicate a nested loop where the inner plan is executed repeatedly. Multiply actual time by loops for total time.

**Trap:** Always check loops — a node with 10ms per execution but 1000 loops is contributing 10s total.

</details>

7. **Q:** What is auto_explain?

<details><summary>Answer</summary>

Extension that logs query plans automatically:

```sql
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 1000; -- log queries over 1 second
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
```

Essential for production monitoring — captures plans of slow queries as they happen.

</details>

8. **Q:** What does pg_stat_statements track?

<details><summary>Answer</summary>

Query execution statistics by normalized query:

```sql
SELECT queryid, query, calls, total_exec_time, mean_exec_time,
       rows, shared_blks_hit, shared_blks_read
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Fields: calls, total_exec_time, mean_exec_time, rows, shared_blks_hit/read, temp_blks_written.

</details>

9. **Q:** What is a Sort node, and when does it spill to disk?

<details><summary>Answer</summary>

Sort node orders result rows. If it exceeds work_mem, it spills to temp files (disk). Detected via EXPLAIN ANALYZE showing "Sort Method: external merge Disk".

```sql
-- Increase work_mem for the session if sorts are spilling
SET work_mem = '64MB';
```

Disk sorts are ~100x slower than in-memory sorts.

</details>

10. **Q:** What is parallel query and when does it activate?

<details><summary>Answer</summary>

PostgreSQL can use multiple workers for scans, joins, and aggregates. Activation requires:
- Table larger than min_parallel_table_scan_size
- Parallel-safe functions/operators
- max_parallel_workers_per_gather > 0

Plan shows "Gather" or "Gather Merge" nodes. Not all queries benefit — small tables or simple point queries are faster without parallel.

</details>

11. **Q:** How do extended statistics help with correlated columns?

<details><summary>Answer</summary>

Default statistics assume columns are independent. With correlation:

```sql
CREATE STATISTICS s (dependencies) ON org_id, status FROM orders;
CREATE STATISTICS s (ndistinct) ON category_id, region FROM inventory;
-- Types: dependencies (functional), ndistinct (distinct combinations), mcv (most common combinations)
```

ANALYZE then collects multivariate statistics for better cardinality estimates.

</details>

12. **Q:** What is a CTE Scan and why can CTEs be optimization barriers?

<details><summary>Answer</summary>

Before PG 12, CTEs were always materialized (optimization fence). PG 12+: MATERIALIZED / NOT MATERIALIZED options:

```sql
WITH t AS NOT MATERIALIZED (
  SELECT * FROM big_table WHERE condition
)
SELECT * FROM t JOIN other_table ...;
```

**Trap:** Without NOT MATERIALIZED, the CTE is evaluated independently and cannot be pushed into the join — causing a Seq Scan on the CTE result.

</details>

13. **Q:** How do you identify a query that's TID (tuple ID) scanning?

<details><summary>Answer</summary>

A Tid Scan node appears when the query uses ctid directly:

```sql
SELECT * FROM t WHERE ctid = '(0,1)';
```

Rare in application code but used by some extensions and replication tools.

</details>

14. **Q:** What is a Subquery Scan and when does it appear?

<details><summary>Answer</summary>

A Subquery Scan (also called "Subquery Scan on <subquery>") wraps a subquery's output. Often a sign the subquery could be rewritten as a join or LATERAL.

**Trap:** Subquery Scan that removes rows (filter) vs. one that passes through — check the filter condition.

</details>

15. **Q:** How do you detect parameterized queries causing plan instability?

<details><summary>Answer</summary>

Prepared statements with bind parameters can cause generic plans (PG decides once) vs. custom plans (per execution). If data is skewed, a plan optimal for 'active' might be terrible for 'archived'.

Fix: Enable adaptive planning or use pg_hint_plan for specific queries.

</details>

16. **Q:** What is the impact of random_page_cost on plan choice?

<details><summary>Answer</summary>

Default: 4 (HDD era). For SSDs/NVMe, set to 1.0–1.5. For cloud storage (EBS with gp3), set to 1.0–2.0. Lower values make index scans more attractive vs. sequential scans.

```sql
ALTER SYSTEM SET random_page_cost = 1.5;
```

**Trap:** Leaving random_page_cost at 4 on SSD hardware causes PostgreSQL to prefer Seq Scans when Index Scans would be faster.

</details>

17. **Q:** What is effective_cache_size and how does it affect plans?

<details><summary>Answer</summary>

Tells the planner how much memory the OS has for page caching (shared_buffers + OS cache). Set to 50-75% of total RAM. Higher values make index scans look more attractive (the planner assumes pages are cached).

**Trap:** effective_cache_size does not allocate memory — it's a planning hint only.

</details>

18. **Q:** How do you identify a query that's waiting on I/O vs CPU?

<details><summary>Answer</summary>

```sql
SELECT pg_stat_get_backend_pid(s.backendid),
       pg_stat_get_backend_activity(s.backendid),
       pg_stat_get_backend_wait_event_type(s.backendid),
       pg_stat_get_backend_wait_event(s.backendid)
FROM (SELECT generate_series(1, pg_stat_get_backend_count()) AS backendid) s;
```

- I/O wait: wait_event_type = IO, events like DataFileRead, WALWrite
- CPU: state = 'active', wait_event_type = NULL
- Lock: wait_event_type = Lock

</details>

19. **Q:** What is the difference between FILTER and WHERE in EXPLAIN?

<details><summary>Answer</summary>

- WHERE: Filter applied at the scan level (e.g., Seq Scan filter)
- FILTER: Filter applied to a join result or subquery output

In plans: "Filter:" is per-row check, "Index Cond:" is index-driven filtering.

</details>

20. **Q:** What is JIT compilation and when does it help?

<details><summary>Answer</summary>

PostgreSQL can JIT-compile expressions, tuple deforming, and quals using LLVM. Enabled by default in PG 12+. Helps for CPU-bound, long-running queries with complex expressions. Not helpful for short OLTP queries.

```sql
SET jit = off; -- for OLTP workloads
```

**Trap:** JIT overhead can slow down simple queries — consider disabling for OLTP.

</details>

---

## 6. Rapid-Fire: Locking & Concurrency (20 questions)

1. **Q:** What are the four row-level lock modes?

<details><summary>Answer</summary>

- **FOR UPDATE:** Most restrictive. Prevents other transactions from locking, updating, or deleting the row.
- **FOR NO KEY UPDATE:** Like FOR UPDATE but doesn't block FOR KEY SHARE (used when updating non-key columns).
- **FOR SHARE:** Shared lock — prevents updates/deletes but allows other FOR SHARE.
- **FOR KEY SHARE:** Lightest — blocks only key-changing updates.

</details>

2. **Q:** What are the eight table-level lock modes in PostgreSQL?

<details><summary>Answer</summary>

1. ACCESS SHARE (SELECT)
2. ROW SHARE (SELECT FOR UPDATE/SHARE)
3. ROW EXCLUSIVE (INSERT, UPDATE, DELETE)
4. SHARE UPDATE EXCLUSIVE (VACUUM, CREATE INDEX CONCURRENTLY)
5. SHARE (CREATE INDEX)
6. SHARE ROW EXCLUSIVE
7. EXCLUSIVE
8. ACCESS EXCLUSIVE (ALTER TABLE, DROP TABLE, VACUUM FULL, CREATE INDEX regular)

**Trap:** ACCESS EXCLUSIVE conflicts with ALL lock modes — it blocks even SELECT.

</details>

3. **Q:** How does deadlock detection work?

<details><summary>Answer</summary>

PostgreSQL detects deadlocks using a timeout-based wait-for graph check. Every `deadlock_timeout` (default: 1s), the deadlock detector runs. If a cycle is found, one transaction is aborted:

```
ERROR: deadlock detected
DETAIL: Process 1 waits for ShareLock on transaction 100; blocked by process 2.
Process 2 waits for ShareLock on transaction 99; blocked by process 1.
```

**Trap:** deadlock_timeout is not how long the deadlock takes — it's how often the detector runs. Reduce detection latency vs. more CPU overhead.

</details>

4. **Q:** What does SKIP LOCKED do?

<details><summary>Answer</summary>

SKIP LOCKED skips rows that are already locked and returns only unlocked rows:

```sql
SELECT id, payload FROM job_queue
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

Essential for job queue workers — prevents contention and doesn't block.

**Follow-up:** What happens without SKIP LOCKED? Workers block on the same rows and compete for locks, reducing throughput.

</details>

5. **Q:** What does NOWAIT do?

<details><summary>Answer</summary>

NOWAIT returns an error immediately if the row is locked instead of waiting:

```sql
SELECT * FROM t WHERE id = 42 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "t"
```

Useful when your application cannot tolerate waiting for locks.

</details>

6. **Q:** What are advisory locks and when would you use them?

<details><summary>Answer</summary>

Application-defined locks not tied to rows/tables:

```sql
-- Session-level (must release explicitly)
SELECT pg_advisory_lock(42);
SELECT pg_advisory_unlock(42);

-- Transaction-level (released on commit/rollback)
SELECT pg_advisory_xact_lock(42);

-- Try (non-blocking)
SELECT pg_try_advisory_lock(42);
```

Use cases: distributed job coordination, custom locking logic, preventing concurrent execution.

</details>

7. **Q:** How do you monitor lock contention in PostgreSQL?

<details><summary>Answer</summary>

```sql
-- Blocked queries
SELECT blocked.pid, blocked.query, blocker.pid, blocker.query
FROM pg_locks blocked
JOIN pg_locks blocker ON blocked.transactionid = blocker.transactionid
WHERE NOT blocked.granted;

-- Using pg_blocking_pids (PG 9.6+)
SELECT pid, pg_blocking_pids(pid), query, state
FROM pg_stat_activity
WHERE pg_blocking_pids(pid) <> '{}';
```

</details>

8. **Q:** What is the difference between relation-level and row-level locks in pg_locks?

<details><summary>Answer</summary>

Relation locks show as locktype = 'relation', row locks as locktype = 'transactionid'. Row-level locks in PostgreSQL are stored in the transaction's XID, not in a separate lock structure — they show as transaction ID locks in pg_locks.

**Trap:** FOR UPDATE row locks appear as transactionid locks, not as 'row' type. This confuses many.

</details>

9. **Q:** What is predicate locking and SERIALIZABLE isolation?

<details><summary>Answer</summary>

SERIALIZABLE isolation uses predicate locks (SSI — Serializable Snapshot Isolation). It detects serialization anomalies by tracking read-write conflicts:

```sql
SET transaction_isolation = 'serializable';
```

**Trap:** SERIALIZABLE is expensive — many applications use REPEATABLE READ or READ COMMITTED. True serializability is only needed for financial reconciliation or similar.

</details>

10. **Q:** How does lock escalation work in PostgreSQL?

<details><summary>Answer</summary>

**PostgreSQL does not do lock escalation.** Unlike SQL Server or Oracle, PostgreSQL never escalates row locks to page/table locks. Each locked row keeps its own lock in memory. This is why bulk UPDATEs can exhaust lock memory.

</details>

11. **Q:** What are the lock types acquired by common DDL commands?

<details><summary>Answer</summary>

| Command | Lock mode | Blocks reads? | Blocks writes? |
|---------|-----------|:---:|:---:|
| SELECT | ACCESS SHARE | No | No |
| INSERT/UPDATE/DELETE | ROW EXCLUSIVE | No | No |
| CREATE INDEX | SHARE | Yes (no writes) | Yes |
| CREATE INDEX CONCURRENTLY | SHARE UPDATE EXCLUSIVE | No | No |
| ALTER TABLE ... ADD COLUMN | ACCESS EXCLUSIVE | Yes | Yes |
| VACUUM | SHARE UPDATE EXCLUSIVE | No | No |
| VACUUM FULL | ACCESS EXCLUSIVE | Yes | Yes |

</details>

12. **Q:** What is the difference between SHARE UPDATE EXCLUSIVE and SHARE?

<details><summary>Answer</summary>

SHARE UPDATE EXCLUSIVE (used by VACUUM, CREATE INDEX CONCURRENTLY): Allows concurrent reads + writes (ROW EXCLUSIVE is compatible). Only conflicts with other SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, and ACCESS EXCLUSIVE.

SHARE (used by regular CREATE INDEX): Blocks writes (conflicts with ROW EXCLUSIVE).

</details>

13. **Q:** How do you safely add a column without downtime?

<details><summary>Answer</summary>

```sql
-- Adding a nullable column with no default is instant (metadata only)
ALTER TABLE t ADD COLUMN new_col TEXT;

-- Adding with default requires a full table rewrite in PG 11+
-- PG 11+ optimizes: adding a non-null column with default is metadata-only
```

**Trap:** In PG 10 and earlier, ALTER TABLE ADD COLUMN DEFAULT rewrites the entire table — ACCESS EXCLUSIVE for the duration. Always check PG version.

</details>

14. **Q:** What is idle_in_transaction_session_timeout?

<details><summary>Answer</summary>

Terminates sessions that are idle in a transaction for longer than the timeout:

```sql
SET idle_in_transaction_session_timeout = '5min';
```

Prevents long-running idle-in-transaction connections from holding locks and blocking vacuum.

</details>

15. **Q:** What is lock_timeout and statement_timeout?

<details><summary>Answer</summary>

- lock_timeout: Maximum time spent waiting for a lock
- statement_timeout: Maximum time for any statement

```sql
SET lock_timeout = '10s';
SET statement_timeout = '30s';
```

**Trap:** Confusing them — statement_timeout cancels the query, lock_timeout only fires if waiting on a lock.

</details>

16. **Q:** What is the difference between READ COMMITTED and REPEATABLE READ in PostgreSQL?

<details><summary>Answer</summary>

- **READ COMMITTED:** Each statement sees rows committed before the statement started. Can see non-repeatable reads (same row queried twice gives different values).
- **REPEATABLE READ:** Transaction sees a snapshot taken at first query. Non-repeatable reads are prevented. Retry on serialization failure.

PostgreSQL's REPEATABLE READ is stronger than SQL standard — no phantom reads either (uses MVCC snapshots).

</details>

17. **Q:** What happens when you FOR UPDATE on a row that's already locked?

<details><summary>Answer</summary>

In READ COMMITTED: The statement waits for the lock. Once acquired, it re-reads the row to get the latest committed version — potentially seeing different data than expected.

In REPEATABLE READ: The statement still waits but may get a serialization failure if the lock-holding transaction committed a conflicting change.

**Trap:** This re-read behavior in READ COMMITTED can cause surprising results — a row that passed the WHERE filter initially may no longer match after re-reading.

</details>

18. **Q:** How do you implement a job queue with SKIP LOCKED?

<details><summary>Answer</summary>

```sql
-- Worker picks next available job atomically
WITH next_job AS (
  SELECT id FROM job_queue
  WHERE status = 'pending'
  ORDER BY priority DESC, created_at ASC
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE job_queue SET status = 'running', started_at = NOW()
WHERE id = (SELECT id FROM next_job)
RETURNING *;
```

**Trap:** Without SKIP LOCKED, all workers try to lock the same row, causing contention and lock waits.

</details>

19. **Q:** What is the difference between FOR UPDATE and FOR NO KEY UPDATE?

<details><summary>Answer</summary>

FOR UPDATE blocks all row-level locks including FOR KEY SHARE. FOR NO KEY UPDATE blocks FOR UPDATE, FOR NO KEY UPDATE, and FOR SHARE but does NOT block FOR KEY SHARE.

Use FOR NO KEY UPDATE when updating non-key columns — it allows concurrent foreign key checks to proceed.

</details>

20. **Q:** How many advisory locks can a backend hold?

<details><summary>Answer</summary>

Limited by max_locks_per_transaction (default: 64 per backend/transaction) for tracked locks, but PostgreSQL can allocate more dynamically. Practical limit is high — advisory locks use shared memory tracking.

**Trap:** Advisory locks are not automatically released on session end if the backend is killed (FORCE). Always use pg_advisory_unlock_all() or let them release naturally.

</details>

---

## 7. Rapid-Fire: Replication & HA (15 questions)

1. **Q:** How does streaming replication work?

<details><summary>Answer</summary>

1. Primary writes WAL records
2. WAL sender process streams them to standby's WAL receiver
3. Standby's WAL receiver writes to standby's pg_wal
4. Standby replays WAL (applying changes) — typically in recovery mode

```sql
-- Primary: wal_level = replica, max_wal_senders > 0
-- Standby: hot_standby = on, primary_conninfo = '...'
```

</details>

2. **Q:** What is the difference between synchronous and asynchronous replication?

<details><summary>Answer</summary>

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Durability | COMMIT waits for standby confirmation | COMMIT returns immediately |
| Data loss | Zero (if synchronous standby is up) | Small window (default: 0) |
| Latency | Higher | Lower |
| Performance impact | Transaction waits for network round-trip | None |

```sql
SET synchronous_standby_names = 'FIRST 1 (standby1, standby2)';
```

</details>

3. **Q:** What are replication slots?

<details><summary>Answer</summary>

Replication slots ensure the primary doesn't remove WAL segments until all standbys have received them:

- **Physical slots:** Track WAL consumption by physical standby
- **Logical slots:** Track logical replication state (decode WAL to logical changes)

**Trap:** Orphaned replication slots cause WAL to pile up indefinitely, filling pg_wal and crashing the primary. Monitor with:

```sql
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
FROM pg_replication_slots;
```

</details>

4. **Q:** What is the difference between logical and physical replication?

<details><summary>Answer</summary>

| Aspect | Physical | Logical |
|--------|----------|---------|
| Level | Block-level (identical copy) | Row-level (SQL operations) |
| Cross-version | Same major version | Yes (PG 10+ can replicate to PG 17) |
| Subset | Full database | Tables, selective columns |
| DDL | Replicated automatically | Not replicated |
| Bi-directional | No | Yes (with caution) |

</details>

5. **Q:** What is Patroni and how does it manage failover?

<details><summary>Answer</summary>

Patroni is a HA tool using distributed consensus (etcd, Consul, ZooKeeper):
1. Monitors primary health via PostgreSQL API
2. On primary failure, promotes best standby
3. Updates DCS (distributed configuration store) to point to new primary
4. Remaining standbys re-point to new primary

```yaml
scope: prod
etcd:
  host: 127.0.0.1:2379
postgresql:
  parameters:
    wal_level: replica
    max_wal_senders: 5
```

</details>

6. **Q:** How does PgBouncer pool connections?

<details><summary>Answer</summary>

PgBouncer supports three pooling modes:
- **Session pooling:** Map one client connection to one server connection for the session duration (most like direct PostgreSQL)
- **Transaction pooling:** Server connection is released after each transaction. Client MUST not use session-level features (prepared statements, temporary tables, LISTEN/NOTIFY)
- **Statement pooling:** Server connection released after each statement. Most restrictive.

**Trap:** Transaction pooling breaks applications using session-scoped features (Cursors, SET, LISTEN, temp tables). Validate compatibility before choosing.

</details>

7. **Q:** What is WAL archiving and why is it needed?

<details><summary>Answer</summary>

WAL archiving copies filled WAL segments to persistent storage. Required for:
- Point-In-Time Recovery (PITR)
- Rebuilding a failed standby
- Long-term backup

```sql
archive_mode = on
archive_command = 'pgbackrest --stanza=prod archive-push %p'
restore_command = 'pgbackrest --stanza=prod archive-get %f %p'
```

</details>

8. **Q:** How does replication lag grow and how do you monitor it?

<details><summary>Answer</summary>

```sql
-- On primary: check standby replay
SELECT pid, application_name, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
       (EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))*1000)::bigint AS lag_ms
FROM pg_stat_replication;
```

**Trap:** WAL bytes behind doesn't equal time lag — a 1GB lag with no transactions is fast to catch up; a 100MB lag with continuous writes may never catch up.

</details>

9. **Q:** What is hot_standby_feedback?

<details><summary>Answer</summary>

When enabled on standby, the standby sends its snapshot's oldest XID to the primary. The primary then avoids vacuuming rows visible to the standby's snapshot. Prevents "query canceled: snapshot too old" errors on the standby.

**Trap:** Can cause table bloat on the primary if a standby has long-running queries. Requires tuning.

</details>

10. **Q:** What happens during a failover?

<details><summary>Answer</summary>

1. Primary is detected as down
2. Best standby is selected (based on WAL position, replication lag)
3. Standby runs pg_ctl promote (or Patroni promotes)
4. Standby becomes the new primary — accepts writes
5. Other standbys re-point to the new primary
6. Applications reconnect (connection string or DNS update)
7. Old primary, if recoverable, becomes a standby of the new primary

</details>

11. **Q:** What is the difference between max_wal_senders and max_replication_slots?

<details><summary>Answer</summary>

- max_wal_senders: Maximum concurrent WAL sender processes (connections from standbys)
- max_replication_slots: Maximum replication slots (logical or physical)

max_wal_senders should be >= max_replication_slots. Each standby needs both a WAL sender and a slot.

</details>

12. **Q:** What is conflict resolution in logical replication?

<details><summary>Answer</summary>

By default: subscriber-side conflicts are handled by subscriber's behavior:
- INSERT conflicts: already exists → error
- UPDATE/DELETE conflicts: row doesn't exist → no action (data loss possible)

PG 15+ has better conflict detection. Monitor:

```sql
SELECT * FROM pg_stat_subscription_workers;
SELECT * FROM pg_replication_origin_status;
```

For production, implement idempotent subscriber logic or use pglogical with conflict handlers.

</details>

13. **Q:** What is cascading replication?

<details><summary>Answer</summary>

A standby replicating from another standby (not the primary). Reduces load on the primary for WAL shipping. Used in geo-distributed setups:

```
Primary → Standby (us-east) → Standby (us-west)
```

The intermediate standby must have wal_level = replica and max_wal_senders > 0.

</details>

14. **Q:** How do you perform a failover test safely?

<details><summary>Answer</summary>

1. Verify all standbys are in-sync (minimal lag)
2. Document current primary: SELECT pg_is_in_recovery(), pg_current_wal_lsn()
3. Gracefully switchover (controlled promotion) or simulate failure
4. Verify new primary accepts writes
5. Verify old primary can become standby
6. Test application connection strings

**Trap:** Never test failover during peak traffic or without verifying backup integrity first.

</details>

15. **Q:** What is timeline forking in PostgreSQL?

<details><summary>Answer</summary>

When a standby is promoted, it starts a new timeline. The old primary's WAL on a different timeline. Timelines allow the old primary to be recovered as a standby of the new timeline (via pg_rewind).

```sql
-- Check current timeline
SELECT timeline_id FROM pg_control_checkpoint();
```

</details>

---

## 8. Rapid-Fire: Performance & Operations (15 questions)

1. **Q:** How do you size shared_buffers?

<details><summary>Answer</summary>

Set to ~25% of total RAM. Beyond ~8-10 GB, diminishing returns — PostgreSQL needs OS cache too. For an 64 GB server: shared_buffers = 16 GB.

**Trap:** Setting shared_buffers > 40% of RAM can cause double buffering and performance regression.

</details>

2. **Q:** How do you tune work_mem?

<details><summary>Answer</summary>

work_mem is per-operation (sort, hash join, hash aggregate), per-query (multiple operations possible). Default: 4 MB. Too high: risk of OOM. Calculation:

```
RAM × 0.25 / (max_connections × max_parallel_workers_per_gather)
```

For 64 GB, 100 connections, 2 workers: (64 × 0.25 × 1024) / (100 × 2) ≈ 82 MB.

**Trap:** Setting work_mem too high (>256 MB) without considering concurrent sorts causes swapping.

</details>

3. **Q:** How do you tune checkpoint parameters?

<details><summary>Answer</summary>

```sql
-- Spread I/O over longer window
checkpoint_timeout = 15min        -- default: 5min
max_wal_size = 32GB               -- default: 1GB
min_wal_size = 4GB
checkpoint_completion_target = 0.9 -- spread writing over 90% of checkpoint interval
```

Higher max_wal_size reduces checkpoint frequency (fewer I/O spikes) but increases crash recovery time and disk usage.

**Trap:** checkpoint_completion_target > 0.9 can cause checkpoints never finishing before the next one starts.

</details>

4. **Q:** How do you tune autovacuum for a high-write table?

<details><summary>Answer</summary>

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,   -- was 0.2
  autovacuum_vacuum_threshold = 1000,       -- was 50
  autovacuum_vacuum_cost_limit = 1000,      -- was 200 (more aggressive)
  autovacuum_vacuum_cost_delay = 2,         -- was 20 (less throttling)
  autovacuum_naptime = 30                   -- check more frequently
);
```

Goal: vacuum more frequently but lighter, preventing massive dead tuple accumulation.

</details>

5. **Q:** What is the difference between pg_dump and pgBackRest?

<details><summary>Answer</summary>

| Tool | Type | Speed | Parallel | PITR | Use case |
|------|------|-------|----------|------|----------|
| pg_dump | Logical | Slow | Limited | No | Schema, small DBs, selective restore |
| pgBackRest | Physical | Fast | Yes | Yes | Large DBs, full recovery, PITR |
| pg_basebackup | Physical | Fast | No | Needs WAL | Building standbys, base backup |

pgBackRest features: compression, encryption, delta restore, parallel backup/restore, verification.

</details>

6. **Q:** How does table partitioning work in PostgreSQL?

<details><summary>Answer</summary>

Declarative partitioning (PG 10+):

```sql
CREATE TABLE events (
  id BIGSERIAL,
  created_at TIMESTAMPTZ NOT NULL,
  data JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_01 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

Benefits: partition pruning (skip irrelevant partitions), easier data retention (DROP PARTITION), parallel processing.

</details>

7. **Q:** What is pg_partman and when would you use it?

<details><summary>Answer</summary>

Extension for automated partition management:

```sql
SELECT partman.create_parent(
  p_parent_table := 'public.events',
  p_control := 'created_at',
  p_type := 'native',
  p_interval := '1 month',
  p_premake := 3
);
```

Automatically creates future partitions and detaches old ones. Essential for time-series partitioning at scale.

</details>

8. **Q:** What are the must-have PostgreSQL extensions for production?

<details><summary>Answer</summary>

- **pg_stat_statements:** Query performance monitoring
- **pg_trgm:** Fuzzy text search (ILIKE, similarity)
- **pg_partman:** Partition management
- **pgBackRest:** Backup and restore
- **pg_repack:** Online bloat removal
- **pg_hint_plan:** Plan stability
- **pg_cron:** Scheduled jobs inside PostgreSQL
- **PostGIS:** Geospatial (if applicable)
- **uuid-ossp / pgcrypto:** UUID generation
- **btree_gin / btree_gist:** Mixed index types

</details>

9. **Q:** What is pg_trgm and what operations does it support?

<details><summary>Answer</summary>

pg_trgm (trigram) indexes support fuzzy string matching:

```sql
CREATE INDEX idx_trgm ON users USING GIN (name gin_trgm_ops);

-- ILIKE (case-insensitive like)
SELECT * FROM users WHERE name ILIKE '%alice%';

-- Similarity
SELECT * FROM users WHERE similarity(name, 'alice') > 0.6;
ORDER BY name <-> 'alice'; -- distance operator
```

**Trap:** B+Tree indexes cannot support `LIKE '%pattern%'` (wildcard prefix). Only GIN/GiST with pg_trgm can.

</details>

10. **Q:** How do you handle backup verification?

<details><summary>Answer</summary>

```sql
-- pgBackRest check (verifies configuration)
pgbackrest --stanza=prod check

-- pgBackRest restore to test environment
pgbackrest --stanza=prod restore --db-path=/var/lib/postgresql/test --delta

-- Validate backups
pgbackrest --stanza=prod --type=full backup
```

**Trap:** Unverified backups are worthless. Always automate restore testing to a staging environment.

</details>

11. **Q:** What is effective_io_concurrency?

<details><summary>Answer</summary>

Controls how many concurrent I/O operations PostgreSQL assumes can be executed. For SSD/NVMe: 200–300 (or more). For HDD: 1–2. Affects bitmap scans and parallel scans.

```sql
ALTER SYSTEM SET effective_io_concurrency = 200;
```

</details>

12. **Q:** What is WAL compression?

<details><summary>Answer</summary>

wal_compression = on compresses full page images in WAL (from full_page_writes). Reduces WAL volume by 2-5x at minimal CPU cost. Enabled by default in PG 15+.

**Trap:** Only compresses full page images, not regular WAL records. Benefit depends on write pattern.

</details>

13. **Q:** How do you handle sequence exhaustion or high sequence contention?

<details><summary>Answer</summary>

```sql
-- Sequences cache in memory per backend
ALTER SEQUENCE orders_id_seq CACHE 1000;

-- Monitor remaining
SELECT seq_name, last_value, seq.max_value - last_value AS remaining
FROM ...;

-- For very high rates: use UUIDs or snowflake IDs instead
```

**Trap:** CACHE > 1 means sequences can have gaps (lost if backend crashes). Acceptable for most OLTP systems.

</details>

14. **Q:** What is pg_cron and when would you use it?

<details><summary>Answer</summary>

Extension for job scheduling inside PostgreSQL:

```sql
CREATE EXTENSION pg_cron;
SELECT cron.schedule('vacuum-db', '0 3 * * *', 'VACUUM ANALYZE');

-- Monitor scheduled jobs
SELECT * FROM cron.job;
SELECT * FROM cron.job_run_details;
```

Use cases: periodic maintenance, partition management, materialized view refresh, data retention.

</details>

15. **Q:** What is the difference between ON_ERROR_ROLLBACK and savepoints?

<details><summary>Answer</summary>

ON_ERROR_ROLLBACK (psql feature, not PostgreSQL) automatically creates a savepoint before each statement — on error, rollback to savepoint. Useful in interactive psql sessions. Not for production.

Savepoints (SQL standard):

```sql
SAVEPOINT sp1;
UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- might fail
ROLLBACK TO SAVEPOINT sp1; -- only this statement is rolled back
RELEASE SAVEPOINT sp1;
```

</details>

---

## 9. Code Puzzles (8 puzzles)

### Puzzle 1: EXPLAIN ANALYZE — Find the bottleneck

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at >= '2025-01-01'
  AND o.status = 'active'
ORDER BY o.created_at DESC
LIMIT 100;
```

Output:
```
Limit  (cost=45231.21..45231.46 rows=100 width=42) (actual time=48234.12..48234.15 rows=100 loops=1)
  ->  Nested Loop  (cost=0.00..45231.21 rows=382000 width=42) (actual time=0.12..48234.05 rows=100 loops=1)
        ->  Seq Scan on orders o  (cost=0.00..42812.90 rows=191000 width=22) (actual time=0.05..48192.89 rows=480000 loops=1)
              Filter: ((created_at >= '2025-01-01'::date) AND (status = 'active'))
              Rows Removed by Filter: 15000000
              Buffers: shared hit=5 read=142000
        ->  Index Scan using customers_pkey on customers c  (cost=0.42..2.64 rows=1 width=28) (actual time=0.01..0.01 rows=1 loops=480000)
              Index Cond: (id = o.customer_id)
              Buffers: shared hit=960000
Planning Time: 0.45 ms
Execution Time: 48234.32 ms
```

**Q:** What is the bottleneck? What index(es) would you add?

<details><summary>Answer</summary>

**Bottleneck:** The Seq Scan on `orders` (15M rows filtered down to 480K). The filter removed 15M rows — that means 15M rows were scanned sequentially. The estimated rows (191K) are far off from actual (480K) — poor stats.

The Nested Loop runs 480,000 times (loops=480000 on the inner index scan) — each matching order fetches customer via pkey.

**Fix:**
```sql
-- Composite index on (status, created_at) for equality + range
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at DESC);

-- Then ANALYZE to update stats
ANALYZE orders;
```

With this index, the Seq Scan becomes an Index Scan using the composite index, finding only the matching rows without scanning 15M rows. The nested loop iterations drop from 480K to ~100 (already limited by LIMIT).

</details>

---

### Puzzle 2: Concurrent Transactions — Deadlock or Not?

**Transaction A:**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- (delay)
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Transaction B:**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- (delay)
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;
```

**Q:** Does a deadlock occur? What if both transactions use FOR UPDATE on both rows at the start?

<details><summary>Answer</summary>

**Yes, deadlock occurs.** A locks id=1, B locks id=2. A tries to lock id=2 (blocked by B), B tries to lock id=1 (blocked by A). Deadlock detected. PostgreSQL kills one transaction.

**Fix — consistent lock ordering:** Always lock rows in the same order (e.g., by id ASC):

```sql
-- Both transactions:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Trap:** Even with FOR UPDATE on both rows at the start, if order differs, deadlock occurs. FOR UPDATE doesn't prevent deadlock — only consistent ordering does.

</details>

---

### Puzzle 3: MVCC Snapshot Visibility

Given:
- XID 100 creates row r1
- XID 101 updates r1 (creating r1v2)
- XID 102 deletes r1v2
- XID 103 is running

**Q:** Which tuple version does each snapshot see?

<details><summary>Answer</summary>

Depends on snapshot boundaries:

| Snapshot (xmin, xmax) | Visible version |
|----------------------|-----------------|
| (95, 99) — before all | r1 (v1) |
| (100, 101) — during | r1v2 (update committed) |
| (101, 102) — during | r1 (v1) — deletion not yet committed |
| (103, 104) — after all | Nothing — row deleted |

**Key rule:** A tuple with xmax = committed XID < snapshot xmin is dead. A tuple with xmin > snapshot xmax is invisible.

</details>

---

### Puzzle 4: Migration Safety — Adding a NOT NULL Column

```sql
-- We need to add a non-null column with a default to a 15M row table
ALTER TABLE orders ADD COLUMN region TEXT NOT NULL DEFAULT 'us-east';
```

**Q:** Is this safe for zero-downtime? What's the impact? What PG version matters?

<details><summary>Answer</summary>

**PG 11+:** Safe. Adding a NOT NULL column with a non-volatile DEFAULT is metadata-only — PostgreSQL stores the default in pg_attrdef and doesn't rewrite the table. Instant.

**PG 10 and earlier:** UNSAFE — rewrites the entire table, holding ACCESS EXCLUSIVE lock for the duration. On 15M rows, this could take minutes, blocking all reads/writes.

**Trap:** Even in PG 11+, adding a column WITHOUT a DEFAULT or with a VOLATILE default rewrites the table. Also, adding NOT NULL with no default requires a separate validation step:

```sql
-- PG 11+ safe approach:
ALTER TABLE orders ADD COLUMN region TEXT DEFAULT 'us-east';
-- Backfill any NULLs in existing rows
UPDATE orders SET region = 'us-east' WHERE region IS NULL;
-- Add NOT NULL constraint with separate validation
ALTER TABLE orders ALTER COLUMN region SET NOT NULL; -- Blocks writes briefly
```

</details>

---

### Puzzle 5: Index Usage Analysis

Given:
```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

**Q:** Which queries can use this index effectively? Which cannot?

```sql
-- Query 1
SELECT * FROM orders WHERE customer_id = 42;

-- Query 2
SELECT * FROM orders WHERE status = 'active';

-- Query 3
SELECT * FROM orders WHERE customer_id = 42 AND status = 'active';

-- Query 4
SELECT * FROM orders WHERE status = 'active' AND created_at > '2025-01-01';

-- Query 5
SELECT customer_id, status FROM orders WHERE customer_id = 42;
```

<details><summary>Answer</summary>

- **Query 1:** YES — uses leftmost column customer_id
- **Query 2:** NO — status is second column, left-prefix rule prevents use
- **Query 3:** YES — uses both columns
- **Query 4:** NO — doesn't include customer_id at all; created_at not in index
- **Query 5:** Yes (Index Only Scan if visibility map permits) — all needed columns in index (customer_id, status)

**Fix for Query 2:** Add an index on (status) alone or a partial index.

**Fix for Query 4:** Add a composite index on (status, created_at) or partial index on status.

</details>

---

### Puzzle 6: WAL/Replication Impact

**Scenario:** You run on a table with 10 million rows, each row is ~200 bytes:

```sql
UPDATE orders SET status = 'archived' WHERE created_at < '2020-01-01';
```

This updates 3 million rows.

**Q:** How much WAL does this generate? What's the impact on replication?

<details><summary>Answer</summary>

**WAL generated:** Each row update generates:
- Tuple header changes (~24 bytes)
- Full page image (first change to page after checkpoint): 8KB
- WAL record overhead

If full_page_writes = on (default) and this is the first change to many pages:
~3M rows / ~200 rows per page = ~15,000 pages. Each page generates ~8KB WAL for the full page image = ~120 MB + tuple data.

If pages were already dirty (modified after last checkpoint):
~3M rows × ~200 bytes per WAL record = ~600 MB.

**Impact on replication:** Standby must replay all this WAL. If WAL generation rate exceeds standby replay rate, lag grows. This type of bulk UPDATE during peak hours can cause replication to fall behind by GB, translating to minutes of lag.

**Mitigation:** Batch in chunks:
```sql
WITH batch AS (
  SELECT id FROM orders
  WHERE created_at < '2020-01-01' AND status != 'archived'
  LIMIT 10000
  FOR UPDATE SKIP LOCKED
)
UPDATE orders SET status = 'archived'
WHERE id IN (SELECT id FROM batch);
```

</details>

---

### Puzzle 7: Advisory Lock Deadlock

**Session 1:**
```sql
SELECT pg_advisory_lock(1);
SELECT pg_advisory_lock(2);
```

**Session 2:**
```sql
SELECT pg_advisory_lock(2);
SELECT pg_advisory_lock(1);
```

**Q:** Does deadlock occur? How does PostgreSQL handle it? Can advisory lock deadlocks be detected?

<details><summary>Answer</summary>

**Yes, deadlock occurs.** Session 1 holds lock 1, waits for lock 2. Session 2 holds lock 2, waits for lock 1. Deadlock detected by the same deadlock_timeout mechanism as row locks.

PostgreSQL detects it as a deadlock (since PG 9.6). The error:
```
ERROR: deadlock detected
DETAIL: Process 1 waits for ExclusiveLock on advisory lock [16394,1,1,2]; blocked by process 2.
```

**Trap:** Not all advisory lock deadlocks are detected — only those using pg_advisory_lock. pg_try_advisory_lock (non-blocking) returns false instead of blocking. Use try_advisory for queue-type coordination to avoid deadlocks entirely.

**Mitigation:** Always acquire advisory locks in the same order globally, or use pg_try_advisory_lock.

</details>

---

### Puzzle 8: Vacuum Tuning

**Given table:**
- Size: 50 GB
- Row count: 100 million
- Write pattern: ~500K rows/day updated, ~200K rows/day deleted
- Read pattern: Heavy OLTP reads on recent data, dashboard queries on aggregated data
- Current autovacuum: defaults (scale_factor = 0.2, threshold = 50)

```sql
SELECT n_dead_tup FROM pg_stat_user_tables WHERE relname = 'orders';
-- Result: n_dead_tup = 8,500,000
```

**Q:** Is autovacuum keeping up? What tuning changes would you make?

<details><summary>Answer</summary>

**Current state:** With defaults, autovacuum triggers at 50 + 0.2 × 100M = ~20M dead tuples. At 8.5M dead tuples, it hasn't triggered yet. By the time it does, there will be 20M dead tuples — a massive vacuum operation.

**This is about the bloat:** 8.5M dead tuples consume significant space. At ~200 bytes per row = ~1.7 GB of dead space.

**Tuning:**
```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_vacuum_threshold = 50000,
  autovacuum_vacuum_cost_limit = 2000,
  autovacuum_vacuum_cost_delay = 5,
  autovacuum_naptime = 30
);
```

- scale_factor = 0.01 → triggers at 1M dead tuples (every ~2 days of churn)
- cost_limit increased + cost_delay reduced → vacuum runs more aggressively but shorter
- naptime = 30 → checks more frequently

**Follow-up:** Also consider adding fillfactor = 90 to enable HOT updates on non-indexed column changes.

</details>

---

## 10. Debugging Scenarios (6 scenarios)

### Scenario 1: Query Degraded After 15M Rows

**Situation:** A dashboard query that took 200ms now takes 45 seconds. The table grew from 2M to 15M rows over 6 months. The query filters on `organization_id` and `status`, ordered by `created_at DESC LIMIT 50`.

```sql
SELECT id, name, total, created_at
FROM orders
WHERE organization_id = 42
  AND status = 'active'
ORDER BY created_at DESC
LIMIT 50;
```

**Q:** What's the most likely cause? What diagnostic would you run? What's the fix?

<details><summary>Answer</summary>

**Most likely:** Missing composite index on (organization_id, status, created_at). With 2M rows, a Seq Scan was "fast enough." At 15M rows, it becomes a Seq Scan on millions of rows only to find 50 matching.

**Diagnostic:**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Likely shows: Seq Scan on orders, filter on org_id and status, sort

-- Check existing indexes
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

**Fix:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_org_status_created
  ON orders (organization_id, status, created_at DESC);
ANALYZE orders;
```

**Root cause:** Stale statistics + no covering index. The planner didn't know the table grew (ANALYZE was running but stats weren't current with 10M new rows in the same status).

**Guardrails:**
- Monitor pg_stat_user_tables.n_dead_tup and idx_scan
- Set up auto_explain on queries > 1s
- Add pg_stat_statements monitoring for execution time regression

</details>

---

### Scenario 2: Deadlock on Order Submission

**Situation:** A trading platform (20K DAU) experiences "deadlock detected" errors during peak hours on order submission. Two orders matching against each other (buy/sell) cause deadlocks.

```
ERROR: deadlock detected
DETAIL: Process 1 waits for ShareLock on transaction 12345; blocked by process 2.
Process 2 waits for ShareLock on transaction 12344; blocked by process 1.
```

**Q:** What's happening? How would you fix it without changing isolation level to SERIALIZABLE?

<details><summary>Answer</summary>

**Root cause:** Two concurrent order matching transactions locking accounts/wallets in different orders. Example:
- Transaction A: Debit buyer's wallet (wallet_id=1), credit seller's wallet (wallet_id=2)
- Transaction B: Debit buyer's wallet (wallet_id=2), credit seller's wallet (wallet_id=1)

**Fix — consistent lock ordering:**
```sql
-- Within each matching transaction, always lock wallets in ascending ID order
BEGIN;
SELECT balance FROM wallets WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Now safe to update both wallets in any order
UPDATE wallets SET balance = balance - 100 WHERE id = 1;
UPDATE wallets SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Additional mitigations:**
- Use `SKIP LOCKED` for order matching queue to avoid contention
- Set `deadlock_timeout = 500ms` for faster detection (lower from default 1s)
- Log lock waits: `log_lock_waits = on`
- Retry logic on deadlock (catch and retry up to 3 times)

**Trap:** Even with single-table updates, a trigger or CASCADE can cause unexpected lock ordering. Audit all triggers and foreign keys.

</details>

---

### Scenario 3: Replication Lag During Peak

**Situation:** At 10 AM daily (peak trading), replication lag on the reporting standby grows from <100ms to 15+ minutes. The standby serves read-only dashboards. Lag spikes coincide with batch jobs.

**Q:** What are the three most likely causes? How do you investigate each?

<details><summary>Answer</summary>

**Cause 1: Long-running write transaction on primary**
- A batch job runs a long transaction that generates WAL (bulk UPDATE)
- Standby can't replay until transaction commits (commit dependency)

**Investigate:** Look for long-running active queries on primary:

```sql
SELECT pid, now() - query_start, query, state
FROM pg_stat_activity
WHERE state = 'active' ORDER BY query_start;
```

**Solution:** Break batch into smaller transactions, use chunked UPDATEs with LIMIT.

**Cause 2: Autovacuum storm**
- Aggressive autovacuum on large tables generates massive WAL
- VACUUM FREEZE on tables approaching wraparound

**Investigate:**
```sql
SELECT pid, query, state, wait_event FROM pg_stat_activity
WHERE query LIKE 'autovacuum%';
```

**Solution:** Tune autovacuum to be more frequent but lighter. Proactive VACUUM FREEZE during maintenance window.

**Cause 3: WAL archiver falling behind**
- archive_command is slow (network latency, S3 uploads)
- WAL sender blocked by archiver

**Investigate:** Check pg_stat_archiver:
```sql
SELECT * FROM pg_stat_archiver;
```

**Solution:** Increase max_wal_senders, optimize archive_command, consider wal_compression = on.

</details>

---

### Scenario 4: ALTER TABLE Hanging

**Situation:** You run:
```sql
ALTER TABLE orders ADD COLUMN discount NUMERIC(5,2);
```
It hangs indefinitely. No errors, just waiting.

**Q:** How do you diagnose and resolve?

<details><summary>Answer</summary>

**Diagnosis:**
```sql
-- Find blocked session
SELECT pid, pg_blocking_pids(pid) AS blocked_by, query, state, wait_event
FROM pg_stat_activity
WHERE pg_blocking_pids(pid) <> '{}';

-- Find what's holding ACCESS EXCLUSIVE lock
SELECT pid, locktype, mode, granted, relation::regclass
FROM pg_locks
WHERE NOT granted;
```

The ALTER TABLE needs ACCESS EXCLUSIVE lock. A long-running query (SELECT, UPDATE) holds ACCESS SHARE/ROW EXCLUSIVE that conflicts.

**Resolution options:**
1. **Cancel the blocker:** `SELECT pg_cancel_backend(pid);` (cancel query)
2. **Terminate the blocker:** `SELECT pg_terminate_backend(pid);` (kill connection)
3. **Set lock_timeout:** `SET lock_timeout = '10s';` before ALTER TABLE
4. **Use a two-step approach:** Check for blocking first, then proceed

**Prevention:** Use `lock_timeout` or `statement_timeout` in migration scripts. Schedule DDL during low traffic.

**Trap:** Never kill a long-running DDL blindly — could be mid-critical operation. Check what the blocking session is doing first.

</details>

---

### Scenario 5: PgBouncer Connections Exhausted

**Situation:** Application reports "could not connect to server: too many connections". PgBouncer shows all connections in use. max_connections in PostgreSQL is 200, PgBouncer pool_size = 100.

```
ERROR: pooler connection limit exceeded for database "app"
```

**Q:** What's happening and how do you debug?

<details><summary>Answer</summary>

**Root cause:** PgBouncer's pool is exhausted — 100 connections are all in use. This happens when:
1. **Slow queries:** Queries take longer → connections held longer → fewer available
2. **Pool too small:** pool_size not sized for peak concurrency
3. **Connection leaks:** Application not releasing connections
4. **Transaction pooling broken:** Application using session-level features (temp tables, SET) forcing session pooling

**Investigation:**
```sql
-- In PostgreSQL (show actual backend connections)
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';

-- Check PgBouncer stats (connect to PgBouncer admin DB)
SHOW STATS;
SHOW POOLS;
SHOW CLIENTS;
SHOW SERVERS;
```

**Solutions:**
1. **Right-size the pool:** pool_size = CPU cores × 2 + disk spindles (rule: 50-100 is often enough)
2. **Fix slow queries:** Optimize query performance to reduce connection hold time
3. **Add reserve_pool:** `reserve_pool_size = 5`, `reserve_pool_timeout = 3` for peak bursts
4. **Connection pooling:** Ensure application uses connection release properly (return to pool)
5. **Monitoring:** Set up PgBouncer metrics in Datadog/Prometheus

**Trap:** Throwing more connections at the problem (increasing pool_size) often makes it worse — more connections = more context switching = slower queries = longer hold times.

</details>

---

### Scenario 6: NOT VALID Constraint Fails After Migration

**Situation:** After a zero-downtime migration using NOT VALID:

```sql
ALTER TABLE orders ADD CONSTRAINT chk_total_positive
  CHECK (total > 0) NOT VALID;
-- Backfill: validate existing rows
ALTER TABLE orders VALIDATE CONSTRAINT chk_total_positive;
```

Two days later, a dashboard query shows orders with total = 0. The VALIDATE CONSTRAINT completed without errors.

**Q:** How is this possible? What went wrong?

<details><summary>Answer</summary>

**Root cause:** VALIDATE CONSTRAINT only validates rows that exist *at the time of validation*. Concurrent INSERT/UPDATE happening during VALIDATE can insert invalid rows that skip validation.

VALIDATE CONSTRAINT takes SHARE UPDATE EXCLUSIVE lock — allows concurrent writes. New rows inserted/updated during validation are NOT checked.

**Fix:**
1. **Re-run validation:** More may have slipped in
2. **Use a safer approach:** Take a brief ACCESS EXCLUSIVE lock to prevent writes during validation:

```sql
BEGIN;
LOCK TABLE orders IN ACCESS EXCLUSIVE MODE; -- blocks writes
ALTER TABLE orders VALIDATE CONSTRAINT chk_total_positive;
COMMIT;
```

Or use a trigger-based approach for strict enforcement.

3. **Fix bad data:**
```sql
SELECT * FROM orders WHERE NOT (total > 0);
UPDATE orders SET total = 0.01 WHERE total <= 0;
```

**Prevention:** The expand→migrate→contract pattern is safer:
1. Add column (nullable or with default) — safe, no lock
2. Backfill data in batches (with SKIP LOCKED)
3. Add NOT NULL or CHECK constraint with NOT VALID
4. LOCK TABLE briefly + VALIDATE CONSTRAINT
5. Drop old column if applicable

**Trap:** Never assume VALIDATE CONSTRAINT is safe from concurrent violations. It's not — it has a known race condition.

</details>

---

## 11. System Design Prompts (6 prompts)

### Prompt 1: Multi-Tenant Database Schema

**Context:** You're designing the database schema for a multi-tenant SaaS platform (inventory management). Each tenant is an organization with its own data.

**Q:** Design the schema with organization_id scoping, composite FKs, tenant-scoped indexes, RLS, and connection pooling. Discuss trade-offs.

<details><summary>Answer</summary>

**Schema design:**

```sql
-- Every table has organization_id as first column
CREATE TABLE organizations (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'starter',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE orders (
  organization_id BIGINT NOT NULL REFERENCES organizations(id),
  id BIGSERIAL,
  customer_id BIGINT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  total NUMERIC(12,2) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (organization_id, id) -- composite PK
) PARTITION BY LIST (organization_id);
```

**Indexing strategy:**
- Every index starts with organization_id (leftmost prefix for tenant isolation)
- Covering indexes for dashboard queries
- Partial indexes for status=active filtering per tenant

```sql
CREATE INDEX idx_orders_tenant_status
  ON orders (organization_id, created_at DESC)
  WHERE status = 'active';

CREATE INDEX idx_orders_tenant_customer
  ON orders (organization_id, customer_id)
  INCLUDE (total, status);
```

**Row-Level Security (RLS):**

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (organization_id = current_setting('app.organization_id')::BIGINT);
```

**Connection pooling:** PgBouncer with transaction pooling. Each connection is stateless; organization_id set via SET LOCAL at the start of each request.

**Trade-offs:**
- Shared DB vs. separate DB per tenant: Shared DB is simpler, cheaper, but noisier neighbor risk. Separate DB is stronger isolation but operationally expensive.
- RLS overhead: ~3-5% performance cost for policy checks on every query
- Composite PKs: More complex joins but ensure tenant isolation at the constraint level

</details>

---

### Prompt 2: Zero-Downtime NOT NULL Column Migration

**Context:** You need to add a NOT NULL column with a default to a 15M row production table without downtime.

**Q:** Design the expand→migrate→contract approach. Include rollback plan.

<details><summary>Answer</summary>

**Phase 1: Expand**

```sql
-- Add nullable column (instant metadata operation in PG 11+)
ALTER TABLE orders ADD COLUMN region TEXT DEFAULT 'us-east';
```

**Phase 2: Migrate**

Backfill in batches to avoid long-running transaction:

```sql
-- Backfill in small batches
WITH batch AS (
  SELECT id FROM orders
  WHERE region IS NULL
  ORDER BY id
  LIMIT 1000
  FOR UPDATE SKIP LOCKED
)
UPDATE orders SET region = 'us-east'
WHERE id IN (SELECT id FROM batch);
```

Run this in a background job until no NULLs remain.

**Phase 3: Contract**

```sql
-- Add NOT NULL constraint with validation
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;
```

This requires ACCESS EXCLUSIVE briefly (PG 11+ is metadata-only if no NULLs exist). Schedule during low traffic or use NOT VALID + LOCK TABLE approach.

**Rollback plan:**

```sql
-- If issues arise:
ALTER TABLE orders ALTER COLUMN region DROP NOT NULL;
ALTER TABLE orders DROP COLUMN region;
```

**Monitoring:**
- Track backfill progress: `SELECT count(*) FROM orders WHERE region IS NULL;`
- Track lock contention: `pg_blocking_pids()`
- Monitor replication lag during backfill (batches minimize impact)

**Trap:** In PG 12+, adding a column with a non-volatile DEFAULT is instant. In PG 10 and earlier, it rewrites the table. Know your PG version.

</details>

---

### Prompt 3: Trading Order Book Schema

**Context:** Design the database schema for a trading order book (buy/sell matching). Needs SKIP LOCKED for order matching, idempotency, and a ledger.

<details><summary>Answer</summary>

```sql
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  symbol TEXT NOT NULL,
  side TEXT NOT NULL CHECK (side IN ('buy', 'sell')),
  price NUMERIC(16,8) NOT NULL,
  quantity NUMERIC(16,8) NOT NULL,
  filled_quantity NUMERIC(16,8) NOT NULL DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'open'
    CHECK (status IN ('open', 'partial', 'filled', 'cancelled')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  idempotency_key TEXT UNIQUE  -- prevent duplicate submissions
);

CREATE INDEX idx_order_book
  ON orders (symbol, side, price DESC, created_at)
  WHERE status IN ('open', 'partial');

CREATE TABLE trades (
  id BIGSERIAL PRIMARY KEY,
  buy_order_id BIGINT NOT NULL REFERENCES orders(id),
  sell_order_id BIGINT NOT NULL REFERENCES orders(id),
  price NUMERIC(16,8) NOT NULL,
  quantity NUMERIC(16,8) NOT NULL,
  executed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ledger_entries (
  id BIGSERIAL PRIMARY KEY,
  account_id BIGINT NOT NULL,
  trade_id BIGINT NOT NULL REFERENCES trades(id),
  amount NUMERIC(16,8) NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('debit', 'credit')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Order matching with SKIP LOCKED:**

```sql
-- Matching engine picks best available order
WITH matched_sell AS (
  SELECT id FROM orders
  WHERE symbol = 'BTC/USD'
    AND side = 'sell'
    AND status IN ('open', 'partial')
    AND price <= 50000.00
  ORDER BY price ASC, created_at ASC
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE orders SET filled_quantity = filled_quantity + 1, ...
WHERE id = (SELECT id FROM matched_sell)
RETURNING *;
```

**Idempotency:** Each order submission has a unique idempotency_key. Duplicate submission returns existing order.

```sql
INSERT INTO orders (symbol, side, price, quantity, idempotency_key)
VALUES ('BTC/USD', 'buy', 50000, 1, 'unique-key-123')
ON CONFLICT (idempotency_key) DO UPDATE SET
  -- no-op, just return existing
  updated_at = orders.updated_at
RETURNING *;
```

**Trap:** SERIALIZABLE isolation for the entire matching cycle can cause many serialization failures. Use READ COMMITTED with SKIP LOCKED and retry logic instead.

</details>

---

### Prompt 4: Time-Series Monitoring/Audit System

**Context:** Design a time-series audit log system with PostgreSQL: 100M events/month, 3-month retention, monthly rollups.

<details><summary>Answer</summary>

**Partitioning by month with pg_partman:**

```sql
CREATE TABLE audit_events (
  id BIGSERIAL,
  organization_id BIGINT NOT NULL,
  event_type TEXT NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create initial partitions
SELECT partman.create_parent(
  p_parent_table := 'public.audit_events',
  p_control := 'created_at',
  p_type := 'native',
  p_interval := '1 month',
  p_premake := 3
);

-- Update retention: drop partitions older than 3 months
SELECT partman.partition_data_proc(
  'public.audit_events',
  p_retention := '3 months',
  p_keep_table := false
);
```

**Indexing:**

```sql
-- BRIN index for time-ordered data (extremely compact)
CREATE INDEX idx_audit_brin ON audit_events
  USING BRIN (created_at) WITH (pages_per_range = 32);

-- B+Tree for tenant-specific queries
CREATE INDEX idx_audit_tenant ON audit_events (organization_id, created_at DESC);

-- GIN for JSONB queries
CREATE INDEX idx_audit_payload ON audit_events USING GIN (payload jsonb_path_ops);
```

**Materialized views for rollups:**

```sql
CREATE MATERIALIZED VIEW audit_daily_rollup AS
SELECT organization_id, event_type,
       date_trunc('day', created_at) AS day,
       count(*) AS event_count
FROM audit_events
GROUP BY organization_id, event_type, date_trunc('day', created_at);

-- Refresh via pg_cron
SELECT cron.schedule('refresh-rollup', '0 1 * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY audit_daily_rollup');
```

**Trap:** BRIN indexes require data to be physically ordered by the indexed column. INSERTs in time order work well. Updates/reorders break BRIN effectiveness.

</details>

---

### Prompt 5: Job Scheduling System

**Context:** Design a job queue/worker system using PostgreSQL (ties to Chronos). Workers pick jobs atomically, retry on failure, and avoid double-processing.

<details><summary>Answer</summary>

```sql
CREATE TABLE jobs (
  id BIGSERIAL PRIMARY KEY,
  queue_name TEXT NOT NULL,
  job_type TEXT NOT NULL,
  payload JSONB NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
  priority INT NOT NULL DEFAULT 0,
  max_retries INT NOT NULL DEFAULT 3,
  retry_count INT NOT NULL DEFAULT 0,
  scheduled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  error_message TEXT,
  idempotency_key TEXT UNIQUE
);

CREATE INDEX idx_jobs_pending
  ON jobs (priority DESC, scheduled_at ASC)
  WHERE status = 'pending';

CREATE INDEX idx_jobs_running
  ON jobs (queue_name, started_at)
  WHERE status = 'running';
```

**Worker pickup (atomic claim):**

```sql
WITH next_job AS (
  SELECT id FROM jobs
  WHERE status = 'pending'
    AND scheduled_at <= NOW()
  ORDER BY priority DESC, scheduled_at ASC
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE jobs SET
  status = 'running',
  started_at = NOW()
WHERE id = (SELECT id FROM next_job)
RETURNING *;
```

**Advisory lock for coordination (prevent concurrent pickup of same job across workers):**

```sql
-- Worker 1 tries to claim job 42
SELECT pg_try_advisory_lock(42); -- returns true → owns the job
-- Worker 2 trying same: returns false → skips
```

**Status tracking with heartbeat:**

```sql
UPDATE jobs SET started_at = NOW()
WHERE id = 42 AND status = 'running';
```

**Dead letter queue:**

Move jobs that exceed max_retries to a separate table or add a flag.

**Trap:** Without SKIP LOCKED, multiple workers compete for the same row — lock contention kills throughput. Without idempotency_keys, network retries can create duplicate jobs.

</details>

---

### Prompt 6: Caching Layer with PostgreSQL

**Context:** Your multi-tenant app needs to cache dashboard aggregations and API responses. Design a caching strategy using PostgreSQL and an external cache (Redis/Memcached).

<details><summary>Answer</summary>

**Two-tier caching:**

1. **L1: Redis** — Sub-millisecond, for hot data, TTL-based eviction
2. **L2: PostgreSQL materialized views** — Freshness guarantees, no separate infrastructure

**Materialized views for dashboards:**

```sql
CREATE MATERIALIZED VIEW dashboard_summary AS
SELECT organization_id,
       count(*) AS order_count,
       sum(total) AS revenue,
       count(*) FILTER (WHERE status = 'active') AS active_count
FROM orders
GROUP BY organization_id;

CREATE UNIQUE INDEX ON dashboard_summary (organization_id);

-- Refresh on a schedule
SELECT cron.schedule('refresh-dashboard', '*/5 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_summary');
```

**Cache invalidation with LISTEN/NOTIFY:**

```sql
-- After an order is created/updated
NOTIFY dashboard_update, '{"org_id": 42}';

-- Application listens and invalidates Redis cache for org 42
```

**Redis integration strategy:**

```
1. On request: Check Redis. Hit → return.
2. Miss → Query PostgreSQL (via materialized views or directly)
3. Store in Redis with TTL (60s for dashboards, 300s for aggregate data)
4. Return
```

**Trap:** Materialized views need PROPER refresh strategy. CONCURRENTLY prevents blocking reads but adds overhead. REFRESH MATERIALIZED VIEW (without CONCURRENTLY) blocks reads — don't use during peak.

</details>

---

## 12. STAR Stories (4 templates)

### STAR 1: The 88% Query Reduction

**Situation:**
- Multi-tenant inventory SaaS, growing from 2M to 15M rows
- A frequently-run dashboard query degraded from 200ms to 45s
- Customer-facing pages were timing out

**Task:**
- Reduce query time to under 500ms
- No downtime for the fix
- Prevent future regression

**Action:**

1. **Measure:** Loaded the query into EXPLAIN (ANALYZE, BUFFERS). Found Seq Scan on orders (15M rows scanned, 480K matching). Nested Loop ran 480K iterations to join customers — but only 100 rows were needed (LIMIT 100).

2. **Profile:** Wait event analysis showed DataFileRead — pure I/O bottleneck. pg_stat_statements confirmed the query was the #1 by total_time.

3. **Root cause:** Two missing composite indexes:
   - `(organization_id, status, created_at DESC)` for the main query
   - Covering `(organization_id, customer_id) INCLUDE (total)` for the dashboard

4. **Fix:** Created both indexes concurrently:
```sql
CREATE INDEX CONCURRENTLY idx_orders_lookup
  ON orders (organization_id, status, created_at DESC);
CREATE INDEX CONCURRENTLY idx_orders_customer_cover
  ON orders (organization_id, customer_id) INCLUDE (total);
ANALYZE orders;
```

5. **Verify:** Query dropped from 45s to 340ms — **99.2% reduction.** Index Only Scan replaced the Seq Scan and Nested Loop.

6. **Guardrails:**
- Added auto_explain to log any query > 1s
- Set up pg_stat_statements monitoring dashboard
- Added weekly unused index scan (daily cron checks idx_scan = 0)
- Implemented migration checklist requiring EXPLAIN ANALYZE before merging

**Result:**
- 45s → 280-340ms (99.2% reduction)
- 0 downtime, no deployments needed
- Dashboard team unblocked

**Lesson:** The true fix was not just adding indexes — it was understanding that the optimizer chose a plan for 2M rows that was catastrophic at 15M. The covering index was the key insight: the query only needed 4 columns, and all 4 fit in the index.

---

### STAR 2: The 15M Zero-Downtime Migration

**Situation:**
- Multi-tenant SaaS needed to split a monolith `orders` table into `orders` + `order_line_items`
- 15M orders with 45M line items in production
- 99.99% uptime requirement (contractual SLA)

**Task:**
- Zero-downtime schema migration
- No data loss
- Rollback capability within 15 minutes

**Action:**

**Phase 1 — Expand:**
- Added nullable `order_id` FK column to `line_items`
- Created new `order_line_items` table (partitioned by org_id)
- Created partial indexes on new table

**Phase 2 — Migrate:**
- Wrote a backfill job using SKIP LOCKED batches of 1000:
```sql
WITH batch AS (
  SELECT li.id FROM line_items li
  WHERE li.order_id IS NULL
  LIMIT 1000
  FOR UPDATE SKIP LOCKED
)
UPDATE line_items SET order_id = ...
WHERE id IN (SELECT id FROM batch);
```
- Backfilled 45M rows over 4 hours during low traffic
- Monitored replication lag, lock contention, and error rate

**Phase 3 — Contract:**
- Added NOT NULL constraint with NOT VALID
- Validated with brief LOCK TABLE + VALIDATE
- Made new table the source of truth
- Deployed code change to read from new schema
- Dropped old columns after 1 week of monitoring

**Rollback plan:**
- Scripted reverse migration (copy data back)
- Toggle flag in config to use old schema
- Validated rollback in staging with production-sized dataset

**Result:**
- Migration completed with 0 downtime
- 0 data inconsistencies
- Rollback not needed

**Key technical insight:** The NOT VALID constraint race condition was avoided by adding a brief ACCESS EXCLUSIVE lock during VALIDATE CONSTRAINT (blocking writes for ~2 seconds during off-peak).

---

### STAR 3: Trading Platform Concurrency Bug

**Situation:**
- Trading platform (20K DAU) introduced a new order matching algorithm
- Production reported random "deadlock detected" errors
- Errors spiked during volatile market conditions (high trading volume)

**Task:**
- Eliminate deadlock errors without changing the matching algorithm fundamentally
- Maintain order matching throughput
- No SERIALIZABLE isolation (performance impact)

**Action:**

1. **Discovered:** Two concurrent matching transactions were locking wallets in opposite order (A locks wallet 1 → wallet 2; B locks wallet 2 → wallet 1).

2. **Evaluated 4 strategies:**
   - **SERIALIZABLE isolation:** Would work but throughput drops 40% in benchmarks
   - **Single-threaded matching:** Wouldn't scale to peak volume
   - **Redis-based queue:** Additional infrastructure, consistency concerns
   - **Consistent lock ordering:** Simple, zero new infra, minimal code change

3. **Chosen: Consistent lock ordering + SKIP LOCKED queue:**
```sql
-- Before matching, sort wallet IDs and lock in order
BEGIN;
SELECT balance FROM wallets
WHERE id IN (SELECT id FROM pending_wallets)
ORDER BY id FOR UPDATE;
-- Now safe to update in any order
-- ...
COMMIT;
```

4. **Deadlock handling:**
- Retry logic: catch deadlock error, retry up to 3 times with exponential backoff
- deadlock_timeout reduced to 500ms for faster detection
- log_lock_waits = on for monitoring

5. **Monitoring:**
- Added pg_locks polling every 5s to detect lock chains
- Grafana dashboard for lock contention
- Alert on > 5 deadlocks per minute

**Result:**
- 0 deadlocks after fix (consistent ordering eliminated the cycle)
- Throughput maintained
- Retry logic never triggered in production

**Lesson:** Lock ordering is a discipline enforced at the code level, not the database level. Documentation and code review are critical.

---

### STAR 4: Multi-Tenant Indexing Strategy

**Situation:**
- Platform had 200+ tenants, from small (100 rows) to large (5M rows)
- Dashboards were slow for large tenants, fast for small ones
- DB size 500GB, 60% was indexes

**Task:**
- Design a tenant-aware indexing strategy
- Reduce index bloat for small tenants
- Improve dashboard query performance for large tenants

**Action:**

**1. Tenant-scoped indexes:**
Every query filters by `organization_id`. Made it the first column in every composite index:

```sql
CREATE INDEX idx_orders_org_status ON orders (organization_id, status)
  INCLUDE (total, created_at);
```

**2. Covering indexes for dashboards:**
The dashboard query selected `id, total, status, created_at, customer_name`. Added all as INCLUDE columns:

```sql
CREATE INDEX idx_dashboard ON orders (organization_id, status, created_at DESC)
  INCLUDE (total, customer_name);
```

Index Only Scan eliminated heap fetches entirely. Query time dropped from 12s to 120ms.

**3. Partial indexes for status filtering:**
80% of queries filtered on `status = 'active'`. Added partial indexes:

```sql
CREATE INDEX idx_active_orders ON orders (organization_id, created_at DESC)
  WHERE status = 'active';
```

These are 5x smaller than full indexes.

**4. Unused index cleanup:**
Ran weekly unused index scan. Found 12 indexes with idx_scan = 0:

```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

Dropped 8 unused indexes — reclaimed 30GB.

**5. BRIN for audit tables:**
Replaced B+Tree with BRIN on append-only audit tables:

```sql
CREATE INDEX idx_audit_brin ON audit_log USING BRIN (created_at)
  WITH (pages_per_range = 64);
```

Reduced index size from 12GB to 40MB.

**Result:**
- Index size reduced 40% (from 300GB to 180GB)
- Large-tenant dashboard queries improved 20-100x
- Write throughput improved 15% (fewer indexes to maintain)
- BRIN indexes: 12GB → 40MB for audit tables

**Lesson:** Different tenant sizes need different indexes. Partial and covering indexes were the highest-impact changes.

---

## 13. Questions to Ask the Interviewer (10 questions)

1. **How do you currently handle schema migrations?** (Looking for: CI/CD pipeline, zero-downtime practice, rollback strategy)

2. **What's your PostgreSQL version and upgrade strategy?** (Looking for: are they stuck on an old version, do they upgrade regularly)

3. **How do you manage connection pooling at scale?** (Looking for: PgBouncer, pool sizing knowledge, transaction vs session pooling)

4. **What's your backup and recovery strategy, and have you tested it this quarter?** (Looking for: PITR, pgBackRest, restore testing)

5. **How do you monitor query performance and detect regressions?** (Looking for: pg_stat_statements, auto_explain, APM integration)

6. **What's your approach to multi-tenancy: separate databases or shared with RLS?** (Looking for: trade-off awareness, noisiness vs. operational complexity)

7. **How do you handle autovacuum tuning on large tables?** (Looking for: scale factor vs. threshold knowledge, per-table overrides, freeze management)

8. **What's the most interesting PostgreSQL production incident you've resolved?** (Looking for: depth of experience, learning culture, blameless postmortems)

9. **How do you approach zero-downtime schema changes?** (Looking for: expand→migrate→contract, NOT VALID, CREATE INDEX CONCURRENTLY)

10. **What's your strategy for partition management?** (Looking for: pg_partman, range partitioning, retention, partition pruning)

---

## 14. Red Flags to Avoid (10 red flags)

1. **"We never touch postgresql.conf"** — Means no tuning. Shared_buffers probably at default (128MB). Run.

2. **"We just use whatever defaults autovacuum gives us"** — On a multi-TB database, this guarantees bloat, wraparound panic, and eventual outage.

3. **"We don't test backups"** — Untested backups are worthless. When recovery is needed, you'll discover corruption too late.

4. **"We run ALTER TABLE during business hours"** — Blocking DDL on production tables indicates no migration process. ACCESS EXCLUSIVE kills your site.

5. **"We use 500 max_connections"** — With process-based architecture, this means 500 backends fighting for CPU. PgBouncer exists for a reason.

6. **"CREATE INDEX is instant, right?"** — On a 100M row table, a regular CREATE INDEX blocks writes for minutes/hours. No CIC in their vocabulary.

7. **"We just SELECT * everywhere"** — No covering indexes, no awareness of index-only scans, no understanding of tuple deforming cost.

8. **"SERIALIZABLE will fix our concurrency problems"** — It masks the problem while cratering throughput. Consistent lock ordering is the answer.

9. **"We don't monitor replication lag"** — When the standby is 2 hours behind and a failover happens, data loss is permanent and shocking.

10. **"VACUUM FULL is our maintenance strategy"** — Blocks everything, doesn't solve the root cause (autovacuum tuning), and shows fundamental misunderstanding.

---

> **End of Question Bank.** Drill these daily. Mark confidence levels. Focus on sections where "trap" caught you.

> **Next topic in sequence:** MongoDB
