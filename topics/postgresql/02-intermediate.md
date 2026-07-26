# PostgreSQL — Tier 2: Intermediate (EXPLAIN, Indexing, Locking & Replication)

This is the toolkit behind the **88% query reduction** story. Every technique here was battle-tested on a multi-tenant SaaS platform doing 15M+ row migrations and running a real-time trading engine. If Tier 1 was about knowing PostgreSQL, Tier 2 is about **diagnosing, fixing, and architecting** at scale.

**What you proved with this knowledge:**
- Took a dashboard endpoint from 520 queries → 63 → 10 → 1
- Cut database CPU from 94% → 28% with composite indexes and query rewrites
- Migrated 15M rows to a new PG version with **zero downtime** via logical replication
- Designed a multi-tenant schema where `organization_id` leads every index
- Built a trading order-matching engine that uses `SKIP LOCKED` and advisory locks
- Configured PgBouncer in transaction mode for 1000+ Laravel workers

**Senior Engineer signal:** You don't just know these features — you know *when not to use them* and *what breaks when you do*.

---

## Table of Contents

1. EXPLAIN ANALYZE — Deep Reading
2. Indexing Strategy for Multi-Tenant Apps
3. Query Optimization Patterns
4. Locking — Complete Reference
5. Advisory Locks
6. Deadlock Detection & Prevention
7. Replication — Streaming & Logical
8. Connection Pooling (PgBouncer)
9. Declarative Partitioning
10. Tier 2 Q&A Drill

---

## 1. EXPLAIN ANALYZE — Deep Reading

### Reading Plans

PostgreSQL's planner builds a tree of plan nodes. Each node has a cost and an output. You read plans **bottom-up, right-to-left** — the most deeply indented node executes first.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT i.*, o.name AS organization_name
FROM inventory_items i
JOIN organizations o ON o.id = i.organization_id
WHERE i.organization_id = 42
  AND i.status = 'active'
  AND i.created_at >= NOW() - INTERVAL '30 days'
ORDER BY i.created_at DESC
LIMIT 25;
```

```text
Limit  (cost=12.34..45.67 rows=25 width=128) (actual time=0.45..1.23 rows=25 loops=1)
  ->  Nested Loop  (cost=12.34..456.78 rows=250 width=128) (actual time=0.45..1.20 rows=25 loops=1)
        ->  Index Scan using idx_inventory_org_status_created on inventory_items i
              (cost=6.17..234.56 rows=250 width=100) (actual time=0.32..0.89 rows=250 loops=1)
              Index Cond: ((organization_id = 42) AND (status = 'active'::text) AND (created_at >= NOW() - '30 days'::interval))
        ->  Index Scan using organizations_pkey on organizations o
              (cost=0.29..0.88 rows=1 width=28) (actual time=0.01..0.01 rows=1 loops=250)
              Index Cond: (id = i.organization_id)
```

### Node Types

| Node | Description | When You See It |
|------|-------------|-----------------|
| **Seq Scan** | Full table scan — reads every page | No useful index, or table so small an index is overhead |
| **Index Scan** | Walks B-tree, then fetches heap pages | Non-covering filter or returning columns not in index |
| **Index Only Scan** | All needed columns in the index — no heap visit | Covering index or index includes queried columns |
| **Bitmap Index Scan + Bitmap Heap Scan** | Builds a bitmap of matching page locations, then fetches pages | Multiple conditions combined, or conditions return many rows scattered across pages |
| **Tid Scan** | Scans by tuple ID (ctid) | Rare — used by `UPDATE`/`DELETE` after index lookup, or `WHERE ctid = ANY(...)` |

### Join Strategies

| Join Type | When Planner Picks It | Risk |
|-----------|----------------------|------|
| **Nested Loop** | Small outer + indexed inner | If inner is **not** indexed, every outer row triggers a full scan — O(n*m) |
| **Hash Join** | Unindexed join — hash table built on inner | High memory usage on the hash table; `work_mem` spills to disk |
| **Merge Join** | Both inputs sorted (by index or explicit Sort node) | Sorting has cost; often slower than Hash for small sets |

**The 88% pattern:** Your correlated subquery caused a Nested Loop with 50 loops on an unindexed inner relation. Rewriting to `LEFT JOIN` with a covering index flipped it to a Hash Join.

```sql
-- Before (50 sequential scans — 520 queries)
SELECT *,
  (SELECT COUNT(*) FROM orders o WHERE o.item_id = i.id) AS order_count
FROM inventory_items i
WHERE i.organization_id = 42;

-- After (1 query, Hash Join, Index Only Scan)
SELECT i.*, COALESCE(o.order_count, 0) AS order_count
FROM inventory_items i
LEFT JOIN (
  SELECT item_id, COUNT(*) AS order_count
  FROM orders
  WHERE organization_id = 42
  GROUP BY item_id
) o ON o.item_id = i.id
WHERE i.organization_id = 42;
```

### Cost Model

```
total_cost = startup_cost + (rows * per_tuple_cost)
```

The planner picks the plan with the **lowest total_cost**. Startup cost is the cost before the first row is emitted (e.g., building a hash table, sorting). Total cost includes everything.

**Critical skill:** Compare `rows` (estimates) vs `actual rows` (after ANALYZE). A mismatch of 10x+ means the planner has bad statistics.

```sql
ANALYZE inventory_items;  -- refresh stats
```

### ANALYZE Output Fields

| Field | Meaning |
|-------|---------|
| `actual time=0.45..1.23` | First number = time to first row, second = time to last row |
| `rows=25` | Actual number of rows returned |
| `loops=250` | How many times this node executed |
| `buffers: shared hit=42 read=3 dirtied=1 written=0` | `hit` = from shared buffers (RAM), `read` = from OS/disk (physical I/O) |

**The real skill:** Multiply rows × loops to get the **true** row count. A node that says `rows=1 loops=100000` processed 100,000 rows, not 1.

### Identifying Problems

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Row estimate mismatch (rows=250, actual rows=25000) | Stale stats | `ANALYZE` or increase `default_statistics_target` |
| Seq Scan on 10M-row table returning 2 rows | Missing index, or stats say all rows match | Add composite index starting with filter column |
| Seq Scan on 10M-row table returning 5M rows | Correct — Seq Scan is fine for large fractions | Nothing (or consider partitioning) |
| Nested Loop on unindexed inner, 500K loops | Planner didn't find index on inner | Add index, or increase `work_mem` to encourage Hash Join |
| `Buffers: shared read` count high | Cache miss — working set doesn't fit in `shared_buffers` | Increase `shared_buffers`, check `pg_buffercache`, optimize query to touch fewer pages |
| Node with `loops=5000` | Plan is in a function/subquery executed many times | Rewrite to avoid row-by-row processing |

### EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)

For programmatic analysis:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT ...;
```

### auto_explain Module

Log slow queries with plans automatically — no more catching queries in the act.

```sql
-- In postgresql.conf or ALTER SYSTEM
-- shared_preload_libraries = 'auto_explain'
-- auto_explain.log_min_duration = '500ms'  -- log plans for queries >= 500ms
-- auto_explain.log_analyze = on
-- auto_explain.log_buffers = on
-- auto_explain.log_nested_statements = on  -- include functions
```

**Production-safe:** `auto_explain.log_min_duration = '1s'` — only logs the 1% of queries that matter.

<!-- TRAP -->
> **⚠️ Trap: ignoring the "loops" value**
>
> A plan node that says `rows=1 loops=100000` is processing ~~1 row — it is processing 100,000 rows total. Always multiply: `total rows = rows × loops`. I've seen people proudly claim "the index scan only returned 1 row" when it returned 100,000.
>
> **⚠️ Trap: `Buffers: shared read` vs `shared hit`**
>
> `shared read` means physical I/O. If `shared read` dominates `shared hit`, your cache hit ratio is poor — the working set doesn't fit in `shared_buffers`. This is not a query problem; it's a memory sizing problem. For multi-tenant SaaS, if every tenant's hot data doesn't fit in RAM, queries will be slow for *every* tenant.
>
> **⚠️ Trap: EXPLAIN ANALYZE executes the query**
>
> `EXPLAIN ANALYZE` runs the query to completion. Don't paste it into production thinking it's dry-run. For writes:
> ```sql
> BEGIN;
> EXPLAIN (ANALYZE, BUFFERS) UPDATE inventory_items SET quantity = 0 WHERE id = 1;
> ROLLBACK;
> ```
> This still acquires locks and writes WAL, but the ROLLBACK undoes it. Use with care.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "How would you identify which queries are causing the most total database time in a production system?"
>
> Use `pg_stat_statements` sorted by `total_time DESC` (or `total_exec_time` in PG 13+). Multiply `mean_time × calls` to get total impact. A query that runs 10,000× at 5ms each (50s total) is more impactful than a query that runs 10× at 2s each (20s total). Tie to the 88% story: we found the top 3 queries by total_time accounted for 72% of database CPU.

---

## 2. Indexing Strategy for Multi-Tenant Apps

### The Rule: organization_id Leads Every Index

Every query in a multi-tenant app starts with `WHERE organization_id = $1`. If your index doesn't start with `organization_id`, PostgreSQL can't use it for the equality filter — it will either Seq Scan or need a Bitmap Scan combining multiple indexes.

```sql
-- ✅ Correct: organization_id first
CREATE INDEX idx_items_org_status_created
  ON inventory_items (organization_id, status, created_at DESC)
  INCLUDE (sku, name);

-- ❌ Wrong: organization_id is second — useless for WHERE org_id = ?
CREATE INDEX idx_items_status_org
  ON inventory_items (status, organization_id);
```

### Composite Index Ordering

```
Equality (WHERE) → Range (>, <, BETWEEN) → Sort (ORDER BY) → INCLUDE (covering)
```

| Position | Purpose | Example |
|----------|---------|---------|
| **1st — Equality** | Exact match filter | `organization_id = ?`, `status = 'active'` |
| **2nd — Range** | Inequality or partial match | `created_at >= ?`, `amount < ?` |
| **3rd — Sort** | ORDER BY without extra sort | `id DESC` |
| **INCLUDE** | Covering columns for index-only scan | `sku`, `name`, `price` |

```sql
-- Covers: WHERE org_id = 42 AND status = 'active' AND created_at > '2024-01-01'
--   ORDER BY created_at DESC
-- Index Only Scan possible if we only select org_id, status, created_at, sku, name
CREATE INDEX idx_items_tenant
  ON inventory_items (organization_id, status, created_at DESC)
  INCLUDE (sku, name);
```

### Partial Indexes

Only index rows that match a condition — smaller, faster, less write overhead.

```sql
-- Only index active (non-deleted) items — WHERE deleted_at IS NULL is implicit
CREATE INDEX idx_items_active
  ON inventory_items (organization_id, status, created_at DESC)
  WHERE deleted_at IS NULL;

-- Tenant-specific low-stock alert index (tiny)
CREATE INDEX idx_org5_low_stock
  ON inventory_items (quantity)
  WHERE organization_id = 5 AND quantity < 10;
```

**88% story tie:** We had a `WHERE deleted_at IS NULL` condition on every query, but the index included millions of soft-deleted rows. Adding `WHERE deleted_at IS NULL` cut index size by 60% and improved scan speed proportionally.

### Covering Indexes with INCLUDE

When you `INCLUDE` columns, PostgreSQL can satisfy the query entirely from the index — no heap fetch. This is **Index Only Scan**.

```sql
-- Without INCLUDE: Index Scan on org_id, then heap fetch for sku and name
-- With INCLUDE: Index Only Scan — everything is in the index
CREATE INDEX idx_items_org_covering
  ON inventory_items (organization_id, status)
  INCLUDE (sku, name, quantity, price, created_at);
```

### Skip Scan (PG 15+)

Before PG 15, an index on `(organization_id, status)` required `organization_id` in the WHERE to be useful. PG 15's Skip Scan can skip distinct values of the leading column:

```sql
-- PG 15+ can use this index even without WHERE organization_id = ?
-- It scans distinct org_id values, then looks for status = 'active' within each
CREATE INDEX idx_items_org_status ON inventory_items (organization_id, status);

-- This query now uses Skip Scan:
SELECT * FROM inventory_items WHERE status = 'active';
```

This is useful for admin-wide queries, but not a replacement for tenant-scoped indexes.

### Finding Unused Indexes

```sql
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
    SELECT indexrelid FROM pg_index WHERE indisprimary
  )
ORDER BY pg_relation_size(indexrelid) DESC;
```

Drop unused indexes to speed up writes and reduce vacuum overhead.

<!-- TRAP -->
> **⚠️ Trap: index on (sku) alone in multi-tenant app**
>
> In a single-tenant app, `WHERE sku = 'ABC123'` uses an index on `(sku)` perfectly. In a multi-tenant app, every query filters by `organization_id`. An index on `(sku)` is scanned for every query, but PostgreSQL must then filter by organization_id — if many orgs have that SKU, it's a lot of useless rows. The correct index is `(organization_id, sku)`.
>
> **⚠️ Trap: redundant indexes**
>
> If you have `idx_org_status` on `(organization_id, status)` and `idx_org` on `(organization_id)`, `idx_org` is completely redundant (the first is a superset). PostgreSQL knows this and may still use the better index, but writes pay for both. Use `pg_stat_user_indexes` and drop the redundant one.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "How do you decide the column order in a composite index for a multi-tenant query like `WHERE org_id = ? AND status IN ('active', 'pending') AND created_at > ? ORDER BY id DESC LIMIT 10`?"
>
> Columns used in `=` or `IN` conditions go first (equality). `created_at > ?` is a range — it goes after the equality columns. `id DESC` is the sort column — if it matches the index direction, no explicit sort needed. So: `(organization_id, status, created_at, id DESC)`. If the query only selects columns in the index, add `INCLUDE` for index-only scans.

---

## 3. Query Optimization Patterns

### The 88% Story in Practice

Before:
```
Dashboard endpoint: 520 queries per request
  - 50 items × 10 relations each (lazy-loaded)
  - 1 × per-row COUNT subquery
  - 1 × per-row SUM subquery
  - Missing composite index → Seq Scan on 5M rows
  - Database CPU: 94%
```

After:
```
Dashboard endpoint: 1 query
  - Eager-loaded with LEFT JOIN + GROUP BY
  - withCount/withSum as subqueries
  - Composite covering index
  - Database CPU: 28%
  - Response time: 4.2s → 47ms
```

### Pattern: Avoid Function Wrapping on Indexed Columns

```sql
-- ❌ Bad: function on indexed column — cannot use index
SELECT * FROM orders
WHERE DATE(created_at) = '2024-01-15';

-- ✅ Good: range condition — can use index
SELECT * FROM orders
WHERE created_at >= '2024-01-15 00:00:00'
  AND created_at < '2024-01-16 00:00:00';
```

### LIKE / ILIKE Optimization

```sql
-- Requires pg_trgm extension and GIN index
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_items_sku_trgm ON inventory_items USING GIN (sku gin_trgm_ops);

-- Now ILIKE can use the trigram index
SELECT * FROM inventory_items
WHERE organization_id = 42
  AND sku ILIKE '%widget%';
```

The trigram index enables sub-string matching. Without it, `ILIKE '%term%'` forces a Seq Scan because the leading `%` prevents B-tree index usage.

### OR Optimization

```sql
-- ❌ Planner often guesses badly with OR on different columns
SELECT * FROM inventory_items
WHERE organization_id = 42
  AND (status = 'active' OR priority = 'high');

-- ✅ UNION — each branch can use its own index
SELECT * FROM inventory_items
WHERE organization_id = 42 AND status = 'active'
UNION
SELECT * FROM inventory_items
WHERE organization_id = 42 AND priority = 'high';
```

### NOT IN vs NOT EXISTS

```sql
-- ❌ NOT IN: slower, can return wrong results if subquery has NULLs
SELECT * FROM inventory_items i
WHERE i.id NOT IN (SELECT item_id FROM orders WHERE organization_id = 42);

-- ✅ NOT EXISTS: usually faster, NULL-safe
SELECT * FROM inventory_items i
WHERE NOT EXISTS (
  SELECT 1 FROM orders o
  WHERE o.item_id = i.id AND o.organization_id = 42
);
```

### DISTINCT vs GROUP BY

For deduplication, `GROUP BY` on the same columns is identical. For distinct on one column with other columns, use `DISTINCT ON`:

```sql
-- Latest order per item
SELECT DISTINCT ON (item_id) *
FROM orders
WHERE organization_id = 42
ORDER BY item_id, created_at DESC;
```

### Keyset Pagination (O(1))

Offset pagination scans all discarded rows. Keyset pagination uses the index to start from a known position.

```sql
-- ❌ Offset pagination — O(n) — scans discarded rows
SELECT * FROM inventory_items
WHERE organization_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 25 OFFSET 50000;

-- ✅ Keyset pagination — O(1) — uses index to jump
SELECT * FROM inventory_items
WHERE organization_id = 42
  AND (created_at, id) < ('2024-06-15 10:00:00', 5000)
ORDER BY created_at DESC, id DESC
LIMIT 25;
```

**Trade-off:** Keyset pagination can't jump to an arbitrary page number. For infinite scroll, this is perfect. For "page 1,341" links, you need offset or a different approach (or accept that nobody clicks page 1,341).

### Query Analysis with pg_stat_statements

```sql
-- Find the worst queries by total time
SELECT
  queryid,
  LEFT(query, 80) AS query_preview,
  calls,
  ROUND(total_exec_time::numeric, 2) AS total_time_ms,
  ROUND(mean_exec_time::numeric, 2) AS avg_time_ms,
  ROUND((total_exec_time / SUM(total_exec_time) OVER ()) * 100, 2) AS pct_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Table Health with pg_stat_user_tables

```sql
-- Dead tuples and bloat indicators
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 1) AS dead_pct,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

<!-- TRAP -->
> **⚠️ Trap: optimizing by slowest query, not most impactful**
>
> You find a query running at 5 seconds once an hour. You spend a day optimizing it to 50ms. Meanwhile, a query running 10,000 times a day at 30ms each (300s/day) is untouched. Optimize by `total_time = calls × avg_time`. Use `pg_stat_statements` sorted by total time.
>
> **⚠️ Trap: index false sense of security**
>
> Adding an index makes reads faster but writes slower (every INSERT/UPDATE/DELETE must update each index). On a write-heavy trading platform, a single insert writes to 8 indexes → 8× the write amplification. Monitor `pg_stat_user_indexes` and drop unused or redundant indexes.
>
> **⚠️ Trap: LIMIT/OFFSET with non-unique ORDER BY**
>
> If `ORDER BY created_at DESC` and multiple rows share the same timestamp, offset-based pagination can skip or duplicate rows as new rows are inserted. Always include a unique column (or combination) in ORDER BY: `ORDER BY created_at DESC, id DESC`.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "How would you debug a query that's fast in development but slow in production?"
>
> 1. Check if the query plan is different (stale stats — run `ANALYZE`)
> 2. Check if indexes exist in production (schema drift)
> 3. Check `shared_buffers` and cache hit ratio (`pg_stat_bgwriter`)
> 4. Use `auto_explain` to capture the production plan
> 5. Check for lock contention (`pg_locks`, `pg_blocking_pids()`)
> 6. Check if parameterized queries are getting generic plans (`plan_cache_mode`)

---

## 4. Locking — Complete Reference

### Table-Level Locks

Listed from weakest to strongest. A stronger lock conflicts with all weaker locks.

| Lock Mode | Acquired By | Conflicts With |
|-----------|-------------|----------------|
| `ACCESS SHARE` | `SELECT` | `ACCESS EXCLUSIVE` only |
| `ROW SHARE` | `SELECT FOR UPDATE` / `FOR SHARE` | `EXCLUSIVE`, `ACCESS EXCLUSIVE` |
| `ROW EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` | `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` |
| `SHARE UPDATE EXCLUSIVE` | `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` |
| `SHARE` | `CREATE INDEX` (non-concurrent) | `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` |
| `SHARE ROW EXCLUSIVE` | — | Everything except `ACCESS SHARE` |
| `EXCLUSIVE` | `REFRESH MATERIALIZED VIEW CONCURRENTLY` | Everything except `ACCESS SHARE` |
| `ACCESS EXCLUSIVE` | `ALTER TABLE`, `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`, `CLUSTER`, `REINDEX` | Everything |

### Lock Conflict Matrix

```
                      AS  RS  RE  SUE  S   SRE  E   AE
ACCESS SHARE         ✓   ✓   ✓   ✓    ✓   ✓    ✓   ✗
ROW SHARE            ✓   ✓   ✓   ✓    ✓   ✓    ✗   ✗
ROW EXCLUSIVE        ✓   ✓   ✓   ✓    ✗   ✗    ✗   ✗
SHARE UPDATE EXCL    ✓   ✓   ✓   ✗    ✗   ✗    ✗   ✗
SHARE                ✓   ✓   ✗   ✗    ✗   ✗    ✗   ✗
SHARE ROW EXCLUSIVE  ✓   ✗   ✗   ✗    ✗   ✗    ✗   ✗
EXCLUSIVE            ✗   ✗   ✗   ✗    ✗   ✗    ✗   ✗
ACCESS EXCLUSIVE     ✗   ✗   ✗   ✗    ✗   ✗    ✗   ✗
```

(✓ = compatible, ✗ = conflicting — wait)

### Row-Level Locks

| Lock | Acquired By | Blocks |
|------|-------------|--------|
| `FOR UPDATE` | `SELECT ... FOR UPDATE` | `UPDATE`, `DELETE`, `SELECT FOR UPDATE`, `SELECT FOR NO KEY UPDATE`, `SELECT FOR SHARE`, `SELECT FOR KEY SHARE` |
| `FOR NO KEY UPDATE` | `UPDATE` (weaker) | Same as above except `SELECT FOR KEY SHARE` |
| `FOR SHARE` | `SELECT ... FOR SHARE` | `UPDATE`, `DELETE`, `SELECT FOR UPDATE`, `SELECT FOR NO KEY UPDATE` |
| `FOR KEY SHARE` | `DELETE` (weakest) | `SELECT FOR UPDATE`, `FOR NO KEY UPDATE` |

### SKIP LOCKED

Skip rows that are already locked — returns only what's available. Used for job queues, trading order matching.

```sql
-- Pick next unprocessed order, skip locked ones
-- Used in your trading platform for order matching
UPDATE orders
SET status = 'processing', locked_at = NOW()
WHERE id = (
  SELECT id FROM orders
  WHERE status = 'pending'
  ORDER BY created_at ASC
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

### NOWAIT

Fail immediately if a row is locked — don't wait.

```sql
-- Fail fast if row is locked — no waiting
SELECT * FROM inventory_items
WHERE id = 42
FOR UPDATE NOWAIT;
```

### Lock Monitoring

```sql
-- See all current locks
SELECT
  pid,
  locktype,
  relation::regclass AS relation,
  mode,
  granted,
  pg_blocking_pids(pid) AS blocked_by
FROM pg_locks
WHERE NOT granted;

-- Find the blocking query
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

### lock_timeout

```sql
-- Set per-session — fail after 2 seconds instead of waiting forever
SET lock_timeout = '2s';
```

<!-- TRAP -->
> **⚠️ Trap: creating an index without CONCURRENTLY**
>
> `CREATE INDEX index_name ON table (column)` acquires `SHARE` lock, which blocks writes (`ROW EXCLUSIVE` conflicts). On a production table, this means your app stops accepting writes for the entire index build duration. Always use `CREATE INDEX CONCURRENTLY` on tables > 100k rows — it uses `SHARE UPDATE EXCLUSIVE` (only blocks other DDL and vacuum).
>
> ```sql
> CREATE INDEX CONCURRENTLY idx_items_org_status
>   ON inventory_items (organization_id, status);
> ```
>
> Trade-off: it takes longer and can fail (left in INVALID state — check with `pg_index.indisvalid`).
>
> **⚠️ Trap: FOR UPDATE waiting on uncommitted transactions**
>
> If Tx1 has UPDATEd row A but not committed, Tx2's `SELECT ... FOR UPDATE` on row A will wait until Tx1 commits or rolls back. This can cause chain blocking. Use `NOWAIT` or `SKIP LOCKED` to avoid the wait.
>
> **⚠️ Trap: confusing SKIP LOCKED with NOWAIT**
>
> `SKIP LOCKED` = skip locked rows, return what's available (may return 0 rows). `NOWAIT` = if any target row is locked, fail immediately with an error. They are different tools for different jobs.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "How would you debug a production deadlock?"
>
> 1. Set `log_lock_waits = on` and check `deadlock_timeout` (default 1s)
> 2. Enable deadlock logging: `log_line_prefix = '%t [%p]: [%l] '`
> 3. When deadlock occurs, PostgreSQL logs both queries and locks involved
> 4. Query `pg_stat_activity` to see blocked sessions
> 5. Query `pg_locks` with `pg_blocking_pids()` to trace the blocking chain
> 6. Fix: ensure consistent lock ordering in application code

---

## 5. Advisory Locks

Application-level locks managed by PostgreSQL — not tied to any table row.

### Session-Scoped

```sql
-- Acquire (blocks until available)
SELECT pg_advisory_lock(12345);

-- Do work...
-- ... transaction commits ...

-- Release manually (REQUIRED — or the lock leaks)
SELECT pg_advisory_unlock(12345);
```

**Danger:** If your application crashes or the connection pool reuses the connection, the lock leaks. Session-scoped locks survive transaction boundaries.

### Transaction-Scoped (PREFERRED)

```sql
-- Acquire, auto-released at COMMIT or ROLLBACK
BEGIN;
SELECT pg_advisory_xact_lock(12345);
-- Do work...
COMMIT;  -- lock released automatically
```

### Non-Blocking Versions

```sql
-- Returns true if lock acquired, false if not (no waiting)
SELECT pg_try_advisory_lock(12345);
SELECT pg_try_advisory_xact_lock(12345);
```

### Use Cases

```sql
-- 1. Distributed mutex — one import per organization at a time
BEGIN;
SELECT pg_advisory_xact_lock(42, 1);  -- (org_id, feature_code)

-- 2. Trading order serialization per instrument
BEGIN;
SELECT pg_advisory_xact_lock(instrument_id, crc32('order_matching'));
-- Match orders for this instrument...
COMMIT;

-- 3. Rate limiting — check if lock available before processing
SELECT pg_try_advisory_xact_lock(user_id, crc32('rate_limit_email'));
```

### Key Design: Two-Argument Form

Always use the two-argument form for namespacing:

```sql
SELECT pg_advisory_xact_lock(
  organization_id,
  hashtext('stock_deduction')  -- or use crc32 for smaller numbers
);
```

This prevents collisions between different lock domains.

<!-- TRAP -->
> **⚠️ Trap: session-scoped locks with connection poolers (PgBouncer)**
>
> PgBouncer in transaction mode reuses connections across transactions. If you take `pg_advisory_lock` (session-scoped), you must release it before the connection is returned to the pool — or the next transaction on that connection inherits the lock. Always use `pg_advisory_xact_lock` with PgBouncer.
>
> **⚠️ Trap: same key across different uses**
>
> If you use advisory lock key `42` for both "import process" and "email sending", they will block each other. Namespace your keys: `pg_advisory_xact_lock(user_id, hashtext('import'))` vs `pg_advisory_xact_lock(user_id, hashtext('email'))`.
>
> **⚠️ Trap: advisory locks are not row locks**
>
> Advisory locks protect application-level resources, not database rows. They don't prevent other transactions from modifying rows. They work *beside* MVCC, not within it. Use row locks (`FOR UPDATE`) for row-level protection.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "Why would you choose advisory locks over a separate locking service like Redis or ZooKeeper?"
>
> Advisory locks are transactional — they auto-release on COMMIT, which is impossible with external lock services (you'd need a distributed transaction or TTL-based lock with fencing tokens). For database-local coordination (same connection pool, same transaction boundary), advisory locks are simpler and safer. For cross-service coordination, use an external service.

---

## 6. Deadlock Detection & Prevention

### How PostgreSQL Detects Deadlocks

PostgreSQL builds a **wait-for graph**: Tx1 waits for a lock held by Tx2, Tx2 waits for lock held by Tx1 → cycle → deadlock.

The `deadlock_timeout` parameter (default 1 second) controls how often PostgreSQL checks for deadlocks. After waiting this long, it checks the wait-for graph. If it finds a cycle, it kills one transaction.

```sql
SHOW deadlock_timeout;  -- default: 1s
```

### Deadlock Example

```sql
-- Transaction 1
BEGIN;
UPDATE inventory_items SET quantity = quantity - 1 WHERE id = 1;  -- locks row 1
UPDATE inventory_items SET quantity = quantity + 1 WHERE id = 2;  -- waits for Tx2

-- Transaction 2 (concurrent)
BEGIN;
UPDATE inventory_items SET quantity = quantity - 1 WHERE id = 2;  -- locks row 2
UPDATE inventory_items SET quantity = quantity + 1 WHERE id = 1;  -- waits for Tx1

-- PostgreSQL detects deadlock, kills one transaction:
-- ERROR:  deadlock detected
-- DETAIL:  Process 12345 waits for ShareLock on transaction 67890; blocked by process 67890.
-- HINT:  See server log for query details.
```

### Prevention: Consistent Lock Ordering

```sql
-- ❌ Bad: different lock orders in different code paths
-- Path A: UPDATE id=1, then id=2
-- Path B: UPDATE id=2, then id=1

-- ✅ Good: always sort IDs ascending
UPDATE inventory_items
SET quantity = quantity - 1
WHERE id IN (1, 2)
ORDER BY id;  -- locks in consistent order
```

### Prevention: Keep Transactions Short

Every millisecond a transaction holds locks increases deadlock probability. The longer a transaction runs, the more locks it accumulates, and the more chances for another transaction to need one.

### Monitoring

```sql
-- Enable in postgresql.conf
log_lock_waits = on          -- log any wait > deadlock_timeout
deadlock_timeout = 100ms     -- reduce from 1s for high-throughput trading
log_line_prefix = '%t [%p] ' -- include timestamp and PID in logs
```

### Application Retry

PostgreSQL kills **one** deadlock victim with error code `40P01`. The application **must** retry:

```python
import psycopg2
from psycopg2 import errors

MAX_RETRIES = 3
for attempt in range(MAX_RETRIES):
    try:
        cursor.execute("UPDATE inventory_items SET quantity = ? WHERE id = ?", (qty, item_id))
        conn.commit()
        break
    except errors.SerializationFailure:
        conn.rollback()
        if attempt == MAX_RETRIES - 1:
            raise
        # Exponential backoff
        time.sleep(0.1 * (2 ** attempt))
```

<!-- TRAP -->
> **⚠️ Trap: not retrying on deadlock**
>
> PostgreSQL picks a deadlock victim arbitrarily. The victim's transaction is rolled back. If your application doesn't catch `40P01` and retry, the user sees an error. In a trading platform, this means lost orders. Always retry.
>
> **⚠️ Trap: deadlock_timeout of 1s in high-throughput systems**
>
> Default `deadlock_timeout = 1s` means PostgreSQL doesn't check for deadlocks for 1 second. During that second, blocked transactions accumulate. For a trading platform doing 1000 TPS, set `deadlock_timeout = 100ms` to detect and resolve deadlocks faster. Check with: `SHOW deadlock_timeout;`

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "What's the difference between a deadlock and a lock wait?"
>
> **Lock wait:** Tx1 holds lock A, Tx2 waits for lock A. This resolves when Tx1 releases lock A (commits or rolls back). This is normal and expected.
>
> **Deadlock:** Tx1 holds lock A and waits for lock B; Tx2 holds lock B and waits for lock A. Neither can proceed. PostgreSQL must kill one transaction to break the cycle.
>
> Monitoring: `pg_stat_activity.wait_event_type = 'Lock'` for lock waits; errors with `40P01` for deadlocks.

---

## 7. Replication — Streaming & Logical

### Streaming Replication

Primary ships WAL (Write-Ahead Log) segments to standby(s) in real-time.

```
Primary (WAL Sender process)
  │
  ├─▶ Standby 1 (WAL Receiver → WAL Apply)
  ├─▶ Standby 2 (WAL Receiver → WAL Apply)
  └─▶ Standby 3 (cascading → Standby 4)
```

```sql
-- On primary: check replication status
SELECT
  application_name,
  state,
  sync_state,
  write_lag,
  flush_lag,
  replay_lag,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_bytes_behind
FROM pg_stat_replication;
```

### Synchronous vs Asynchronous

| Mode | Behavior | RPO | Write Latency |
|------|----------|-----|---------------|
| **Asynchronous** | Primary doesn't wait | Potential data loss on primary crash | Low |
| **Synchronous** | Primary waits for standby(s) ack via `synchronous_standby_names` | RPO = 0 | Higher (network round trip) |

```sql
-- In postgresql.conf on primary
synchronous_standby_names = 'FIRST 2 (standby1, standby2)'
```

### Replication Slots

Prevent WAL deletion until the standby confirms consumption:

```sql
-- Physical slot (for streaming replication)
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Logical slot (for logical replication)
SELECT pg_create_logical_replication_slot('logical_slot', 'pgoutput');
```

**Critical:** If a standby using a slot is down for too long, WAL accumulates on the primary and fills the disk. Monitor slot lag.

### Cascading Replication

```
Primary → Standby A → Standby B
```

Standby B replicates from Standby A, not the primary. Reduces load on primary.

### Hot Standby

Standby accepts read-only queries:

```sql
-- On standby
SELECT * FROM inventory_items WHERE organization_id = 42;
-- Read-only queries work — no writes
```

### Logical Replication (PG 10+)

Publish/subscribe model — row-level replication, not WAL-level.

```sql
-- Publisher (source database)
CREATE PUBLICATION my_pub FOR TABLE inventory_items, orders;
ALTER PUBLICATION my_pub ADD TABLE organizations;

-- Subscriber (target database)
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=primary-host dbname=app user=replicator'
PUBLICATION my_pub;
```

**Key properties:**
- Row-level: only DML (INSERT, UPDATE, DELETE) — no DDL
- Column selection: `CREATE PUBLICATION my_pub FOR TABLE inventory_items (id, sku, quantity)`
- Cross-version: PG 12 → PG 16 works
- No DDL replication: schema changes must be applied manually on subscriber

**Use case — Zero-downtime migration (your 15M story):**

```
1. Set up logical replication from old PG to new PG
2. Let it catch up (15M rows sync in background)
3. Run app in dual-write mode (write to both, read from new)
4. Verify data consistency
5. Switch app connection string to new PG
6. Drop subscription
```

### pg_rewind

After a failover, the old primary may be ahead of the new primary. `pg_rewind` synchronizes it by replaying WAL from the new primary:

```bash
pg_rewind --target-pgdata /var/lib/postgresql/data \
         --source-server="host=new-primary dbname=app user=replicator"
```

<!-- TRAP -->
> **⚠️ Trap: streaming replication requires same major version**
>
> You cannot stream replicate PG 14 → PG 16. Both primary and standby must run the same major version. For cross-version replication, use logical replication.
>
> **⚠️ Trap: logical replication requires wal_level = logical**
>
> If `wal_level = replica` (default for replication), logical replication won't work. Must set `wal_level = logical` and restart PostgreSQL.
>
> **⚠️ Trap: DDL changes are not replicated by logical replication**
>
> `ALTER TABLE inventory_items ADD COLUMN price DECIMAL;` — must run this on both publisher and subscriber manually. If schema drifts, replication can break.
>
> **⚠️ Trap: cascading replication increases failover complexity**
>
> If Standby A (upstream of Standby B) fails, Standby B must re-point to the primary or another standby. Promotions require careful orchestration.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "How would you monitor replication lag in production?"
>
> Query `pg_stat_replication` for `write_lag`, `flush_lag`, `replay_lag`. Set up alerts when any lag exceeds a threshold (e.g., 10 seconds for async, 100ms for sync). Also monitor replication slot lag with `pg_replication_slots` — a slot with stalled consumer causes WAL accumulation and disk fill.
>
> For logical replication, check `pg_stat_subscription` and `pg_stat_subscription_stats` (PG 16+).

---

## 8. Connection Pooling (PgBouncer)

### Why Pool

PostgreSQL uses one OS process per connection. 500 connections = 500 processes × ~10MB each = 5GB+ RAM before any query runs. PgBouncer multiplexes many client connections through a smaller pool of PostgreSQL connections.

```
Clients (1000) → PgBouncer → PostgreSQL (100 connections)
```

### PgBouncer Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Session pooling** | One PG connection per client session — kept for the session's lifetime | Reduces connection setup cost (TCP + auth), but still high memory for long-lived sessions |
| **Transaction pooling** | One PG connection per active transaction — released to pool after COMMIT/ROLLBACK | **Best for web apps** — 1000 Laravel workers can share 100 PG connections |
| **Statement pooling** | Connection released after each statement | Rare — breaks most ORMs |

### PgBouncer Configuration

```ini
[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool configuration
pool_mode = transaction
default_pool_size = 100
max_client_conn = 1000
reserve_pool_size = 10
reserve_pool_timeout = 3

# Timeouts
server_idle_timeout = 300
query_wait_timeout = 10
```

### Transaction Pooling Restrictions

Because connections are returned to the pool after each transaction, **session-scoped state is lost**:

| Feature | Breaks? | Alternative |
|---------|---------|-------------|
| `SET SESSION` statements | ❌ | Use `SET LOCAL` in transaction, or connect directly |
| `LISTEN` / `NOTIFY` | ❌ | Use polling or message queue |
| Temporary tables | ❌ | Use regular tables with cleanup |
| Prepared statements (named) | ❌ | Use `ATTR_EMULATE_PREPARES = true` (Laravel) |
| `pg_advisory_lock` (session-scoped) | ❌ | Use `pg_advisory_xact_lock` |
| Cursors (`DECLARE ... FETCH`) | ❌ | Fetch in single query with LIMIT/OFFSET |

### Laravel with PgBouncer

```php
// config/database.php
'pgsql' => [
    'driver' => 'pgsql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '6432'), // PgBouncer port
    'database' => env('DB_DATABASE', 'forge'),
    'username' => env('DB_USERNAME', 'forge'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => 'utf8',
    'prefix' => '',
    'prefix_indexes' => true,
    'schema' => 'public',
    'sslmode' => 'prefer',
    // Critical for PgBouncer
    'options' => [
        PDO::ATTR_EMULATE_PREPARES => true,
    ],
],
// Sticky read connection — bypasses PgBouncer
'pgsql_direct' => [
    'driver' => 'pgsql',
    'host' => env('DB_HOST_DIRECT', '127.0.0.1'),
    'port' => env('DB_PORT_DIRECT', '5432'), // Direct PG connection
    'database' => env('DB_DATABASE', 'forge'),
    ...
],
```

**Your SaaS pattern:** PgBouncer in transaction mode with `default_pool_size = (max_connections / 2)`. Sticky reads use a separate connection pool that bypasses PgBouncer directly to PostgreSQL.

### Pool Sizing

```text
default_pool_size = (max_connections / 2) - reserved_for_admin
```

If `max_connections = 200`, set `default_pool_size = 90`, `reserve_pool_size = 10`. The reserve pool is used when clients queue up — it ensures admin connections can still get through.

<!-- TRAP -->
> **⚠️ Trap: PgBouncer in transaction mode breaks session-scoped features**
>
> The most common trap: "SET SESSION works in my local dev (direct connection) but not in production (PgBouncer)." Any session-scoped feature is lost after COMMIT because the PG connection is returned to the pool. Use `SET LOCAL` (transaction-scoped) instead, or make a direct connection for DDL/admin tasks.
>
> **⚠️ Trap: pool size too small vs too large**
>
> Pool too small: queries queue up (`query_wait_timeout` expires → clients see errors). Pool too large: PostgreSQL handles 200 connections concurrently, but you lose the pooling benefit. Start with `default_pool_size = max_connections / 2` and monitor `pgbouncer_stats` for queue length.
>
> **⚠️ Trap: not setting server_idle_timeout**
>
> Without `server_idle_timeout`, PgBouncer keeps idle PG connections forever. At low traffic, you still have 100 PG connections sitting idle. Set `server_idle_timeout = 300` (5 minutes) so idle connections are released.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "How do you handle prepared statements with PgBouncer in transaction mode?"
>
> Prepared statements are session-scoped — they're lost after COMMIT. Solutions:
> 1. **Most ORMs:** Set `ATTR_EMULATE_PREPARES = true` (Laravel, PDO) — the driver emulates prepared statements client-side, sending regular queries with parameters interpolated client-side.
> 2. **Named prepared statements:** Don't use them with PgBouncer. Use simple parameterized queries.
> 3. **Direct connection:** For queries that absolutely need server-side prepared statements, bypass PgBouncer.

---

## 9. Declarative Partitioning

### Syntax

```sql
-- Create partitioned table
CREATE TABLE audit_logs (
  id BIGSERIAL,
  organization_id INTEGER NOT NULL,
  action TEXT NOT NULL,
  details JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE audit_logs_202401
  PARTITION OF audit_logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE audit_logs_202402
  PARTITION OF audit_logs
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Partition by LIST (e.g., by region)
CREATE TABLE orders_na PARTITION OF orders FOR VALUES IN ('NA');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('EU');
CREATE TABLE orders_apac PARTITION OF orders FOR VALUES IN ('APAC');

-- Partition by HASH (e.g., for even distribution)
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

### Partition Pruning

PostgreSQL eliminates irrelevant partitions at plan time:

```sql
EXPLAIN SELECT * FROM audit_logs
WHERE organization_id = 42
  AND created_at >= '2024-01-15'
  AND created_at < '2024-01-20';
```

Planner only scans `audit_logs_202401` — other partitions are pruned.

### Automatic Partition Creation

Using `pg_partman` extension:

```sql
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
  p_parent_table := 'public.audit_logs',
  p_control := 'created_at',
  p_type := 'native',
  p_interval := '1 month',
  p_premake := 3  -- create 3 partitions in advance
);
```

### Partitioning Use Cases

| Use Case | Partition Key | Strategy | Benefit |
|----------|---------------|----------|---------|
| Time-series (audit logs) | `created_at` | RANGE (monthly) | DROP old partitions instantly |
| Multi-region | `region` | LIST | Physically separate data by region |
| Large fact tables | `organization_id` | HASH | Even distribution, parallel queries |
| Soft-delete cleanup | `deleted_at` | RANGE | DROP partition where `deleted_at < cutoff` |

### Your Multi-Tenant SaaS Pattern

```sql
CREATE TABLE audit_logs (
  id BIGSERIAL,
  organization_id INTEGER NOT NULL,
  action TEXT NOT NULL,
  details JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE audit_logs_202401
  PARTITION OF audit_logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- BRIN index on partition + tenant for fast scans
CREATE INDEX idx_audit_logs_org_created
  ON audit_logs_202401 (organization_id, created_at);

-- Drop old data instantly (no vacuum, no VACUUM FULL)
DROP TABLE IF EXISTS audit_logs_202201;
```

### Partitioning vs Inheritance

| Feature | Declarative Partitioning (Recommended) | Table Inheritance (Legacy) |
|---------|---------------------------------------|---------------------------|
| Partition pruning | ✅ Automatic | ❌ Manual constraints + `constraint_exclusion` |
| Unique/PK constraints | ✅ Must include partition key | ✅ Flexible |
| `UPDATE` routing | ✅ PG 11+ — rows move between partitions | ❌ |
| `PARTITION BY` | ✅ Native syntax | ❌ Manual inheritance tree |

### Sub-Partitioning

Partitions of partitions:

```sql
CREATE TABLE audit_logs_2024
  PARTITION OF audit_logs
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
  PARTITION BY RANGE (created_at);

CREATE TABLE audit_logs_202401
  PARTITION OF audit_logs_2024
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### Indexing

Indexes must be created on each partition individually (or on the parent — PG 12+ propagates):

```sql
CREATE INDEX ON audit_logs_202401 (organization_id, created_at);
CREATE INDEX ON audit_logs_202402 (organization_id, created_at);
-- Or create on parent — PG 12+ automatically creates on children:
CREATE INDEX ON audit_logs (organization_id, created_at);
```

<!-- TRAP -->
> **⚠️ Trap: unique index must include partition key**
>
> A PRIMARY KEY or UNIQUE index on a partitioned table must include the partition key. PG 11+ allows a PK without the partition key IF a global index is created — but this is a common misconception. Without the partition key, the PK can't enforce global uniqueness.
>
> ```sql
> -- This works:
> CREATE TABLE audit_logs (
>   id BIGSERIAL,
>   created_at TIMESTAMPTZ NOT NULL,
>   PRIMARY KEY (id, created_at)  -- partition key in PK
> ) PARTITION BY RANGE (created_at);
>
> -- This does NOT work (PG < 16):
> CREATE TABLE audit_logs (
>   id BIGSERIAL PRIMARY KEY,  -- PK doesn't include partition key
>   created_at TIMESTAMPTZ NOT NULL
> ) PARTITION BY RANGE (created_at);
> -- ERROR:  unique constraint on partitioned table must include all partitioning columns
> ```
>
> **⚠️ Trap: partition DETACH is expensive**
>
> `ALTER TABLE audit_logs DETACH PARTITION audit_logs_202401` — this copies the data to verify constraints. Use `DROP TABLE` instead for old partitions. If you need to keep the data as a standalone table (for archival), use `ALTER TABLE ... DETACH PARTITION CONCURRENTLY` (PG 15+).
>
> **⚠️ Trap: too many partitions degrade planning**
>
> 5,000+ partitions cause the planner to spend significant time evaluating partition pruning. Stick to 100-500 partitions unless you need more. For daily partitions, keep them for 30 days then merge to monthly.

<!-- FOLLOW-UP -->
> **🔍 Follow-up:** "When should you NOT use partitioning?"
>
> 1. Tables under 10M rows — partitioning adds complexity with no benefit
> 2. Tables without a clear partition key — partitioning on a column that doesn't appear in WHERE clauses won't prune
> 3. High-frequency UPDATE/DELETE — rows moving between partitions (UPDATE on partition key) is expensive
> 4. Read-heavy tables with random access — partitioning doesn't help if you're looking up by a unique key; an index is sufficient

---

## 10. Tier 2 Q&A Drill

**Q1:** You run `EXPLAIN ANALYZE` and see a Nested Loop with `rows=1 loops=50000`. What does this mean, and what's your fix?

```text
A: The plan processes 50,000 rows total (1 × 50,000 loops), not 1 row. The Nested Loop is
re-executed 50,000 times, likely from a correlated subquery or function call. Fix: rewrite
to a JOIN with a composite index on the inner table's join columns + WHERE clause.
```

---

**Q2:** In a multi-tenant app with `WHERE organization_id = ? AND status = ? AND created_at > ? ORDER BY id DESC LIMIT 20`, what index do you create?

```sql
CREATE INDEX idx_org_status_created_id
  ON table (organization_id, status, created_at, id DESC);
```

Equality columns first (org_id, status), then range (created_at), then sort (id DESC).

---

**Q3:** You have `SELECT COUNT(*) FROM orders WHERE status = 'paid'` in a multi-tenant app. It's slow. What do you do?

```text
A: First, check if every query needs organization_id. If yes, add it. If this is an admin-wide
report, consider a partial index on (status) WHERE status = 'paid' or a materialized view.
Better: push counts into a separate summary table updated by triggers or application events.
```

---

**Q4:** What's the difference between `FOR UPDATE`, `FOR SHARE`, and `FOR KEY SHARE`?

```text
A: FOR UPDATE: strongest — blocks UPDATE, DELETE, and any SELECT...FOR. FOR SHARE:
blocks UPDATE/DELETE but not reads. FOR KEY SHARE: weakest — only blocks
DELETE and FOR UPDATE/FOR NO KEY UPDATE. Used internally by DELETE.
```

---

**Q5:** How do you prevent deadlocks in a trading order-matching engine?

```text
A: 1. Consistent lock ordering — always match orders sorted by instrument_id, price ASC
2. Use SKIP LOCKED to skip rows already locked by another matching engine
3. Keep transactions short — match and commit, don't batch
4. Set deadlock_timeout = 100ms for faster detection
5. Always retry on 40P01 (SerializationFailure)
```

---

**Q6:** You need to migrate 15M rows from PG 12 to PG 16 with zero downtime. How?

```text
A: Use logical replication.
1. Set up PG 16 with wal_level = logical
2. Create publication on PG 12 for all tables
3. Create subscription on PG 16 pointing to PG 12
4. Let replication catch up (monitor pg_stat_subscription)
5. Run dual-writes briefly (write to both, verify)
6. Switch application connection string to PG 16
7. Drop subscription
DDL changes must be applied manually on both sides.
```

---

**Q7:** Your Laravel app is using PgBouncer in transaction mode, but `pg_advisory_lock` doesn't seem to work. Why?

```text
A: pg_advisory_lock() is session-scoped. PgBouncer returns the connection to the pool after every
COMMIT, releasing the session and the lock. Use pg_advisory_xact_lock() instead — it's
transaction-scoped and auto-releases at COMMIT/ROLLBACK. Also ensure PDO::ATTR_EMULATE_PREPARES
is true for prepared statements.
```

---

**Q8:** `EXPLAIN (ANALYZE, BUFFERS)` shows `Buffers: shared hit=3 shared read=47` for a query that runs every second. What does this tell you?

```text
A: 47 of 50 buffer accesses were physical reads (cache miss). The working set doesn't fit in
shared_buffers. This isn't a query problem — it's a sizing problem. Either:
- Increase shared_buffers (if system has RAM)
- Optimize the query to touch fewer pages (covering index with INCLUDE)
- Check if pg_buffercache shows the table isn't cached
```

---

**Q9:** You have a table with `(organization_id, sku)` and `(organization_id, created_at)` indexes. Should you also have an index on `(sku)` alone?

```text
A: No. In a multi-tenant app, every query filters by organization_id. An index starting
without organization_id is useless for tenant-scoped queries. However, for admin-wide
queries (WHERE sku = 'ABC' across all orgs), it might help — but that should be rare.
Use pg_stat_user_indexes to confirm idx_scan = 0, then drop it.
```

---

**Q10:** Your `ORDER BY created_at DESC LIMIT 25 OFFSET 10000` pagination is slow at high page numbers. How do you fix it?

```text
A: Switch to keyset (cursor-based) pagination:
SELECT * FROM inventory_items
WHERE (created_at, id) < (last_created, last_id)
ORDER BY created_at DESC, id DESC
LIMIT 25;
This uses the index directly and doesn't scan discarded rows. Trade-off: can't jump to
arbitrary page numbers.
```

---

**Q11:** What's the difference between `CREATE INDEX` and `CREATE INDEX CONCURRENTLY`?

```text
A: CREATE INDEX acquires SHARE lock — blocks writes (INSERT, UPDATE, DELETE) for the duration.
CREATE INDEX CONCURRENTLY uses SHARE UPDATE EXCLUSIVE — only blocks other DDL and VACUUM.
Writes can continue during the build. Trade-off: CONCURRENTLY takes longer (two passes),
uses more resources, and can leave the index in INVALID state if it fails (check
pg_index.indisvalid).
```

---

**Q12:** `pg_stat_statements` shows a query with `calls=50000, total_exec_time=250000ms`. Is it slow?

```text
A: Average = 5ms per call. That's fast individually. But 50,000 calls × 5ms = 250 seconds
of database time (4+ minutes). If this is one of many similar queries, reducing it to
1 call (by batching or eager-loading) saves 245 seconds of DB time per monitoring window.
Optimize by total_time, not by mean_time.
```

---

**Q13:** You partition `audit_logs` by month. Queries for a specific organization across 12 months are slow. What indexing strategy do you use?

```sql
CREATE INDEX idx_audit_logs_org_id
  ON audit_logs (organization_id, created_at);

-- PG 12+ — create on parent, propagates to all partitions
-- PG 11 — create on each partition individually
```

Partition pruning eliminates irrelevant months. The index on (org_id, created_at) makes the per-partition scan efficient. For very old partitions that are never queried, DROP them.

---

**Q14:** Explain the `auto_explain` module and when you'd use it in production.

```text
A: auto_explain logs query plans for slow queries automatically. Configured via:
auto_explain.log_min_duration = '1s' — log plans for queries taking > 1 second.
auto_explain.log_analyze = on — includes actual timing.
auto_explain.log_buffers = on — includes buffer usage.

Use it in production to capture plans of slow queries without manual EXPLAIN. The overhead
at log_min_duration of 1s is negligible (1% of queries or fewer). For trading platforms,
consider 500ms to catch all slow order-matching queries.
```

---

**Q15:** You have a dashboard that does `SELECT item_id, COUNT(*) FROM orders GROUP BY item_id` for 5,000 items. It's doing a Seq Scan on a 10M-row orders table. How do you optimize it?

```text
A: 1. Ensure there's an index on (item_id) or (organization_id, item_id)
2. If filtering by org_id: add WHERE organization_id = ? — reduces scan rows
3. If this runs frequently: create a materialized view or summary table:
   CREATE MATERIALIZED VIEW item_order_counts AS
   SELECT item_id, COUNT(*) AS order_count
   FROM orders GROUP BY item_id;
   REFRESH MATERIALIZED VIEW CONCURRENTLY item_order_counts;
4. If counts don't need to be real-time: cache in application (Redis)
```

---

**Q16:** What's the difference between synchronous and asynchronous replication? When would you use each?

```text
A: Synchronous: primary waits for standby acknowledgment. RPO = 0 (no data loss).
Increases write latency (at least network round trip). Use for critical financial data,
trading ledgers.
Async: primary doesn't wait. Potential data loss on primary crash (RPO > 0).
Much lower write latency. Use for reporting, analytics, read replicas, non-critical data.
```

---

**Q17:** How do you monitor and resolve lock contention in production?

```sql
-- Find blocked queries
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  blocked.wait_event_type,
  blocked.wait_event,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query,
  blocking.state AS blocking_state
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';

-- If you need to kill a blocking process (last resort):
SELECT pg_terminate_backend(blocking_pid);
```

---

**Q18:** You're migrating from OFFSET pagination to keyset pagination. What edge cases do you need to handle?

```text
A: 1. Non-unique sort columns — always include a unique column as tiebreaker (id)
2. Deleting rows — a row deleted between pages won't appear, but the cursor doesn't break
3. Inserting rows — new rows that sort before the cursor won't appear (expected in "load more")
4. API design — cursors are opaque strings (base64-encoded tuple), not page numbers
5. No "page X" navigation — keyset only supports "next" and "previous"
```

---

**Q19:** What's the `work_mem` setting and when would you increase it?

```text
A: work_mem is per-operation, per-sort/hash memory limit. Used for:
- Sort operations (ORDER BY, DISTINCT, merge join)
- Hash tables (Hash Join, Hash Aggregate)
- Bitmap operations

If you see "external sort" (disk spill) in EXPLAIN ANALYZE, increase work_mem:
SET work_mem = '64MB';  -- or globally: ALTER SYSTEM SET work_mem = '64MB';

Don't set it too high globally — work_mem * max_connections * sort_operations = memory.
For a connection pool of 100, each running 2 sorts at 64MB: 100 × 2 × 64MB = 12.8GB.
```

---

**Q20:** You have a table with 500M rows. `VACUUM FULL` is blocking writes. How do you reclaim disk space without downtime?

```text
A: Don't use VACUUM FULL. Solutions:
1. pg_repack: rebuilds table online (ACCESS SHARE lock), no downtime
2. Partitioning: truncate/drop old partitions instead of VACUUM FULL
3. If bloat is from UPDATE/DELETE: increase autovacuum frequency
4. If index bloat: REINDEX CONCURRENTLY
5. If extreme: set up logical replication to a new table, then switch
```

---

---

**Q21:** Explain the difference between `shared_preload_libraries` and custom `ALTER SYSTEM SET`. When would you need to restart PostgreSQL vs reload?

```text
A: shared_preload_libraries requires a full restart — libraries are loaded at process start.
Examples: auto_explain, pg_stat_statements, pg_partman.
Most other settings (work_mem, shared_buffers, effective_cache_size) require restart only
for shared memory changes.

Settings that need restart: shared_buffers, wal_level, max_connections, max_worker_processes.
Settings that can be reloaded: work_mem, maintenance_work_mem, log_min_duration_statement.
Reload: SELECT pg_reload_conf(); or pg_ctl reload.
```

---

**Q22:** You're designing a job queue in PostgreSQL for your trading platform. Each order needs to be processed exactly once. How do you prevent duplicate processing?

```sql
-- Use SKIP LOCKED with UPDATE ... RETURNING
WITH next_order AS (
  SELECT id FROM orders
  WHERE status = 'pending'
  ORDER BY created_at ASC
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE orders
SET status = 'processing', locked_at = NOW(), locked_by = pg_backend_pid()
FROM next_order
WHERE orders.id = next_order.id
RETURNING orders.*;

-- Atomic: SELECT FOR UPDATE SKIP LOCKED picks a row, UPDATE claims it.
-- No two workers can process the same order.
-- If worker crashes, the row stays 'processing' — have a timeout/retry job.
```

---

**Q23:** How do you handle schema migrations on a partitioned table without downtime?

```text
A: 1. For adding a column: ALTER TABLE audit_logs ADD COLUMN source TEXT;
   PG 11+ — DDL on parent propagates to all partitions (ACCESS EXCLUSIVE, brief).
2. For adding an index: CREATE INDEX CONCURRENTLY ON audit_logs (source);
   PG 12+ — concurrent index creation works on partitioned tables.
3. For changing data type: create new partition set, use logical replication or
   batch INSERT ... SELECT to migrate, then swap.
4. For renaming/dropping: do in low-traffic window or use pt-online-schema-change
   pattern (trigger-based).
```

---

> **Pro tip for the interview:** When a Tier 2 question comes up, start your answer with a real example from the 88% reduction or the 15M migration. Interviewers interviewing Senior Engineers don't want textbook definitions — they want to hear "we did X, we observed Y, and we fixed it with Z."
>
> Example framing: *"We had a correlated subquery causing a Nested Loop with 50K loops. When I read the EXPLAIN ANALYZE output, I noticed the loops value was 50,000 but the estimated rows were 1 — classic correlated subquery pattern. We rewrote it to use a LEFT JOIN with a GROUP BY subquery and added a composite covering index. That single change cut 60% of the database CPU."*
>
> Alternative framing for the 15M migration: *"We needed to move from PG 12 to PG 16 with zero downtime for a trading platform. We set up logical replication between the two clusters, verified consistency with row-count checksums, then did a controlled switchover in under 30 seconds. The key lesson: apply DDL changes manually on both sides and never assume logical replication handles schema changes."*
>
> Alternative framing for the trading platform: *"Order matching deadlocks were causing 3% of orders to fail. We identified the root cause as inconsistent lock ordering — two code paths locked order book entries in different sequences. The fix was a single rule: always sort instrument IDs ascending before locking. Combined with SKIP LOCKED for the matching queue, deadlocks went to zero."*
