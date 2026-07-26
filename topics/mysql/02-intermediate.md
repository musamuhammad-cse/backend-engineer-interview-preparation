# MySQL — Tier 2: Intermediate (Indexing, Query Optimization, Transactions & Locking)

> **Target:** Senior Backend Engineer (8+ years, PHP/Laravel with Postgres, Go, JS)  
> **Prerequisite:** Tier 1 basics — MySQL architecture, InnoDB storage engine, B+Tree indexes, basic EXPLAIN, SQL vs MySQL extensions  
> **Goal:** Deep understanding of MySQL-specific indexing behavior, query optimization, transaction isolation (InnoDB-specific), locking internals, deadlock diagnosis, and replication. You know PostgreSQL well — every section highlights what MySQL does differently.

---

## Table of Contents

1. [Advanced Indexing](#1-advanced-indexing)
2. [EXPLAIN Deep Dive](#2-explain-deep-dive)
3. [Query Optimization Patterns](#3-query-optimization-patterns)
4. [Performance Schema & Sys Schema](#4-performance-schema--sys-schema)
5. [Transactions & Isolation in InnoDB](#5-transactions--isolation-in-innodb)
6. [Locking in InnoDB — Complete Reference](#6-locking-in-innodb--complete-reference)
7. [Deadlock Diagnosis & Prevention](#7-deadlock-diagnosis--prevention)
8. [Replication](#8-replication)
9. [Replication Lag & Consistency](#9-replication-lag--consistency)
10. [Tier 2 Q&A Drill](#10-tier-2-qa-drill)

---

## 1. Advanced Indexing

### 1.1 Composite Index Column Ordering

The rule for ordering columns in a composite B+Tree index:

**equality → range → sort → include**

```sql
-- Given this query:
SELECT  user_id, status, created_at
FROM    orders
WHERE   user_id = 42          -- equality
  AND   status IN ('paid', 'shipped')  -- equality (IN is multiple equality)
  AND   created_at > '2024-01-01'      -- range
ORDER BY amount DESC;                  -- sort

-- Optimal index:
CREATE INDEX idx_orders_user_status_created_amount
ON orders(user_id, status, created_at, amount DESC);

-- user_id: equality → index leader
-- status: equality → next
-- created_at: range → after equality
-- amount: appended for ORDER BY (descending index in 8.0+)
```

**PostgreSQL comparison:** Same rule applies, but PostgreSQL can use Index Skip Scan to skip leading columns (MySQL 8.0+ has skip scan but it's limited). MySQL is more rigid — missing a leading column often means full index scan or table scan.

### 1.2 Filtering with Multiple Range Conditions

MySQL can only use an index for **one range condition** per index. Subsequent range predicates are used as filters, not index seeks.

```sql
-- Index: (age, salary)
SELECT * FROM employees
WHERE age BETWEEN 30 AND 40
  AND salary BETWEEN 50000 AND 100000;

-- age uses index range scan
-- salary is filtered by reading rows matched by age range (does not use index for salary range)
```

**PostgreSQL difference:** PostgreSQL can use multiple range conditions via bitmap scans. MySQL's B+Tree index can only descend one range path.

### 1.3 Covering Indexes (Index-Only Scan)

A covering index contains **all columns** referenced by a query. MySQL reads only the index — no table access. Visible in `Extra: Using index`.

```sql
-- Table: users(id, email, name, status, created_at)

-- This query needs only id, email, name:
EXPLAIN SELECT id, email, name FROM users WHERE email = 'a@b.com';
-- If we have:
CREATE INDEX idx_email_covering ON users(email, id, name);
-- Extra: "Using index"  →  index-only scan

-- Without covering index:
-- Extra: "Using index condition" (ICP) or just "Using where"
```

**Trap:** `Extra: "Using index"` does NOT mean "the query used an index." It means **index-only scan** — the index covers all queried columns, so no row lookup. A query can use an index without being "covering."

### 1.4 Partial Indexes

**MySQL does not have partial indexes** like PostgreSQL:

```sql
-- PostgreSQL:
CREATE INDEX idx_active_users ON users(created_at) WHERE status = 'active';
```

**MySQL workarounds:**

1. **Generated column + index:**

```sql
ALTER TABLE users
ADD COLUMN is_active TINYINT(1) GENERATED ALWAYS AS (IF(status = 'active', 1, NULL)) VIRTUAL;

CREATE INDEX idx_active_created ON users(is_active, created_at);

-- Query:
SELECT * FROM users WHERE is_active = 1 AND created_at > '2024-01-01';
```

2. **Functional index** (MySQL 8.0.13+):

```sql
-- MySQL 8.0.13+ can index expression results directly:
CREATE INDEX idx_active_created ON users((IF(status = 'active', 1, NULL)), created_at);
```

3. **Filtered queries with multiple indexes** (MySQL can use Index Merge — see 1.8).

**Follow-up:** "When would you use a generated column vs a functional index?"  
Generated columns are visible in `SHOW CREATE TABLE` and can be used in queries as columns. Functional indexes are invisible to queries — you can't reference the expression in WHERE without repeating it. Generated columns also support being stored (STORED) for re-use across queries.

### 1.5 Functional Indexes

MySQL 8.0.13+ supports indexing expressions directly:

```sql
-- Before 8.0.13: could only index columns
-- After 8.0.13:
CREATE INDEX idx_lower_email ON users((LOWER(email)));

-- Query must match the expression exactly:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- This does NOT use the index:
SELECT * FROM users WHERE LOWER(email) = LOWER('Alice@Example.com');
-- Wait — actually it does, because the entire expression LOWER(email) matches.
```

**Follow-up:** PostgreSQL has native expression indexes (`CREATE INDEX ON users(LOWER(email))`). MySQL's implementation is newer (8.0.13) and has limitations — expressions must be enclosed in parentheses, and you cannot use subqueries or aggregate functions.

### 1.6 Invisible Indexes

MySQL 8.0+ allows marking indexes invisible to the optimizer. The index is still maintained (reads and writes), but the optimizer won't use it for query plans.

```sql
ALTER TABLE users ALTER INDEX idx_email SET INVISIBLE;
ALTER TABLE users ALTER INDEX idx_email SET VISIBLE;

-- Verify:
SELECT INDEX_NAME, VISIBILITY FROM information_schema.STATISTICS
WHERE TABLE_NAME = 'users';
```

Use case: test dropping an index without actually dropping it. If performance doesn't degrade, drop it.

**Trap:** Unlike DROP INDEX, INVISIBLE still updates the index on writes. No write-performance gain. Only for testing read-path removal.

### 1.7 Descending Indexes (MySQL 8.0+)

```sql
CREATE INDEX idx_created_desc ON orders(created_at DESC);

-- Multi-column:
CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);

-- Query benefiting:
SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC;
-- Previously: MySQL would scan index forward and sort, or do backward scan (same cost)
-- Now: index is physically stored in descending order — avoids reverse scan
```

**Trap:** In MySQL 8.0, descending indexes are supported but **only some storage engines** implement them efficiently. InnoDB does. Also, descending indexes don't help `MIN()` / `MAX()` optimization in all cases.

### 1.8 Full-Text Indexes

```sql
-- MyISAM (traditional) and InnoDB (5.6+):
CREATE FULLTEXT INDEX idx_ft_body ON articles(title, body);

-- Search:
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('database optimization' IN BOOLEAN MODE);

-- Boolean mode operators:
-- +   must be present
-- -   must not be present
-- >   increase rank
-- <   decrease rank
-- *   wildcard
-- ""  exact phrase
-- ()  grouping

-- Example: find articles with "database" and "optimization" but NOT "NoSQL":
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('+database +optimization -NoSQL' IN BOOLEAN MODE);

-- Natural language mode (default):
SELECT *, MATCH(title, body) AGAINST('database optimization') AS relevance
FROM articles
ORDER BY relevance DESC;
```

**PostgreSQL difference:** PostgreSQL has `tsvector`/`tsquery` with lexemes, dictionaries, and ranking. MySQL's full-text is simpler: it uses a built-in parser with stopwords and a thesaurus. The `MATCH() AGAINST()` syntax is MySQL-specific.

**Trap:** Full-text stopwords can silently exclude results. `ft_min_word_len` (default 4) means words shorter than 4 characters are ignored. The thesaurus can map unexpected words. Always verify with `IN BOOLEAN MODE` (which ignores stopwords for operators like `+` but not for general matching).

### 1.9 Spatial Indexes

```sql
CREATE TABLE locations (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    coord GEOMETY NOT NULL SRID 4326,
    SPATIAL INDEX idx_spatial(coord)
);

-- Find points within a bounding box:
SELECT id, name FROM locations
WHERE MBRContains(ST_GeomFromText('POLYGON((...))'), coord);
```

Spatial indexes use **R-Tree** (not B+Tree). Only on GEOMETRY columns. InnoDB supports spatial indexes since MySQL 5.7.

### 1.10 Index Merge

MySQL can use multiple indexes per table in limited scenarios via **Index Merge**:

```sql
-- Indexes: idx_status, idx_created
SELECT * FROM orders WHERE status = 'pending' OR created_at > '2024-06-01';

-- Possible plans:
-- 1. Full scan (worst)
-- 2. Index merge union (merged results from both indexes)
-- 3. Composite index (best)

-- Index merge intersection:
SELECT * FROM orders WHERE status = 'pending' AND category = 'premium';
-- Indexes: idx_status, idx_category
-- MySQL can merge intersections: rows found in both indexes
```

Index Merge types in `Extra`:
- `Using intersect(...)` — AND of multiple conditions
- `Using union(...)` — OR of multiple conditions  
- `Using sort_union(...)` — OR where conditions can't be directly merged

**Trap:** Index merge is **rarely optimal**. MySQL's optimizer often chooses it only when no single composite index covers the query. A well-designed composite index almost always outperforms index merge. The optimizer also has cost thresholds — if merging seems expensive, it falls back to full scan.

**Trap:** MySQL uses **only one index per table per query** unless index merge kicks in. This is a major difference from PostgreSQL, which can use multiple indexes freely (bitmap scans in PostgreSQL combine them efficiently).

---

## 2. EXPLAIN Deep Dive

### 2.1 Output Columns

```sql
EXPLAIN SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id
ORDER BY order_count DESC
LIMIT 10;
```

Output columns:

| Column | Meaning |
|--------|---------|
| `id` | SELECT identifier (sequential, higher = earlier execution) |
| `select_type` | SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION, UNION RESULT |
| `table` | Table name (aliased) |
| `partitions` | Partition pruning info |
| `type` | Access method (ordered best to worst below) |
| `possible_keys` | Indexes MySQL considered |
| `key` | Actual index used |
| `key_len` | Bytes used from the index |
| `ref` | Columns or constants used with key |
| `rows` | **Estimated** rows examined |
| `filtered` | Percentage estimated to survive WHERE |
| `Extra` | Additional execution details |

### 2.2 `type` Access Methods (Best → Worst)

```
system → const → eq_ref → ref → fulltext → ref_or_null → index_merge →
unique_subquery → index_subquery → range → index → ALL
```

- **`system`**: Table has one row (system table). Trivial.
- **`const`**: Primary key or unique index lookup with constant. One row max.
- **`eq_ref`**: JOIN using primary key / unique index. One row per joined row.
- **`ref`**: Non-unique index lookup. Multiple rows possible.
- **`fulltext`**: Full-text index used via `MATCH ... AGAINST`.
- **`ref_or_null`**: Like `ref` but also searches for NULL.
- **`index_merge`**: Index merge algorithm used.
- **`unique_subquery`**: `IN(subquery)` using unique index.
- **`index_subquery`**: `IN(subquery)` using non-unique index.
- **`range`**: Index range scan (BETWEEN, >, <, LIKE 'prefix%').
- **`index`**: Full index scan (scans entire index, not table).
- **`ALL`**: Full table scan. Worst.

**PostgreSQL comparison:** PostgreSQL's `EXPLAIN` output is plan-node-based (Seq Scan, Index Scan, Bitmap Heap Scan, etc.) rather than a single `type` column. MySQL's `type` summarizes the most expensive access method.

### 2.3 Extra Values — What They Mean

```sql
-- "Using index": index-only scan (covering index)
EXPLAIN SELECT id, email FROM users WHERE email = 'a@b.com';
-- Extra: Using index

-- "Using index condition": Index Condition Pushdown (ICP)
-- MySQL 5.6+ pushes parts of WHERE into storage engine
EXPLAIN SELECT * FROM users WHERE name LIKE '%john%' AND status = 'active';
-- Extra: Using index condition
-- (Note: if index is (name, status), ICP evaluates status during index traversal)

-- "Using where": rows filtered after storage engine returns them
EXPLAIN SELECT * FROM users WHERE status = 'active';
-- Extra: Using where

-- "Using filesort": sort couldn't use index (needs optimization)
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC;
-- If no index: Extra: Using filesort
-- Fix: CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);

-- "Using temporary": temporary table needed (GROUP BY, DISTINCT)
EXPLAIN SELECT status, COUNT(*) FROM users GROUP BY status;
-- If no index on status: Extra: Using temporary; Using filesort
-- Fix: CREATE INDEX idx_status ON users(status);

-- "Using join buffer (Block Nested Loop)" or "(Batched Key Access)"
-- JOIN without index on the inner table
EXPLAIN SELECT * FROM users u LEFT JOIN orders o ON o.user_id = u.id;
-- If no index on orders.user_id: Extra: Using join buffer (Block Nested Loop)

-- "Using MRR": Multi-Range Read optimization
-- Sorts retrieved keys by primary key for sequential lookups

-- "Using index for group-by": Loose index scan for GROUP BY
-- Only possible when GROUP BY matches index left prefix

-- "Using index for skip scan": MySQL 8.0+ skip scan optimization
-- Allows using composite index when leading column is not in WHERE
```

### 2.4 EXPLAIN ANALYZE (MySQL 8.0.18+)

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id;

-- Output (iterative format):
-- -> Group aggregate: count(o.id)
--     -> Left hash join (o.user_id = u.id)  (cost=... rows=... actual time=... rows=...)
--         -> Table scan on u  (cost=... actual time=... rows=... loops=...)
--         -> Hash
--             -> Table scan on o  (cost=... actual time=... rows=... loops=...)
```

**PostgreSQL difference:** PostgreSQL's `EXPLAIN ANALYZE` output is more granular and supports `BUFFERS` for cache hit info. MySQL's version is newer (8.0.18) but doesn't provide buffer/cache stats directly.

### 2.5 EXPLAIN FORMAT=JSON

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM users WHERE id = 1;

-- Output provides:
--   - Cost estimates
--   - Attached conditions
--   - Used key parts
--   - Prefix/range info
-- Useful for programmatic analysis
```

**Trap:** `rows` is an **estimate**, not actual row count. MySQL's optimizer uses index statistics (cardinality estimates). After bulk loads, stale statistics cause bad plans. Run `ANALYZE TABLE`.

**Trap:** `Using filesort` does not mean a file on disk — it means MySQL couldn't use an index for sorting. Sorting happens in memory (`sort_buffer_size`) or on disk if the result set exceeds the buffer.

**Trap:** `Using temporary` means a temporary table was created (in memory or on disk). Common with `GROUP BY` or `DISTINCT` when no index is available. Can be mitigated with proper indexing.

**Trap:** `Extra: Using index` does NOT mean "the query used an index." It **specifically** means **index-only scan** — all columns in the query exist in the index. A query with `type: ref` that **does** use an index but needs table lookups will NOT show `Using index`.

---

## 3. Query Optimization Patterns

### 3.1 Avoid `SELECT *`

```sql
-- Bad: reads all columns, prevents covering indexes
SELECT * FROM users WHERE email = 'a@b.com';

-- Good: reads only needed columns, can use covering index
SELECT id, email, name FROM users WHERE email = 'a@b.com';
```

MySQL sends the **full row** from storage engine to server layer for every `*`. With a covering index, the storage engine never touches the table.

**PostgreSQL comparison:** Same principle applies to PostgreSQL. MySQL's storage engine architecture (InnoDB vs server layer) makes the overhead more visible — the storage engine must unpack the row.

### 3.2 Use LIMIT

```sql
-- Bad: loads all matching rows, even if only 10 needed
SELECT id, name FROM users WHERE status = 'active';

-- Good:
SELECT id, name FROM users WHERE status = 'active' LIMIT 10;
```

MySQL optimizes LIMIT with index ordering — it can stop scanning after finding enough rows.

### 3.3 Prefer JOINs Over Correlated Subqueries

```sql
-- Slow: correlated subquery runs for each row of users
SELECT id, name,
    (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) AS order_count
FROM users;

-- Fast: JOIN with GROUP BY
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

**PostgreSQL comparison:** PostgreSQL's optimizer can convert correlated subqueries to joins. MySQL's optimizer is less aggressive. You should write the explicit JOIN.

### 3.4 EXISTS vs IN

```sql
-- EXISTS can be faster when subquery result is large:
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'paid');

-- IN can be faster when subquery result is small:
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE status = 'paid');
```

**Trap:** MySQL 8.0 optimizes both better than earlier versions, but the rule of thumb still holds: EXISTS for large subquery results, IN for small/lists.

### 3.5 Functions on Indexed Columns

```sql
-- Bad: function on indexed column → can't use index
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-01';

-- Good: range condition → can use index
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';
```

Same applies to `YEAR()`, `MONTH()`, `UPPER()`, `LOWER()`, arithmetic.

### 3.6 UNION ALL vs UNION

```sql
-- UNION removes duplicates (temporary table + sort)
SELECT id, name FROM active_users
UNION
SELECT id, name FROM archived_users;

-- UNION ALL does not remove duplicates (faster)
SELECT id, name FROM active_users
UNION ALL
SELECT id, name FROM archived_users;
```

**Trap:** `UNION` must deduplicate across all columns. This requires a temporary table and sorting. If you don't need deduplication, `UNION ALL` is always faster.

### 3.7 OR Conditions

```sql
-- Bad: OR is hard to optimize
SELECT * FROM orders WHERE status = 'pending' OR user_id = 42;

-- Option 1: UNION
SELECT * FROM orders WHERE status = 'pending'
UNION
SELECT * FROM orders WHERE user_id = 42;

-- Option 2: Composite index that supports both
-- CREATE INDEX idx_status_user ON orders(status, user_id);
```

**PostgreSQL comparison:** PostgreSQL handles OR better with bitmap scans. MySQL often falls back to full table scan or index merge (which is suboptimal).

### 3.8 LIKE Patterns

```sql
-- Prefix LIKE uses B-Tree index:
SELECT * FROM users WHERE email LIKE 'alice@%';

-- Suffix/contains LIKE cannot use B-Tree:
SELECT * FROM users WHERE email LIKE '%@example.com';  -- Can't use index
SELECT * FROM users WHERE email LIKE '%alice%';          -- Can't use index

-- Alternative for full-text search in text columns:
-- Add FULLTEXT index (if content, not email)
-- For prefix/suffix matching on short strings: consider reverse index column
```

### 3.9 ORDER BY with LIMIT

MySQL can optimize `ORDER BY ... LIMIT N` to stop after finding N rows if the index supports the ordering:

```sql
-- Index: (status, created_at)
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 100;
-- MySQL scans the index in order, stops after 100 rows matching status
```

Without the right index, MySQL sorts the entire result set first, then takes N.

### 3.10 GROUP BY Optimization

Loose index scan for GROUP BY:

```sql
-- Index: (status, created_at)
SELECT status, COUNT(*) FROM orders GROUP BY status;
-- Extra: "Using index for group-by"

-- Requires GROUP BY column(s) to be the leftmost prefix of an index
```

### 3.11 Avoid HAVING When WHERE Suffices

```sql
-- Bad: HAVING filters after aggregation (can't use index for filter)
SELECT user_id, COUNT(*) AS cnt
FROM orders
GROUP BY user_id
HAVING user_id > 100;

-- Good: WHERE filters before aggregation (uses index)
SELECT user_id, COUNT(*) AS cnt
FROM orders
WHERE user_id > 100
GROUP BY user_id;
```

### 3.12 BETWEEN for Inclusive Ranges

```sql
-- These are equivalent:
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at <= '2024-01-31';
```

BEWARE: `BETWEEN` is **inclusive** on both ends. For timestamps, this can cause off-by-one errors. Prefer `>=` and `<` for datetime ranges.

**Trap:** MySQL optimizer stats can be **stale** after bulk loads. Oracle's optimizer is cost-based but relies on index cardinality estimates. Always run:

```sql
ANALYZE TABLE orders;
```

after loading significant data. In PostgreSQL, `ANALYZE` runs automatically with `autovacuum`. MySQL depends on `innodb_stats_auto_recalc` (default ON, but not triggered by all operations).

**Trap:** `ORDER BY` + `LIMIT` with **non-unique sort column** produces non-deterministic pagination. Rows with the same sort value may flip order between queries.

```sql
-- Non-deterministic: two rows with same created_at can swap
SELECT * FROM orders ORDER BY created_at LIMIT 10 OFFSET 20;

-- Fix: add tiebreaker column
SELECT * FROM orders ORDER BY created_at, id LIMIT 10 OFFSET 20;
```

**Trap:** `ISNULL(col)` is not indexable. Use `col IS NULL` instead:

```sql
-- Correct:
SELECT * FROM users WHERE deleted_at IS NULL;
```

---

## 4. Performance Schema & Sys Schema

### 4.1 performance_schema Overview

`performance_schema` is an event-based instrumentation layer in MySQL. Low overhead (enabled by default in MySQL 8.0). Collects query execution, waits, I/O, memory usage, and more.

```sql
-- Check if enabled:
SHOW VARIABLES LIKE 'performance_schema';

-- Key setup tables:
SELECT * FROM performance_schema.setup_consumers;
SELECT * FROM performance_schema.setup_instruments;
```

### 4.2 Key performance_schema Tables

**Query analysis:**

```sql
-- Top queries by total execution time (query fingerprint digest):
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT/1000000000 AS total_ms,
       AVG_TIMER_WAIT/1000000000 AS avg_ms,
       SUM_ROWS_EXAMINED, SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Currently running queries:
SELECT * FROM performance_schema.events_statements_current\G

-- Wait events:
SELECT * FROM performance_schema.events_waits_current\G

-- Wait summary:
SELECT EVENT_NAME, COUNT_STAR, SUM_TIMER_WAIT/1000000000 AS total_ms
FROM performance_schema.events_waits_summary_global_by_event_name
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- I/O per file:
SELECT FILE_NAME, COUNT_READ, SUM_NUMBER_OF_BYTES_READ, COUNT_WRITE,
       SUM_NUMBER_OF_BYTES_WRITE
FROM performance_schema.file_summary_by_instance
ORDER BY SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE DESC
LIMIT 10;

-- Table I/O waits:
SELECT OBJECT_SCHEMA, OBJECT_NAME, COUNT_STAR,
       SUM_TIMER_WAIT/1000000000 AS total_io_ms
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Table lock waits:
SELECT * FROM performance_schema.table_lock_waits_summary_by_table;
```

### 4.3 sys Schema

The `sys` schema provides easy-to-use views over `performance_schema`. Not installed by default, but widely used.

```sql
-- Query statistics (ordered by total latency):
SELECT * FROM sys.statement_analysis
WHERE last_seen > NOW() - INTERVAL 1 HOUR
ORDER BY total_latency DESC
LIMIT 10;

-- Unused indexes:
SELECT * FROM sys.schema_unused_indexes;

-- Current table lock waits:
SELECT * FROM sys.schema_table_lock_waits;

-- I/O by file (global):
SELECT * FROM sys.io_global_by_file_by_bytes
ORDER BY total DESC
LIMIT 10;

-- Client/connection summary:
SELECT * FROM sys.host_summary
ORDER BY total_connections DESC;

-- InnoDB lock waits:
SELECT * FROM sys.innodb_lock_waits;
```

**PostgreSQL comparison:** PostgreSQL has `pg_stat_statements` (extension) for query fingerprinting, `pg_stat_activity` for running queries, and `pg_locks` for lock information. MySQL's `performance_schema` is more comprehensive but has higher overhead.

### 4.4 Configuration

```sql
-- performance_schema memory is configurable:
performance_schema_events_statements_history_size = 10   -- default
performance_schema_events_statements_history_long_size  = 10000
performance_schema_max_digest_length                   = 1024
performance_schema_max_sql_text_length                  = 1024

-- Disable specific instruments (reduce overhead):
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO', TIMED = 'NO'
WHERE NAME LIKE 'wait/io/file/innodb/%';
```

**Trap:** `performance_schema` can use significant memory. In high-memory environments, tune `performance_schema_*_size` parameters. Defaults are conservative (MySQL 8.0) but custom installations may need adjustment.

**Trap:** `sys` schema is **not installed by default** in MySQL. You need to run:

```bash
mysql -u root -p < /usr/share/mysql/sys_schema.sql
# or depending on distribution:
mysql -u root -p < /usr/share/mysql/mysql_sys_install.sql
```

**Trap:** `performance_schema` overhead is **low** (1-3%) but **not zero**. Disable it for benchmark runs. In production, keep it enabled — the visibility trade-off is worth it.

---

## 5. Transactions & Isolation in InnoDB

### 5.1 Transaction Basics

```sql
-- Explicit transaction:
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Or:
ROLLBACK;

-- AUTOCOMMIT mode (default: ON for mysql CLI, varies by driver):
SHOW VARIABLES LIKE 'autocommit';  -- ON means each statement is its own transaction
SET AUTOCOMMIT = 0;  -- Manual commit required

-- Savepoints:
START TRANSACTION;
INSERT INTO audit_log SET action = 'start';
SAVEPOINT sp1;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
ROLLBACK TO SAVEPOINT sp1;
COMMIT;  -- audit_log inserted, update rolled back
```

### 5.2 Isolation Levels — InnoDB Specific Behavior

```sql
-- Set isolation level:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- 8.0+

-- Or in my.cnf:
-- transaction-isolation = READ-COMMITTED
```

**READ UNCOMMITTED:**

- Dirty reads possible (see uncommitted data).
- Rarely used. MySQL allows it but InnoDB's MVCC still prevents dirty reads in practice for consistent reads — dirty reads only happen with `SELECT ... FOR UPDATE` or `LOCK IN SHARE MODE` at this level.

**READ COMMITTED:**

- Statement-level snapshot (each statement gets its own read view).
- Non-repeatable reads possible (same transaction reads same row twice, gets different values).
- **Gap locking is disabled for indexing and searching.** Only record locks are used.
- Recommended for most production workloads (reduces deadlocks).

```sql
-- Session 1:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Session 2:
UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT;

-- Session 1 re-reads:
SELECT balance FROM accounts WHERE id = 1;  -- 200 (non-repeatable read)
```

**REPEATABLE READ (MySQL Default):**

- Transaction-level snapshot (read view created at first read in the transaction).
- Phantoms prevented by **next-key locking** (record lock + gap lock).
- Consistent non-locking read: reads see snapshot from transaction start.

```sql
-- Default RR behavior:
START TRANSACTION;
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- creates snapshot

-- Session 2 inserts a new order > '2024-01-01' and commits.

SELECT * FROM orders WHERE created_at > '2024-01-01';
-- Same result as first read. Phantom prevented by consistent read.
```

**SERIALIZABLE:**

- All reads implicitly become `SELECT ... LOCK IN SHARE MODE`.
- Highest isolation, lowest concurrency.
- Rarely used in practice.

### 5.3 MVCC in InnoDB

**PostgreSQL comparison:** Both implement MVCC, but differently:

| Aspect | MySQL (InnoDB) | PostgreSQL |
|--------|---------------|------------|
| Old row versions | Undo log (separate tablespace) | Heap with dead tuples |
| Cleanup | Purge thread cleans undo log | VACUUM reclaims dead tuples |
| Read view | Created at TX start (RR) or stmt start (RC) | Same (snapshot isolation) |
| Write skew | Next-key locking prevents phantoms | Serializable via SSI |
| Tuple visibility | System columns (DB_TRX_ID, DB_ROLL_PTR) | Tuple header (xmin, xmax) |

**InnoDB undo log:**

```sql
-- Undo tablespace management:
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';  -- Number of undo tablespaces
SHOW VARIABLES LIKE 'innodb_max_undo_log_size';  -- Max undo log size (8.0+)
```

**Consistent non-locking read:** InnoDB reads do **not** lock rows (even at REPEATABLE READ). This is different from `SELECT ... FOR UPDATE` or `LOCK IN SHARE MODE`. MVCC provides a snapshot without blocking readers.

```sql
-- Session 1:
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1;  -- No locks held

-- Session 2 can also read and even UPDATE:
UPDATE accounts SET balance = 200 WHERE id = 1;  -- No conflict with Session 1's read
```

**Trap:** MySQL's REPEATABLE READ **prevents phantoms via gap locking** (next-key locks), unlike PostgreSQL's snapshot isolation which allows write skew. This means MySQL's RR has different trade-offs — more locking, fewer anomalies.

**Trap:** READ COMMITTED with **statement-based binary logging** can cause replication inconsistencies. If an UPDATE matches different rows on master vs slave (due to data drift), statement-based binlog replicates the statement, not the row changes.

```sql
-- Safe config for RC:
binlog_format = ROW
transaction_isolation = READ-COMMITTED
```

**Trap:** Long-running transactions cause **undo log bloat**. The undo log grows because old row versions can't be purged (needed for the transaction's snapshot). This can cause massive `ibdata1` growth. Monitor `SHOW ENGINE INNODB STATUS` for undo log size. Set `innodb_max_undo_log_size` and `innodb_purge_rseg_truncate_frequency` in 8.0+.

---

## 6. Locking in InnoDB — Complete Reference

### 6.1 Row-Level Locks

```sql
-- Shared (S) lock: SELECT ... LOCK IN SHARE MODE (MySQL 8.0+: FOR SHARE)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- Exclusive (X) lock: SELECT ... FOR UPDATE
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- UPDATE and DELETE implicitly acquire exclusive locks:
UPDATE accounts SET balance = 200 WHERE id = 1;  -- X-lock on row
```

S and X locks compatibility:

| Requested \ Held | S | X |
|:---:|:---:|:---:|
| **S** | Compatible | Conflict |
| **X** | Conflict | Conflict |

### 6.2 Intention Locks

MySQL acquires intention locks **before** acquiring row-level locks. They don't block anything except full-table requests (e.g., `LOCK TABLES ... WRITE`).

- **Intention Shared (IS):** Transaction intends to set S locks on individual rows.
- **Intention Exclusive (IX):** Transaction intends to set X locks on individual rows.

```sql
-- SELECT ... FOR SHARE acquires IS on table, then S on rows
-- SELECT ... FOR UPDATE acquires IX on table, then X on rows
-- UPDATE/DELETE acquires IX on table, then X on rows
```

**Purpose:** Fast conflict detection for table-level locks. If transaction A has IX on the table, transaction B wanting `LOCK TABLES ... WRITE` immediately knows there's a conflict.

### 6.3 Record Lock

Locks a single index record.

```sql
-- On a unique index (e.g., primary key):
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Record lock on accounts.id = 1

-- On a non-unique index:
SELECT * FROM accounts WHERE status = 'active' FOR UPDATE;
-- Record locks on all matching rows in the secondary index
-- AND also their corresponding primary key entries (clustered index)
```

### 6.4 Gap Lock

Locks the **gap between index records** to prevent inserts. Only active at REPEATABLE READ and SERIALIZABLE.

```sql
-- Table: accounts(id PK, balance)
-- Data: id = 1, 5, 10

-- Transaction 1:
START TRANSACTION;
SELECT * FROM accounts WHERE balance = 100 FOR UPDATE;  -- No matching rows
-- Gap locks on: (-∞, 1), (1, 5), (5, 10), (10, +∞)

-- Transaction 2: tries to insert a row with balance = 100
INSERT INTO accounts (id, balance) VALUES (7, 100);  -- BLOCKED by gap lock
```

**Follow-up:** "When are gap locks acquired?"  
Gap locks are acquired when searching with a range condition, or when a query with equal condition finds no matching row (locking the gap where it would have been). They exist only to prevent phantom rows under REPEATABLE READ.

### 6.5 Next-Key Lock

**Record lock + gap lock before it.** Combined into one lock.

```sql
-- Data: id = 1, 5, 10

-- Next-key lock on (1, 5] means:
--   - Record lock on id=5
--   - Gap lock on (1, 5) — prevents insert of id = 2, 3, 4
```

This is what actually prevents phantoms in REPEATABLE READ. The gap prevents inserts, the record lock prevents modifications.

### 6.6 Insert Intention Lock

A special type of gap lock for `INSERT`. Multiple transactions can insert into the same gap as long as their values don't conflict on a unique index.

```sql
-- Transaction 1:
INSERT INTO accounts (id, balance) VALUES (3, 200);
-- Acquires insert intention lock on gap (1, 5)

-- Transaction 2: (not blocked, even though same gap)
INSERT INTO accounts (id, balance) VALUES (4, 300);
-- Also acquires insert intention lock on gap (1, 5)
-- Both proceed concurrently

-- But if both try to insert the same unique key:
INSERT INTO accounts (id, balance) VALUES (3, 400);
-- Transaction 2 will wait for Transaction 1's lock
```

### 6.7 AUTO-INC Lock

Table-level lock for `AUTO_INCREMENT`. Performance impact depends on `innodb_autoinc_lock_mode`:

| Mode | Name | Behavior |
|:----:|------|----------|
| 0 | Traditional | `LOCK TABLES ... AUTOINC` — table-level lock for all INSERTs (serializes) |
| 1 | Consecutive (default) | Bulk inserts get table lock; simple inserts use lightweight mutex |
| 2 | Interleaved | No table lock — highest concurrency, but values may be non-monotonic with mixed INSERT types |

```sql
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
-- 1 = consecutive (default)
```

**RDS/Aurora note:** In Aurora MySQL, auto-increment behavior can differ. Verify with your environment.

### 6.8 Predicate Locks

For **spatial indexes** (R-Tree). Lock based on spatial predicates (MBR, geometry relationships), not on index records. Complex — generally don't worry about them unless doing heavy GIS work under SERIALIZABLE.

### 6.9 Lock Read Statements

```sql
-- Exclusive lock (X):
SELECT ... FOR UPDATE;

-- Shared lock (S):
SELECT ... LOCK IN SHARE MODE;  -- Old syntax
SELECT ... FOR SHARE;            -- MySQL 8.0+ syntax

-- NOWAIT and SKIP LOCKED (MySQL 8.0+):
SELECT * FROM jobs WHERE status = 'pending' ORDER BY created_at LIMIT 1
FOR UPDATE SKIP LOCKED;
-- SKIP LOCKED: skip rows locked by other transactions
-- NOWAIT: return error immediately if row is locked (don't wait)
```

These are critical for implementing queues, counters, and optimistic concurrency.

### 6.10 Lock Escalation

**InnoDB does NOT escalate row locks to page/table locks.** Unlike Microsoft SQL Server or some other databases, InnoDB always locks at the row level (plus gap/next-key). This means many locked rows may consume more memory, but you never get surprise table scans due to escalation.

### 6.11 Lock Monitoring

```sql
-- Classic lock info:
SHOW ENGINE INNODB STATUS\G
-- Look for "LATEST DETECTED DEADLOCK" and "TRANSACTIONS" sections

-- Performance schema (MySQL 8.0+):
SELECT * FROM performance_schema.data_locks\G
SELECT * FROM performance_schema.data_lock_waits\G

-- Sys schema:
SELECT * FROM sys.innodb_lock_waits;
```

**Trap:** `SELECT ... FOR UPDATE` locks **all rows scanned by the query**, not just the matched rows.

```sql
-- DISASTER: No index on status → full table scan → every row locked!
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;

-- SAFE: Index on status → only matching rows locked
-- (but still all scanned rows, not just returned rows!)
SELECT * FROM orders WHERE status = 'pending' LIMIT 10 FOR UPDATE;
-- This locks 10 rows (the ones returned), not all pending rows
-- But only if the index is used for the scan
```

This is a **major production incident** pattern. A query without an index on the WHERE condition locks every row MySQL scans. In a busy table, this brings all writes to a halt.

**Trap:** Gap locking under REPEATABLE READ can cause **insert deadlocks**. Two transactions reading the same range with `FOR UPDATE` both acquire gap locks. When both try to insert into the same gap, they deadlock. Using READ COMMITTED (which disables gap locking) eliminates this.

**Trap:** `innodb_autoinc_lock_mode = 0` (traditional) **serializes all INSERTs** with auto-increment. Never use this in production. Mode 1 (default) is safe and performant. Mode 2 can cause gaps and is only for maximum insert throughput under statement-based replication (rare).

---

## 7. Deadlock Diagnosis & Prevention

### 7.1 Deadlock Detection

InnoDB automatically detects deadlocks (transaction waits in a cycle). It chooses a **victim** (usually the transaction with the fewest locks) and rolls it back with error:

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

```sql
-- Enable logging all deadlocks (MySQL 8.0+):
SET GLOBAL innodb_print_all_deadlocks = 1;
-- Writes every deadlock to the error log
```

### 7.2 Analyzing Deadlocks

```sql
SHOW ENGINE INNODB STATUS\G
```

Output contains (look for `LATEST DETECTED DEADLOCK`):

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-06-15 10:30:45 0x7f1234
*** (1) TRANSACTION:
TRANSACTION 123456, ACTIVE 10 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 8, OS thread handle 1234, query id 1234 localhost root updating
UPDATE accounts SET balance = balance - 100 WHERE id = 1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5 page no 3 n bits 72 index PRIMARY of table `accounts`
trx id 123456 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
TRANSACTION 123457, ACTIVE 8 sec starting index read
mysql tables in use 1, locked 1
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 9, OS thread handle 5678, query id 1235 localhost root updating
UPDATE accounts SET balance = balance - 50 WHERE id = 2
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5 page no 3 n bits 72 index PRIMARY of table `accounts`
trx id 123457 lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (1)
```

### 7.3 Common Deadlock Scenarios

**Scenario 1: Different lock order**

```sql
-- Transaction 1:
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Transaction 2 (concurrent):
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;

-- Deadlock: T1 locks 1, waits for 2. T2 locks 2, waits for 1.
```

**Solution:** Always acquire locks in the same order (e.g., sort IDs).

**Scenario 2: Gap lock conflicts**

```sql
-- Transaction 1:
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;
-- Acquires gap locks on status index

-- Transaction 2:
INSERT INTO orders (status) VALUES ('pending');
-- Waits for gap lock from T1

-- Transaction 1:
INSERT INTO orders (status) VALUES ('pending');
-- Deadlock: T1 waits for T2's insert intention lock, T2 waits for T1's gap lock
```

**Solution:** Use READ COMMITTED (disables gap locks for indexing/searching).

**Scenario 3: Full table lock (missing index)**

```sql
-- No index on user_id
-- Transaction 1:
UPDATE orders SET status = 'paid' WHERE user_id = 42;
-- Locks ALL rows scanned (entire table)

-- Transaction 2:
UPDATE orders SET status = 'shipped' WHERE user_id = 99;
-- Also needs to scan entire table → blocked by T1's locks → deadlock on concurrent writes
```

**Solution:** Always index WHERE columns used in UPDATE/DELETE with `FOR UPDATE`.

### 7.4 Deadlock Prevention Strategies

1. **Consistent lock ordering:** Always acquire locks in the same order (by primary key, by sorted list, etc.).
2. **Keep transactions short:** Minimize time between lock acquisition and commit.
3. **Lock only what you need:** Use `SELECT ... FOR UPDATE` only on rows you actually modify.
4. **Use READ COMMITTED:** Reduces gap locking significantly.
5. **Index all WHERE clauses:** Prevents full-table locking.
6. **Use low isolation when possible:** Avoid REPEATABLE READ if not needed.
7. **Use `NOWAIT` or `SKIP LOCKED`:** For queue/job workers, skip locked rows instead of waiting.

### 7.5 Deadlock Retry Logic

**Trap:** Not having retry logic for deadlock victims is a common production bug. MySQL kills one transaction with error 1213 — the client **must** retry.

```php
// PHP example (Laravel):
for ($attempts = 0; $attempts < 3; $attempts++) {
    try {
        DB::transaction(function () {
            // Your queries
        });
        break;  // Success
    } catch (\Illuminate\Database\QueryException $e) {
        if ($e->errorInfo[1] != 1213 && $e->errorInfo[1] != 1205) {
            throw $e;  // Not a deadlock/lock wait timeout
        }
        usleep(100000 * ($attempts + 1));  // 100ms, 200ms, 300ms backoff
    }
}
```

```go
// Go example:
for attempts := 0; attempts < 3; attempts++ {
    err := db.Transaction(func(tx *sql.Tx) error {
        // Your queries
        return nil
    })
    if err == nil {
        break
    }
    if mysqlErr, ok := err.(*mysql.MySQLError); ok && mysqlErr.Number == 1213 {
        time.Sleep(time.Duration(100*(attempts+1)) * time.Millisecond)
        continue
    }
    return err
}
```

**Follow-up:** "Should deadlocks ever happen in production?"  
Yes — under high concurrency, deadlocks are inevitable even with perfect code. Design for retries. The difference between senior and junior is: seniors build retry logic, monitor deadlock rates, and investigate spikes rather than assuming deadlocks = zero.

**Trap:** Ignoring `SHOW ENGINE INNODB STATUS` in production. Enable `innodb_print_all_deadlocks = 1` and collect deadlock logs. Without this, you're debugging blind.

**Trap:** Assuming deadlocks are **always** a code bug. Under high contention (e.g., counter increments, hot rows), deadlocks are normal. Design transactions to be short, and implement retries.

---

## 8. Replication

### 8.1 Asynchronous Replication Overview

```
[Master] -- writes → [Binary Log]
                    ↓ (binlog dump thread)
              [Slave I/O Thread] -- writes → [Relay Log]
                                              ↓ (Slave SQL Thread)
                                        [Slave Data]
```

```sql
-- On master:
SHOW MASTER STATUS;
-- File: mysql-bin.000123, Position: 456, Binlog_Do_DB: ...

-- On slave:
SHOW SLAVE STATUS\G
```

### 8.2 Binary Log Formats

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

| Format | Behavior | Best for |
|--------|----------|----------|
| `STATEMENT` | Logs SQL statements | Small binlogs, but non-deterministic |
| `ROW` | Logs row changes (before/after images) | Safe, recommended |
| `MIXED` | Statement by default, ROW for unsafe statements | Compromise |

**Recommendation:** Use `ROW` in production. Statement-based can produce inconsistent replicas (e.g., `LIMIT` without ORDER BY, `NOW()`, `UUID()`).

**PostgreSQL comparison:** PostgreSQL uses WAL (Write-Ahead Log) streaming, which is always physical (block-level), not logical statement/row. PostgreSQL's logical replication (10+) is similar to MySQL's row-based. MySQL also has `mysqlbinlog` for reading binary logs, analogous to `pg_waldump`.

### 8.3 Semi-Synchronous Replication

```sql
-- Install plugins (on master):
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
-- On slave:
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- Enable (my.cnf):
-- Master:
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_master_timeout = 1000  -- ms (fall back to async if no slave ack)

-- Slave:
rpl_semi_sync_slave_enabled = 1
```

**Behavior:** Master waits for at least one slave to acknowledge receiving the binlog event before committing. This doesn't guarantee the slave has **applied** the event — only that it's in the slave's relay log.

**Trap:** Semi-sync is **not** synchronous replication. It's "at least one slave has the binlog." For true synchronous replication, use Group Replication (InnoDB Cluster).

### 8.4 GTID-Based Replication (MySQL 5.6+)

GTID = Global Transaction Identifier. Format: `server_uuid:transaction_id`.

```sql
-- Enable GTID:
gtid_mode = ON
enforce_gtid_consistency = ON

-- Show GTIDs on master:
SHOW MASTER STATUS;
-- Executed_Gtid_Set: 3e11fa47-71ca-11e9-9e0c-42010a800123:1-42

-- On slave:
SHOW SLAVE STATUS;
-- Retrieved_Gtid_Set: 3e11fa47-71ca-11e9-9e0c-42010a800123:1-42
-- Executed_Gtid_Set: 3e11fa47-71ca-11e9-9e0c-42010a800123:1-42

-- Failover: simply point new slave to the new master.
-- No need to track binlog file + position.
CHANGE MASTER TO
  MASTER_HOST = 'new-master',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'secret',
  MASTER_AUTO_POSITION = 1;
```

**Advantage:** GTID simplifies failover. No need to find correct `MASTER_LOG_FILE` and `MASTER_LOG_POS`.

### 8.5 Replication Threads

```sql
-- Check slave thread status:
SHOW SLAVE STATUS\G
-- Slave_IO_State: Waiting for master to send event
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0
```

- **Slave I/O thread:** Connects to master, reads binlog, writes to relay log.
- **Slave SQL thread:** Reads relay log, applies events to slave.

### 8.6 Multi-Threaded Slave (Parallel Replication)

```sql
-- MySQL 8.0 configuration:
slave_parallel_workers = 4      -- Number of applier threads
slave_parallel_type = LOGICAL_CLOCK  -- Use commits that were parallel on master
```

- `DATABASE` (old): parallel apply only if changes are in different databases. Useless for single-database workloads.
- `LOGICAL_CLOCK` (MySQL 8.0): applies transactions within the same binary log commit group in parallel. Significantly reduces lag.

**Trap:** Database-level parallelism (`slave_parallel_type=DATABASE`) doesn't help single-database workloads. Always use `LOGICAL_CLOCK` in MySQL 8.0.

### 8.7 Replication Filters

```sql
-- my.cnf:
replicate-do-db = myapp
replicate-ignore-db = mysql,test
replicate-wild-do-table = myapp.orders%
```

**Caution:** Filters are applied per event, not per statement. Statement-based mode with filters can produce inconsistent results.

### 8.8 Monitoring Replication

```sql
SHOW SLAVE STATUS\G
```

Key fields:

| Field | What it tells you |
|-------|-------------------|
| `Slave_IO_Running` | IO thread connected to master? |
| `Slave_SQL_Running` | SQL thread applying events? |
| `Seconds_Behind_Master` | Approximate lag (see trap below) |
| `Last_IO_Error` | IO thread error |
| `Last_SQL_Error` | SQL thread error |
| `Retrieved_Gtid_Set` | GTIDs fetched from master |
| `Executed_Gtid_Set` | GTIDs applied on slave |
| `Last_IO_Errno` | Error code |
| `Seconds_Behind_Master_NULL` | NULL if SQL thread running but IO thread not caught up |

**Trap:** `Seconds_Behind_Master` is **unreliable**. It's computed as: (master's timestamp in binlog) - (slave's current timestamp). If clocks skew, this number is wrong. It can also be NULL or 0 when there's actual lag (e.g., large transaction still applying but timestamps match).

Better: Use **pt-heartbeat** (Percona Toolkit) for precise lag measurement:

```bash
pt-heartbeat --update --database percona --daemonize
pt-heartbeat --monitor --database percona --master-server-id=1
```

**Trap:** Row-based replication + full blob columns causes **massive binlog growth**. `UPDATE ... SET large_text_col = 'new value'` logs the **entire row** (before and after). If you have large TEXT/BLOB columns, binlog can explode. Mitigation: normalize large columns, use `binlog_row_image = minimal` (logs only changed columns), or avoid row-based for tables with large blobs.

```sql
-- Minimize binlog size for large columns:
SET GLOBAL binlog_row_image = MINIMAL;
-- Only logs before/after images for changed columns, not full rows
```

**Trap:** Replication lag causes **read-after-write inconsistencies**. You write to master, read from replica, and the replica hasn't caught up. Mitigations:
- Use semi-sync replication (reduces window but doesn't eliminate it)
- ProxySQL read/write splitting with query rules (send deterministic reads to master)
- Application-level tracking: store last-write-timestamp and check replica lag before reading

---

## 9. Replication Lag & Consistency

### 9.1 Causes of Replication Lag

1. **Long-running writes on master:** A single large UPDATE/INSERT blocks the binlog and delays other events.
2. **Slave apply is single-threaded** (without parallel replication): The SQL thread applies events sequentially, while master may have committed many transactions concurrently.
3. **Slave hardware slower than master:** Weaker CPU, less memory, slower disks, network latency.
4. **Large transactions:** A single row change and a 1M-row bulk insert produce the same binlog event size, but the bulk insert takes longer to apply. Then `Seconds_Behind_Master` spikes.
5. **Lock contention on slave:** Long-running queries on slave can block the SQL thread.
6. **DDL operations:** `ALTER TABLE` on master replicates as a single event — slave must execute it while holding exclusive metadata locks.

### 9.2 Impact of Lag

- Stale reads (reporting shows old data)
- Read-after-write inconsistency (user writes, refresh, sees old data)
- Cascading failures (replica promotion with missing data)
- Backup inconsistency (backup from lagging replica is behind master)

### 9.3 Monitoring Lag Precisely

```sql
-- pt-heartbeat (Percona Toolkit):
-- On master:
pt-heartbeat --update --database percona --create-table --daemonize

-- On slave:
pt-heartbeat --monitor --database percona --master-server-id=1
-- Output: 0.01s  0.02s  0.01s  (precise, clock-independent lag)

-- Or query the heartbeat table:
SELECT * FROM percona.heartbeat;
```

### 9.4 Mitigation Strategies

**1. Semi-sync replication:** Master waits for at least one slave ack. Reduces lag window but doesn't eliminate it.

**2. ProxySQL read/write splitting:**

```
-- ProxySQL rule:
-- All writes to hostgroup 0 (master)
-- Reads to hostgroup 1 (replicas), but with:
--   If replica lag > threshold → send to master
--   If connection has written within N ms → send to master
```

**3. Application-level read-after-write consistency:**

```php
// Track last write time per user:
session(['last_write_at' => now()]);

// When reading, check if replicas have caught up:
$lag = DB::select("SHOW SLAVE STATUS")[0]->Seconds_Behind_Master ?? 0;
$lastWriteAt = session('last_write_at');

if ($lastWriteAt && $lag > 0) {
    // Read from master for this request
    $orders = DB::connection('master')->select(...);
} else {
    $orders = DB::connection('replica')->select(...);
}
```

**4. Load balancing with lag awareness:**

```go
// Go pseudocode:
type ReplicaPool struct {
    replicas []*sql.DB
    lag      map[string]time.Duration
}

func (p *ReplicaPool) getReplica(lastWrite time.Time) *sql.DB {
    // Filter replicas that have caught up past lastWrite
    current := time.Now()
    for _, replica := range p.replicas {
        lag := p.lag[replica.key]  // from pt-heartbeat
        if current.Add(-lag).After(lastWrite) {
            return replica
        }
    }
    return master  // fallback
}
```

### 9.5 Monitoring Replication Health

```sql
-- Check all replicas:
SHOW SLAVE STATUS\G

-- Critical queries:
SELECT
    VARIABLE_VALUE AS seconds_behind_master
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Seconds_behind_master';

-- Check for errors:
SELECT * FROM mysql.slave_relay_log_info;
```

**Trap:** Assuming replication lag is **always small**. Under load or schema changes, lag can grow to **hours**. Monitor it with thresholds. Alert if `Seconds_Behind_Master > 300` (5 minutes).

**Trap:** Using replication as a backup strategy. **Replication is not backup.** If a `DROP TABLE` is executed on master, the slave immediately drops the table too. You can't recover from logical errors via replication. Always have actual backups (XtraBackup, mysqldump, PITR via binlog).

**Trap:** Not monitoring replication health. A slave can be stopped (`Slave_SQL_Running: No`) for days while serving stale data. Set up monitoring that alerts on:
- `Slave_IO_Running != Yes`
- `Slave_SQL_Running != Yes`
- `Seconds_Behind_Master > threshold`
- `Last_IO_Error` or `Last_SQL_Error` non-empty

---

## 10. Tier 2 Q&A Drill

**Q1:** You have a composite index `(a, b, c)`. Which queries can use the index efficiently?

```sql
-- (a) WHERE a = 1 AND b = 2 AND c > 3
-- (b) WHERE b = 2 AND c = 3
-- (c) WHERE a = 1 ORDER BY b
-- (d) WHERE a = 1 AND c = 3
```

<details>
<summary>Answer</summary>

- (a) Yes — full index usage: equality → equality → range.
- (b) No — missing leading column `a`. MySQL can't use this (no skip scan in older versions; even in 8.0, skip scan is limited).
- (c) Yes — `a` filters, `b` provides sort order.
- (d) Partially — `a` uses the index, but `c` cannot be used because `b` is skipped. The index filters on `a` and then `c` is evaluated as a filter (Using index condition if ICP is used, or Using where).
</details>

---

**Q2:** What's the difference between `Extra: Using index` and `Extra: Using index condition` in EXPLAIN?

<details>
<summary>Answer</summary>

- `Using index` = **index-only scan**. All columns needed by the query are in the index. No table access. The index covers the query.
- `Using index condition` = **Index Condition Pushdown (ICP)**. The storage engine evaluates part of the WHERE clause during index traversal, reducing the number of rows read from the table. But the query does need to access the table for some columns.

Both indicate an index is used, but `Using index` is more efficient (no table access at all).
</details>

---

**Q3:** How does MySQL's MVCC differ from PostgreSQL's? List at least 3 differences.

<details>
<summary>Answer</summary>

1. **Old row version storage:** MySQL uses undo log (separate tablespace); PostgreSQL stores dead tuples in the heap (VACUUM reclaims).
2. **Phantom prevention:** MySQL's REPEATABLE READ uses next-key locking (gap + record) to prevent phantoms; PostgreSQL's snapshot isolation prevents phantom reads via MVCC but allows write skew.
3. **Cleanup:** MySQL has a purge thread that periodically removes undo log entries; PostgreSQL relies on VACUUM to reclaim dead tuple space.
4. **Tuple visibility:** MySQL uses system columns (DB_TRX_ID, DB_ROLL_PTR); PostgreSQL uses tuple header fields (xmin, xmax) with CLOG for transaction status.
5. **READ COMMITTED implementation:** In MySQL, each statement creates a new read view (statement-level snapshot). In PostgreSQL, READ COMMITTED also re-evaluates plan nodes for each row to get latest committed data.
</details>

---

**Q4:** A `SELECT ... FOR UPDATE` on a non-indexed column locked every row in a 10M-row table for 30 seconds. Explain why and how to prevent it.

<details>
<summary>Answer</summary>

Without an index, MySQL performs a **full table scan** to find matching rows. For every row scanned, InnoDB acquires a next-key lock (even if it doesn't match). This locked all 10M rows.

Prevention:
1. Create an index on the WHERE column.
2. Use `LIMIT` (if applicable) so only returned rows are locked (but still need an index for efficient scan).
3. Use READ COMMITTED isolation (reduces gap locking).
4. Consider `NOWAIT` or `SKIP LOCKED` if the scenario allows.

Rule: **Every UPDATE/DELETE must use an indexed WHERE clause.**
</details>

---

**Q5:** What causes deadlocks under REPEATABLE READ that wouldn't happen under READ COMMITTED?

<details>
<summary>Answer</summary>

Gap locking is the primary difference. Under REPEATABLE READ:
- `SELECT ... FOR UPDATE` acquires next-key locks (record + gap).
- Two transactions can acquire overlapping gap locks.
- When both try to INSERT into the same gap, they wait on each other's insert intention locks → deadlock.

Under READ COMMITTED:
- Gap locking is disabled for indexing and searching.
- The deadlock scenario doesn't occur.

Example:
```
T1: SELECT * FROM orders WHERE id > 10 FOR UPDATE;  -- gap lock on (10, ∞)
T2: SELECT * FROM orders WHERE id > 10 FOR UPDATE;  -- same gap lock (shared)
T1: INSERT INTO orders (id) VALUES (15);  -- waits for T2's gap
T2: INSERT INTO orders (id) VALUES (20);  -- waits for T1's gap → DEADLOCK
```
</details>

---

**Q6:** You see `Extra: Using filesort; Using temporary` in EXPLAIN for a query. What's happening and how do you fix it?

<details>
<summary>Answer</summary>

- `Using filesort`: MySQL couldn't use an index to satisfy ORDER BY. It's sorting in memory (or on disk).
- `Using temporary`: MySQL created a temporary table (for GROUP BY, DISTINCT, or UNION.

Fix: Create an index that satisfies both the WHERE filter and the ORDER BY/GROUP BY. For example, if query is:

```sql
SELECT status, COUNT(*) FROM orders
WHERE created_at > '2024-01-01'
GROUP BY status
ORDER BY status;
```

Index: `(created_at, status)` — but this is suboptimal for GROUP BY.

Better: `(status, created_at)` — GROUP BY matches index left prefix, range on created_at.

If the GROUP BY columns are different from ORDER BY, you may need to either accept a filesort or redesign the query.
</details>

---

**Q7:** What are the pros and cons of semi-synchronous replication vs asynchronous replication?

<details>
<summary>Answer</summary>

| Aspect | Async | Semi-sync |
|--------|-------|-----------|
| Write latency | Lower (master doesn't wait) | Higher (master waits for slave ack) |
| Data loss risk | Higher (master crash before slave pulls binlog) | Lower (slave has binlog before master commits) |
| Availability | Master remains available if all slaves fail | Master falls back to async if no slave acks (timeout) |
| Throughput | Higher | Slightly lower (network round trip) |

Semi-sync doesn't guarantee slave has **applied** the transaction — only that it's in the relay log. In MySQL 8.0, WAIT_AFTER_SYNC (default) waits after the write to binlog but before the commit.

For true zero-data-loss, use Group Replication with `group_replication_consistency = BEFORE_ON_PRIMARY_FAILOVER`.
</details>

---

**Q8:** `Seconds_Behind_Master` shows 0 but your application reads stale data from the replica. Why?

<details>
<summary>Answer</summary>

`Seconds_Behind_Master` is unreliable because:
1. **Clock skew:** It's calculated as master binlog timestamp - slave system timestamp. If clocks differ, it's wrong.
2. **Large transactions:** If a large transaction is applying on the slave, `Seconds_Behind_Master` may show 0 or NULL temporarily (the slave is "catching up" but the single large transaction hasn't committed).
3. **Reporting thread:** If the replica has its own long-running queries, they may see stale snapshots even though the SQL thread is up to date.

Use **pt-heartbeat** for precise lag measurement, or use GTID-based checks (compare `gtid_subset`/`gtid_subtract` between master and replica).
</details>

---

**Q9:** Design a job queue in MySQL that handles high concurrency without deadlocks.

<details>
<summary>Answer</summary>

```sql
-- Table:
CREATE TABLE job_queue (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    status ENUM('pending', 'processing', 'done', 'failed') NOT NULL DEFAULT 'pending',
    payload JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index:
CREATE INDEX idx_job_status_created ON job_queue(status, created_at);

-- Dequeue (MySQL 8.0+):
START TRANSACTION;
SELECT id, payload
FROM job_queue
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- SKIP LOCKED avoids contention

UPDATE job_queue SET status = 'processing' WHERE id = ?;
COMMIT;
```

Key design decisions:
- `SKIP LOCKED` (8.0+) allows multiple workers to dequeue without blocking each other.
- `LIMIT 1` minimizes locked rows.
- Index on `(status, created_at)` enables efficient scan without table lock.
- READ COMMITTED isolation reduces gap locking.
- Retry logic for deadlocks (error 1213).
</details>

---

**Q10:** You're migrating from PostgreSQL to MySQL. What are the top 5 things you need to change in your queries/design?

<details>
<summary>Answer</summary>

1. **Partial indexes → generated columns:** Replace `CREATE INDEX ... WHERE` with generated column + index, or functional index (8.0.13+).

2. **REPEATABLE READ vs SERIALIZABLE:** PostgreSQL SSI (Serializable Snapshot Isolation) uses conflict detection; MySQL's SERIALIZABLE uses locking. Don't just flip the isolation level — redesign for MySQL's RR with gap locking.

3. **RETURNING clause:** MySQL doesn't support `INSERT ... RETURNING`, `UPDATE ... RETURNING`, or `DELETE ... RETURNING`. Use `LAST_INSERT_ID()` for auto-increment, or `SELECT after INSERT` inside a transaction.

4. **DISTINCT ON / window functions:** MySQL supports window functions (8.0+), but `DISTINCT ON` is a PostgreSQL extension. Use `ROW_NUMBER() OVER (PARTITION BY ...)` instead.

5. **Arrays/JSONB → JSON:** MySQL's JSON type is less capable than PostgreSQL's JSONB (no GIN index for path operations, no indexed operators). Use generated columns with indexes for JSON path queries.

Bonus: MySQL's default REPEATABLE READ with gap locking means more deadlocks. Use READ COMMITTED in production.
</details>

---

> **End of Tier 2.** Next: [`03-senior.md`](./03-senior.md) — High availability, sharding, partitioning, performance tuning, schema migration tools, backup & recovery, production runbooks.
