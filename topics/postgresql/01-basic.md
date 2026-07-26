# PostgreSQL — Tier 1: Basic (Architecture, SQL & Fundamentals)

PostgreSQL is your primary database. Every story you tell in an interview — the 88% query reduction on the SaaS multi-tenant platform, the 15-million-record migration with zero downtime, the trading platform with strict consistency requirements — all of it runs on Postgres. You aren't a "MySQL person" or a "NoSQL person." You're a Postgres person who happens to also know other databases. This tier covers everything you must own cold before walking into any Senior Backend Engineer interview.

PostgreSQL is an ORDBMS (Object-Relational Database Management System) with a process-per-connection architecture, full ACID compliance, multi-version concurrency control (MVCC), extensible indexing, and a decade-plus head start on JSON support among traditional relational databases.

---

## Table of Contents

1. [PostgreSQL Architecture](#1-postgresql-architecture)
2. [WAL (Write-Ahead Log)](#2-wal-write-ahead-log)
3. [MVCC (Multi-Version Concurrency Control)](#3-mvcc-multi-version-concurrency-control)
4. [Vacuum & Autovacuum](#4-vacuum--autovacuum)
5. [Data Types — Complete Reference](#5-data-types--complete-reference)
6. [Index Types Deep Dive](#6-index-types-deep-dive)
7. [Basic Query Patterns](#7-basic-query-patterns)
8. [Transaction Isolation Levels](#8-transaction-isolation-levels)
9. [Roles, Permissions & Security](#9-roles-permissions--security)
10. [Tier 1 Q&A Drill](#10-tier-1-qa-drill)

---

## 1. PostgreSQL Architecture

### Process Model

PostgreSQL uses a **process-per-connection** architecture (unlike MySQL which uses threads). When PostgreSQL starts, it launches the **postmaster** process — the supervisor daemon that manages everything.

```
postmaster (PID 1, parent of all)
├── backend process (for client connection #1)
├── backend process (for client connection #2)
├── backend process (for client connection #3)
├── checkpointer
├── autovacuum launcher
│   ├── autovacuum worker (table A)
│   └── autovacuum worker (table B)
├── WAL writer
├── archiver
├── stats collector
├── logical replication launcher
│   └── logical replication worker
└── WAL sender (for streaming replica)
    └── WAL receiver (on replica side)
```

**Postmaster**: The first process. Forks child processes for every connection and every background worker. Monitors child processes — if a backend crashes, postmaster performs crash recovery (replays WAL) and restarts. If `shared_buffers` is corrupted, postmaster kills all backends and forces a full restart.

**Backend Processes**: One per client connection. Forked by postmaster when a connection is accepted. Each backend has its own copy of process memory (hence the "process-per-connection" model). Communication between backends happens through **shared memory** (not IPC pipes or signals — though signals are used for interruption).

**Background Workers:**

| Worker | Responsibility |
|---|---|
| Checkpointer | Writes all dirty shared buffers to disk. Controlled by `checkpoint_timeout` (default 5 min) and `max_wal_size`. |
| Autovacuum Launcher | Schedules autovacuum runs. Wakes every `autovacuum_naptime` (default 1 min) to check which tables need vacuuming. |
| Autovacuum Workers | Perform actual vacuum/analyze. Up to `autovacuum_max_workers` (default 3) run concurrently. |
| WAL Writer | Flushes WAL from WAL buffer to WAL segments on disk every `wal_writer_delay` (default 200ms). |
| Archiver | Copies completed WAL segments to the archive location (if `archive_mode = on`). |
| Stats Collector | Gathers pg_stat_* metrics. Lightweight process, minimal overhead. |
| Logical Replication Launcher | Starts logical replication workers for subscribed publications. |
| WAL Sender | Sends WAL to streaming replicas. One sender per replica. |

### Shared Memory

PostgreSQL allocates a large shared memory segment at startup, accessible by all backends:

- **`shared_buffers`**: The primary data cache. Default is 128MB — in production, typically set to 25% of total RAM. All backends read/write through shared_buffers. Dirty buffers are flushed by the checkpointer.
- **WAL Buffer**: `wal_buffers` (default 16MB). Pending WAL writes before they're flushed to disk. Written continuously by backends, flushed by WAL writer.
- **CLOG (Commit Log)**: Tracks the status of every transaction — 2 bits per transaction: in-progress (00), committed (01), aborted (10), sub-committed (11). Stored in `pg_xact` (formerly `pg_clog`). Required for MVCC visibility checks.
- **ProcArray**: Array of all active backend processes and their transaction IDs. Used for snapshot generation.
- **Lock Manager**: Memory for heavyweight locks (table-level, row-level) and lightweight locks (LWLocks — internal buffer/process management).

### Comparison to MySQL

| Aspect | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Connection model | Process-per-connection (fork) | Thread-per-connection |
| Memory per connection | Higher — each backend has its own memory context | Lower — threads share process memory |
| Scalability at high connections | Connection pooling essential beyond ~200 connections | Can handle more connections natively |
| Crash isolation | One backend crashing doesn't affect others | One thread crash can affect process |
| Fork overhead | `fork()` has memory copy semantics (though COW helps) | Thread creation is lighter |

### Signal-Based IPC

PostgreSQL uses POSIX signals for inter-process control:

- `SIGINT` (Ctrl+C): Cancel current query (backend remains)
- `SIGTERM`: Smart shutdown — terminate all backends gracefully after current queries finish
- `SIGQUIT`: Immediate shutdown — abort all backends, then perform crash recovery on restart
- `SIGHUP`: Reload configuration files (pg_reload_conf())
- `SIGUSR1`: Internal signaling (used by stats collector, checkpointer coordination)

Backend processes do NOT directly communicate with each other via signals. All coordination goes through shared memory (locks, flags) with signals used only to wake sleeping processes.

> **Trap**: PostgreSQL's process-per-connection model means each connection consumes 5–10 MB just for the backend process overhead (before query work memory). With `max_connections = 100` (default), this is fine. But if you set `max_connections = 1000` on a 16 GB machine, you've already lost 5–10 GB to process overhead before any query runs. **Connection pooling (PgBouncer, pgcat) is mandatory** for any application with >200 concurrent connections. Your multi-tenant SaaS should absolutely use transaction-mode pooling.

> **Follow-up**: "How would you size `max_connections`?" — Start with `max_connections = 4 * vCPU_cores + (pooler_connections)`. Each backend needs ~10MB base + work_mem + hash_mem_multiplier. On a 16-core, 64GB machine: `max_connections = 200`, connection pooler caps connections at 50, remaining headroom for maintenance.

### Key Configuration Parameters

```sql
-- Memory settings (adjust to your hardware)
SHOW shared_buffers;          -- 25% of RAM (e.g., 16GB on 64GB machine)
SHOW effective_cache_size;    -- 50-75% of RAM (helps planner estimate index scans)
SHOW work_mem;                -- Per-sort/per-hash operation (default 4MB, tune per query)
SHOW maintenance_work_mem;    -- For VACUUM, CREATE INDEX (default 64MB, can go higher)
SHOW wal_buffers;             -- Default 16MB, 32-64MB for write-heavy workloads

-- Parallelism
SHOW max_parallel_workers_per_gather;  -- Parallel query execution
SHOW max_parallel_workers;
SHOW parallel_tuple_cost;              -- Planner cost for parallel worker setup
SHOW parallel_setup_cost;
```

| Parameter | Recommendation | Context |
|---|---|---|
| `shared_buffers` | 25% of RAM | PostgreSQL caches in shared memory. More is not always better — OS cache also matters. Above ~40% of RAM, PostgreSQL's buffer management overhead increases. |
| `effective_cache_size` | 50-75% of RAM | Tells the planner how much cache is available (including OS cache). Higher values encourage index scans. |
| `work_mem` | 4-64 MB per operation | Used for sorts, hash tables, bitmap operations. On a 64GB machine with 200 connections: `64GB * 0.25 / 200 = 80MB` per operation as a crude estimate. But be careful — a single query can use multiple `work_mem` slots (each sort, each hash join). |
| `maintenance_work_mem` | 256 MB - 1 GB | Used by VACUUM, CREATE INDEX, ALTER TABLE. Higher = faster maintenance operations. |
| `wal_buffers` | 16-64 MB | Write-ahead log buffer. Write-heavy workloads benefit from larger values. |
| `max_worker_processes` | CPU cores | Total background worker limit. |
| `max_parallel_workers_per_gather` | CPU cores / 2 | Parallel query workers per query. |
| `random_page_cost` | 1.1 (SSD) / 4.0 (HDD) | Lower on SSDs = planner prefers index scans. Default is 4.0 (assumes HDD). Change to 1.1 for SSD. |

```sql
-- Tune for SSD-based production
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_cache_size = '32GB';
ALTER SYSTEM SET shared_buffers = '16GB';
-- Requires restart
SELECT pg_reload_conf();
```

> **Trap**: Raising `work_mem` too high without considering connection count can exhaust RAM. Each backend can make multiple passes through `work_mem` — a single complex query with 3 hash joins and 2 sorts could use 5× `work_mem`. On a server with 200 connections, that's potentially explosive. Start conservative and bump per-query using `SET LOCAL work_mem = '64MB'` for known expensive queries.

> **Trap**: PostgreSQL's default `random_page_cost` (4.0) assumes HDD. On SSDs, it should be 1.1. If you leave it at 4.0 on an SSD, the planner systematically underestimates the value of index scans and may choose sequential scans for large tables. This was part of the 88% query reduction — fixing planner cost constants.

---

## 2. WAL (Write-Ahead Log)

### What Is WAL?

The fundamental rule: **WAL is written before the data file is modified.** Every change that will be applied to the database is first written to the Write-Ahead Log (WAL). If the system crashes, PostgreSQL replays the WAL from the last checkpoint to restore consistency.

WAL enables three critical features:
1. **Crash recovery** — replay WAL from last checkpoint
2. **Point-in-Time Recovery (PITR)** — restore from base backup + replay WAL to any point
3. **Replication** — streaming replicas consume WAL from primary

### WAL Segments

WAL is stored in `pg_wal/` (formerly `pg_xlog`). Each segment is 16MB by default (controlled by `wal_segment_size`, cluster-wide fixed at initdb).

**WAL segment lifecycle:**

```
online → filled → archived (optional) → recycled
```

- **Online**: Currently being written to. Two segments are always "active" — the current one being written and a reserved one.
- **Fill**: When a segment reaches 16MB, PostgreSQL switches to the next segment. `pg_switch_wal()` forces a manual switch.
- **Archive**: If `archive_mode = on`, the archiver process copies each filled segment to the archive location.
- **Recycle**: Unused segments are renamed (not deleted) and reused to avoid filesystem fragmentation. Controlled by `wal_keep_size` and replication slots.

### WAL Insert vs. WAL Flush

- **WAL Insert**: Appending to the WAL buffer (in shared memory). This is fast (memory-only).
- **WAL Flush**: Writing the WAL buffer to disk (fsync). This is slow (I/O).

Every committed transaction must flush WAL to disk (guaranteed by `synchronous_commit = on`). If you set `synchronous_commit = off`, a transaction can report "committed" before the WAL flush — if the server crashes, that transaction is lost (but the database is still consistent).

```sql
-- Check current WAL insert and flush locations
SELECT pg_current_wal_insert_lsn(), pg_current_wal_flush_lsn();
```

### Checkpoint

A checkpoint forces all dirty shared_buffers to disk and advances the checkpoint LSN. After a checkpoint, crash recovery only needs to replay WAL from the checkpoint LSN, not from the beginning of time.

**Checkpoint triggers:**
- `checkpoint_timeout` expires (default 5 minutes)
- `max_wal_size` is approached (default 1 GB)
- Manual: `CHECKPOINT;`

```sql
-- Checkpoint-related settings
SHOW checkpoint_timeout;   -- 5min
SHOW max_wal_size;         -- 1GB
SHOW min_wal_size;         -- 80MB
```

### full_page_writes

After a checkpoint, the **first modification** to any page writes the entire page to WAL. This handles **partial page writes** — if PostgreSQL crashes while writing an 8KB page and only half the page made it to disk, the full page image in WAL allows recovery to restore the page completely before replaying WAL changes.

```sql
SHOW full_page_writes;  -- on (default and recommended)
```

> **Trap**: Never set `full_page_writes = off` on a primary or a standby that could be promoted. If a crash occurs during a page write, the partial page cannot be recovered and data corruption is guaranteed. The only time it's safe to disable is on a standby that will never be promoted and when filesystem guarantees atomic page writes (e.g., ZFS, certain SSDs with power-loss protection). Even then, be very cautious.

> **Trap**: If checkpoints happen too frequently (because `max_wal_size` is too low), PostgreSQL writes a constant stream of full page images, amplifying WAL writes and degrading performance. Monitor `pg_stat_bgwriter` — if `checkpoints_timed` is low and `checkpoints_req` is high, increase `max_wal_size`.

### wal_level

Controls how much information is written to WAL:

| Level | Use Case | Supports |
|---|---|---|
| `minimal` | Development, no replication | Crash recovery only |
| `replica` | Production default | WAL archiving, streaming replication |
| `logical` | Logical replication, CDC | All of replica + logical decoding |

```sql
SHOW wal_level;  -- replica (should be default in production)
```

> **Trap**: Setting `wal_level = minimal` breaks streaming replication and WAL archiving. If you change to `minimal` while replicas exist, they will immediately fall behind and cannot be recovered. Never set it below `replica` in any production deployment.

> **Follow-up**: "What is the difference between WAL archiving and streaming replication?" — WAL archiving (continuous archiving) copies completed WAL segments to an archive location (e.g., S3, NFS) and is used for PITR. Streaming replication sends WAL in real-time from primary to replica over a TCP connection. Streaming is lower-latency; archiving provides a durable backup. In production, use **both** — streaming replicas for HA, WAL archiving for disaster recovery and PITR.

---

## 3. MVCC (Multi-Version Concurrency Control)

### How PostgreSQL Implements MVCC

PostgreSQL doesn't use rollback segments or an undo log (like Oracle/MySQL). Instead, **multiple versions of a row exist in the table itself**. Each row version is called a "tuple."

**Tuple header fields:**
- `xmin`: Transaction ID (XID) that created this tuple
- `xmax`: Transaction ID that deleted/updated this tuple (0 if not deleted)
- `ctid`: Physical location of the tuple within the page (page number + item index). Also used to follow H.O.T. chains.

### Operation Details

**INSERT:**
```sql
INSERT INTO users (name) VALUES ('Alice');
-- New tuple: xmin = current XID, xmax = 0 (invisible)
-- Tuple is visible to future transactions once XID commits
```

**DELETE:**
```sql
DELETE FROM users WHERE id = 1;
-- Original tuple: xmax = current XID (marked as "dead")
-- Tuple is invisible to future transactions
```

**UPDATE (logically DELETE + INSERT in PG):**
```sql
UPDATE users SET name = 'Bob' WHERE id = 1;
-- Step 1: Old tuple gets xmax = current XID (same as DELETE)
-- Step 2: New tuple inserted: xmin = current XID, xmax = 0
-- The two tuples are linked via ctid chain
```

### Visibility Rules

A tuple is visible to a transaction if:

```
visible = (xmin IS COMMITTED) AND (xmax == 0 OR xmax IS NOT COMMITTED)
```

More precisely for a snapshot with `xmin` (oldest active XID), `xmax` (first unassigned XID), and `xip_list` (active XIDs at snapshot time):

```
visible = (xmin < snap.xmax AND xmin NOT IN snap.xip_list)
       AND (xmax == 0 OR xmax >= snap.xmax OR xmax NOT IN snap.xip_list)
```

### Transaction Snapshots

```sql
-- Get current snapshot
SELECT txid_current_snapshot();
-- Output: "100:110:100,102,105"
-- Format: xmin:xmax:xip_list
--   xmin: oldest active XID
--   xmax: first unassigned XID
--   xip_list: active (in-progress) XIDs at snapshot time
```

**Snapshot behavior by isolation level:**

| Isolation Level | Snapshot Timing |
|---|---|
| READ COMMITTED | New snapshot per statement |
| REPEATABLE READ | One snapshot for entire transaction |
| SERIALIZABLE | One snapshot + serialization monitoring |

### H.O.T. Updates (Heap-Only Tuples)

When an UPDATE doesn't change any indexed columns, PostgreSQL tries to place the new tuple in the **same page** as the old tuple. The index entry still points to the old tuple's ctid, which now points to the new tuple via the page's item pointer chain.

**Benefits:**
- No index maintenance (no new index entry)
- No index bloat from updates
- Only touches one page

**Prerequisites:**
- No indexed columns are modified
- There is free space in the page
- `fillfactor` is below 100 (e.g., `FILLFACTOR = 90` leaves room for H.O.T.)

```sql
-- Configure fillfactor for a table with many HOT updates
ALTER TABLE orders SET (fillfactor = 90);
```

> **Trap**: Long-running transactions are the #1 cause of table bloat in PostgreSQL. Why? VACUUM can only clean tuples whose xmin/xmax is older than the oldest active transaction's XID. If a transaction runs for an hour, no dead tuples created during that hour can be reclaimed. You accumulate bloat during that hour. Monitor `pg_stat_activity` for long-running transactions and use `statement_timeout` to prevent them.

> **Trap**: XID wraparound is a hard database shutdown. XIDs are 32-bit (4 billion transactions). A transaction sees tuples with XIDs in "its past." When XIDs wrap around, old tuples suddenly appear to be in the future. PostgreSQL prevents this by freezing tuples — marking them as committed regardless of XID age. If autovacuum fails to freeze, the database will shut down to prevent data loss. `autovacuum_freeze_max_age` defaults to 200 million — when approached, PostgreSQL forces aggressive anti-wraparound vacuums.

> **Follow-up**: "How would you detect and fix XID wraparound risk?" — Query `SELECT datname, age(datfrozenxid) FROM pg_database;` If any database has `age(datfrozenxid) > 1.5 billion`, you're in danger zone. Run `VACUUM FREEZE` on affected tables during maintenance. The real fix: ensure autovacuum is running properly and not disabled.

---

## 4. Vacuum & Autovacuum

### Why Vacuum Exists

PostgreSQL's MVCC means dead tuples stay in pages until cleaned. Vacuum is essential for:

1. **Reclaiming storage**: Dead tuples occupy space that should be reusable
2. **Preventing transaction ID wraparound**: Freezing tuples marks them as permanently visible
3. **Updating the visibility map**: Enables index-only scans
4. **Updating planner statistics**: When combined with ANALYZE

### VACUUM vs. VACUUM FULL

```sql
-- Standard vacuum: removes dead tuples, marks space reusable
-- Does NOT return space to OS (space remains in the table file)
VACUUM users;

-- Full vacuum: rewrites entire table, returns space to OS
-- Requires ACCESS EXCLUSIVE LOCK = blocks ALL reads and writes
VACUUM FULL users;

-- Combined vacuum + analyze
VACUUM ANALYZE users;
```

| Operation | Locks | Returns space to OS | Production safe? |
|---|---|---|---|
| `VACUUM` | SHARE UPDATE EXCLUSIVE (allows reads) | No | Yes |
| `VACUUM FULL` | ACCESS EXCLUSIVE (blocks everything) | Yes | **No** (downtime required) |
| `CLUSTER` | ACCESS EXCLUSIVE | Yes (rewrites in order) | **No** |

### ANALYZE

```sql
-- Analyze a table (update statistics for the planner)
ANALYZE users;

-- Analyze all tables in the database
ANALYZE;
```

Without up-to-date statistics, the query planner makes terrible decisions (sequential scans on large tables, nested loops when hash joins are better). Auto-ANALYZE is part of autovacuum.

### Autovacuum Configuration

```sql
-- Core autovacuum settings (defaults)
SHOW autovacuum;                          -- on
SHOW autovacuum_vacuum_threshold;         -- 50
SHOW autovacuum_vacuum_scale_factor;      -- 0.2 (20%)
SHOW autovacuum_analyze_threshold;        -- 50
SHOW autovacuum_analyze_scale_factor;     -- 0.1 (10%)
SHOW autovacuum_vacuum_cost_limit;        -- -1 (uses vacuum_cost_limit = 200)
SHOW autovacuum_max_workers;              -- 3
```

**Trigger formula:**
```
VACUUM triggers when: dead_tuples > threshold + (scale_factor * row_count)
Example: 1M row table with defaults: > 50 + (0.2 * 1,000,000) = > 200,050 dead tuples
```

**Per-table tuning** (your hot tables need this):

```sql
-- Aggressive vacuum for high-write table
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.01,
                        autovacuum_vacuum_threshold = 1000,
                        autovacuum_analyze_scale_factor = 0.05);

-- Disable autovacuum on a table (WARNING: dangerous)
ALTER TABLE archive_logs SET (autovacuum_enabled = false);
```

### Freeze and XID Wraparound

```sql
-- Check freeze age of all databases
SELECT datname, age(datfrozenxid) FROM pg_database;

-- Check freeze age of all tables (sorted by worst first)
SELECT relname, age(relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_all_tables
WHERE relfrozenxid != 0
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

When a table's `age(relfrozenxid)` approaches `autovacuum_freeze_max_age` (default 200M), PostgreSQL forces an **anti-wraparound vacuum**. This ignores cost limits and runs at full speed — it can consume significant I/O and CPU, impacting production performance.

### Visibility Map

Each table has a visibility map stored alongside the data file. It tracks which pages have **all tuples visible to all active transactions**.

**Benefits of visibility map:**
- **Index-only scans**: If a page is VM-marked, PostgreSQL doesn't need to check the heap at all — the index alone suffices
- **Partial vacuum**: VACUUM skips pages marked "all visible" (dramatic speedup for partially-touched tables)

```sql
-- Check visibility map status
SELECT relname, relpages, pg_visibility_map(relid) AS vm_status
FROM pg_class
WHERE relname = 'orders';
```

> **Trap**: The most common vacuum-related production incident is **autovacuum falling behind on high-write tables**. Monitor `pg_stat_all_tables.n_dead_tup` over time. If `n_dead_tup` keeps growing, autovacuum isn't keeping up. Fixes: reduce `autovacuum_vacuum_scale_factor`, increase `autovacuum_max_workers`, increase `autovacuum_vacuum_cost_limit`.

> **Trap**: Running `VACUUM FULL` on production is a classic footgun. It takes ACCESS EXCLUSIVE LOCK and rewrites the entire table. Your multi-tenant SaaS app will get 500 errors for every request during that window. Use `pg_repack` instead (reorganizes tables with only a brief SHARE UPDATE EXCLUSIVE lock at the end). Or schedule `VACUUM FULL` during explicit maintenance windows.

> **Trap**: Never disable autovacuum on any table unless you have a very specific reason and monitoring in place. If you disable autovacuum and forget, you *will* hit XID wraparound and the database *will* shut down. There's no warning window — PostgreSQL forces shutdown at 1 million transactions before wraparound.

> **Follow-up**: "We had a table with 500M rows and high churn. Autovacuum couldn't keep up. What did we do?" — We tuned per-table: `ALTER TABLE t SET (autovacuum_vacuum_scale_factor = 0.001, autovacuum_vacuum_threshold = 10000)`. This made vacuum continuous but incremental. We also increased `autovacuum_vacuum_cost_limit = 2000` to let workers run faster, and set `fillfactor = 90` to enable HOT updates (reducing the number of dead tuples created). The 88% query reduction story? Proper autovacuum tuning was part of it — stale stats were causing bad query plans.

---

## 5. Data Types — Complete Reference

### Numeric Types

| Type | Size | Range / Precision | Notes |
|---|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Small range, rarely used |
| `INTEGER` | 4 bytes | -2^31 to 2^31-1 | Default for most integer columns |
| `BIGINT` | 8 bytes | -2^63 to 2^63-1 | IDs that might exceed INTEGER |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | Variable | Up to 131,072 digits | Exact precision — use for money |
| `REAL` | 4 bytes | 6 decimal digits | float32, approximate |
| `DOUBLE PRECISION` | 8 bytes | 15 decimal digits | float64, approximate |
| `SMALLSERIAL` | 2 bytes | 1 to 32,767 | Auto-incrementing SMALLINT |
| `SERIAL` | 4 bytes | 1 to 2^31-1 | Auto-incrementing INTEGER |
| `BIGSERIAL` | 8 bytes | 1 to 2^63-1 | Auto-incrementing BIGINT |

### Character Types

| Type | Size | Notes |
|---|---|---|
| `CHAR(n)` | n bytes (fixed) | Blank-padded — rarely used |
| `VARCHAR(n)` | Variable + 1 byte overhead | Length constraint enforced |
| `TEXT` | Variable + 1 byte overhead | No length constraint, same perf as VARCHAR |

```sql
-- TEXT and VARCHAR are identical in performance
CREATE TABLE example (
    a TEXT,           -- unlimited
    b VARCHAR(255),   -- limited to 255
    c CHAR(10)        -- always 10 chars, blank padded
);
```

### Binary Types

| Type | Size | Notes |
|---|---|---|
| `BYTEA` | Variable + overhead | Binary data, hex or escape format |

### Temporal Types

| Type | Size | Description |
|---|---|---|
| `DATE` | 4 bytes | Date only (no time) |
| `TIME` | 8 bytes | Time of day (no date, no TZ) |
| `TIMESTAMP` | 8 bytes | Date + time, no timezone |
| `TIMESTAMPTZ` | 8 bytes | Date + time, with timezone (stored as UTC) |
| `INTERVAL` | 16 bytes | Time duration |

```sql
-- TIMESTAMPTZ stores UTC, displays in session timezone
SET timezone = 'America/New_York';
SELECT '2024-06-15 12:00:00 UTC'::timestamptz;
-- Shows: 2024-06-15 08:00:00-04

SET timezone = 'UTC';
SELECT '2024-06-15 12:00:00 UTC'::timestamptz;
-- Shows: 2024-06-15 12:00:00+00

-- TIMESTAMP is a literal — no timezone conversion
SELECT '2024-06-15 12:00:00'::timestamp;
-- Always shows: 2024-06-15 12:00:00 (no TZ conversion)
```

### UUID

```sql
-- 16 bytes, globally unique
CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT
);
```

### JSON Types

| Type | Storage | Re-parsed? | Indexed? |
|---|---|---|---|
| `JSON` | Text | Yes (on every access) | No (function indexes possible) |
| `JSONB` | Binary (decomposed) | No | Yes (GIN indexes) |

```sql
-- Always use JSONB unless you need key ordering preservation
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    payload JSONB
);

-- GIN index for JSONB querying
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- Query operators
SELECT * FROM events WHERE payload @> '{"type": "click"}';
SELECT * FROM events WHERE payload ? 'user_id';
SELECT * FROM events WHERE payload ?| ARRAY['click', 'view'];
```

### Arrays

Any data type can be an array:

```sql
CREATE TABLE articles (
    tags TEXT[],
    ratings INTEGER[3]
);

INSERT INTO articles VALUES (ARRAY['postgres', 'sql'], ARRAY[5, 4, 5]);

-- Array access (1-indexed in PostgreSQL!)
SELECT tags[1] FROM articles;  -- 'postgres'

-- Array operators
SELECT * FROM articles WHERE tags @> ARRAY['postgres'];
```

### Network Types

| Type | Description |
|---|---|
| `INET` | IPv4 or IPv6 address (e.g., '192.168.1.1/24') |
| `CIDR` | Network specification (e.g., '192.168.0.0/16') |
| `MACADDR` | MAC address (e.g., '08:00:2b:01:02:03') |

### Geometric Types

| Type | Description |
|---|---|
| `POINT` | (x, y) |
| `LINE` | Infinite line |
| `LSEG` | Line segment |
| `BOX` | Rectangular box |
| `PATH` | Closed/open path |
| `POLYGON` | Closed polygon |
| `CIRCLE` | Circle |

### Range Types

| Type | Description |
|---|---|
| `int4range` | Range of INTEGER |
| `int8range` | Range of BIGINT |
| `numrange` | Range of NUMERIC |
| `tsrange` | Range of TIMESTAMP |
| `tstzrange` | Range of TIMESTAMPTZ |
| `daterange` | Range of DATE |

```sql
-- Range type operators: @>, <@, &&, -|-
SELECT * FROM bookings
WHERE daterange(start_date, end_date, '[]') @> '2024-06-15'::date;
```

### Composite Types

```sql
CREATE TYPE address AS (
    street TEXT,
    city TEXT,
    zip VARCHAR(10)
);

CREATE TABLE companies (
    name TEXT,
    location address
);
```

### DOMAIN

A domain is a constrained type:

```sql
CREATE DOMAIN email AS VARCHAR(255)
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- Use it like any type
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email_address email NOT NULL
);
```

### Enums

```sql
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');

CREATE TABLE people (
    name TEXT,
    current_mood mood
);
```

> **Trap**: `TIMESTAMP` vs `TIMESTAMPTZ` is a classic "tell me you don't understand PostgreSQL" question. `TIMESTAMPTZ` stores UTC internally and converts to the session timezone on display. `TIMESTAMP` stores the literal value with no timezone conversion. If your app stores `2024-01-15 10:00:00` in a `TIMESTAMP` column, then the server timezone changes, the *same* absolute value is displayed — but it represents a *different* moment in time. **Always use `TIMESTAMPTZ`** for application timestamps. Only use `TIMESTAMP` for things like "store opens at 10:00 AM regardless of timezone."

> **Trap**: `SERIAL` is legacy. Use `GENERATED ... AS IDENTITY` (SQL standard) for new tables:
```sql
-- Old way (SERIAL — creates implicit sequence)
CREATE TABLE t (id SERIAL PRIMARY KEY);

-- New way (IDENTITY — SQL standard, better dependency tracking)
CREATE TABLE t (id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY);
```
Why? If you drop a column with a `SERIAL` default, the sequence remains (orphaned). `IDENTITY` correctly ties the sequence to the column. Also, `SERIAL` is `INTEGER` — on big tables you'll hit the 2B limit.

> **Trap**: `TEXT` vs `VARCHAR(n)` — there is zero performance difference in PostgreSQL. `TEXT` and `VARCHAR` are the same internal type (varlena). The only difference is that `VARCHAR(n)` enforces a length constraint. Use `TEXT` with application-level validation, or `VARCHAR(n)` for DB-level enforcement. Don't use `VARCHAR(255)` as a default habit — it's meaningless without domain reasoning.

> **Trap**: `JSON` vs `JSONB` — always use `JSONB`. The text-based `JSON` type re-parses the entire document on every access. `JSONB` stores it in a decomposed binary format that supports indexing. The only case for `JSON` is if you need to preserve key ordering (JSONB sorts keys) or if you need to store duplicate keys (JSONB deduplicates).

> **Follow-up**: "Why did you migrate from SERIAL to IDENTITY in your Laravel app?" — We had a schema migration that dropped a column with a SERIAL default. The sequence was orphaned and we didn't notice until we hit the sequence's max value in production. Migrating all new tables to `IDENTITY` fixed this — sequences are properly dropped when their column is dropped. We also switched from INTEGER to BIGINT for user IDs after projecting past 2B users.

---

## 6. Index Types Deep Dive

### B+Tree (Default)

The default index type in PostgreSQL. Implements a B+Tree (balanced tree with leaf pages containing index entries and heap TIDs).

```sql
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_name_email ON users (name, email);
CREATE INDEX idx_users_email_include_name ON users (email) INCLUDE (name);
CREATE INDEX idx_users_created_at ON users (created_at DESC);
```

**Operators:** `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `IS NOT NULL`, `LIKE 'prefix%'` (prefix wildcard only)

**Multi-column B-Tree:** (a, b, c) — can be used for:
- `WHERE a = 1` (uses first column)
- `WHERE a = 1 AND b = 2` (uses first two columns)
- `WHERE a = 1 AND b = 2 AND c = 3` (uses all columns)
- `WHERE a = 1 ORDER BY b` (uses first two columns for ordering)
- `WHERE b = 2` — **cannot** use index (skips first column)

**INCLUDE columns (PG 11+):** Add non-key payload columns that are stored at the leaf level but don't participate in the index-tree structure. Enables index-only scans without adding to index tree size.

**NULLS behavior in B-Tree:**
- NULLs sort after all values by default (highest)
- `NULLS FIRST` / `NULLS LAST` in ORDER BY can leverage multi-column ordering
- Partial indexes are common for NULL filtering

### GiST (Generalized Search Tree)

A balanced tree structure for extensible indexing. Not a specific algorithm — it's an interface you implement for custom data types.

```sql
-- Geometry (2D points, polygons, etc.)
CREATE INDEX idx_geo_points ON locations USING GiST (point_column);

-- Range type overlap
CREATE INDEX idx_booking_dates ON bookings USING GiST (daterange(
    start_date, end_date, '[]'
));

-- Full-text search (alternative to GIN)
CREATE INDEX idx_fts ON documents USING GiST (tsvector_column);
```

**Operators:** `@@` (full-text match), `@>` (contains), `<@` (contained by), `&&` (overlaps)

### GIN (Generalized Inverted Index)

Inverted index — maps each element to the list of rows containing it. Best for JSONB, arrays, and full-text search.

```sql
-- JSONB
CREATE INDEX idx_events_payload ON events USING GIN (payload jsonb_path_ops);

-- Array
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);

-- Full-text search (preferred over GiST for write-heavy workloads)
CREATE INDEX idx_docs_fts ON documents USING GIN (tsvector_column);
```

**JSONB operators:** `@>`, `?`, `?|`, `?&`
**Array operators:** `@>`, `<@`, `&&`

### BRIN (Block Range INdex)

Indexes blocks of pages (not individual rows). Extremely space-efficient for append-only data with natural ordering.

```sql
-- Time-series data — minimal index size
CREATE INDEX idx_orders_created_at ON orders USING BRIN (created_at);

-- With custom pages_per_range (default 128)
CREATE INDEX idx_orders_created_at_brin
ON orders USING BRIN (created_at) WITH (pages_per_range = 32);
```

**Size comparison:** A B-Tree on a 1B row table might be 30GB. A BRIN on the same column could be 10MB — 3000x smaller.

**Requirements:** Data must be physically ordered by the indexed column (append-only tables with a monotonic timestamp work perfectly).

### SP-GiST (Space-Partitioned GiST)

Best for k-dimensional data, quad-trees, radix trees, network address trees.

```sql
CREATE INDEX idx_ip_ranges ON networks USING SP-GiST (ip_range inet_ops);
```

### Hash

Equality-only index. Limited use — B-Tree already handles `=` efficiently.

```sql
CREATE INDEX idx_users_id_hash ON users USING HASH (id);
```

**Pre-PG10:** Hash indexes are NOT WAL-logged — they're lost on crash and need `REINDEX`. **Post-PG10:** Fully WAL-logged and crash-safe. Still, B-Tree is preferred for equality unless you have a specific reason.

### Partial Indexes

Index only a subset of rows — dramatically reduces index size.

```sql
-- Only index active users (enormous space savings)
CREATE INDEX idx_users_active ON users (email) WHERE active = true;

-- Only index large orders
CREATE INDEX idx_orders_large ON orders (id) WHERE amount > 1000;
```

### Covering Indexes (INCLUDE)

```sql
-- Index-only scan for email lookup without hitting the heap
CREATE INDEX idx_users_email_include_name
ON users (email) INCLUDE (name, created_at);
```

### Reindex

```sql
-- Rebuild all indexes on a table (takes lock)
REINDEX TABLE users;

-- Rebuild a specific index (takes lock)
REINDEX INDEX idx_users_email;

-- Concurrent reindex (PG 12+, allows writes during rebuild)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

> **Trap**: GiST vs GIN for full-text search. GiST indexes are smaller but slower for writes — every insert/update touches multiple index entries. GIN is faster for reads and bulk writes but larger. GIN also supports `tsvector` better via `gin_tsvector_ops`. **Rule of thumb**: use GIN for full-text search on any table with more reads than writes.

> **Trap**: BRIN indexes on a column with no correlation to physical row order are completely useless. If your `updated_at` column gets random values scattered across the physical table, BRIN's block-summary approach will scan almost every block — same as a sequential scan. Only use BRIN on append-only time-series data, log tables, or monotonic sequences.

> **Trap**: Creating B-Tree indexes on low-cardinality columns (boolean, enum with 3 values, status with 5 values) is wasteful and counterproductive. A B-Tree on a boolean column will do a bitmap scan over 50% of the table — slower than a sequential scan. Use **partial indexes** instead: `CREATE INDEX idx_orders_pending ON orders (id) WHERE status = 'pending';`

> **Trap**: Concurrent REINDEX (PG 12+) is safe but can be slow because it builds the new index alongside the existing one (double disk usage). Also, it waits for all conflicting transactions to finish before starting, and at the end it waits again for all transactions using the old index to finish. Plan for this.

> **Follow-up**: "How did you achieve the 88% query reduction?" — One major factor was index analysis. We found tables with 15 indexes (including redundant multi-column indexes), B-Trees on low-cardinality columns, and missing composite indexes for our most common query patterns. We used `pg_stat_user_indexes` to find unused indexes and dropped them. We used `pg_stat_all_tables.seq_scan` to find tables needing indexes. We added BRIN indexes on our time-series event tables. The query planner started making sane choices.

---

## 7. Basic Query Patterns

### DML Fundamentals

```sql
-- SELECT
SELECT id, name, email FROM users WHERE active = true ORDER BY created_at DESC LIMIT 20;

-- INSERT with RETURNING
INSERT INTO orders (user_id, amount, status)
VALUES (42, 1500.00, 'pending')
RETURNING *;

-- UPDATE with RETURNING
UPDATE orders SET status = 'paid', paid_at = now()
WHERE id = 1001
RETURNING id, status, paid_at;

-- DELETE with RETURNING
DELETE FROM temp_logs WHERE created_at < now() - interval '90 days'
RETURNING id, created_at;

-- TRUNCATE (DDL — can't be rolled back in some contexts, resets storage)
TRUNCATE TABLE staging_imports;
```

### JOIN Types

```sql
-- INNER JOIN
SELECT o.*, u.name
FROM orders o
JOIN users u ON u.id = o.user_id;

-- LEFT JOIN
SELECT u.*, o.id AS order_id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;

-- RIGHT JOIN (rare — rewrite as LEFT JOIN for readability)
SELECT u.*, o.*
FROM orders o
RIGHT JOIN users u ON u.id = o.user_id;

-- FULL OUTER JOIN
SELECT COALESCE(u.id, o.user_id) AS user_id, u.name, o.id AS order_id
FROM users u
FULL OUTER JOIN orders o ON o.user_id = u.id;

-- CROSS JOIN (Cartesian product)
SELECT a.*, b.* FROM table_a a CROSS JOIN table_b b;

-- NATURAL JOIN (uses shared column names — implicit, avoid in production)
SELECT * FROM orders NATURAL JOIN users;  -- joins on columns with same name

-- LATERAL JOIN (subquery can reference outer columns)
SELECT u.name, recent_orders.amount, recent_orders.created_at
FROM users u
LEFT JOIN LATERAL (
    SELECT amount, created_at
    FROM orders
    WHERE user_id = u.id
    ORDER BY created_at DESC
    LIMIT 3
) recent_orders ON true;
```

### CTEs (Common Table Expressions)

```sql
-- Non-recursive CTE
WITH active_users AS (
    SELECT id, name, email FROM users WHERE active = true
)
SELECT * FROM active_users WHERE email LIKE '%@company.com';

-- Recursive CTE (tree traversal — e.g., org chart)
WITH RECURSIVE org_tree AS (
    -- Anchor: start with the CEO
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: join back to employees
    SELECT e.id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    JOIN org_tree ot ON ot.id = e.manager_id
)
SELECT * FROM org_tree ORDER BY level, name;

-- Multiple CTEs
WITH
deleted AS (
    DELETE FROM temp_sessions WHERE expires_at < now()
    RETURNING user_id
),
updated AS (
    UPDATE users SET last_cleanup = now()
    WHERE id IN (SELECT user_id FROM deleted)
)
SELECT count(*) FROM deleted;
```

**PG 12+ optimization fence removed:** Before PG 12, CTEs were optimization fences — the planner always materialized them. From PG 12, CTEs can be inlined into the outer query. You can control this:

```sql
-- Force materialization (useful for expensive CTEs run once)
WITH t AS MATERIALIZED (
    SELECT * FROM massive_table WHERE expensive_filter()
)
SELECT * FROM t WHERE id IN (SELECT id FROM another_table);

-- Force inlining (default behavior for simple CTEs)
WITH t AS NOT MATERIALIZED (
    SELECT * FROM small_table
)
SELECT * FROM t WHERE active = true;
```

### Window Functions

```sql
-- Row numbering
SELECT id, amount,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM orders;

-- Ranking
SELECT id, amount,
       RANK() OVER (ORDER BY amount DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank
FROM orders;

-- NTILE (divide into N buckets)
SELECT id, amount,
       NTILE(4) OVER (ORDER BY amount DESC) AS quartile
FROM orders;

-- LEAD/LAG (access previous/next row)
SELECT id, amount, created_at,
       LAG(amount) OVER (ORDER BY created_at) AS prev_amount,
       LEAD(amount) OVER (ORDER BY created_at) AS next_amount,
       amount - LAG(amount) OVER (ORDER BY created_at) AS diff
FROM orders
WHERE user_id = 42;

-- FIRST_VALUE / LAST_VALUE / NTH_VALUE
SELECT id, amount, created_at,
       FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS first_order_amount,
       LAST_VALUE(amount) OVER (
           PARTITION BY user_id ORDER BY created_at
           RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_order_amount
FROM orders;

-- Aggregate with OVER
SELECT id, amount,
       SUM(amount) OVER (PARTITION BY user_id) AS user_total,
       AVG(amount) OVER () AS avg_order_amount
FROM orders;

-- ROWS/RANGE frames
SELECT id, amount, created_at,
       SUM(amount) OVER (
           ORDER BY created_at
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_sum_3
FROM orders;

-- EXCLUDE (PG 13+)
SELECT id, amount,
       AVG(amount) OVER (
           ORDER BY amount
           ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
           EXCLUDE CURRENT ROW
       ) AS neighbors_avg
FROM orders;
```

### GROUPING SETS, CUBE, ROLLUP

```sql
-- ROLLUP: hierarchical aggregation
SELECT user_id, status, count(*), sum(amount)
FROM orders
GROUP BY ROLLUP (user_id, status);

-- CUBE: all combinations
SELECT user_id, status, count(*)
FROM orders
GROUP BY CUBE (user_id, status);

-- GROUPING SETS: explicit combinations
SELECT user_id, status, count(*)
FROM orders
GROUP BY GROUPING SETS (
    (user_id, status),
    (user_id),
    (status),
    ()
);
```

### DISTINCT vs. DISTINCT ON

```sql
-- DISTINCT: unique rows across all selected columns
SELECT DISTINCT status FROM orders;

-- DISTINCT ON: first row per group (PostgreSQL-specific!)
SELECT DISTINCT ON (user_id) id, user_id, amount, created_at
FROM orders
ORDER BY user_id, created_at DESC;
```

### DISTINCT ON details

`DISTINCT ON` returns the **first row** for each group defined by the ON expression. The ORDER BY determines which row is "first."

> **Trap**: `DISTINCT ON` without `ORDER BY` is non-deterministic. PostgreSQL will return an arbitrary row from each group. Always specify `ORDER BY` to control which row is selected.

```sql
-- WRONG: Which order is "first" for each user?
SELECT DISTINCT ON (user_id) * FROM orders;

-- RIGHT: Returns the latest order per user
SELECT DISTINCT ON (user_id) id, user_id, amount, created_at
FROM orders
ORDER BY user_id, created_at DESC;
```

### Query Analysis with EXPLAIN

```sql
-- Basic explain
EXPLAIN SELECT * FROM orders WHERE id = 42;

-- Explain with actual execution (runs the query!)
EXPLAIN ANALYZE SELECT * FROM orders WHERE id = 42;

-- Machine-readable formats for tooling
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT * FROM orders WHERE id = 42;
EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM orders WHERE id = 42;
```

**EXPLAIN output sections:**
- **Seq Scan**: Sequential scan — scanning entire table (bad for large tables)
- **Index Scan**: B-Tree index lookup (good, exact match)
- **Index Only Scan**: All needed columns are in the index (best)
- **Bitmap Heap Scan + Bitmap Index Scan**: Multiple index matches combined
- **Nested Loop**: For each outer row, probe inner (good for small inner sets)
- **Hash Join**: Build hash table on one side, probe with other (good for equi-joins)
- **Merge Join**: Sort both sides, merge (good for large sorted datasets)
- **Sort**: Explicit sort (can be expensive, may indicate missing index)
- **Materialize**: CTE or subquery evaluated once and cached
- **Gather**: Parallel query workers coordinator

```text
-- Example EXPLAIN ANALYZE output:
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.85..28.45 rows=5 width=24) (actual time=0.02..0.04 rows=5 loops=1)
   ->  Index Scan using idx_users_email on users u  (cost=0.42..8.44 rows=1 width=8)
         Index Cond: (email = 'alice@example.com'::text)
         (actual time=0.01..0.02 rows=1 loops=1)
   ->  Index Scan using idx_orders_user_id on orders o  (cost=0.43..19.96 rows=5 width=16)
         Index Cond: (user_id = u.id)
         (actual time=0.01..0.02 rows=5 loops=1)
 Planning Time: 0.18 ms
 Execution Time: 0.08 ms
```

### RETURNING Clause

PostgreSQL's `RETURNING` clause is extremely powerful — it returns the affected rows after DML operations. This avoids a follow-up `SELECT`.

```sql
-- Laravel-like pattern: create a record and get it back
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com')
RETURNING *;

-- Update records and return them
UPDATE orders SET status = 'shipped', shipped_at = now()
WHERE status = 'paid' AND id IN (101, 102, 103)
RETURNING id, status, shipped_at;

-- Delete old records and log them
DELETE FROM sessions WHERE expires_at < now()
RETURNING user_id, expires_at;
```

> **Trap**: CTEs in PG 12+ can be `MATERIALIZED` or `NOT MATERIALIZED`. By default, PostgreSQL inlines simple CTEs and materializes CTEs with side effects or expensive computations. If you have a CTE used multiple times in the query, forcing `MATERIALIZED` can prevent repeated evaluation. If you have a CTE used once, forcing `NOT MATERIALIZED` allows the planner to push down filters into the CTE. Use explicit directives when the planner makes the wrong choice.

> **Trap**: `LATERAL` subqueries are PostgreSQL's superpower for "for each row in the outer table, do this subquery." They're the equivalent of a correlated subquery but can return multiple columns and multiple rows. They're confusing to read but can eliminate N+1 queries in a single SQL statement. Always consider `LATERAL` when you need to do a "top N per group" query.

> **Follow-up**: "How did you reduce queries by 88%?" — We identified N+1 query patterns in Laravel. Where we had 100 queries fetching orders per user, we replaced them with a single query using `LATERAL` joins or `DISTINCT ON`. The `RETURNING` clause eliminated follow-up SELECTs after INSERT/UPDATE. Recursive CTEs replaced PHP-level tree traversals. Every endpoint was audited for query count reduction.

---

## 8. Transaction Isolation Levels

### Overview

PostgreSQL supports four isolation levels as defined by the SQL standard, but **not all behaviors match the standard exactly**.

```sql
-- Set isolation level for a transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- ... queries ...
COMMIT;

-- Snapshot control (REPEATABLE READ and SERIALIZABLE only)
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... queries: all see the same snapshot ...
COMMIT;
```

### Isolation Level Details

| Level | Implementation | Snapshot |
|---|---|---|
| `READ UNCOMMITTED` | Same as READ COMMITTED in PG | New snapshot per statement |
| `READ COMMITTED` | Default — statement-level snapshot | New snapshot per statement |
| `REPEATABLE READ` | Transaction-level snapshot | Single snapshot at first query |
| `SERIALIZABLE` | SSI (Serializable Snapshot Isolation) | Single snapshot + conflict detection |

### READ UNCOMMITTED

PostgreSQL ignores READ UNCOMMITTED — it behaves identically to READ COMMITTED. PostgreSQL physically cannot read uncommitted data because MVCC tuples are only visible when the writing transaction commits.

```sql
-- In PostgreSQL, this is the same as READ COMMITTED
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM users;  -- only sees committed data
COMMIT;
```

### READ COMMITTED

**Default isolation level.** Each statement within the transaction gets its own snapshot. If another transaction commits between our statements, we see the new data.

```sql
-- Transaction A
BEGIN;
SELECT amount FROM orders WHERE id = 1;  -- sees $100

-- Transaction B (concurrent)
UPDATE orders SET amount = 200 WHERE id = 1;
COMMIT;

-- Transaction A (same transaction, second statement)
SELECT amount FROM orders WHERE id = 1;  -- sees $200! (new snapshot)
-- This is the "non-repeatable read" anomaly
COMMIT;
```

**Anomalies allowed:**
- Dirty read: Prevented (PG can't read uncommitted data)
- Non-repeatable read: Allowed (demonstrated above)
- Phantom read: Allowed
- Lost update: Allowed (last concurrent UPDATE wins)

### REPEATABLE READ

**Transaction-level snapshot.** The first query establishes a snapshot, and all subsequent queries in the same transaction see the same data — even if other transactions commit.

```sql
-- Transaction A
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT amount FROM orders WHERE id = 1;  -- sees $100

-- Transaction B (concurrent)
UPDATE orders SET amount = 200 WHERE id = 1;
COMMIT;

-- Transaction A (same transaction)
SELECT amount FROM orders WHERE id = 1;  -- still sees $100! (repeatable read)
COMMIT;

-- But if A tries to UPDATE the same row:
UPDATE orders SET amount = 300 WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update
-- (A gets a serialization failure because the row has been modified since A's snapshot)
```

**Anomalies prevented:**
- Non-repeatable read: Prevented
- Phantom read: Prevented (same snapshot sees same rows)
- Lost update: Prevented (concurrent UPDATE causes serialization error)

**Anomalies still allowed:**
- Write skew: **Allowed** (see below)

### SERIALIZABLE

**True serializability via Serializable Snapshot Isolation (SSI).** PostgreSQL detects read-write conflicts between concurrent transactions using a dependency graph. If a cycle is detected, one transaction is aborted with error code `40001`.

```sql
-- Example write skew that REPEATABLE READ allows but SERIALIZABLE prevents
-- Scenario: Doctor on-call system
-- Rule: at least one doctor must always be on call
-- Table: doctors (id, name, on_call boolean)

-- Transaction A (Doctor 1 going off call)
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM doctors WHERE on_call = true;  -- 2
UPDATE doctors SET on_call = false WHERE id = 1;
COMMIT;

-- Concurrent Transaction B (Doctor 2 going off call)
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM doctors WHERE on_call = true;  -- 2
UPDATE doctors SET on_call = false WHERE id = 2;
COMMIT;

-- Result: both see 2 on-call, both update, now 0 on-call
-- REPEATABLE READ allows this! Only SERIALIZABLE detects the conflict
```

Under `SERIALIZABLE`, the transactions would be analyzed for rw-conflicts:
- T1 reads `on_call=true` set (includes T2's row)
- T2 reads `on_call=true` set (includes T1's row)
- T1 writes row 1
- T2 writes row 2
- → rw-dependency detected → one transaction aborted with `40001`

### Anomaly Comparison Table

| Anomaly | READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|---|---|---|---|---|
| Dirty read | Prevented* | Prevented | Prevented | Prevented |
| Non-repeatable read | Allowed | Allowed | Prevented | Prevented |
| Phantom read | Allowed | Allowed | Prevented | Prevented |
| Lost update | Allowed | Allowed | Prevented | Prevented |
| Write skew | Allowed | Allowed | Allowed | Prevented |

*PostgreSQL doesn't allow dirty reads at any isolation level because of its MVCC implementation.

> **Trap**: REPEATABLE READ in PostgreSQL does NOT prevent write skew. This differs from MySQL InnoDB, where REPEATABLE READ + gap locks can prevent certain write skew patterns. If your application logic depends on invariant enforcement across multiple rows, you need SERIALIZABLE (with retry logic) or explicit locking (`SELECT ... FOR UPDATE`).

> **Trap**: SERIALIZABLE isolation requires transaction retry logic. PostgreSQL aborts one of the conflicting transactions with error `40001`. Your application MUST catch this error and retry the transaction. In Laravel, this means wrapping serializable transactions in a retry loop:
```php
$maxRetries = 5;
for ($i = 0; $i < $maxRetries; $i++) {
    try {
        DB::transaction(function () {
            // serializable work
        }, 5);  // 5 retries for deadlock
        break;
    } catch (\Illuminate\Database\QueryException $e) {
        if ($e->getCode() !== '40001') throw;
        // retry
    }
}
```

> **Trap**: Long-running transactions under ANY isolation level increase the autovacuum tax. The oldest active XID in any transaction sets the cutoff for dead tuple cleanup. A 30-minute transaction blocks vacuum from reclaiming any tuples created during those 30 minutes. Set `statement_timeout` and `idle_in_transaction_session_timeout` to prevent this. For your trading platform, keep transactions as short as possible — seconds, not minutes.

> **Follow-up**: "Your trading platform uses REPEATABLE READ or SERIALIZABLE?" — SERIALIZABLE for the order matching engine. We accept the retry cost because correctness is non-negotiable. The read models and reporting queries use READ COMMITTED (default). We also use `SELECT ... FOR UPDATE` with `NOWAIT` to serialize access to specific account balances without the overhead of full SSI detection. Every transaction has a timeout and retry loop.

---

## 9. Roles, Permissions & Security

### Roles

PostgreSQL uses **roles** (not users). A role can be a user (LOGIN), a group, or both.

```sql
-- Create a login role (user)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secure_password';

-- Create a group role
CREATE ROLE read_only;
CREATE ROLE read_write;

-- Grant group membership
GRANT read_only TO app_user;

-- Create a superuser (avoid in production if possible)
CREATE ROLE admin WITH LOGIN SUPERUSER PASSWORD 'admin_pw';
```

**Role attributes:**

| Attribute | Description |
|---|---|
| `LOGIN` | Can connect to database |
| `SUPERUSER` | Bypasses all permission checks |
| `CREATEDB` | Can create databases |
| `CREATEROLE` | Can create/manage other roles |
| `REPLICATION` | Can connect in replication mode |
| `BYPASSRLS` | Bypasses Row-Level Security |
| `INHERIT` | Default — inherits permissions of granted roles |
| `CONNECTION LIMIT n` | Limits concurrent connections |

### GRANT

```sql
-- Schema-level
GRANT USAGE ON SCHEMA public TO app_user;
GRANT ALL ON SCHEMA public TO app_user;

-- Table-level
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO read_write;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO read_only;

-- Column-level (fine-grained)
GRANT SELECT (id, name, email) ON users TO support_agent;
-- support_agent cannot SELECT salary, ssn columns

-- Function-level
GRANT EXECUTE ON FUNCTION calculate_tax TO app_user;

-- Sequence-level
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO read_write;
```

### Default Privileges

`GRANT` only affects existing objects. New objects created after the GRANT have default permissions unless you use `ALTER DEFAULT PRIVILEGES`.

```sql
-- All future tables created by the migration user will be readable by app_user
ALTER DEFAULT PRIVILEGES FOR ROLE migration_user IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;

-- All future functions
ALTER DEFAULT PRIVILEGES FOR ROLE migration_user IN SCHEMA public
GRANT EXECUTE ON FUNCTIONS TO app_user;
```

> **Trap**: `ALTER DEFAULT PRIVILEGES` only applies to objects created **by the user specified in `FOR ROLE`**, not by other users. This is a common source of permission bugs: the migration user runs migrations, so default privileges must be set `FOR ROLE migration_user`. If you set them `FOR ROLE postgres`, they won't apply to tables created by `migration_user`.

### Row-Level Security (RLS)

RLS enables row-level access control — different roles see different subsets of rows in the same table.

```sql
-- Enable RLS on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create a policy
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::int);

-- For multi-tenant SaaS
CREATE POLICY tenant_select ON orders FOR SELECT
    USING (tenant_id = current_setting('app.tenant_id')::int);

CREATE POLICY tenant_insert ON orders FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);

CREATE POLICY tenant_update ON orders FOR UPDATE
    USING (tenant_id = current_setting('app.tenant_id')::int)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);

CREATE POLICY tenant_delete ON orders FOR DELETE
    USING (tenant_id = current_setting('app.tenant_id')::int);
```

Setting the tenant context in the application:

```php
// In Laravel middleware
DB::statement("SET app.tenant_id = {$tenantId}");
// All subsequent queries in this connection respect RLS
```

**Additional RLS controls:**

```sql
-- Force RLS even for table owner (default: owner bypasses RLS)
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Create permissive vs. restrictive policies
CREATE POLICY ... AS PERMISSIVE;   -- OR combined (default)
CREATE POLICY ... AS RESTRICTIVE;  -- AND combined
```

### SSL/TLS

```sql
-- postgresql.conf SSL settings
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
```

**Connection strings with SSL modes:**

```text
# sslmode options:
# disable       - no SSL
# allow         - prefer no SSL (tries SSL first)
# prefer        - try SSL first, fallback to no SSL (default)
# require       - SSL required, but no CA verification
# verify-ca     - SSL + verify server certificate
# verify-full   - SSL + verify server cert matches hostname

# Connection string
postgresql://user:pass@host:5432/db?sslmode=verify-full
```

### Password Encryption

```sql
-- Check encryption type
SHOW password_encryption;  -- scram-sha-256 (default since PG 13)

-- Set password (uses current password_encryption setting)
ALTER USER app_user PASSWORD 'new_password';

-- md5 is legacy — avoid
```

### pg_hba.conf

Host-Based Authentication controls which hosts/users/databases can connect and how they authenticate.

```text
# TYPE  DATABASE  USER      ADDRESS         METHOD
local   all       all                       peer
host    all       all       127.0.0.1/32    scram-sha-256
host    all       all       ::1/128         scram-sha-256
host    all       app_user  10.0.0.0/8      scram-sha-256
hostssl all       all       0.0.0.0/0       scram-sha-256
```

> **Trap**: RLS policies are bypassed by superusers and the table owner by default. If you implement RLS for multi-tenant isolation, make sure your application doesn't connect as superuser or table owner. Use a dedicated application role that is NOT the table owner, and set `ALTER TABLE t FORCE ROW LEVEL SECURITY` so even the owner must respect policies.

> **Trap**: `GRANT USAGE ON SCHEMA public TO app_user` gives access to the schema but NOT the tables inside it. You need separate `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user`. Schema USAGE is like "can see the directory" — table grants are "can open the files."

> **Trap**: `ALTER DEFAULT PRIVILEGES` only affects objects created AFTER the ALTER, and only by the user specified. If you run `ALTER DEFAULT PRIVILEGES FOR ROLE postgres ...` but your migrations run as `FOR ROLE migration_user`, the defaults won't apply. Run ALTER DEFAULT PRIVILEGES for each role that creates objects.

> **Follow-up**: "How did you implement multi-tenancy with RLS?" — Each tenant has an `organization_id` in every table. We set `app.tenant_id` via `SET` at the beginning of each request in Laravel middleware. RLS policies use `current_setting('app.tenant_id')::int` to filter rows. This gives us tenant isolation at the database level — even if a query somehow bypasses Laravel's scoping, RLS prevents data leakage. We use `FORCE ROW LEVEL SECURITY` so even the deployment user (table owner) can't see cross-tenant data without explicitly setting the parameter.

---

## 10. Tier 1 Q&A Drill

The following questions are designed to be answered out loud in 30-60 seconds. Practice each one until your answer is smooth and confident.

---

**Q1**: What is the difference between PostgreSQL's process model and MySQL's thread model?

**A1**: PostgreSQL uses process-per-connection — each client connection gets a forked backend process. MySQL InnoDB uses threads. This means PostgreSQL connections use more memory per connection (5-10 MB overhead each), making connection pooling essential beyond ~200 concurrent connections. The upside: better crash isolation. A PostgreSQL backend crash only kills that one connection; a MySQL thread crash can bring down the entire server. PostgreSQL uses shared memory for inter-process coordination; MySQL uses shared memory within the process.

---

**Q2**: What happens when you execute `UPDATE` in PostgreSQL (step by step)?

**A2**: (1) The backend acquires a row-level lock on the target tuple. (2) It writes a WAL record describing the change (WAL insert into the WAL buffer). (3) It marks the old tuple's `xmax` = current XID (effectively deleting it). (4) It inserts a new tuple with `xmin` = current XID, `xmax` = 0. This is a HOT update if no indexed columns changed and there's free space in the page. (5) WAL flush on commit. (6) On commit, the CLOG is updated to mark the XID as committed. (7) Future transactions check visibility against their snapshot — the old tuple is "dead" (xmax committed), the new tuple is "visible" (xmin committed).

---

**Q3**: What is the difference between `VACUUM` and `VACUUM FULL`?

**A3**: `VACUUM` (standard) removes dead tuples and marks the space as reusable within the table file. It does NOT return space to the OS. It only takes a SHARE UPDATE EXCLUSIVE lock (concurrent reads allowed). `VACUUM FULL` rewrites the entire table to a new file, returning space to the OS. It takes ACCESS EXCLUSIVE LOCK — blocking ALL reads and writes. **Never run `VACUUM FULL` on production.** Use `pg_repack` instead, or schedule it during explicit maintenance windows.

---

**Q4**: What is XID wraparound and how does PostgreSQL prevent it?

**A4**: PostgreSQL uses 32-bit transaction IDs. After ~4 billion transactions, the XID counter wraps around. Old tuples suddenly appear to be from the future. PostgreSQL prevents this by **freezing** tuples — marking them with a special frozen XID (2) that's always in the past. Autovacuum freezes tuples as part of its normal operation. If autovacuum falls behind and `age(datfrozenxid)` approaches `autovacuum_freeze_max_age` (200M), PostgreSQL forces an anti-wraparound vacuum. If this also fails and the age reaches 1M before wraparound, PostgreSQL shuts down to prevent data loss.

---

**Q5**: When would you use `TIMESTAMPTZ` vs `TIMESTAMP`?

**A5**: Always use `TIMESTAMPTZ` for application timestamps. It stores UTC internally and converts to the session timezone on display. `TIMESTAMP` stores a literal with no timezone conversion — changing the server's timezone changes the meaning of the stored value (because the same clock time represents a different absolute moment). The only case for `TIMESTAMP` is domain-specific: "store opens at 10:00 AM" where the wall-clock time is what matters, regardless of timezone.

---

**Q6**: What is the difference between `SERIAL` and `IDENTITY`?

**A6**: `IDENTITY` (SQL standard: `GENERATED BY DEFAULT AS IDENTITY`) is preferred over `SERIAL`. `SERIAL` is legacy — it creates a sequence implicitly that's not properly tied to the column. Dropping a column with `SERIAL` leaves the sequence orphaned. `IDENTITY` correctly drops the sequence when the column is dropped. Also, `SERIAL` is always `INTEGER` (4 bytes, ~2B max). You can specify `BIGINT GENERATED BY DEFAULT AS IDENTITY` for large tables.

---

**Q7**: What isolation level should you use for a financial transaction system like a trading platform?

**A7**: `SERIALIZABLE` for critical operations (order matching, balance transfers) to prevent write skew and other serialization anomalies. But you MUST implement retry logic because SERIALIZABLE aborts conflicting transactions with error 40001. For non-critical reads (reporting, read models), `READ COMMITTED` is fine. Use `SELECT ... FOR UPDATE` with `NOWAIT` to serialize access to specific hot rows without SSI overhead. Keep all transactions short — long-running transactions under any isolation level block autovacuum progress.

---

**Q8**: How does a GiST index differ from a GIN index?

**A8**: GiST is a balanced tree structure good for geometry, range types, and custom data types. It's smaller but slower for writes. GIN is an inverted index — each element maps to a list of rows. It's best for JSONB, arrays, and full-text search. GIN is faster for reads and bulk writes but larger. For full-text search specifically: GiST is good for small, read-heavy workloads; GIN is the default choice for any production full-text search. GIN also has better concurrency (FGD — Fast Update with pending list).

---

**Q9**: What causes table bloat in PostgreSQL and how do you prevent it?

**A9**: Table bloat is caused by dead tuples accumulating faster than VACUUM can clean them. Common causes: (1) Long-running transactions set the XID horizon — VACUUM can't reclaim tuples newer than the oldest active XID. (2) Autovacuum is too slow for the write rate — increase `autovacuum_vacuum_scale_factor` (make it smaller) or decrease `autovacuum_naptime`. (3) HOT updates aren't happening — set `fillfactor` below 100 (e.g., 90) to leave space for in-page updates. (4) Disabled autovacuum on some tables. Monitor `pg_stat_all_tables.n_dead_tup` and `n_live_tup` over time.

---

**Q10**: What is the visibility map and why does it matter?

**A10**: The visibility map tracks which pages in a table have all tuples visible to all current transactions. It's stored alongside the data file. Two benefits: (1) **Index-only scans** — if a page is all-visible, PostgreSQL can satisfy a query from the index alone without checking the heap. (2) **Partial vacuum** — VACUUM skips all-visible pages, dramatically speeding up vacuum on partially-modified tables. The visibility map is updated by VACUUM and can become outdated after crashes (but is repaired on the next VACUUM).

---

**Q11**: What is `full_page_writes` and when would you disable it?

**A11**: After a checkpoint, the first modification to any page writes the entire page image to WAL. This protects against partial page writes — if the server crashes while writing an 8KB page and only part of it reached disk, the full page image in WAL allows recovery to reconstruct the page before replaying WAL. Disabling it (`full_page_writes = off`) reduces WAL volume but risks data corruption on crash. The only safe time to disable is on a standby that will never be promoted, AND when the filesystem guarantees atomic page writes (e.g., ZFS, batteries-backed RAID controller). In general, never disable it.

---

**Q12**: How would you implement multi-tenant data isolation in PostgreSQL?

**A12**: Three strategies: (1) **Separate databases per tenant** — strongest isolation, hardest to manage, can't query across tenants easily. (2) **Separate schemas per tenant** — good isolation, shared connection pool, but more complex migrations. (3) **Shared schema with RLS** — all tenants in the same tables, RLS policies filter by `tenant_id`. This is what I'd recommend for most SaaS apps: lowest operational overhead, single connection pool, easy cross-tenant analytics (with a superuser role that bypasses RLS). Implement RLS with `current_setting('app.tenant_id')` and force it with `FORCE ROW LEVEL SECURITY`. The application sets the tenant context at the start of each request.

---

**Q13**: What does `EXPLAIN ANALYZE` show you and how do you read it?

**A13**: `EXPLAIN ANALYZE` executes the query and shows the execution plan with actual timings. Key items: (1) **cost** — planner's estimate (startup..total). Lower is better. (2) **actual time** — ms, (3) **rows** — actual rows vs planner's `rows= estimate`. If `actual rows` differs wildly from estimated, your table statistics are stale — run `ANALYZE`. (4) **loops** — how many times the node ran. A nested loop that runs 100,000 times is bad. (5) **Buffers** — shared hit (in cache) vs read (disk read). Watch for Seq Scan on large tables, high loop counts, and row count mismatches between planner estimates and actuals.

---

**Q14**: What is `random_page_cost` and how does it affect query planning on SSD?

**A14**: `random_page_cost` is the planner's cost estimate for a non-sequential disk fetch. Default is 4.0 (HDD era). On SSDs, random I/O is nearly as fast as sequential I/O — set it to 1.1. If left at 4.0, the planner overestimates the cost of index scans and may incorrectly choose sequential scans. This was a significant factor in the 88% query reduction — the planner was avoiding valid indexes because it thought random I/O was 4× slower than sequential. On an all-SSD infrastructure, `random_page_cost = 1.1` and `effective_cache_size = 75% of RAM` are essential starting points.

---

**Q15**: What is a partial index and when would you use one?

**A15**: A partial index indexes only a subset of rows that match a `WHERE` clause. It's dramatically smaller than a full index. Examples: (1) `CREATE INDEX idx_orders_pending ON orders (id) WHERE status = 'pending'` — only pending orders are indexed, the index is tiny and fast. (2) `CREATE INDEX idx_users_active ON users (email) WHERE active = true` — only active users. Use partial indexes when: you always query with the same filter, and the filtered set is much smaller than the full table. The planner uses them when the query's `WHERE` clause implies the index's filter condition. The space savings can be 90%+ compared to a full index on a low-cardinality column.

---

**Q16**: How do you monitor PostgreSQL performance? What views do you query?

**A16**: Core monitoring views:
```sql
-- Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle' ORDER BY duration DESC;

-- Cache hit ratio (should be >99%)
SELECT sum(blks_hit)::float / (sum(blks_hit) + sum(blks_read)) AS cache_hit_ratio
FROM pg_stat_database;

-- Index usage (find unused indexes to drop)
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0 ORDER BY tablename;

-- Table bloat (dead tuple ratio)
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::float / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000 ORDER BY dead_pct DESC;

-- Autovacuum activity
SELECT schemaname, relname, last_autovacuum, last_autoanalyze, vacuum_count, autovacuum_count
FROM pg_stat_user_tables
ORDER BY last_autovacuum NULLS FIRST;

-- Wait events
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL AND state != 'idle';
```

---

**Q17**: Your 15M record migration — how did you do it with zero downtime?

**A17**: We used a **batched, application-level migration** pattern. Key principles: (1) **Double-write** — old and new tables/columns are written simultaneously during the migration window. (2) **Backfill in batches** — 10,000 rows at a time using `COPY` or batched INSERTs with `id` range tracking. (3) **Verify each batch** — row counts, checksums, constraint validation. (4) **Incremental switchover** — read from new once backfilled, fallback to old if verification fails. (5) **Atomic cutover** — a single transaction renames tables or flips a feature flag. The actual pattern: `CREATE TABLE new, double-write to both, backfill old rows in batches with ORDER BY id LIMIT offset, verify, drop old`. PostgreSQL's `LISTEN/NOTIFY` was used to coordinate the cutover across application servers. The 15M records took ~3 hours of backfill with no user-facing downtime.
