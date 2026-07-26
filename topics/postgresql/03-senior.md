# PostgreSQL — Tier 3: Senior (Zero-Downtime Migrations, HA & Operations)

This is PostgreSQL at the senior level. You've been running it in production for years — your primary database, supporting multi-tenant SaaS, a trading platform, and millions of records. These are the topics that separate operational experience from textbook knowledge: zero-downtime migrations, high availability, performance tuning at scale, and the hard-won lessons from real incidents.

This tier assumes you know the fundamentals (Tier 1) and the advanced query/design patterns (Tier 2). Here we focus on keeping PostgreSQL alive, fast, and safe under production load.

---

## Table of Contents

1. [Zero-Downtime Migrations — The Complete Playbook](#1-zero-downtime-migrations--the-complete-playbook)
2. [CREATE INDEX CONCURRENTLY — Deep Dive](#2-create-index-concurrently--deep-dive)
3. [Advanced Backfill Strategies](#3-advanced-backfill-strategies)
4. [Performance Tuning — Memory & Checkpoint](#4-performance-tuning--memory--checkpoint)
5. [Autovacuum Tuning](#5-autovacuum-tuning)
6. [Patroni — High Availability with Auto-Failover](#6-patroni--high-availability-with-auto-failover)
7. [Backup & Recovery — pgBackRest, pg_dump, PITR](#7-backup--recovery-pgbackrest-pg_dump-pitr)
8. [Advanced Monitoring — pg_stat_statements & pg_stat_* Views](#8-advanced-monitoring--pg_stat_statements--pg_stat_-views)
9. [PostgreSQL vs MySQL — Deep Comparison for Senior Engineers](#9-postgresql-vs-mysql--deep-comparison-for-senior-engineers)
10. [Extension Ecosystem](#10-extension-ecosystem)
11. [PostgreSQL in the Cloud](#11-postgresql-in-the-cloud)
12. [Tier 3 Q&A Drill](#12-tier-3-qa-drill)

---

## 1. Zero-Downtime Migrations — The Complete Playbook

This is YOUR story. 15M+ records. Live traffic. A schema change that cannot block reads or writes. The naive approach — `ALTER TABLE ... ADD COLUMN` with a default — acquires `ACCESS EXCLUSIVE` lock, blocking all reads and writes for the duration of the table rewrite (minutes to hours at 15M rows). You cannot afford that.

### The Problem with ALTER TABLE

PostgreSQL adds a column with a non-null default by rewriting the entire table:

```sql
-- This will rewrite the entire table, holding ACCESS EXCLUSIVE lock
ALTER TABLE items ADD COLUMN warehouse_id INTEGER NOT NULL DEFAULT 1;
```

With 15M rows, that's a full table rewrite — every row is read, the new column is appended, and the old row is marked dead. During this operation, no other transaction can read or write to the table.

### The Expand → Migrate → Contract Pattern

The production-safe approach breaks the migration into three phases, each using the least-locking DDL available.

#### Expand

Add the column as nullable (metadata only in PG 11+, instant — no table rewrite):

```sql
ALTER TABLE items ADD COLUMN warehouse_id INTEGER;
```

In PG 11+, `ADD COLUMN` with no default is a metadata-only operation — it does not rewrite the table and completes in milliseconds regardless of table size.

Create the index concurrently (non-blocking):

```sql
CREATE INDEX CONCURRENTLY idx_items_warehouse_id ON items (warehouse_id);
```

Add the constraint as NOT VALID (no row validation, acquires no heavy lock):

```sql
ALTER TABLE items ADD CONSTRAINT fk_warehouse
    FOREIGN KEY (warehouse_id) REFERENCES warehouses (id) NOT VALID;
```

The `NOT VALID` clause means PostgreSQL records the constraint but does not scan existing rows to validate them. The `SHARE UPDATE EXCLUSIVE` lock only blocks other DDL — reads and writes continue normally. New rows are checked against the constraint; only pre-existing rows may violate it.

#### Migrate

Backfill the column in batches using keyset pagination:

```sql
-- Keyset pagination: stable, resumable, works under concurrent writes
SELECT id FROM items
WHERE id > :last_processed_id
ORDER BY id
LIMIT 1000
FOR UPDATE SKIP LOCKED;
```

```sql
-- Update batch using the paginated IDs
UPDATE items
SET warehouse_id = (
    SELECT id FROM warehouses
    WHERE warehouses.organization_id = items.organization_id
    ORDER BY id LIMIT 1
)
WHERE id = ANY(:batch_ids);
```

Rolling your own in Laravel:

```php
// Equivalent to chunkById with lazy loading — keyset pagination internally
Item::query()
    ->whereNull('warehouse_id')
    ->orderBy('id')
    ->chunkById(1000, function ($chunk) {
        foreach ($chunk as $item) {
            $item->update(['warehouse_id' => $item->organization->defaultWarehouse->id]);
        }
        // Update checkpoint after each batch
        Checkpoint::updateOrCreate(
            ['migration' => 'add_warehouse_id'],
            ['last_processed_id' => $chunk->last()->id]
        );
    });
```

Checkpoint tracking with a tracking table:

```sql
CREATE TABLE migration_checkpoints (
    id BIGSERIAL PRIMARY KEY,
    migration_name TEXT NOT NULL UNIQUE,
    last_processed_id BIGINT NOT NULL,
    rows_processed BIGINT DEFAULT 0,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    status TEXT DEFAULT 'running'
);
```

Replica-lag backpressure:

```php
while ($hasMore) {
    $lag = DB::select("
        SELECT GREATEST(
            COALESCE(pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn(), 0)
        ) AS lag_bytes
    ")[0]->lag_bytes;

    if ($lag > 100 * 1024 * 1024) {  // 100MB lag
        sleep(10);
        continue;
    }

    // Process next batch
    $batch = processBatch();
    $hasMore = $batch > 0;
}
```

#### Validate

Validation scans the table for constraint violations, holding only `SHARE UPDATE EXCLUSIVE` lock:

```sql
ALTER TABLE items VALIDATE CONSTRAINT fk_warehouse;
```

This acquires `SHARE UPDATE EXCLUSIVE` — it blocks other DDL (like `ALTER TABLE`) but does **not** block reads or writes. It must scan the full table, so expect I/O pressure.

```sql
-- Verify no violations before validating
SELECT COUNT(*) FROM items
WHERE warehouse_id IS NOT NULL
  AND NOT EXISTS (SELECT 1 FROM warehouses WHERE id = items.warehouse_id);
```

#### Contract

Now clean up the old column, old index, or set the column to NOT NULL:

```sql
ALTER TABLE items ALTER COLUMN warehouse_id SET NOT NULL;
ALTER TABLE items DROP COLUMN old_warehouse_code;
DROP INDEX CONCURRENTLY IF EXISTS idx_items_old_warehouse_code;
```

```sql
-- If you needed a NOT NULL constraint, use NOT VALID + validate pattern again
ALTER TABLE items ADD CONSTRAINT warehouse_id_not_null
    CHECK (warehouse_id IS NOT NULL) NOT VALID;

-- After backfill completes
ALTER TABLE items VALIDATE CONSTRAINT warehouse_id_not_null;

-- Then promote to true NOT NULL (table rewrite)
ALTER TABLE items ALTER COLUMN warehouse_id SET NOT NULL;
ALTER TABLE items DROP CONSTRAINT warehouse_id_not_null;
```

### Your 15M Migration — The Full Narrative

**Schema change**: Added `warehouse_id INTEGER` to `items` (15M rows), with a foreign key to `warehouses`. This was a multi-tenant SaaS platform — `items` belonged to organizations, and each item needed a default warehouse assignment.

**Backfill strategy**: Keyset pagination via `chunkById` (Laravel's internal keyset cursor). Batches of 1,000 rows. Checkpoint saved in `migration_checkpoints` table every 100 batches (~100K rows). Total backfill: ~4 hours.

**Rollback plan**: Full `pgBackRest` backup taken before migration. WAL archiving enabled. If anything went wrong, restore the backup + replay WAL to pre-migration point. Additionally, the new column was nullable — rolling back meant just dropping the column (instant, metadata-only).

**Validation**: Ran a pre-validation query to check for orphaned warehouse_ids. Then `VALIDATE CONSTRAINT`. The scan took ~8 minutes with some I/O pressure — we ran it during low traffic.

**Production cutover**: Application was deployed in two phases. Phase 1: deploy app code that writes to both old and new columns (dual-write). Phase 2: after backfill complete and validated, deploy code that reads from new column only. Phase 3: drop old column in next release cycle.

> **Trap**: `VALIDATE CONSTRAINT` requires a full table scan. On a 15M-row table this can take minutes and generates significant I/O. Plan accordingly — monitor disk read throughput. The NOT VALID constraint does **not** enforce against new rows (only validated rows are guaranteed, but new rows are indeed checked — actually, NOT VALID does check new rows; the trap is that existing rows are not checked until VALIDATE runs, so there CAN be violating existing rows that you must fix before validation). `CREATE INDEX CONCURRENTLY` can fail and leave an invalid index behind — you must detect and drop it.

> **Follow-up**: What happens if the backfill fails after 2 hours? — You restart from the last checkpoint; no data loss. What if the constraint validation fails? — Find the violating rows, fix them, try VALIDATE again. How do you handle the case where the application reads `NULL` warehouse_id during the backfill window? — The app must handle null gracefully (use a default at read time), or you use a partial index that excludes nulls during the backfill.

---

## 2. CREATE INDEX CONCURRENTLY — Deep Dive

Building an index on a live table without blocking writes.

### Syntax

```sql
CREATE INDEX CONCURRENTLY idx_items_org_sku ON items (organization_id, sku);
```

That's it. One keyword — `CONCURRENTLY` — turns a blocking operation into a non-blocking one.

### How It Works (Four Phases)

1. **Build**: PostgreSQL takes a snapshot of the table and builds the index from that snapshot. Writes the index to a temporary file (not yet visible).
2. **Catch-up**: A second table scan applies all changes that occurred since the snapshot was taken. This catches up the temporary index.
3. **Validation**: For unique indexes, validates that all entries are unique. This is a third scan.
4. **Swap**: The temporary index is renamed and made visible. The entry in `pg_index` is updated from `indisvalid = false` to `indisvalid = true`.

### Locks

`SHARE UPDATE EXCLUSIVE` — held during the entire operation. This lock does not block:

- `SELECT` (reads)
- `INSERT`, `UPDATE`, `DELETE` (writes)
- Other `CREATE INDEX CONCURRENTLY` commands

It does block:
- `DROP TABLE`, `TRUNCATE`
- `ALTER TABLE` (some variants)
- `VACUUM FULL`

### Failure Modes

```sql
-- Create an invalid index (build fails partway through)
CREATE INDEX CONCURRENTLY idx_bad ON items (bad_column);
-- ERROR: column "bad_column" does not exist
-- But wait — what if it's a validation failure instead?
```

```sql
-- Unique constraint violation during validation
CREATE UNIQUE INDEX CONCURRENTLY idx_items_sku_unique ON items (sku);
-- ERROR: could not create unique index "idx_items_sku_unique"
-- DETAIL: Key (sku)=(ABC-123) is duplicated.
```

On failure, the index is left in `INVALID` state:

```sql
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE NOT indisvalid AND indrelid = 'items'::regclass;

-- Returns: idx_items_sku_unique | f
```

You must manually drop and retry:

```sql
DROP INDEX CONCURRENTLY idx_items_sku_unique;
-- Fix the duplicate data
-- Then retry
CREATE UNIQUE INDEX CONCURRENTLY idx_items_sku_unique ON items (sku);
```

### Transaction Considerations

**Cannot run inside a transaction block:**

```sql
BEGIN;
CREATE INDEX CONCURRENTLY idx ON items (col);  -- ERROR!
COMMIT;
```

`CREATE INDEX CONCURRENTLY` requires auto-commit because it needs to commit its progress across multiple transactions internally. If you try to wrap it in `BEGIN/COMMIT`, PostgreSQL rejects it.

### Monitoring

```sql
SELECT phase, tuples_total, tuples_done, locks_waiting
FROM pg_stat_progress_create_index;
```

Useful phases:
- `initializing`
- `waiting for writers before build`
- `building index: scanning table`
- `building index: sorting index tuples`
- `building index: writing index`
- `waiting for writers before validation`
- `index validation: scanning index`
- `index validation: sorting index tuples`
- `index validation: scanning table`
- `waiting for old snapshots`
- `waiting for readers before marking dead`
- `cleaning up`

### Your 15M Migration Connection

> "We needed to add a unique index on `(organization_id, sku)` to `items` with 15M rows serving live traffic. We used `CREATE INDEX CONCURRENTLY` in a maintenance window with retry logic — it took 12 minutes, non-blocking. The monitoring query showed `building index: scanning table` for most of that time."

Your 88% story also involved concurrent indexes:

> "Adding a covering INCLUDE index concurrently for the dashboard aggregation query eliminated the seq scan and reduced query time from 310ms to 30ms."

```sql
CREATE INDEX CONCURRENTLY idx_orders_dashboard
ON orders (organization_id, created_at)
INCLUDE (total_amount, status);
```

> **Trap**: Cannot run inside a transaction — surprising if you're used to wrapping DDL in transactions for safety. Costs more total work (two full table scans for non-unique, three for unique). Leaves `INVALID` indexes on failure that must be monitored and cleaned. Can deadlock with concurrent DDL (use `lock_timeout` to avoid indefinite waits).

> **Follow-up**: What happens if a concurrent `INSERT` conflicts with the unique index validation? — The validation phase detects the conflict and fails the entire build. You must drop and retry. Can you build multiple indexes concurrently? — Yes, each `CREATE INDEX CONCURRENTLY` runs independently and acquires its own `SHARE UPDATE EXCLUSIVE` lock, but they serialize on the lock. They don't run truly in parallel.

---

## 3. Advanced Backfill Strategies

Backfilling large tables without performance impact is a core senior skill.

### Keyset Pagination (Cursor-Based)

The only stable pagination method for backfilling under concurrent writes:

```sql
-- Correct: stable, resumable
SELECT id, col1, col2
FROM items
WHERE id > :last_processed_id
ORDER BY id
LIMIT 1000;
```

```sql
-- Wrong: OFFSET pagination is unstable — new rows shift the window
SELECT id, col1, col2
FROM items
ORDER BY id
LIMIT 1000 OFFSET :offset;  -- DON'T DO THIS
```

Why OFFSET is bad: when new rows are inserted with lower IDs (or you're not ordering by a stable column), the OFFSET shifts and you either miss rows or process duplicates.

### Tracking Progress

```sql
CREATE TABLE migration_checkpoints (
    id BIGSERIAL PRIMARY KEY,
    migration_name TEXT NOT NULL UNIQUE,
    last_processed_id BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ,
    rows_processed BIGINT DEFAULT 0,
    checkpoint_at TIMESTAMPTZ DEFAULT NOW(),
    status TEXT DEFAULT 'running',
    error TEXT
);
```

```sql
-- Update checkpoint
INSERT INTO migration_checkpoints (migration_name, last_processed_id, rows_processed)
VALUES ('add_warehouse_id', :last_id, :rows)
ON CONFLICT (migration_name)
DO UPDATE SET
    last_processed_id = EXCLUDED.last_processed_id,
    rows_processed = migration_checkpoints.rows_processed + EXCLUDED.rows_processed,
    updated_at = NOW();
```

### Resumability

On restart, resume from last checkpoint:

```sql
SELECT COALESCE(
    (SELECT last_processed_id FROM migration_checkpoints
     WHERE migration_name = 'add_warehouse_id' AND status = 'running'),
    0
) AS last_id;
```

```sql
SELECT id FROM items
WHERE id > :last_id
ORDER BY id
LIMIT 1000;
```

### Rate Limiting

```sql
-- Simple delay between batches
SELECT pg_sleep(0.1);  -- 100ms delay
```

```php
// Dynamic rate limiting
$batchTime = microtime(true) - $batchStart;
if ($batchTime < 0.5) {
    usleep((0.5 - $batchTime) * 1_000_000);  // Ensure at least 500ms per batch
}
```

### Replica-Lag Backpressure

```sql
-- Check replica lag in bytes
SELECT pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS lag_bytes;

-- Check in time (PG 10+)
SELECT GREATCE(0, EXTRACT(EPOCH FROM (NOW() - replay_lag))) AS lag_seconds
FROM pg_stat_replication
WHERE application_name = 'replica1';
```

```php
$lag = DB::select("
    SELECT COALESCE(
        pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()),
        0
    ) AS lag_bytes
")[0]->lag_bytes;

if ($lag > 500 * 1024 * 1024) {  // 500MB
    Log::warning("Replica lag too high ($lag bytes), pausing backfill");
    sleep(30);
    continue;
}
```

### Transaction Size

Small batches prevent:
- Long-running transactions (block `VACUUM` from reclaiming dead tuples)
- Excessive lock contention
- High memory usage for sorting

```sql
-- Good: 1,000 rows per batch
BEGIN;
UPDATE items SET warehouse_id = :wid WHERE id = ANY(:batch_ids);
COMMIT;

-- Bad: 100,000 rows per batch (long transaction, bloat risk)
BEGIN;
UPDATE items SET warehouse_id = :wid
WHERE id BETWEEN :start AND :end;  -- DON'T DO THIS FOR LARGE RANGES
COMMIT;
```

### Batching for Different Scenarios

**UPDATE batch — set column value:**

```sql
UPDATE items
SET warehouse_id = subquery.warehouse_id
FROM (
    SELECT id, (SELECT id FROM warehouses WHERE organization_id = items.organization_id LIMIT 1) AS warehouse_id
    FROM items
    WHERE id > :last_id
    ORDER BY id
    LIMIT 1000
) AS subquery
WHERE items.id = subquery.id;
```

**Backfill foreign key — set FK column to reference new parent:**

```sql
UPDATE line_items
SET order_id = orders.id
FROM orders
WHERE line_items.order_uuid = orders.uuid
  AND line_items.order_id IS NULL
  AND line_items.id > :last_id
ORDER BY line_items.id
LIMIT 1000;
```

**Backfill new table — INSERT INTO ... SELECT FROM:**

```sql
INSERT INTO item_warehouses (item_id, warehouse_id, created_at)
SELECT id, warehouse_id, NOW()
FROM items
WHERE warehouse_id IS NOT NULL
  AND id > :last_id
ORDER BY id
LIMIT 1000
ON CONFLICT (item_id) DO NOTHING;
```

### Your 15M Migration Connection

> "We backfilled 15M rows in batches of 1,000, using keyset pagination with `chunkById`, with a checkpoint saved every 100 batches. The backfill ran for ~4 hours with replica-lag backpressure pausing it during peak hours. We had a 'pause' switch so we could stop for incident response without losing progress."

> **Trap**: `UPDATE ... LIMIT` is not valid SQL in PostgreSQL. You must use a subquery with `ORDER BY` and `LIMIT`: `UPDATE ... SET ... WHERE id IN (SELECT id FROM ... ORDER BY id LIMIT ... FOR UPDATE SKIP LOCKED)`. Beware of lock ordering — `FOR UPDATE SKIP LOCKED` prevents batches from blocking each other but does not order by ID. Long transactions for large backfills cause bloat (dead tuples accumulate until the transaction commits). Always test the backfill on a replica first.

> **Follow-up**: How do you handle backfilling when there's no integer primary key (UUID)? — Use a `created_at` timestamp cursor: `WHERE created_at > :last AND created_at <= :next ORDER BY created_at, id`. What if the backfill needs to be paused midway? — The checkpoint system handles this; just kill the backfill process and restart from the checkpoint. How do you validate the backfill completed correctly? — `SELECT COUNT(*) FROM items WHERE warehouse_id IS NULL` should be 0.

---

## 4. Performance Tuning — Memory & Checkpoint

PostgreSQL's default configuration is designed for a small development database on a Raspberry Pi. In production, you must tune it.

### Memory Configuration

```ini
# postgresql.conf — Memory Settings

# 25% of RAM (16GB server → 4GB)
shared_buffers = 4GB

# 50-75% of RAM — tells planner how much cache is available
effective_cache_size = 12GB

# Per-operation sort/join memory (default 4MB — too low)
work_mem = 8MB

# For VACUUM, CREATE INDEX, ADD FOREIGN KEY
maintenance_work_mem = 1GB

# 1-2% of shared_buffers
wal_buffers = 64MB
```

**shared_buffers**: PostgreSQL's own cache. Setting it too high (> 40% of RAM) causes double caching — the OS also caches the same pages. Sweet spot is 25% of RAM.

**effective_cache_size**: Not a cache allocation — it's a planner hint. Tells the query planner how much cache is available (OS page cache + shared_buffers). Higher values make index scans look cheaper than sequential scans.

**work_mem**: Used per sort/join/hash operation per query. A query with 2 sort operations and 200 connections can consume `8MB * 2 * 200 = 3.2GB`. Increase carefully.

**maintenance_work_mem**: Used by VACUUM, CREATE INDEX, and ADD FOREIGN KEY. Raising to 1GB significantly speeds up index creation and vacuum on large tables.

**wal_buffers**: WAL write buffer. Default 16MB is usually fine, but for write-heavy workloads, increase to 1-2% of shared_buffers.

### Checkpoint Tuning

```ini
# Checkpoint settings

# Total WAL size between checkpoints (increase to reduce frequency)
max_wal_size = 20GB

# Minimum WAL size between checkpoints
min_wal_size = 5GB

# Maximum time between checkpoints
checkpoint_timeout = 15min

# Spread checkpoint writes over this fraction of checkpoint interval
checkpoint_completion_target = 0.9

# Compress WAL (reduces I/O, adds CPU overhead)
wal_compression = on
```

**max_wal_size**: Default 1GB means checkpoints happen every ~1GB of WAL. On a write-heavy system, that's every few minutes — causing checkpoint write storms. Increase to 10-20GB.

**checkpoint_timeout**: Default 5 minutes. Even if WAL hasn't reached max_wal_size, a checkpoint fires at this interval. Increase to 15-30 minutes.

**checkpoint_completion_target**: Default 0.5 — checkpoint writes are spread over 50% of the checkpoint interval. Setting to 0.9 spreads writes over 90%, smoothing I/O. But be careful: if a checkpoint takes too long to complete, the next one starts overlapping.

**wal_compression**: Reduces WAL volume by ~50-70% at the cost of CPU overhead. Worth it on I/O-bound systems.

### Planner Cost Constants for SSD

```ini
# Planner cost constants — tune for SSD storage

# Default 4 (HDD random seek cost). SSD: 1.0-1.5
random_page_cost = 1.5

# Default 1. SSD: 200-500 (SSDs handle concurrent I/O well)
effective_io_concurrency = 200
```

**random_page_cost**: The planner uses this to compare index scans (random I/O) vs sequential scans. Default 4 is for HDDs where a random seek costs ~4x a sequential read. On SSD, random reads are nearly as fast as sequential — set to 1.0-1.5.

**effective_io_concurrency**: How many concurrent I/O operations the system can handle. Default 1 (single HDD). SSD can handle 200+ concurrent operations.

### Your 88% Story Connection

> "After the query optimization (N+1 to join + covering index), we also tuned PG memory settings. We increased `shared_buffers` from 128MB to 4GB on the 16GB instance, bumped `work_mem` to 8MB, and set `random_page_cost` to 1.5 for SSDs. This further improved the dashboard latency from 310ms to ~200ms — the planner started using the covering index more aggressively."

> **Trap**: `work_mem` is per-operation, not per-session. A single query with 3 sort nodes uses 3x work_mem. 200 connections × 3 sorts × 50MB = 30GB — instant OOM. `shared_buffers` > 40% of RAM causes double caching (OS also caches the same pages). Checkpoint too frequent causes WAL write storms (I/O spikes every few minutes). Checkpoint too infrequent means crash recovery takes longer (must replay all WAL since last checkpoint).

> **Follow-up**: How do you calculate the right `shared_buffers`? — 25% of RAM is a good starting point. Monitor cache hit ratio in `pg_stat_bgwriter` — if `blks_read` / (`blks_hit` + `blks_read`) is below 99%, you may need more. What's the downside of too-large `checkpoint_completion_target`? — If it's too close to 1.0, checkpoints may not complete before the next one starts, causing overlapping checkpoints and I/O storms.

---

## 5. Autovacuum Tuning

The most common operational pain point for PostgreSQL in production.

### Default Settings (Don't Work for Large Tables)

```ini
# Default autovacuum settings — designed for <10GB databases
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
autovacuum_vacuum_cost_limit = 200
autovacuum_vacuum_cost_delay = 20ms
autovacuum_naptime = 1min
autovacuum_max_workers = 3
```

The problem: `scale_factor = 0.2` means vacuum triggers when 20% of rows are dead. For a 10M-row table, that's **2 million dead tuples** before vacuum runs. By that time, you have significant bloat and query performance degradation.

### Per-Table Tuning

```sql
-- Aggressive vacuum for hot write tables
ALTER TABLE inventory_items SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_vacuum_cost_limit = 1000,
    autovacuum_vacuum_cost_delay = 0
);

-- Less aggressive for read-heavy tables
ALTER TABLE historical_reports SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_vacuum_threshold = 10000,
    autovacuum_vacuum_cost_limit = 200,
    autovacuum_vacuum_cost_delay = 20
);

-- Disable auto-analyze threshold override (use default)
ALTER TABLE inventory_items SET (
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_analyze_threshold = 1000
);
```

### Global Tuning

```ini
# postgresql.conf — Autovacuum Tuning for Large Databases

# More workers for many large tables
autovacuum_max_workers = 6

# More aggressive cost limit (allow more I/O)
autovacuum_vacuum_cost_limit = 2000

# Lower delay (less throttling)
autovacuum_vacuum_cost_delay = 10ms

# Check more frequently
autovacuum_naptime = 30s
```

### Monitoring Vacuum Activity

```sql
-- Check per-table vacuum status
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct,
    last_autovacuum,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Active vacuum progress
SELECT
    pid,
    datname,
    relid::regclass AS relation,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count,
    max_dead_tuples,
    num_dead_tuples
FROM pg_stat_progress_vacuum;
```

### Bloat Detection

```sql
-- Rough table bloat estimation
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(
        (n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0)) * 100,
        2
    ) AS dead_tuple_pct,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

A more precise bloat query uses `pgstattuple` extension:

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT * FROM pgstattuple('inventory_items');
-- Look for: dead_tuple_count, dead_tuple_percent, free_percent
```

### Your Multi-Tenant SaaS Connection

> "The `inventory_items` table was growing fast — each of our tenants had thousands of SKUs, and the update-heavy workload generated dead tuples quickly. Default autovacuum settings (scale_factor = 0.2) meant vacuum wouldn't fire until we had 2M+ dead tuples. By then, index bloat was causing seq scans and query slowdown. We set per-table vacuum scale factor to 0.01 for hot tables, increased autovacuum_max_workers to 6, and the problem disappeared."

> **Trap**: Not tuning autovacuum for large tables is the #1 operational mistake. The defaults are designed for small databases. Disabling autovacuum "for performance" is catastrophic — you WILL get bloat so severe that queries time out, and eventually you hit transaction ID wraparound (database shuts down). Too-aggressive vacuum on high-write tables causes CPU overhead from constant vacuuming — set `cost_delay = 0` only if you have the I/O budget. `autovacuum_max_workers` at default (3) is insufficient when you have many large tables — workers may be busy on small tables while a large table bloat.

> **Follow-up**: What happens when autovacuum can't keep up? — The table accumulates dead tuples, queries slow down due to index bloat, and eventually the table reaches `autovacuum_freeze_max_age` (200M transactions), forcing an aggressive anti-wraparound vacuum that is NOT rate-limited and will saturate I/O. How do you detect bloat without extensions? — Compare `pg_relation_size` to `n_live_tup * avg_tuple_width`, or use the `pgstattuple` extension. What's the relationship between autovacuum and long-running transactions? — Long-running transactions prevent vacuum from removing dead tuples that were created after the transaction started, causing bloat to accumulate.

---

## 6. Patroni — High Availability with Auto-Failover

Patroni is the de-facto standard for PostgreSQL HA in production (outside of managed cloud services).

### What Patroni Provides

- HA layer on top of streaming replication
- Automatic failover when primary fails
- Consensus via DCS (etcd, Consul, ZooKeeper)
- REST API for health checks and management
- Configuration management (PG parameters via Patroni config)
- `pg_rewind` integration for rejoining former primaries

### Architecture

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  etcd 1  │     │  etcd 2  │     │  etcd 3  │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     └────────────────┼────────────────┘
                      │
       ┌──────────────┼──────────────┐
       │              │              │
┌──────┴──────┐ ┌─────┴──────┐ ┌─────┴──────┐
│ Patroni/PG  │ │ Patroni/PG │ │ Patroni/PG │
│  Primary    │ │  Replica 1 │ │  Replica 2 │
└─────────────┘ └────────────┘ └────────────┘
```

### Key Concepts

**Leader election**: When the primary fails, replicas contact DCS. The first replica to acquire the leader lock becomes the new primary. Other replicas repoint their replication to the new primary.

**Synchronous replication**: Configurable number of sync standbys. With `synchronous_mode = true` and `synchronous_mode_strict = true`, writes commit only when confirmed by N sync standbys.

```yaml
# patroni.yml — synchronous replication
postgresql:
  parameters:
    synchronous_commit: "remote_write"
    synchronous_standby_names: "*"
```

**Switchover** (planned): Gracefully move the primary to a different node. No data loss. Zero downtime if applications use a proxy (HAProxy, pgBouncer) that reroutes connections.

```bash
patronictl -c /etc/patroni/patroni.yml switchover my-cluster
# Prompts for target, then gracefully demotes current primary and promotes replica
```

**Failover** (unplanned): Automatic when primary is unreachable. Detection via DCS leader key TTL (default 30s). Configurable retry and timeout.

```bash
# Manual failover (if needed)
patronictl -c /etc/patroni/patroni.yml failover my-cluster --master my-primary --candidate my-replica
```

### Configuration Example

```yaml
# /etc/patroni/patroni.yml
scope: my-cluster
namespace: /service/
name: pg-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.1:8008

etcd:
  host: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 500
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: on  # Required for pg_rewind
        shared_buffers: 4GB
        work_mem: 8MB
        random_page_cost: 1.5

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 10.0.0.0/8 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.1:5432
  data_dir: /data/postgresql
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: strongpassword
    superuser:
      username: postgres
      password: strongpassword
```

### Restoring After Split-Brain

`pg_rewind` allows a former primary to rejoin the cluster without a full rebuild:

```sql
-- Patroni uses this automatically:
pg_rewind --target-pgdata /var/lib/postgresql/14/main \
         --source-server='host=new-primary port=5432 user=postgres dbname=postgres'
```

Prerequisites: `wal_log_hints = on` or `data_checksums` enabled at initdb time.

### Comparison with Other HA Solutions

| Feature | Patroni | repmgr | Stolon |
|---------|---------|--------|--------|
| Consensus | DCS (etcd/Consul/ZK) | No consensus | K8s-native |
| Auto-failover | Yes | Yes | Yes |
| DCS dependency | Required | No | K8s API |
| pg_rewind | Yes | Yes | Yes |
| REST API | Yes | No | Yes |
| Complexity | Medium | Low | Medium |
| K8s native | No | No | Yes |
| Widespread use | Most common | Common | Niche |

### Your Chronos Connection

> "Patroni's leader election via etcd is similar to how Chronos uses Raft for leader election — both solve the same problem for different systems. In Chronos, the leader manages cron job scheduling; in Patroni, the leader manages PostgreSQL writes. The failure detection and lock-based handoff patterns are identical."

> **Trap**: Patroni requires DCS quorum — an etcd cluster of 3+ nodes minimum. If DCS is unavailable, Patroni cannot perform failover (the primary continues running, but automated failover is broken). Network partitions can cause multiple primaries if DCS fails — Patroni uses the DCS leader key as a fence, but in rare scenarios (DCS cluster itself partitions), both sides may think they have the lock. Patroni does NOT prevent data loss in async replication (unacknowledged writes on the old primary are lost on failover). Switchover is not instant — leader lock transfer takes seconds, and the old primary must gracefully shut down its connections.

> **Follow-up**: How does Patroni prevent split-brain? — Through the DCS leader key. Only one node can hold the leader key at a time. Before promoting, a replica must acquire the leader lock. The old primary, if still alive, will detect it lost the lock and demote itself. What's the RPO with async replication? — The last few transactions committed on the old primary before failure are lost (typically < 1 second). With synchronous replication, RPO = 0. Can you use Patroni without etcd? — Yes, Consul and ZooKeeper are supported. Consul is simpler to operate than etcd.

---

## 7. Backup & Recovery — pgBackRest, pg_dump, PITR

Backup strategy is a senior responsibility. You need to answer: what's our RPO (Recovery Point Objective) and RTO (Recovery Time Objective)?

### Physical vs Logical Backup

**Physical backup** (pgBackRest, pg_basebackup, WAL-G):
- Binary copy of the database files
- Supports PITR (Point-in-Time Recovery) via WAL archiving
- Faster restore (file-level copy)
- Must restore entire cluster
- Required for PITR

```bash
# pg_basebackup (built-in, basic)
pg_basebackup -h primary -D /backup/base -X stream -P -v

# pgBackRest (advanced, feature-rich)
pgbackrest --stanza=myapp --type=full backup
```

**Logical backup** (pg_dump, pg_dumpall):
- SQL-level dump (schema + data as SQL commands or custom format)
- Per-database or per-table granularity
- Slower backup and restore
- Cannot do PITR
- Good for selective export, migration to different PG version

```bash
# Logical backup — per-database
pg_dump -h primary -Fc -f myapp.dump myapp

# Restore
pg_restore -h new-server -d myapp -j 4 myapp.dump
```

### pgBackRest — Production Backup

```ini
# /etc/pgbackrest/pgbackrest.conf
[myapp]
pg1-path=/var/lib/postgresql/14/main
pg1-port=5432

[global]
repo1-path=/backup/pgbackrest
repo1-retention-full=4
repo1-retention-diff=14
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=strong-encryption-key

compress-type=zst
compress-level=6

process-max=4  # Parallel backup/restore
```

```bash
# Full backup (weekly)
pgbackrest --stanza=myapp --type=full backup

# Differential backup (daily) — captures changes since last full
pgbackrest --stanza=myapp --type=diff backup

# Incremental backup (hourly) — captures changes since last backup of any type
pgbackrest --stanza=myapp --type=incr backup

# List backups
pgbackrest --stanza=myapp info

# Restore to specific point in time
pgbackrest --stanza=myapp --type=time "--target=2026-07-25 14:30:00+00" restore
```

### WAL Archiving Configuration

```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'pgbackrest --stanza=myapp archive-push %p'
archive_timeout = 60  # Force archive every 60 seconds even on idle systems
```

```ini
# For restore — recovery.conf (PG 12) or postgresql.conf (PG 13+)
restore_command = 'pgbackrest --stanza=myapp archive-get %f "%p"'
recovery_target_time = '2026-07-25 14:30:00+00'
recovery_target_action = 'promote'
```

### PITR — Point-in-Time Recovery

```sql
-- Recover to just before a:
-- Dropped table
-- Accidental DELETE
-- Bad migration

-- Example: recover to 1 minute before disaster
pgbackrest --stanza=myapp --type=time \
    "--target=2026-07-25 14:29:00+00" \
    --target-action=promote \
    restore
```

Transaction ID recovery:

```sql
-- Find the transaction ID that dropped the table
SELECT xmin, xmax, cmin, cmax, * FROM pg_prepared_xacts;
-- Then recover to xid-1
-- Recovery configuration:
-- recovery_target_xid = '1234567'
-- recovery_target_inclusive = false
```

### WAL Cleanup

```bash
# On standby — clean archived WAL that's been applied
archive_cleanup_command = 'pg_archivecleanup /var/lib/pgsql/14/archive %r'

# Or use pgbackrest's built-in retention
pgbackrest --stanza=myapp expire
```

### Recovery Testing — Critical

```bash
# Monthly restore test script
#!/bin/bash
set -e

# Create temporary PG instance
mkdir -p /tmp/restore-test
pgbackrest --stanza=myapp --db-path=/tmp/restore-test/14/main restore

# Start temporary instance
pg_ctl -D /tmp/restore-test/14/main -l /tmp/restore-test/log start

# Run validation queries
psql -p 5433 -d myapp -c "SELECT count(*) FROM items"
psql -p 5433 -d myapp -c "SELECT tablename FROM pg_tables WHERE schemaname = 'public'"

# Verify data integrity
psql -p 5433 -d myapp <<EOF
DO $$
DECLARE
    row_count BIGINT;
BEGIN
    -- Check critical tables have expected row counts
    SELECT COUNT(*) INTO row_count FROM items;
    ASSERT row_count > 0, 'Items table is empty!';
END $$;
EOF

echo "Restore test PASSED at $(date)"
pg_ctl -D /tmp/restore-test/14/main stop
rm -rf /tmp/restore-test
```

### Your 15M Migration Connection

> "Before the migration, we took a full pgBackRest backup and archived WAL continuously with `archive_timeout = 60`. If the migration failed (corrupted data, constraint violation, wrong foreign key mapping), we could restore the backup + replay WAL to the point before the migration started — zero data loss. This gave us the confidence to proceed with the schema change under live traffic."

> **Trap**: Not testing backups is the #1 operational sin. A backup you've never restored is not a backup. Schedule monthly restore tests. pg_dump cannot do PITR — it's a logical snapshot at a point in time. Relying only on pg_dump means you lose all data since the last dump. pgBackRest stanza misconfiguration (wrong pg1-path, wrong port) means backup silently fails — always check `pgbackrest --stanza=myapp check`. Archive_command failure blocks WAL recycling — if WAL files can't be archived, PostgreSQL holds onto them, potentially filling the disk. Monitor `archive_command` success rate and use `archive_timeout` to ensure regular WAL segments are produced.

> **Follow-up**: What's the difference between incremental and differential backups? — Differential captures changes since the last full backup. Incremental captures changes since the last backup of any type (full, diff, or incr). Differential restores require only the last full + last diff. Incremental restores require the full + all incrementals in sequence. How do you minimize RTO? — Use pgBackRest's parallel restore (`process-max=4`), use delta restore (only overwrites changed files), and pre-warm the buffer cache after restore. Can you backup a standby? — Yes, with pgBackRest's `repo1-host` and `repo1-host-user` settings for backup from standby (reduces primary load).

---

## 8. Advanced Monitoring — pg_stat_statements & pg_stat_* Views

You can't fix what you can't measure. These views are your first line of defense.

### pg_stat_statements — Query Performance

Must-have extension:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Required in postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- (requires restart)
```

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
pg_stat_statements.save = on
```

**Top queries by total time:**

```sql
SELECT
    queryid,
    LEFT(query, 100) AS query_preview,
    ROUND(total_exec_time::NUMERIC / 1000, 2) AS total_seconds,
    ROUND(mean_exec_time::NUMERIC, 2) AS mean_ms,
    calls,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER (), 2) AS pct_total_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    temp_blks_written
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Top queries by mean time (slowest individual executions):**

```sql
SELECT
    queryid,
    LEFT(query, 100) AS query_preview,
    calls,
    ROUND(mean_exec_time::NUMERIC, 2) AS mean_ms,
    ROUND(max_exec_time::NUMERIC, 2) AS max_ms,
    rows,
    shared_blks_hit,
    shared_blks_read
FROM pg_stat_statements
WHERE calls > 10  -- Filter out one-off queries
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Top queries by frequency (most calls):**

```sql
SELECT
    queryid,
    LEFT(query, 100) AS query_preview,
    calls,
    ROUND(mean_exec_time::NUMERIC, 2) AS mean_ms,
    ROUND(total_exec_time::NUMERIC / 1000, 2) AS total_seconds,
    rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

**Queries with high temp file usage (disk sorts):**

```sql
SELECT
    queryid,
    LEFT(query, 100) AS query_preview,
    temp_blks_written,
    temp_blk_write_time,
    calls,
    ROUND(mean_exec_time::NUMERIC, 2) AS mean_ms
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 20;
```

### pg_stat_user_tables — Table-Level Activity

```sql
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_tup_hot_upd,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### pg_stat_user_indexes — Index Usage

```sql
-- Unused indexes (zero scans — consider dropping)
SELECT
    schemaname,
    tablename,
    indexrelname::regclass AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### pg_stat_activity — Current Query Activity

```sql
-- Current running queries
SELECT
    pid,
    state,
    query_start,
    NOW() - query_start AS duration,
    wait_event_type,
    wait_event,
    LEFT(query, 150) AS query,
    application_name,
    client_addr,
    backend_type
FROM pg_stat_activity
WHERE state = 'active'
  AND backend_type = 'client backend'
ORDER BY query_start;

-- Blocked queries (waiting for lock)
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_locks blocked
JOIN pg_locks blocking ON blocked.locktype = blocking.locktype
    AND blocked.database = blocking.database
    AND blocked.relation = blocking.relation
    AND blocked.page = blocking.page
    AND blocked.tuple = blocking.tuple
    AND blocked.pid != blocking.pid
WHERE NOT blocked.granted;

-- Idle in transaction (long-running transactions — cause bloat)
SELECT
    pid,
    state,
    NOW() - query_start AS idle_duration,
    LEFT(query, 150) AS query,
    application_name
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < NOW() - INTERVAL '5 minutes'
ORDER BY query_start;
```

### pg_stat_replication — Replica Health

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), write_lag) AS write_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lag) AS flush_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lag) AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag,
    backend_start,
    CASE WHEN pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lag) > 100 * 1024 * 1024
         THEN 'LAGGING'
         ELSE 'OK'
    END AS status
FROM pg_stat_replication;
```

### pg_stat_bgwriter — Checkpoint Behavior

```sql
SELECT
    checkpoints_timed,       -- Scheduled checkpoints (good)
    checkpoints_req,         -- Requested checkpoints (bad — WAL filled up before timeout)
    ROUND(100.0 * checkpoints_req / NULLIF(checkpoints_timed + checkpoints_req, 0), 2) AS req_pct,
    buffers_checkpoint,      -- Buffers written by checkpoint
    buffers_clean,           -- Buffers written by bgwriter
    buffers_backend,         -- Buffers written by backends (high = checkpoint is behind)
    maxwritten_clean,        -- Times bgwriter hit max limit
    buffers_alloc
FROM pg_stat_bgwriter;
```

A high `checkpoints_req` percentage (> 20%) means `max_wal_size` is too low — checkpoints are triggered by WAL filling up, not by timeout.

### pg_stat_progress_* — Long-Running Operations

```sql
-- Active vacuum
SELECT * FROM pg_stat_progress_vacuum;

-- Active create index
SELECT * FROM pg_stat_progress_create_index;

-- Active cluster
SELECT * FROM pg_stat_progress_cluster;
```

### Your 88% Story Connection

> "We installed `pg_stat_statements` in staging, ran the endpoint, and the top query by `total_time` was the per-row count query. That's how we found the N+1 — a query with 50M calls (one per row, times 50 pages) dominated the total time. We added it to production, enabled `track = all`, and within a day we had a ranked list of the worst queries. The covering index was added based on what we saw in `shared_blks_read` — the query was doing seq scans with millions of blocks read."

> **Trap**: `pg_stat_statements` resets on restart (or manually via `pg_stat_statements_reset()`). You need a monitoring tool (pgBadger, Datadog, Grafana) to persist and trend this data. Statement normalization cannot hide security-sensitive data — the query text is logged as-is. Don't store secrets in queries. High-call-count fast queries can accumulate more total time than a few slow queries — always sort by `total_time` when prioritizing optimization targets.

> **Follow-up**: How do you track query performance over time? — Use pgBadger for log analysis, or a monitoring tool that calls `pg_stat_statements` periodically and stores snapshots. How do you find queries that could use an index? — Look for queries with high `shared_blks_read` and low `shared_blks_hit` ratio. What's the difference between `blks_hit` and `blks_read`? — `blks_hit` is found in shared buffers, `blks_read` requires OS or disk read. A low hit ratio (< 99%) indicates memory pressure.

---

## 9. PostgreSQL vs MySQL — Deep Comparison for Senior Engineers

This is a common senior interview question. The answer should demonstrate depth — not just feature lists, but architectural tradeoffs.

### Architecture

| Aspect | PostgreSQL | MySQL (InnoDB) |
|--------|-----------|----------------|
| Process model | Process-per-connection | Thread-per-connection |
| Storage | Heap-based with MVCC via old tuples | Clustered index (PK = data) |
| Memory | Shared buffers + OS cache | Buffer pool + log buffer |
| Connection overhead | Higher (fork per connection) | Lower (thread per connection) |

### Storage and MVCC

**PostgreSQL** uses heap storage with MVCC via old row versions:

- Each update creates a new row version (dead tuple)
- Old versions remain until VACUUM reclaims them
- No clustered index by default (heap has no order; indexes reference CTIDs)
- `HOT (Heap-Only Tuple)` updates avoid index bloat when indexed columns aren't changed

**MySQL (InnoDB)** uses clustered index as primary storage:

- The primary key IS the table data
- Secondary indexes reference the primary key value
- MVCC via undo log (old versions in undo tablespace)
- No vacuum needed — undo is cleaned by purge threads

### Indexing

| Index Type | PostgreSQL | MySQL |
|-----------|-----------|-------|
| B-Tree | Yes (default) | Yes (default) |
| GiST | Yes | No |
| GIN | Yes | No |
| BRIN | Yes | No |
| SP-GiST | Yes | No |
| Hash | Yes | Yes (MEMORY only) |
| Full-text | tsvector/tsquery (advanced) | Built-in (less sophisticated) |
| Partial | Yes | No |
| Covering (INCLUDE) | Yes (PG 11+) | No (uses composite) |
| Expression | Yes | Yes (functional, PG 7+) |

### Replication

**PostgreSQL** — streaming WAL:
- Physical (block-level) replication via WAL
- Logical replication (table-level, PG 10+)
- Synchronous replication per-transaction (configurable)
- Cascading replication
- Bi-directional (via logical, PG 14+)

**MySQL** — binlog replication:
- Statement-based, row-based, or mixed
- Group replication (InnoDB Cluster)
- Semi-synchronous replication
- GTID-based replication

### Concurrency Control

**PostgreSQL**: MVCC with Serializable Snapshot Isolation (SSI):
- Serializable uses predicate locking (aborts on conflict — pessimistic)
- No gap locks
- True `SERIALIZABLE` isolation with SSI (PG 9.1+)

```sql
-- PostgreSQL SERIALIZABLE — aborts if conflict detected
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If another concurrent transaction touches overlapping predicate range:
-- ERROR: could not serialize access due to read/write dependencies among transactions
```

**MySQL**: MVCC with gap locks:
- `SERIALIZABLE` uses gap locks (blocks inserts in range)
- Lower concurrency under serializable isolation
- No SSI

```sql
-- MySQL SERIALIZABLE — uses gap locks, blocks inserts
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM accounts WHERE balance > 1000 FOR UPDATE;
-- No other transaction can INSERT with balance > 1000 until this commits
```

### JSON Support

**PostgreSQL** — `JSONB`:
- Binary format, no keyspace limits
- Indexed via GIN (full document, path, or expression indexes)
- Rich operators: `->`, `->>`, `#>`, `@>`, `?`, `?|`, `?&`
- JSON path queries (PG 12+)

```sql
CREATE INDEX idx_items_metadata ON items USING GIN (metadata jsonb_path_ops);

SELECT * FROM items WHERE metadata @> '{"color": "red"}';

-- JSON path (PG 12+)
SELECT * FROM items WHERE metadata @? '$.tags[*] ? (@ == "sale")';
```

**MySQL** — JSON:
- Binary optimized (stored as JSON internally)
- Indexed via generated columns + virtual indexes
- JSON path expressions
- Fewer operators than PostgreSQL

### Extensions

**PostgreSQL** has a rich extension ecosystem:
- PostGIS (spatial)
- pg_stat_statements (monitoring)
- pg_partman (partition management)
- pg_cron (scheduled jobs)
- pg_trgm (fuzzy string matching)
- pgvector (vector similarity)
- TimescaleDB (time-series)
- HypoPG (hypothetical indexes)
- pg_hint_plan (force query plans)
- dblink/postgres_fdw (foreign data wrappers)

**MySQL** has pluggable storage engines:
- InnoDB (default, ACID)
- MyISAM (no transactions, table-level locks)
- MEMORY (heap-based, volatile)
- NDB Cluster (distributed)

### Partitioning

**PostgreSQL**: Declarative partitioning (mature in PG 12-13):
- Range, List, Hash (PG 11+)
- Partition pruning
- Partition-wise join (PG 12+)

**MySQL**: Built-in partitioning (since 5.1):
- Range, List, Hash, Key, Columns
- Subpartitioning

### CTEs and Window Functions

Both support CTEs and window functions, but PostgreSQL has additional features:

- PostgreSQL: Recursive CTEs, `SELECT ... FROM (VALUES ...)` without a table
- PostgreSQL: `FILTER (WHERE ...)` clause for aggregates
- PostgreSQL: `DISTINCT ON` clause

```sql
-- PostgreSQL-only: FILTER clause
SELECT
    department_id,
    COUNT(*) FILTER (WHERE status = 'active') AS active_count,
    COUNT(*) FILTER (WHERE status = 'inactive') AS inactive_count
FROM employees
GROUP BY department_id;

-- PostgreSQL-only: DISTINCT ON
SELECT DISTINCT ON (department_id)
    department_id,
    employee_id,
    salary
FROM employees
ORDER BY department_id, salary DESC;
```

### When to Choose Each

**Choose PostgreSQL when:**
- Complex queries, joins, CTEs, window functions
- Data integrity is critical (constraints, SERIALIZABLE isolation)
- Extensibility needed (custom types, operators, extensions)
- Geospatial data (PostGIS)
- Full-text search (beyond basic)
- JSON/NoSQL-style documents with indexing
- Partial, covering, expression indexes

**Choose MySQL when:**
- Simpler, read-heavy workloads
- Need easy replication with many read replicas
- Compatibility required (WordPress, Magento, many PHP apps)
- Team has more MySQL expertise
- Need lightweight, easy-to-operate deployment
- Using managed MySQL (RDS, Aurora) with minimal tuning

### Your Experience Connection

> "My multi-tenant SaaS is on PostgreSQL because of its indexing flexibility — partial indexes per-tenant, covering indexes for reporting queries, GIN indexes for JSONB metadata — and CTEs/window functions for complex reporting. The trading platform uses PostgreSQL with `SERIALIZABLE` isolation and advisory locks for inventory operations. MySQL would have been a worse fit for both: the SaaS needed extensibility, the trading platform needed strict serializability without gap locks."

> **Trap**: Saying one is universally better than the other shows lack of senior perspective. They excel in different areas — choose based on workload, not dogma. Ignoring the MySQL ecosystem (tools, hosting, mature replication, performance schema) shows bias. Not knowing locking differences — MySQL's gap locks under `SERIALIZABLE` block inserts in ranges, which would be a problem for the trading platform. PostgreSQL's SSI aborts on conflict instead — better for some workloads but requires retry logic.

> **Follow-up**: How would you migrate a MySQL app to PostgreSQL? — Use `pgloader` for schema and data migration, or logical replication via Confluent/Kafka for zero-downtime. Key differences to watch: MySQL's `AUTO_INCREMENT` → PostgreSQL `SERIAL`/`IDENTITY`, MySQL's `TINYINT(1)` → `BOOLEAN`, MySQL's `ON DUPLICATE KEY UPDATE` → `ON CONFLICT DO UPDATE`. What's harder to migrate? — Stored procedures (PL/pgSQL is different from MySQL's procedure language), full-text search queries use different syntax, and MySQL's `GROUP BY` is more lenient than PostgreSQL's strict `GROUP BY`.

---

## 10. Extension Ecosystem

PostgreSQL's extension system is a key differentiator. These are the must-know extensions:

### pg_stat_statements — Query Monitoring (Must-Have)

```sql
-- Essential: track query performance
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Requires shared_preload_libraries (restart required)
-- Tracks: calls, total_time, mean_time, min_time, max_time, rows,
--         shared_blks_hit/read, temp_blks_written, etc.
```

### pg_partman — Automated Partition Management

```sql
CREATE EXTENSION IF NOT EXISTS pg_partman;

-- Create a monthly partition set
SELECT partman.create_parent(
    p_parent_table := 'public.orders',
    p_control := 'created_at',
    p_type := 'native',
    p_interval := '1 month',
    p_premake := 3
);

-- Automatically maintain partitions (run via pg_cron)
SELECT partman.run_maintenance();
```

### pg_cron — Scheduled Jobs Inside PostgreSQL

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Schedule maintenance every hour
SELECT cron.schedule('hourly-vacuum', '0 * * * *',
    $$VACUUM ANALYZE inventory_items$$
);

-- Schedule partition maintenance nightly
SELECT cron.schedule('nightly-partition', '0 2 * * *',
    $$SELECT partman.run_maintenance()$$
);
```

Requires `shared_preload_libraries = 'pg_cron'` and `cron.database_name = 'myapp'`.

### pgBackRest / WAL-G — Backup & Restore

Covered in detail in Section 7.

### PostGIS — Spatial & Geographic Objects

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

-- GeoIP location lookups
SELECT id, name
FROM stores
WHERE ST_DWithin(
    location,
    ST_MakePoint(-73.9857, 40.7484),  -- Empire State Building
    5000  -- 5km radius
);

-- Distance queries with spatial index
CREATE INDEX idx_stores_location ON stores USING GIST (location);
```

### pg_trgm — Fuzzy String Matching

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Trigram index for ILIKE queries
CREATE INDEX CONCURRENTLY idx_items_sku_trgm ON items USING GIN (sku gin_trgm_ops);

-- Faster ILIKE with trigram index
SELECT * FROM items WHERE sku ILIKE '%search%';

-- Similarity ranking
SELECT *, similarity(sku, 'seach') AS sim
FROM items
WHERE sku % 'seach'
ORDER BY sim DESC;
```

Your 88% story connection: "The ILIKE search on `items.name` was doing a seq scan because the table didn't have a trigram index. Adding a GIN trigram index with `gin_trgm_ops` turned the 300ms seq scan into a 2ms index scan."

### btree_gin / btree_gist — GIN/GiST Indexes for B-Tree Types

```sql
CREATE EXTENSION IF NOT EXISTS btree_gin;

-- Enables composite indexes mixing B-Tree and GIN columns
-- Useful for: GIN index with integer column + array column
CREATE INDEX idx_items_composite
ON items USING GIN (organization_id, tags);
```

### pg_hint_plan — Force Query Plan (Last Resort)

```sql
CREATE EXTENSION IF NOT EXISTS pg_hint_plan;

-- Force a specific plan when the planner gets it wrong
SELECT /*+ SeqScan(items) */ * FROM items WHERE organization_id = 1;
SELECT /*+ IndexScan(items idx_items_org_id) */ * FROM items WHERE organization_id = 1;
```

Requires `shared_preload_libraries = 'pg_hint_plan'`.

### pgvector — Vector Similarity Search

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id BIGSERIAL PRIMARY KEY,
    embedding vector(384)  -- 384-dimensional vector
);

-- Nearest neighbor search
SELECT * FROM embeddings
ORDER BY embedding <=> '[0.1, 0.2, ...]'  -- cosine distance
LIMIT 10;

-- IVFFlat index for approximate nearest neighbor (faster)
CREATE INDEX idx_embeddings ON embeddings USING IVFFLAT (embedding vector_cosine_ops) WITH (lists = 100);
```

### TimescaleDB — Time-Series Optimized

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Convert table to hypertable
SELECT create_hypertable('orders', 'created_at');

-- Time-series optimizations: auto-partitioning, compression, continuous aggregates
ALTER TABLE orders SET (timescaledb.compress);
SELECT add_compression_policy('orders', INTERVAL '7 days');

-- Continuous aggregate (materialized view that auto-updates)
CREATE MATERIALIZED VIEW hourly_orders
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', created_at) AS hour,
       organization_id,
       COUNT(*) AS order_count,
       SUM(total_amount) AS revenue
FROM orders
GROUP BY hour, organization_id;
```

### HypoPG — Hypothetical Indexes

```sql
CREATE EXTENSION IF NOT EXISTS hypopg;

-- Test an index without actually creating it
SELECT * FROM hypopg_create_index('CREATE INDEX ON items (organization_id, created_at)');

-- Run the query to see if the planner would use the hypothetical index
EXPLAIN SELECT * FROM items WHERE organization_id = 1 ORDER BY created_at;
```

### auto_explain — Log Slow Queries with Plan

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'   # Log queries slower than 500ms
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
```

No extension creation needed — loaded at startup via `shared_preload_libraries`.

### Your 88% Story Connection

> "We used `pg_trgm` with a GIN index for the ILIKE search as part of the 88% query reduction. The search was doing a seq scan on 15M rows. Adding `CREATE INDEX CONCURRENTLY idx_items_name_trgm ON items USING GIN (name gin_trgm_ops)` reduced the query from 300ms to 2ms."

> **Trap**: Not vetting extensions for production readiness — some extensions are experimental or unmaintained. Check the extension's support status, commit activity, and community usage before deploying to production. Extensions can block PG upgrades — they must be compatible with the new PG version. `pg_cron` and `pg_hint_plan` require `shared_preload_libraries` (PG restart needed). Some extensions require superuser access and are not available on RDS/Aurora (HypoPG, pg_hint_plan, pg_cron).

> **Follow-up**: What extensions are available on RDS? — pg_stat_statements, pg_trgm, postgis, btree_gin/btree_gist, pg_partman (not all), auto_explain, dblink/postgres_fdw. What's NOT available? — pg_cron, pg_hint_plan, hypopg, pg_repack, TimescaleDB (separate, available on RDS as managed service). How do you handle extension upgrades? — `ALTER EXTENSION ... UPDATE TO 'new_version'`. Check compatibility with PG version first.

---

## 11. PostgreSQL in the Cloud

### RDS for PostgreSQL

**Backups:**
- Automated snapshots (daily, retention up to 35 days)
- Automated WAL archiving (continuous, enables PITR)
- Manual snapshots (retained as long as needed)

```bash
# Create manual snapshot
aws rds create-db-snapshot \
    --db-instance-identifier my-postgres \
    --db-snapshot-identifier my-postgres-pre-migration

# Restore to point in time
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier my-postgres \
    --target-db-instance-identifier my-postgres-restored \
    --restore-time "2026-07-25T14:29:00+00:00"
```

**Multi-AZ (High Availability):**
- Synchronous standby in another AZ
- Automatic failover (~60-120 seconds)
- Standby is NOT accessible for reads (waste of resources)
- DNS update on failover (CNAME swap)

```bash
# Modify instance for Multi-AZ
aws rds modify-db-instance \
    --db-instance-identifier my-postgres \
    --multi-az true \
    --apply-immediately
```

**Read Replicas:**
- Up to 15 read replicas
- Cross-region and cross-account replication
- Can be promoted to standalone instance

```bash
# Create read replica
aws rds create-db-instance-read-replica \
    --db-instance-identifier my-postgres-replica-1 \
    --source-db-instance-identifier my-postgres \
    --region us-west-2
```

**Performance Insights:**
- Based on `pg_stat_statements`
- Visualizes top queries by wait, load, or time
- Retention: 7 days free, up to 2 years paid

**Parameter Groups:**
```ini
# Custom parameter group — values that differ from defaults
shared_buffers = {DBInstanceClassMemory*1/4}
effective_cache_size = {DBInstanceClassMemory*3/4}
work_mem = {DBInstanceClassMemory*1/32768}
maintenance_work_mem = {DBInstanceClassMemory*1/16}
random_page_cost = 1.5
effective_io_concurrency = 200
log_min_duration_statement = 500
```

**Storage Auto-Scaling:**
- Increases storage automatically when approaching limit
- Set maximum storage threshold
- No downtime during scaling for some storage types

### Aurora PostgreSQL

**Architecture:**
- PG-compatible, but NOT standard PostgreSQL
- Distributed storage (up to 128TB)
- 6-way replication across 3 AZs (6 copies of data)
- Compute and storage decoupled

```bash
# Create Aurora PostgreSQL cluster
aws rds create-db-cluster \
    --db-cluster-identifier my-aurora-cluster \
    --engine aurora-postgresql \
    --engine-version 14.6 \
    --master-username postgres \
    --master-user-password supersecret
```

**Aurora vs RDS PostgreSQL:**

| Feature | RDS PostgreSQL | Aurora PostgreSQL |
|---------|---------------|-------------------|
| Storage | EBS (block-level) | Distributed (shared) |
| Max storage | 64TB | 128TB |
| Replicas | Up to 15 async | Up to 15 (with 0-2ms lag, no redo) |
| Failover | 60-120s (EBS fail + WAL replay) | < 30s (no WAL replay needed) |
| Storage auto-scaling | Yes (10GB increments) | Yes (10GB increments) |
| Caching | Shared buffers only | Shared buffers + OS cache |
| Recovery | WAL replay needed | Instant (shared storage) |

**Aurora Limitations:**
- Fewer extensions (no pg_cron, pg_hint_plan, HypoPG)
- Some PG features missing or different (e.g., no `pg_stat_statements` reset? — actually supported)
- Higher cost than equivalent RDS instance
- Performance characteristic differences (buffer cache handling)

### Cloud SQL (GCP)

Similar to RDS — managed PostgreSQL with automated backups, failover, read replicas, and Point-in-Time Recovery.

### Migration Strategies to Cloud

**pg_dump/pg_restore** — small databases, some downtime:
```bash
pg_dump -h old-server -Fc -j 4 myapp > myapp.dump
pg_restore -h new-server -d myapp -j 4 myapp.dump
```

**Logical replication** — zero-downtime:
```sql
-- On source
CREATE PUBLICATION myapp_pub FOR TABLE items, orders, inventory_items;

-- On target
CREATE SUBSCRIPTION myapp_sub
CONNECTION 'host=old-server dbname=myapp user=replicator password=...'
PUBLICATION myapp_pub;
```

**DMS (AWS Database Migration Service)** — managed migration:
```bash
# Create replication instance
aws dms create-replication-instance ... --replication-instance-class dms.c5.large

# Create source and target endpoints
aws dms create-endpoint --endpoint-identifier source-pg ... --engine-name postgres
aws dms create-endpoint --endpoint-identifier target-pg ... --engine-name postgres

# Start migration task (full load + CDC)
aws dms create-replication-task ...
```

> **Trap**: RDS Multi-AZ is NOT a read replica — the standby is synchronous but inaccessible for reads. You're paying for compute you can't use. Aurora MySQL vs Aurora PostgreSQL — Aurora MySQL has more features and better maturity than Aurora PostgreSQL in some regions (check feature parity before choosing). RDS `max_connections` is tied to instance memory and cannot be overridden — formula: `LEAST({DBInstanceClassMemory/9531392}, 5000)` for most instances. `pg_hba.conf` is managed by AWS and cannot be customized beyond what parameter groups allow.

> **Follow-up**: How would you migrate a 2TB PostgreSQL database to RDS with minimal downtime? — Use logical replication (pglogical or native PG publication/subscription). Set up the target as a subscriber, let it catch up, then switch the application's connection string. Rollback is just changing the connection string back. What's the RTO for RDS Multi-AZ failover? — Typically 60-120 seconds. Aurora failover is under 30 seconds. Aurora global database provides cross-region failover. What limitations does Aurora PostgreSQL have compared to RDS? — Fewer extensions, some performance edge cases under specific workloads, and it's not open-source PostgreSQL (some behavior differences in edge cases).

---

## 12. Tier 3 Q&A Drill

### Question 1: Zero-Downtime Migration

**Q**: You need to add a NOT NULL column with a default to a 50M-row table serving live traffic. Walk through the approach.

**A**: Use the expand → migrate → contract pattern.
1. **Expand**: `ALTER TABLE t ADD COLUMN c INTEGER` (nullable, metadata-only, instant in PG 11+).
2. **Migrate**: Batch backfill using keyset pagination (`WHERE id > :last ORDER BY id LIMIT 1000`), with checkpoint tracking. Add a default at the application layer during backfill.
3. **Validate**: `ALTER TABLE t VALIDATE CONSTRAINT c_not_null` after verifying no nulls.
4. **Contract**: `ALTER TABLE t ALTER COLUMN c SET NOT NULL` (table rewrite — schedule this). Or, to avoid the rewrite: use a `CHECK (c IS NOT NULL) NOT VALID` constraint instead of true NOT NULL.

### Question 2: CREATE INDEX CONCURRENTLY Failure

**Q**: You ran `CREATE INDEX CONCURRENTLY` on a production table. It failed with an error. What happened and what do you do?

**A**: Two common failure modes:
1. **Unique constraint violation**: A duplicate key was found during the validation phase. The index is left in `pg_index` with `indisvalid = false`. Drop it, fix the duplicates, retry.
2. **Build failure**: The transaction aborted mid-build (e.g., deadlock). Again, an invalid index is left behind.

Check with `SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid AND indrelid = 'items'::regclass;`. Drop with `DROP INDEX CONCURRENTLY idx_name`, fix the issue, and retry.

### Question 3: Autovacuum Not Keeping Up

**Q**: Your `inventory_items` table has 10M rows and 3M dead tuples. Queries are slowing down. What do you do?

**A**: The default `scale_factor = 0.2` means vacuum only triggers at 2M+ dead tuples. Per-table tuning is needed:
```sql
ALTER TABLE inventory_items SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_vacuum_cost_limit = 2000,
    autovacuum_vacuum_cost_delay = 0
);
```

Also check:
- Is `autovacuum_max_workers` sufficient (default 3)? Increase to 6.
- Are there long-running transactions preventing vacuum from advancing? Check `pg_stat_activity` for idle-in-transaction queries.
- Run `VACUUM inventory_items` manually to clear the backlog, then let autovacuum take over with the tuned settings.

### Question 4: Patroni Failover

**Q**: Your Patroni primary crashed. Walk through what happens during failover and what you check after.

**A**: 
1. Patroni on the primary stops heartbeating to etcd.
2. etcd leader key TTL expires (default 30s).
3. Replicas detect the missing leader key and hold an election.
4. The replica with the most recent LSN acquires the leader lock first.
5. That replica promotes itself (runs `pg_ctl promote`).
6. Other replicas repoint their replication to the new primary.
7. Application connections must reconnect (via HAProxy/pgBouncer).

Post-failover checks:
- Verify the new primary is accepting writes.
- Check `pg_stat_replication` on the new primary for connected replicas.
- Identify why the old primary crashed (OOM? disk full? hardware failure?).
- Rejoin the old primary using `pg_rewind` once it's back online.

RPO with async replication: a few transactions may be lost. With synchronous replication: zero data loss.

### Question 5: pg_stat_statements Analysis

**Q**: Your dashboard endpoint is slow. Using `pg_stat_statements`, what do you look for?

**A**: 
```sql
-- 1. Top by total_time
SELECT LEFT(query, 100), total_exec_time, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- 2. Check for seq scans (high shared_blks_read, low shared_blks_hit)
SELECT LEFT(query, 100), shared_blks_read, shared_blks_hit, calls
FROM pg_stat_statements
WHERE shared_blks_read > 10000
ORDER BY shared_blks_read DESC LIMIT 10;

-- 3. Check for temp file spills (work_mem too low)
SELECT LEFT(query, 100), temp_blks_written
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC LIMIT 10;
```

If the dashboard query is N+1, you'll see one query with 50M+ calls and another with a few calls but high total time (the parent query). The fix: replace N+1 with a JOIN + covering index.

### Question 6: Backup Strategy

**Q**: Design a backup strategy with RPO of 5 minutes and RTO of 1 hour.

**A**: 
- **Full backup**: Weekly pgBackRest full backup (Saturday 2AM).
- **Incremental**: Daily differential, hourly incremental.
- **WAL archiving**: Continuous with `archive_timeout = 60` (RPO = ~1 minute of WAL).
- **Recovery testing**: Monthly restore test to verify RTO.
- **PITR**: Recover `pgbackrest --type=time "--target=2026-07-25 14:30:00+00"`.

For RTO of 1 hour: use pgBackRest's parallel restore (`process-max = 4`), keep backups on fast storage (SSD), and pre-warm after restore. Run recovery playbook drills to verify.

### Question 7: PostgreSQL vs MySQL Trade-offs

**Q**: Your CTO asks why we use PostgreSQL instead of MySQL. They heard MySQL is faster. What do you say?

**A**: "PostgreSQL and MySQL excel in different areas. For our workload — complex reporting queries with CTEs and window functions, JSONB with GIN indexes, partial indexes per-tenant, and the trading platform needing SERIALIZABLE isolation — PostgreSQL is the better choice. MySQL would be faster for simple key-value lookups and read-heavy workloads with minimal joins. If our workload was WordPress or a high-throughput key-value store, I'd recommend MySQL. But for OLAP-heavy SaaS with geospatial and JSON requirements, PostgreSQL's extensibility and feature set win."

### Question 8: Checkpoint Tuning

**Q**: Your PostgreSQL instance shows `checkpoints_req` at 60% of total checkpoints. Writes spike every 2 minutes. Diagnose and fix.

**A**: `checkpoints_req` (requested checkpoints) at 60% means checkpoints are triggered by WAL filling up, not by `checkpoint_timeout`. `max_wal_size` is too low — WAL fills up every 2 minutes, triggering a checkpoint. The write spikes are checkpoint writes flushing dirty buffers.

Fix: Increase `max_wal_size` from default 1GB to 10-20GB. Increase `checkpoint_timeout` from 5min to 15min. Set `checkpoint_completion_target` to 0.9 to spread writes. After tuning, `checkpoints_timed` should dominate (80%+).

### Question 9: Migration Rollback

**Q**: You're halfway through a backfill (7M of 15M rows processed). The backfill script crashes. What do you do?

**A**: Because we use keyset pagination with checkpoint tracking, the recovery is straightforward:
1. Check the tracking table for `last_processed_id`.
2. Verify the application writes to the new column are still working (no corruption).
3. Restart the backfill from the checkpoint: `WHERE id > :last_processed_id`.
4. Verify the row count: `SELECT COUNT(*) FROM items WHERE warehouse_id IS NULL` matches expected.

No data loss. No partial state. The checkpoint guarantees resumability.

### Question 10: SERIALIZABLE Isolation

**Q**: Your trading platform uses `SERIALIZABLE` isolation. You're seeing many serialization failures (40001 errors). How do you handle them?

**A**: PostgreSQL's SSI aborts one transaction when a serialization conflict is detected. The solution:
1. **Retry logic** in the application: catch `40001` and retry the transaction.
2. **Reduce conflict window**: keep transactions short (avoid user interaction inside transaction).
3. **Optimistic locking**: if reads far outnumber writes, consider `REPEATABLE READ` + application-level optimistic locking.
4. **Advisory locks**: for high-contention resources (e.g., inventory deduction), use `pg_advisory_xact_lock()` to serialize access explicitly.
5. **Monitor conflicts**: use `SELECT * FROM pg_stat_database_conflicts`.

```php
// Retry logic for serialization failures
$maxRetries = 3;
$retryDelay = 100; // ms

for ($i = 0; $i < $maxRetries; $i++) {
    try {
        DB::transaction(function () {
            // Execute trading operation
        });
        break;
    } catch (\Illuminate\Database\QueryException $e) {
        if ($e->getCode() !== '40001') throw;
        if ($i === $maxRetries - 1) throw;
        usleep($retryDelay * 1000 * ($i + 1)); // Exponential backoff
    }
}
```

### Question 11: Monitoring Bloat

**Q**: How do you detect table bloat, and what do you do about it?

**A**: Detection:
```sql
-- Quick check: dead tuple ratio
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Precise: pgstattuple
SELECT * FROM pgstattuple('inventory_items');
```

Fixes:
- **Tune autovacuum**: per-table `scale_factor`, `cost_limit`, `cost_delay`.
- **pg_repack**: rebuild table without locks (extension, requires superuser).
- **VACUUM FULL**: requires `ACCESS EXCLUSIVE` lock — schedule during maintenance.
- **CLUSTER**: reorder table by index (also `ACCESS EXCLUSIVE`).

Prevention:
- Keep autovacuum running fast enough (`cost_limit` high, `cost_delay` low).
- Avoid long-running transactions (prevent dead tuple cleanup).
- Monitor dead tuple ratio weekly.

### Question 12: Connection Pooling

**Q**: Your application opens 500 connections to PostgreSQL. The `max_connections` is 500. What problems do you expect?

**A**: 
- Each connection uses ~5-10MB of memory (backend process + sort memory). 500 connections × 8MB = 4GB just for connection overhead.
- `work_mem` is multiplied by connections: 8MB × 500 = 4GB potential sort memory.
- Context switching overhead from 500 processes.
- Connection storm when the app restarts (all 500 connect at once).

Solutions:
- **pgBouncer**: transaction-level pooling (reuses connections between transactions). Set pool size to 20-50 connections.
- **Connection pooling in app**: HikariCP, Laravel connection pool.
- **Reduce `max_connections`**: set to 100-200, let pgBouncer manage the rest.
- **Calculate memory budget**: `shared_buffers + (work_mem * max_connections * avg_sort_ops) + maintenance_work_mem + wal_buffers < RAM`.

### Question 13: pg_stat_statements Reset

**Q**: Your `pg_stat_statements` data was reset after a PostgreSQL restart. How do you prevent data loss?

**A**: `pg_stat_statements` is in-memory and resets on restart. Solutions:
1. **Monitoring tool**: Use Datadog, New Relic, or a scheduled task that snapshots `pg_stat_statements` to a history table every 5 minutes.
2. **pg_stat_statements_history**: Create a cron job:
```sql
CREATE TABLE query_stats_history AS SELECT * FROM pg_stat_statements WHERE false;
CREATE INDEX ON query_stats_history (snapshot_at);

-- Every 5 minutes:
INSERT INTO query_stats_history
SELECT NOW(), * FROM pg_stat_statements;
TRUNCATE pg_stat_statements;  -- Reset to avoid unbounded growth
```
3. **pgBadger**: Parse PostgreSQL logs for query analysis instead of relying on `pg_stat_statements` persistence.
4. **RDS Performance Insights**: If on RDS, Performance Insights persists this data.

### Question 14: Rollback Plan for Schema Migration

**Q**: You just deployed a schema migration that adds a column with a default. It broke a critical query. How do you roll back in production without downtime?

**A**: If you followed the expand → migrate → contract pattern:
- **Expand phase (ADD COLUMN nullable, metadata-only)**: Rollback is `ALTER TABLE items DROP COLUMN warehouse_id` — instant, metadata-only.
- **Migrate phase (backfill)**: Rollback is setting the column back to NULL: `UPDATE items SET warehouse_id = NULL`. The old column path is still used by the application (dual-write pattern).
- **Contract phase (SET NOT NULL, DROP OLD)**: Rollback requires restoring the old column from backup or replaying the migration forward.

Phase 1 and 2 are safe to roll back. Phase 3 requires a new forward migration to undo.

### Question 15: Concurrent DDL Risks

**Q**: You run `CREATE INDEX CONCURRENTLY` and `ALTER TABLE ADD COLUMN` at the same time. What happens?

**A**: `CREATE INDEX CONCURRENTLY` acquires `SHARE UPDATE EXCLUSIVE` lock. `ALTER TABLE ADD COLUMN` (no default) in PG 11+ acquires `ACCESS EXCLUSIVE` lock very briefly (metadata-only). They can deadlock if the order of lock acquisition conflicts.

Best practice: serialize DDL operations. Run them sequentially, not in parallel. Use `lock_timeout` to fail fast if a deadlock occurs:

```sql
SET lock_timeout = '10s';
CREATE INDEX CONCURRENTLY idx_items_org_id ON items (organization_id);
SET lock_timeout = '30s';  -- Reset to default
```

If a deadlock does occur, one of the operations is aborted. The `CREATE INDEX CONCURRENTLY` leaves an invalid index — clean it up and retry.
