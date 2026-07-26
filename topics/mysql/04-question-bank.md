# MySQL — Question Bank & Drill Material

A comprehensive MySQL interview question bank for a Senior Backend Engineer (8+ years, PHP/Laravel with PostgreSQL experience, Go, JS). Covers architecture, internals, SQL, indexing, transactions, replication, HA, operations, and production debugging — with comparisons to PostgreSQL throughout.

> **How to use:** Rapid-Fire sections use `<details>` blocks for self-quizzing. Trap/Follow-up callouts highlight edge cases. Code Puzzles include reasoning walkthroughs. Debugging Scenarios simulate real production incidents. System Design Prompts tie to your existing PostgreSQL experience.

---

## Table of Contents

1. [Rapid-Fire: MySQL Architecture & InnoDB (30 questions)](#1-rapid-fire-mysql-architecture--innodb-30-questions)
2. [Rapid-Fire: SQL & Data Types (30 questions)](#2-rapid-fire-sql--data-types-30-questions)
3. [Rapid-Fire: Indexing & Query Optimization (30 questions)](#3-rapid-fire-indexing--query-optimization-30-questions)
4. [Rapid-Fire: Transactions, Locking & Deadlocks (25 questions)](#4-rapid-fire-transactions-locking--deadlocks-25-questions)
5. [Rapid-Fire: Replication (20 questions)](#5-rapid-fire-replication-20-questions)
6. [Rapid-Fire: HA, Sharding & Operations (25 questions)](#6-rapid-fire-ha-sharding--operations-25-questions)
7. [Code Puzzles](#7-code-puzzles)
8. [Debugging Scenarios](#8-debugging-scenarios)
9. [System Design Prompts](#9-system-design-prompts)
10. [Questions to Ask the Interviewer](#10-questions-to-ask-the-interviewer)
11. [Red Flags to Avoid](#11-red-flags-to-avoid)

---

## 1. Rapid-Fire: MySQL Architecture & InnoDB (30 questions)

**Q1:** What are the main storage engines in MySQL and when would you use each?

<details><summary>Answer</summary>

- **InnoDB** (default since 5.5) — transactions, FK constraints, crash recovery, row-level locking. Use for everything unless you have a specific reason not to.
- **MyISAM** — no transactions, table-level locking, full-text search (pre-5.6). Used for read-heavy data warehouses in legacy systems. **Avoid in 8.0** — no crash recovery.
- **Memory (HEAP)** — data in RAM, no durability. Use for temporary tables, caching, session data.
- **CSV** — plain CSV files on disk. Useful for data exchange (Excel/ETL).
- **Archive** — high compression, no indexing (other than auto-increment PK). Use for audit logs/data warehousing.
- **NDB (Cluster)** — distributed in-memory, high availability. Use for carrier-grade HA with <10ms failover.

> **Postgres comparison:** PostgreSQL doesn't have pluggable storage engines — the heap is built-in.

</details>

**Q2:** What is the InnoDB buffer pool? How does it work?

<details><summary>Answer</summary>

The buffer pool is InnoDB's main memory cache for data pages and index pages. It uses a variation of LRU (with midpoint insertion strategy — the page is inserted at the midpoint (5/8 from the head) and only promoted to the "young" sublist if accessed again). This prevents large scans from flushing the entire cache.

Key parameters:
- `innodb_buffer_pool_size` — typically 60-80% of available RAM
- `innodb_buffer_pool_instances` — divides the pool into multiple instances to reduce contention (default: 1 if <1GB, 8 if >=1GB)

**Trap:** Setting buffer pool too large (> available RAM) causes swapping. Setting it too small causes excessive disk I/O.

</details>

**Q3:** Explain redo log (WAL) in InnoDB. How is it different from PostgreSQL's WAL?

<details><summary>Answer</summary>

The redo log records every change made to data pages before they're written to disk (Write-Ahead Logging). On crash recovery, InnoDB replays the redo log to restore committed transactions.

Key points:
- Written sequentially (circular), very fast I/O
- `innodb_log_file_size` (pre-8.0) / `innodb_redo_log_capacity` (8.0.30+)
- `innodb_flush_log_at_trx_commit`:
  - 1 (default): flush to disk on every commit — safest, slowest
  - 2: flush per second only — faster, lose 1s of data on crash
  - 0: no flush — fastest, lose data on crash

**Postgres comparison:** PostgreSQL's WAL is similar but: Postgres WAL is logical at a higher level (row changes) vs InnoDB redo log is page-physical. Postgres can do WAL archiving and PITR via `pg_wal`; MySQL uses binary logs for PITR, not the redo log.

</details>

**Q4:** What is the undo log? How does it relate to MVCC?

<details><summary>Answer</summary>

The undo log stores the "before image" of a row so that:
1. A ROLLBACK can restore the original value
2. MVCC readers can reconstruct an older version of a row (consistent read)

Each transaction gets its own undo segment. The undo log is stored in the system tablespace (ibdata1) or in separate undo tablespaces (8.0+). Old versions in the undo log are purged by the purge system when no active transaction needs them.

**Trap:** Long-running transactions prevent purge, causing undo log bloat and the "history list length" to grow. Monitor `SHOW ENGINE INNODB STATUS` → `History list length`. A high value (millions) signals a transaction retention problem.

</details>

**Q5:** What is the doublewrite buffer? Why is InnoDB safer than MyISAM even on a power loss?

<details><summary>Answer</summary>

InnoDB writes pages in 16KB chunks. If the OS crashes while writing a 16KB page (torn page — only 8KB written), the page is corrupt. The doublewrite buffer:
1. Writes the page to a contiguous 128-page buffer (2MB) on disk
2. Then writes to its final location
3. On recovery, if a torn page is detected, it's repaired from the doublewrite copy

**Trap:** The doublewrite buffer adds write amplification. If you have a filesystem that guarantees atomic page writes (e.g., ZFS, Fusion-IO, certain SSDs with power-loss protection), you can disable it (`innodb_doublewrite = 0`) for ~2x write throughput.

</details>

**Q6:** What is the change buffer? When does it help?

<details><summary>Answer</summary>

The change buffer caches changes (INSERT/UPDATE/DELETE) to **non-unique secondary index pages** when the page is not in the buffer pool. Instead of reading the page from disk immediately, the change is recorded in the buffer and merged later when the page is read or during idle merge operations.

This is a significant optimization for workloads with many secondary indexes where the same pages aren't frequently accessed — it avoids random I/O to read the index page for every DML.

**Trap:** The change buffer can grow large and slow down crash recovery. `innodb_change_buffer_max_size` (default 25% of buffer pool) limits its size.

</details>

**Q7:** What is the Adaptive Hash Index (AHI)?

<details><summary>Answer</summary>

InnoDB monitors index searches. If it detects that a B+Tree index is being accessed with a pattern that could benefit from hashing (e.g., many `WHERE id = ?` lookups over a frequently accessed range), it builds a hash index in memory on top of the B+Tree. This turns O(log n) lookups into O(1).

**Trap:** AHI is built on demand and consumes memory from the buffer pool. Under heavy DML workloads, maintaining the AHI can become a contention point (seen as `btr0sea.c` mutex contention). `innodb_adaptive_hash_index` can be disabled.

</details>

**Q8:** How does InnoDB's MVCC work under REPEATABLE READ vs READ COMMITTED?

<details><summary>Answer</summary>

At REPEATABLE READ, a **consistent read view** (snapshot) is created at the **first read** in the transaction. All subsequent reads within the same transaction see that snapshot.

At READ COMMITTED, each **statement** gets its own read view. You'll see commits from other transactions between your statements.

**Postgres comparison:** PostgreSQL's MVCC is similar but: Postgres stores multiple row versions in the table heap itself (no separate undo log), using `t_xmin`/`t_xmax` system columns. VACUUM cleans up dead tuples.

</details>

**Q9:** How does InnoDB organize data on disk? Describe the page structure.

<details><summary>Answer</summary>

Data is stored in **tablespaces** (`.ibd` files per table by default). The hierarchy is:

```
Tablespace → Extent (1MB = 64 pages) → Page (16KB default) → Records
```

Each 16KB page contains:
- **Page header** (FIL header): page number, checksum, LSN of the last modification
- **Page body**: records in a singly-linked list (infimum, user records, supremum)
- **Page directory**: array of offsets to records for binary search
- **Page trailer**: checksum copy and LSN

Pages for B+Tree leaf nodes store the actual row data. Non-leaf pages store key values + child page numbers.

**Trap:** `innodb_page_size` can be set to 4KB, 8KB, or 32KB at instance init. Cannot be changed after.

</details>

**Q10:** What is a clustered index in InnoDB?

<details><summary>Answer</summary>

Every InnoDB table has a clustered index (the table IS the index):
- If you define a PRIMARY KEY, it becomes the clustered index
- If no PK, InnoDB picks the first UNIQUE NOT NULL index
- If neither, InnoDB generates a hidden 6-byte `GEN_CLUST_INDEX` (row ID)

The clustered index stores the entire row data in its leaf pages. Secondary indexes store the primary key value as a pointer to the row.

**Trap:** A large primary key (e.g., UUID v4) in a secondary index means every secondary index also stores that large UUID — this is why auto-increment BIGINT is often faster in MySQL.

**Postgres comparison:** PostgreSQL uses heap tables with a separate index structure. Index entries point to physical (page, offset) via ctid.

</details>

**Q11:** How is the data dictionary stored in MySQL 8.0? How is it different from 5.7?

<details><summary>Answer</summary>

MySQL 8.0 replaced the MyISAM-based `.frm` files and system tables with a **transactional data dictionary** stored in InnoDB tables (in the `mysql` tablespace). This means DDL operations are now atomic and crash-safe.

Additionally, `INFORMATION_SCHEMA` tables are now implemented as views over the data dictionary tables (much faster than 5.7's file-per-table scans).

**Trap:** In 5.7, you could copy `.frm` and `.ibd` files to restore a table. In 8.0, you can't — SDI is embedded in each `.ibd` file. Use `ALTER TABLE ... DISCARD/IMPORT TABLESPACE` instead.

</details>

**Q12:** What is the `ibdata1` file? Why does it grow and can it shrink?

<details><summary>Answer</summary>

`ibdata1` is the **system tablespace**. It contains:
- The data dictionary (pre-8.0)
- The doublewrite buffer
- The change buffer
- Undo logs (before 8.0)

`ibdata1` never shrinks in size even after data is deleted (unless you dump & reload). Use `innodb_file_per_table` (default in 5.6+) to avoid this — then each table has its own `.ibd` file that can be shrunk via `OPTIMIZE TABLE`.

**Trap:** If `ibdata1` grows to 100GB, you can't just delete it. You must mysqldump all data, delete the file, restart MySQL with new config, and restore.

</details>

**Q13:** What information does `SHOW ENGINE INNODB STATUS` provide?

<details><summary>Answer</summary>

It shows:
- Current transaction list (active/locked transactions)
- Lock waits and deadlocks (latest detected deadlock)
- Buffer pool statistics (hit rate, free pages, dirty pages)
- Row operations (inserts, updates, deletes, reads)
- Log and checkpoint info (LSN, checkpoint age)
- I/O thread status (pending I/O, thread counts)

Use `SHOW ENGINE INNODB STATUS\G` (MySQL client) for the full output. The `LATEST DETECTED DEADLOCK` section is the first place to look when debugging deadlocks.

</details>

**Q14:** What is the purpose of `innodb_flush_method`? What values should you use?

<details><summary>Answer</summary>

Controls how InnoDB flushes data and log files to disk:
- `fsync` (default on Linux) — uses `fsync()` for both data and log
- `O_DSYNC` — uses `O_SYNC` for log files
- `O_DIRECT` — bypasses the OS page cache for data files (reduces double-buffering)
- `O_DIRECT_NO_FSYNC` — like `O_DIRECT` but skips `fsync()` after writes (dangerous)

**Recommendation:** `O_DIRECT` is standard for production Linux with InnoDB.

</details>

**Q15:** What is `innodb_io_capacity` and why does it matter?

<details><summary>Answer</summary>

This parameter tells InnoDB the approximate I/O throughput of your storage. It controls how aggressively InnoDB performs background tasks:
- Page flushing (dirty page writeback)
- Change buffer merging
- Purge operations

Set it based on your storage: HDD ~200, SATA SSD ~1000, NVMe ~2000-10000+. Setting it too low causes the buffer pool to fill with dirty pages (checkpoint age stalls writes). Setting it too high causes unnecessary I/O.

</details>

**Q16:** What happens during InnoDB crash recovery? How long does it take?

<details><summary>Answer</summary>

On startup after a crash, InnoDB:
1. Scans the redo log from the last checkpoint
2. Applies redo log entries (redo phase) — rolls forward changes not yet flushed
3. Rolls back uncommitted transactions using the undo log (undo phase)
4. Purges undo log entries no longer needed

Recovery time depends on redo log size, checkpoint distance, and buffer pool size.

**Trap:** Setting the redo log very large (e.g., 64GB) creates long recovery times.

</details>

**Q17:** How does InnoDB handle `AUTO_INCREMENT`? What is the lock behavior?

<details><summary>Answer</summary>

`innodb_autoinc_lock_mode` controls locking:
- **0 (traditional)**: table-level AUTO-INC lock held until statement completes
- **1 (consecutive, 5.1+ default)**: "simple inserts" use lightweight mutex; "bulk inserts" use AUTO-INC table lock
- **2 (interleaved, 8.0 default)**: all inserts use lightweight mutex — highest concurrency but auto-inc values may have gaps

**Trap:** In mode 2 with statement-based replication, auto-increment values may differ on master and replica. Use ROW format.

</details>

**Q18:** What is the `INFORMATION_SCHEMA.INNODB_TRX` table used for?

<details><summary>Answer</summary>

Shows all currently executing transactions:
- `trx_id`, `trx_state` (RUNNING, LOCK WAIT, COMMITTING, ROLLING BACK)
- `trx_started`, `trx_mysql_thread_id`
- `trx_tables_locked`, `trx_rows_locked`, `trx_rows_modified`
- `trx_isolation_level`

Used with `performance_schema.data_locks`/`data_lock_waits` to debug locking issues.

</details>

**Q19:** Explain InnoDB's checkpoint mechanism.

<details><summary>Answer</summary>

InnoDB uses **fuzzy checkpointing** — not all dirty pages are flushed at once. The checkpoint age (how far behind the current redo log LSN is from the last checkpoint LSN) is tracked. When it exceeds a threshold (controlled by `innodb_max_dirty_pages_pct`), the page cleaner thread flushes dirty pages.

**Adaptive flushing** (8.0) adjusts the flush rate based on redo log generation rate.

**Trap:** If flushing can't keep up, writes stall. Watch `Innodb_buffer_pool_wait_free` — if >0, write stalls are happening.

</details>

**Q20:** What is the difference between a tablespace and a schema?

<details><summary>Answer</summary>

In MySQL, a **schema** is synonymous with a **database** — a logical namespace. A **tablespace** is a physical storage container (file). By default, each table gets its own tablespace (`innodb_file_per_table`).

```sql
CREATE TABLESPACE my_ts ADD DATAFILE 'my_ts.ibd';
CREATE TABLE t1 (id INT) TABLESPACE my_ts;
```

</details>

**Q21:** What is ONLINE DDL in InnoDB?

<details><summary>Answer</summary>

InnoDB supports online DDL using the **row log** — changes made during the ALTER are logged and applied after. Operations have different `ALGORITHM` options:
- `INSTANT` (8.0.12+): metadata-only. Adding column to end, renaming column, setting default.
- `INPLACE`: rebuilds table but allows concurrent DML.
- `COPY`: creates temp table, copies data row-by-row, blocks writes.

Use `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE` for zero-downtime operations.

</details>

**Q22:** How does the Log Sequence Number (LSN) work?

<details><summary>Answer</summary>

LSN is a monotonically increasing number representing bytes written to the redo log. Every page header stores the LSN of the last modification. During recovery, InnoDB compares LSNs: if a page's LSN is less than the checkpoint LSN, it needs redo.

Key LSNs in `SHOW ENGINE INNODB STATUS`:
- `Log sequence number`: current position in the redo log
- `Log flushed up to`: last flushed position
- `Last checkpoint at`: position of the last checkpoint

**Trap:** If `Log sequence number - Last checkpoint at` is consistently large, your redo log is too small.

</details>

**Q23:** What is the MySQL connection handling model?

<details><summary>Answer</summary>

MySQL uses **one-thread-per-connection** — each client connection gets its own OS thread. At thousands of connections, thread context switching becomes expensive.

**Thread Pool** (MySQL Enterprise / Percona / MariaDB): a limited number of threads handle all connections.

**Postgres comparison:** Postgres uses one-process-per-connection, making thousands of connections even more expensive. Postgres uses PgBouncer for pooling.

</details>

**Q24:** What happens when you run `SELECT COUNT(*) FROM t` on a large InnoDB table?

<details><summary>Answer</summary>

InnoDB does **not** store row counts for tables (unlike MyISAM). `SELECT COUNT(*)` triggers a **full index scan** of the smallest secondary index or the clustered index. This can be extremely slow on large tables.

```sql
-- For approximate counts:
SHOW TABLE STATUS LIKE 't'\G
SELECT TABLE_ROWS FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 't';
```

</details>

**Q25:** What are the InnoDB purge threads and what happens when they fall behind?

<details><summary>Answer</summary>

The purge thread (`INNODB_PURGE_THREADS`) asynchronously cleans up undo log entries no longer needed by any active transaction. If it falls behind (growing `History list length`):
- Long-running transactions are holding up purge
- Too many DMLs for the purge threads to handle
- The system is I/O bound

`INNODB_PURGE_THREADS` can be increased (default 1, max 32 in 8.0).

</details>

**Q26:** What is `INNODB_METRICS` in the `INFORMATION_SCHEMA`?

<details><summary>Answer</summary>

Exposes hundreds of counters for InnoDB internals — buffer pool reads/writes, row lock waits, adaptive hash index usage, change buffer merges, etc.

Enable/disable counters:
```sql
SET GLOBAL innodb_monitor_enable = module_name;
```

**Postgres comparison:** Postgres uses `pg_stat_*` views. MySQL's INNODB_METRICS requires explicit enabling.

</details>

**Q27:** What is a mini-transaction (mtr) in InnoDB?

<details><summary>Answer</summary>

A mini-transaction is InnoDB's lowest-level atomic unit — it protects B+Tree operations (page splits, merges, rotations). Multiple mtrs comprise a user transaction. Each mtr acquires latches on pages and writes redo log entries.

</details>

**Q28:** What are the read and write I/O threads in InnoDB?

<details><summary>Answer</summary>

InnoDB uses dedicated background threads for I/O:
- **Read threads** (default 4): handle read-ahead and page reads
- **Write threads** (default 4): handle dirty page flushing and change buffer merges

On high-I/O systems (NVMe, multiple tablespaces), increasing these up to 16-32 can improve throughput.

</details>

**Q29:** What is MySQL's data-at-rest encryption?

<details><summary>Answer</summary>

MySQL 8.0 supports transparent tablespace encryption using a keyring plugin:
```sql
CREATE TABLE t1 (id INT) ENCRYPTION='Y';
```

**Postgres comparison:** Postgres has `pg_tde` extension or file-level encryption. MySQL's TDE is tablespace-based and more mature.

</details>

**Q30:** What is the DATA DIRECTORY clause on CREATE TABLE?

<details><summary>Answer</summary>

Allows placing the `.ibd` file on a different path:
```sql
CREATE TABLE t1 (id INT) DATA DIRECTORY = '/mnt/fast_ssd';
```

**Trap:** The path must be absolute and MySQL must have write permissions.

---

## 2. Rapid-Fire: SQL & Data Types (30 questions)

**Q31:** What is the difference between `VARCHAR(N)` and `CHAR(N)`?

<details><summary>Answer</summary>

- `CHAR(N)`: fixed-length, right-padded with spaces (up to 255). Always uses N characters of storage.
- `VARCHAR(N)`: variable-length (up to 65535 bytes). Uses 1 or 2 bytes overhead + actual data length.

Use `CHAR` for fixed-length codes (ISO country codes `CHAR(2)`, status flags `CHAR(1)`). Use `VARCHAR` for variable text.

**Trap:** `VARCHAR(255)` vs `VARCHAR(256)` — at 255, the length prefix is 1 byte; at 256, it becomes 2 bytes.

**Postgres comparison:** In PostgreSQL, `CHAR(N)` and `VARCHAR(N)` are nearly identical. `TEXT` is the same as VARCHAR without a length limit.

</details>

**Q32:** What is the difference between `DATETIME` and `TIMESTAMP`?

<details><summary>Answer</summary>

| Aspect | DATETIME | TIMESTAMP |
|--------|----------|-----------|
| Range | '1000-01-01' to '9999-12-31' | '1970-01-01' to '2038-01-19' |
| Storage | 8 bytes (5.6.4+) | 4 bytes (before 2038) |
| Timezone | Stored as-is | Converted to/from session TZ to UTC |
| Auto-init | No | `DEFAULT CURRENT_TIMESTAMP`, `ON UPDATE` |

**Trap:** The Year 2038 problem is real for `TIMESTAMP`. Use `DATETIME` for dates beyond 2038.

</details>

**Q33:** What are the storage requirements for `TEXT` and `BLOB` types?

<details><summary>Answer</summary>

| Type | Max Size | Storage Prefix |
|------|----------|---------------|
| `TINYTEXT` / `TINYBLOB` | 255 bytes | 1-byte length |
| `TEXT` / `BLOB` | 65,535 bytes (64KB) | 2-byte length |
| `MEDIUMTEXT` / `MEDIUMBLOB` | 16,777,215 bytes (16MB) | 3-byte length |
| `LONGTEXT` / `LONGBLOB` | 4,294,967,295 bytes (4GB) | 4-byte length |

Text/BLOB values are stored **off-page** (overflow pages) when the row exceeds the page size.

**Trap:** `SELECT *` on tables with large TEXT/BLOB columns causes significant I/O.

</details>

**Q34:** What is `DECIMAL` and how does it differ from `FLOAT`/`DOUBLE`?

<details><summary>Answer</summary>

`DECIMAL(p, s)` is a fixed-point type — exact numeric storage. Stored as a binary string (not IEEE 754). `FLOAT` (4 bytes) and `DOUBLE` (8 bytes) are approximate types.

Use `DECIMAL` for money, tax rates. Use `DOUBLE` for scientific calculations where small rounding is acceptable.

</details>

**Q35:** How does MySQL handle `ENUM`? What are the pros and cons?

<details><summary>Answer</summary>

`ENUM('a', 'b', 'c')` is stored as a **1 or 2 byte integer** (0=error, 1='a', 2='b', ...).

Pros: Compact storage, enforced valid values, fast integer comparison.

Cons: Adding values requires `ALTER TABLE` (metadata lock), ORDER BY sorts by internal index (not alphabetically), not portable.

**Postgres comparison:** PostgreSQL has `CREATE TYPE` for custom enums — can add values with `ALTER TYPE ... ADD VALUE` without table locks.

</details>

**Q36:** What is the `JSON` data type in MySQL 8.0? How is it stored?

<details><summary>Answer</summary>

`JSON` is stored internally as binary JSON (similar to BSON). Allows fast key lookups and partial updates.

```sql
CREATE TABLE t (data JSON);
INSERT INTO t VALUES ('{"name": "Alice", "age": 30}');
SELECT data->'$.name' FROM t;

-- Index a JSON field via generated column:
ALTER TABLE t ADD COLUMN name VARCHAR(100) GENERATED ALWAYS AS (data->>'$.name');
CREATE INDEX idx_name ON t(name);
```

**Postgres comparison:** Postgres JSONB is the inspiration. MySQL's JSON is less feature-rich (no GIN-style index).

</details>

**Q37:** What is the difference between `utf8mb3` and `utf8mb4`?

<details><summary>Answer</summary>

- `utf8mb3` (what MySQL calls `utf8`): 1-3 bytes per character. Cannot store emoji (😀) or certain CJK characters.
- `utf8mb4`: 1-4 bytes per character. Full Unicode support.

**Trap:** MySQL's `utf8` is NOT true UTF-8. It's a 3-byte subset. Always use `utf8mb4`.

```sql
CREATE TABLE t (
    name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
);
```

</details>

**Q38:** What is a generated column? What are VIRTUAL vs STORED?

<details><summary>Answer</summary>

A generated column's value is computed from an expression:
```sql
CREATE TABLE t (
    a INT,
    b INT,
    c INT GENERATED ALWAYS AS (a + b) VIRTUAL
);
```

- `VIRTUAL` (default): computed on read, no storage. Can be indexed (5.7+).
- `STORED`: computed on write/update, stored on disk.

</details>

**Q39:** What does `INSERT ... ON DUPLICATE KEY UPDATE` do? How is it different from `REPLACE`?

<details><summary>Answer</summary>

- `ON DUPLICATE KEY UPDATE`: if a UNIQUE/PK violation occurs, UPDATE the existing row.
- `REPLACE`: deletes the old row and inserts a new one (DELETE + INSERT). Side effects: ON DELETE CASCADE fires, auto-increment always increments.

```sql
-- Prefer this:
INSERT INTO users (id, name, email) VALUES (1, 'Alice', 'a@example.com')
ON DUPLICATE KEY UPDATE name = VALUES(name), email = VALUES(email);

-- Dangerous with FKs:
REPLACE INTO users (id, name, email) VALUES (1, 'Alice', 'a@example.com');
```

**Postgres comparison:** PostgreSQL has `ON CONFLICT DO UPDATE` / `ON CONFLICT DO NOTHING`.

</details>

**Q40:** What does `INSERT IGNORE` do?

<details><summary>Answer</summary>

Silently ignores rows that would cause errors (duplicate key, data truncation, constraint violations).

**Trap:** `INSERT IGNORE` ignores **all** errors, including FK violations and `NOT NULL` violations. Use `ON DUPLICATE KEY` for intentional upserts.

</details>

**Q41:** How does `LOAD DATA INFILE` work?

<details><summary>Answer</summary>

Loads data from a text file efficiently (50-100x faster than INSERT):
```sql
LOAD DATA INFILE '/tmp/data.csv' INTO TABLE users
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(id, name, email)
SET created_at = NOW();
```

**Security:** `LOAD DATA LOCAL INFILE` is a security risk — disable it unless needed.

</details>

**Q42:** What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?

<details><summary>Answer</summary>

| Operation | DML/DDL | Rollback | Speed | FKs |
|-----------|---------|----------|-------|-----|
| `DELETE` | DML | Yes (row-by-row undo) | Slow | Respected |
| `TRUNCATE` | DDL | No (implicit commit) | Very fast | Fails if FK ref |
| `DROP` | DDL | No | Instant | Drops table |

**Trap:** In MySQL, `TRUNCATE` drops and recreates the table. Does **not** invoke `ON DELETE` triggers.

</details>

**Q43:** What is `sql_mode`? Name some important modes.

<details><summary>Answer</summary>

`sql_mode` controls MySQL's SQL syntax and validation behavior. Always set it explicitly.

Important modes:
- `ONLY_FULL_GROUP_BY`: columns in SELECT must be in GROUP BY or be aggregates
- `STRICT_TRANS_TABLES`: reject bad data instead of silently truncating
- `NO_ZERO_DATE`: reject '0000-00-00'
- `ERROR_FOR_DIVISION_BY_ZERO`: reject division by zero
- `PIPES_AS_CONCAT`: treat `||` as string concat

**Trap:** Default modes changed between versions. Applications written for 5.6 may break when upgrading to 8.0.

</details>

**Q44:** What is the difference between `UNION` and `UNION ALL`?

<details><summary>Answer</summary>

- `UNION`: combines results and removes **duplicates** (requires sort/temp table — expensive)
- `UNION ALL`: combines all results, preserving duplicates (no sort — fast)

Use `UNION ALL` when you know result sets don't overlap or duplicates are acceptable.

</details>

**Q45:** What is `ROW_COUNT()` and `FOUND_ROWS()`?

<details><summary>Answer</summary>

- `ROW_COUNT()`: number of rows affected by the last DML/DDL.
- `FOUND_ROWS()`: number of rows that **would have been** returned without LIMIT (used with `SQL_CALC_FOUND_ROWS`).

**Trap:** `SQL_CALC_FOUND_ROWS` is deprecated in MySQL 8.0.17+ and slower than two separate queries (`SELECT COUNT(*)` + `SELECT ... LIMIT`).

</details>

**Q46:** How does MySQL handle implicit type conversion? What are the pitfalls?

<details><summary>Answer</summary>

MySQL automatically converts types in comparisons:
```sql
SELECT * FROM users WHERE name = 0;   -- all rows! 'abc' converted to 0 → match
```

Key rule: If one operand is numeric, the other is converted to a number. String-to-integer: leading numbers extracted; `'abc'` becomes `0`.

**Trap:** The `WHERE name = 0` pitfall returns all rows when name is VARCHAR. Also defeats index usage.

</details>

**Q47:** What is `SELECT ... FOR SHARE` (formerly `LOCK IN SHARE MODE`)?

<details><summary>Answer</summary>

Acquires **shared** (S) locks on the rows read, preventing modifications but allowing other shared locks.

`SELECT ... FOR UPDATE` acquires **exclusive** (X) locks — no other transaction can read (with locks) or modify.

Use `FOR SHARE` when you only need to prevent concurrent modifications but don't intend to modify the rows yourself.

</details>

**Q48:** What is a `VISIBLE` / `INVISIBLE` index in MySQL 8.0?

<details><summary>Answer</summary>

An **invisible index** is maintained (DMLs still update it) but not used by the optimizer:
```sql
CREATE INDEX idx_name ON t(name) INVISIBLE;
ALTER TABLE t ALTER INDEX idx_name VISIBLE;
```

Used for testing query performance before dropping an index.

**Trap:** An INVISIBLE index is still updated on DML. Dropping it would save DML overhead.

</details>

**Q49:** What is `ORDER BY ... DESC` performance implication with indexes?

<details><summary>Answer</summary>

MySQL can read indexes both forward (ASC) and backward (DESC). In 8.0, **descending indexes** allow mixing ASC/DESC in composite indexes efficiently:

```sql
CREATE INDEX idx_abc ON t(a ASC, b DESC);
-- Handles: ORDER BY a ASC, b DESC without filesort
```

Before 8.0, a DESC sort after an ASC column would cause a filesort.

</details>

**Q50:** What is `WAIT_FOR_EXECUTED_GTID_SET` used for?

<details><summary>Answer</summary>

Implements **read-after-write consistency**:
```sql
SET @gtid = LAST_INSERTED_GTID;
SELECT WAIT_FOR_EXECUTED_GTID_SET(@gtid, 10);  -- wait up to 10 seconds
```

**Trap:** Blocks the connection until the GTID is applied or timeout expires. Use short timeout (1-5s) and fall back to master read.

</details>

**Q51:** What are the different `ROW_FORMAT` options for InnoDB?

<details><summary>Answer</summary>

- `REDUNDANT`: old format (MySQL <5.0.3)
- `COMPACT`: default since 5.0.3
- `DYNAMIC`: default since 5.7 — stores TEXT/BLOB completely off-page (20 bytes in-page)
- `COMPRESSED`: like DYNAMIC but page-level compression (zlib/lz4)

**Trap:** `COMPRESSED` adds CPU overhead for every page read/write.

</details>

**Q52:** What is the difference between `NOW()`, `SYSDATE()`, and `UTC_TIMESTAMP()`?

<details><summary>Answer</summary>

- `NOW()`: statement start time (same for all rows in multi-row insert)
- `SYSDATE()`: actual time of evaluation (different for long-running statements)
- `UTC_TIMESTAMP()`: current UTC time

**Trap:** `SYSDATE()` breaks replication with statement-based logging. Set `sysdate_is_now = ON` or use `NOW()`.

</details>

**Q53:** What is `ON UPDATE CURRENT_TIMESTAMP`?

<details><summary>Answer</summary>

Automatically sets a TIMESTAMP/DATETIME column to the current timestamp when the row is updated:
```sql
CREATE TABLE t (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

To avoid auto-update: `UPDATE t SET name='x', updated_at = updated_at WHERE id = 1`.

</details>

**Q54:** What is the `CHECK` constraint in MySQL 8.0?

<details><summary>Answer</summary>

MySQL 8.0.16+ supports `CHECK` constraints (parsed but ignored before):
```sql
CREATE TABLE t (
    age INT CHECK (age >= 0 AND age <= 150),
    status VARCHAR(10) CHECK (status IN ('active', 'inactive'))
);
```

**Postgres comparison:** Postgres has supported CHECK since the beginning.

</details>

**Q55:** What is `FOREIGN KEY` constraint behavior in MySQL?

<details><summary>Answer</summary>

FK constraints are only enforced by **InnoDB**:
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE ON UPDATE RESTRICT
);
```

Referential actions: `RESTRICT`, `CASCADE`, `SET NULL`, `NO ACTION` (same as RESTRICT).

**Trap:** FK constraints require an index on the referencing column(s). MySQL auto-creates one.

**Postgres comparison:** Postgres supports deferrable constraints (`INITIALLY DEFERRED`), which MySQL does not.

</details>

**Q56:** How does MySQL handle `GROUP BY` vs standard SQL?

<details><summary>Answer</summary>

In standard SQL, every column in `SELECT` must be in `GROUP BY` or an aggregate. With `ONLY_FULL_GROUP_BY` enabled (default since 5.7), MySQL enforces this. Without it, MySQL picks an **arbitrary value** per group — non-deterministic and dangerous.

**Trap:** Always enable `ONLY_FULL_GROUP_BY`.

</details>

**Q57:** What happens when you insert a value that exceeds VARCHAR(N)?

<details><summary>Answer</summary>

Depends on `sql_mode`:
- With `STRICT_TRANS_TABLES`: fails with `ERROR 1406: Data too long for column`
- Without strict mode: value is truncated silently (with a warning)

**Trap:** Silent truncation causes data loss. Always use `STRICT_TRANS_TABLES`.

</details>

**Q58:** How do you find and kill a long-running query?

<details><summary>Answer</summary>

```sql
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE COMMAND != 'Sleep' AND TIME > 30 ORDER BY TIME DESC;

KILL CONNECTION 12345;  -- drop session
KILL QUERY 12345;       -- kill only the query (leaves transaction open)
```

**Trap:** `KILL QUERY` leaves the transaction open holding locks. Use `KILL CONNECTION`.

</details>

**Q59:** How does `LOAD DATA` compare to INSERT performance?

<details><summary>Answer</summary>

`LOAD DATA INFILE` is 50-100x faster than row-by-row INSERT because:
- Bulk-loading bypasses the I/O layer for individual rows
- Can disable indexes temporarily: `FOREIGN_KEY_CHECKS=0; UNIQUE_CHECKS=0`

</details>

**Q60:** What is `FLUSH STATUS` used for?

<details><summary>Answer</summary>

Resets session-level status variables for profiling a specific query:
```sql
FLUSH STATUS;
SELECT * FROM users WHERE email = 'test@example.com';
SHOW SESSION STATUS LIKE 'Handler%';
```

Common counters: `Handler_read_rnd_next` (high = full scan), `Handler_read_key` (high = good index usage).

---

## 3. Rapid-Fire: Indexing & Query Optimization (30 questions)

**Q61:** How does a B+Tree index work in InnoDB?

<details><summary>Answer</summary>

A B+Tree index stores data in a balanced tree:
- **Non-leaf nodes**: key values and pointers to child nodes (routing table)
- **Leaf nodes**: actual data (clustered) or PK values (secondary). Linked in a doubly-linked list for range scans.

Characteristics:
- Height is usually 2-4 levels (even for millions of rows)
- Fanout is high (~500-1000 children per node)
- Range scans are fast because leaf pages are linked

**Postgres comparison:** Postgres B-Trees are similar but heap-based (no clustered index). Postgres also supports GiST, GIN, BRIN, hash indexes.

</details>

**Q62:** What is the "leftmost prefix rule" for composite indexes?

<details><summary>Answer</summary>

Given a composite index on `(a, b, c)`:
- `WHERE a = ?` — yes (leftmost prefix)
- `WHERE a = ? AND b = ?` — yes
- `WHERE a = ? AND b = ? AND c = ?` — yes
- `WHERE b = ?` — no (misses leftmost column)
- `WHERE a = ? AND c = ?` — partially (a uses index, c filtered row-by-row)

**Trap:** If the condition on middle column is a range (`b > 10`), subsequent columns are filtered row-by-row.

</details>

**Q63:** What does `EXPLAIN`'s `type` column mean? Order from best to worst.

<details><summary>Answer</summary>

From best to worst:
1. **const**: PK/unique lookup with constant condition
2. **eq_ref**: one row per joined row, using PK/unique in JOIN
3. **ref**: non-unique index lookup
4. **range**: indexed range scan
5. **index**: full index scan
6. **ALL**: full table scan

**Trap:** `ALL` is acceptable for tiny tables (<1000 rows). `range` or `ref` is good for large tables.

</details>

**Q64:** What does the `Extra` column in EXPLAIN reveal?

<details><summary>Answer</summary>

- `Using index`: **covering index** — all data from index (fastest)
- `Using where`: filtered after storage engine retrieval
- `Using index condition`: Index Condition Pushdown — part of WHERE checked in storage engine
- `Using filesort`: extra sort pass (avoid if possible)
- `Using temporary`: temp table created (avoid if possible)
- `Using MRR`: Multi-Range Read optimization

**Trap:** `Using filesort` + `Using temporary` together on large result set = slow query.

</details>

**Q65:** What is a covering index? Why is it faster?

<details><summary>Answer</summary>

A covering index contains **all columns** needed by a query. InnoDB doesn't need to access the clustered index — query satisfied entirely from the index B+Tree.

```sql
-- Index: idx_name_email ON users(name, email)
SELECT name, email FROM users WHERE name = 'Alice';
-- Extra: "Using index" (covering)
```

Benefits: eliminates table access I/O, smaller pages, less buffer pool pressure.

**Trap:** Adding too many columns to make an index covering slows down DML. Balance read vs write cost.

</details>

**Q66:** What is Index Merge? When does the optimizer use it?

<details><summary>Answer</summary>

MySQL can use **multiple indexes** and merge the results. Three algorithms:
1. **Intersection** (AND)
2. **Union** (OR)
3. **Sort-Union**

**Trap:** Index merge is often less efficient than a single composite index. It's often a sign to create a composite index.

</details>

**Q67:** What is `EXPLAIN ANALYZE` (MySQL 8.0.18+)?

<details><summary>Answer</summary>

`EXPLAIN ANALYZE` actually **executes** the query and returns timing for each step:
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 1\G
```

Shows actual time per iteration (first..last in ms), rows returned, and loops. Invaluable for identifying optimizer misestimates.

**Postgres comparison:** Postgres has `EXPLAIN (ANALYZE, BUFFERS)`.

</details>

**Q68:** What is the Performance Schema?

<details><summary>Answer</summary>

`performance_schema` (default enabled in 5.7+) provides instrumentation for MySQL's internal operations:
- Query execution statistics (waits, timing, lock times)
- Memory usage by subsystem
- Digest-based query analysis

```sql
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT/1000000000 AS total_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

**Trap:** Adds ~10-20% CPU overhead with all instruments enabled. Disable selectively.

</details>

**Q69:** What is the `sys` schema?

<details><summary>Answer</summary>

The `sys` schema (5.7+) is a set of views on top of `performance_schema`:
```sql
SELECT * FROM sys.memory_global_total;
SELECT * FROM sys.statements_with_full_table_scans;
SELECT * FROM sys.innodb_lock_waits;
SELECT * FROM sys.statement_analysis ORDER BY avg_latency DESC LIMIT 10;
```

First place to look for performance troubleshooting.

</details>

**Q70:** How do you configure and use the slow query log?

<details><summary>Answer</summary>

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
log_queries_not_using_indexes = 1
```

Analyze with `pt-query-digest`:
```bash
pt-query-digest /var/log/mysql/mysql-slow.log
```

**Trap:** `log_queries_not_using_indexes` can flood the slow log. Use `min_examined_row_limit`.

</details>

**Q71:** What is Index Condition Pushdown (ICP)?

<details><summary>Answer</summary>

ICP pushes part of the WHERE condition to the storage engine. Without ICP, InnoDB reads each row and checks WHERE in the server layer. With ICP, InnoDB evaluates conditions on index columns before reading the row.

```sql
-- Index: (zipcode, lastname)
SELECT * FROM users WHERE zipcode = '10001' AND lastname LIKE '%son%';
-- With ICP: InnoDB checks lastname in the index before reading the row
```

Signaled by `Using index condition` in EXPLAIN.

</details>

**Q72:** What is a hash join in MySQL 8.0?

<details><summary>Answer</summary>

MySQL 8.0.18+ added **hash joins** for join queries without index access. Builds a hash table from the smaller table and probes with the larger table.

```sql
SELECT * FROM big_table t1 JOIN small_table t2 ON t1.col = t2.col;
```

**Trap:** Hash joins require memory (`join_buffer_size`). If hash table doesn't fit, MySQL spills to disk.

**Postgres comparison:** PostgreSQL has had hash joins since 7.x (2000).

</details>

**Q73:** What is the difference between `eq_ref` and `ref` in EXPLAIN?

<details><summary>Answer</summary>

- **eq_ref**: one row per probe. Used when JOIN uses a PK or unique NOT NULL index.
- **ref**: multiple rows may match. Used when the index is non-unique.

```sql
-- eq_ref: JOIN on PK
SELECT * FROM orders JOIN users ON orders.user_id = users.id;
-- ref: non-unique index filter
SELECT * FROM orders WHERE status = 'active';
```

</details>

**Q74:** What is the MRR (Multi-Range Read) optimization?

<details><summary>Answer</summary>

MRR optimizes range scans on secondary indexes: collects a batch of key values, sorts by PK, then reads rows from the clustered index in PK order (sequential I/O instead of random I/O).

Visible as `Using MRR` in EXPLAIN.

</details>

**Q75:** How do you optimize `ORDER BY ... LIMIT` queries?

<details><summary>Answer</summary>

The optimal index for `ORDER BY col LIMIT N` is an index on `col`. MySQL scans the index in order and stops after N rows — no filesort.

```sql
-- Index: idx_created_at ON t(created_at)
SELECT * FROM t ORDER BY created_at DESC LIMIT 10;
```

For composite ORDER BY, ensure the index matches ORDER BY columns in the same order and direction.

</details>

**Q76:** What is the impact of `SELECT *` on query performance?

<details><summary>Answer</summary>

Harmful because:
1. Network overhead (sends unused columns)
2. Buffer pool waste
3. Prevents covering index usage
4. I/O amplification with TEXT/BLOB

```sql
-- Bad: can't use covering index
SELECT * FROM users WHERE email = 'test@example.com';
-- Good: can use covering index
SELECT id, name FROM users WHERE email = 'test@example.com';
```

</details>

**Q77:** What is "filesort"? When does MySQL use a temp table vs sort buffer?

<details><summary>Answer</summary>

Filesort sorts result sets when ORDER BY can't be satisfied by an index. Two algorithms:
1. **Single-pass**: reads all columns, sorts in memory (`sort_buffer_size`).
2. **Two-pass**: reads sort key + row pointer, sorts, then reads full rows.

A **temporary table** is used for GROUP BY with non-grouped columns, DISTINCT combined with ORDER BY, or UNION.

**Trap:** `sort_buffer_size` is allocated per session. Setting it too high (e.g., 256MB) can cause memory exhaustion with many connections.

</details>

**Q78:** How do you find unused indexes?

<details><summary>Answer</summary>

```sql
SELECT * FROM sys.schema_unused_indexes;
-- Or from performance_schema:
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NOT NULL AND COUNT_READ = 0 AND COUNT_FETCH = 0;
```

**Follow-up:** Make unused indexes `INVISIBLE` first (8.0+) before dropping, to verify no queries need them.

</details>

**Q79:** Why was the query cache removed in MySQL 8.0?

<details><summary>Answer</summary>

The query cache caused scalability collapse under write-heavy workloads — the cache invalidation lock (`query_cache_lock`) became a global mutex contention point. Removed in 8.0.

Use application-level caching (Redis, Memcached) instead.

**Postgres comparison:** Postgres never had a query cache. Shared buffers serve as page cache.

</details>

**Q80:** What is "index dive" vs "index statistics"?

<details><summary>Answer</summary>

The optimizer estimates row counts:
- **Index dive** (default): quick lookup into the index for range boundaries. Accurate.
- **Index statistics**: uses `SHOW INDEX` cardinality estimates. Faster but can be inaccurate.

```sql
-- If IN list > eq_range_index_dive_limit (default 200), switches to statistics
SELECT * FROM t WHERE id IN (1, 2, 3, ... 20000);
```

**Trap:** Large IN lists (200+) may cause bad plan choices. Break into batches.

</details>

**Q81:** What is Batched Key Access (BKA) join?

<details><summary>Answer</summary>

BKA uses MRR for JOIN operations. Collects key values from the outer table in batches, sorts them, then reads rows from the inner table in sorted order. Controlled by `optimizer_switch='batched_key_access=on'`.

</details>

**Q82:** What is "derived table merge" vs "materialization"?

<details><summary>Answer</summary>

For subqueries in FROM:
- **Merge**: the derived table is merged into the outer query (no temp table). Faster.
- **Materialize**: the derived table is executed and stored as a temp table.

In 5.7+, `derived_merge` is on by default. `LIMIT` in subquery prevents merge.

</details>

**Q83:** What is semijoin and materialization for IN subqueries?

<details><summary>Answer</summary>

MySQL optimizes `IN (subquery)` using:
- **Semijoin (5.6+)**: converts to join equivalent (DuplicateWeedout, FirstMatch).
- **Subquery materialization**: executes subquery once, stores results in temp table.

**Trap:** `IN (SELECT ...)` with correlated subquery can be extremely slow. EXPLAIN shows `DEPENDENT SUBQUERY` — rewrite as JOIN.

</details>

**Q84:** What is the Skip Scan optimization (MySQL 8.0.13+)?

<details><summary>Answer</summary>

Skip Scan allows a composite index to be used even when the leftmost column is skipped, provided it has low cardinality.

```sql
-- Index: (gender, last_name)
SELECT * FROM users WHERE last_name = 'Smith';
-- Extra: "Using index for skip scan"
-- MySQL scans distinct gender values, then does range on last_name
```

**Trap:** Only beneficial when leftmost column has few distinct values (e.g., gender, status).

</details>

**Q85:** How does `LATERAL` derived table work (MySQL 8.0.14+)?

<details><summary>Answer</summary>

`LATERAL` allows a subquery in FROM to reference columns from preceding tables:
```sql
SELECT u.id, u.name, latest_order.*
FROM users u,
LATERAL (
    SELECT * FROM orders WHERE user_id = u.id
    ORDER BY created_at DESC LIMIT 1
) AS latest_order;
```

Powerful for "top N per group" queries.

</details>

**Q86:** What are windowing functions in MySQL 8.0?

<details><summary>Answer</summary>

MySQL 8.0 added ANSI SQL window functions:
```sql
SELECT name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
FROM employees;

SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) AS running_total
FROM transactions;
```

**Postgres comparison:** Postgres has had window functions since 8.4 (2009).

</details>

**Q87:** What is a CTE (Common Table Expression) in MySQL 8.0?

<details><summary>Answer</summary>

CTEs allow defining a named temporary result:
```sql
WITH active_users AS (
    SELECT id, name FROM users WHERE status = 'active'
)
SELECT a.*, COUNT(o.id) AS order_count
FROM active_users a LEFT JOIN orders o ON o.user_id = a.id
GROUP BY a.id;
```

Recursive CTE for tree traversal:
```sql
WITH RECURSIVE org_tree AS (
    SELECT id, name, manager_id, 1 AS depth FROM employees
    WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, ot.depth + 1
    FROM employees e JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT * FROM org_tree;
```

</details>

**Q88:** What is a FULLTEXT index?

<details><summary>Answer</summary>

```sql
CREATE FULLTEXT INDEX idx_fulltext ON articles(title, body);
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('mysql database' IN NATURAL LANGUAGE MODE);
```

Search modes: `IN NATURAL LANGUAGE MODE`, `IN BOOLEAN MODE`, `WITH QUERY EXPANSION`.

Limitations: minimum word length (`innodb_ft_min_token_size`), stopwords, no prefix search without wildcard.

**Postgres comparison:** Postgres has `tsvector`/`tsquery` — more powerful. MySQL Fulltext is simpler.

</details>

**Q89:** What is a spatial index (R-Tree) in MySQL?

<details><summary>Answer</summary>

Spatial indexes use **R-Tree** for geometry columns:
```sql
CREATE TABLE locations (
    id INT PRIMARY KEY,
    coords POINT NOT NULL SRID 4326,
    SPATIAL INDEX idx_coords(coords)
);
```

**Postgres comparison:** Postgres has PostGIS — far more advanced. MySQL spatial is basic.

</details>

**Q90:** How do you validate that a query is using an index efficiently?

<details><summary>Answer</summary>

Checklist:
1. **EXPLAIN**: `type` is not `ALL` or `index` for large tables
2. **EXPLAIN `key`**: confirms the optimizer chose your intended index
3. **EXPLAIN `rows`**: should be close to actual matching rows
4. **EXPLAIN `Extra`**: `Using index` (good), `Using filesort` (maybe bad), `Using temporary` (bad)
5. **EXPLAIN ANALYZE**: check actual timing vs estimated
6. **Handler status**: `SHOW SESSION STATUS LIKE 'Handler%'` before and after

---

## 4. Rapid-Fire: Transactions, Locking & Deadlocks (25 questions)

**Q91:** What are the four isolation levels in MySQL? What is the default?

<details><summary>Answer</summary>

1. **READ UNCOMMITTED**: dirty reads possible
2. **READ COMMITTED**: no dirty reads, non-repeatable reads possible
3. **REPEATABLE READ** (default in InnoDB): no dirty/non-repeatable reads. Phantoms prevented by next-key locking.
4. **SERIALIZABLE**: all reads are implicit `SELECT ... FOR SHARE`

**Postgres comparison:** PostgreSQL's default is READ COMMITTED. Postgres REPEATABLE READ uses SSI (may return serialization failures requiring retry).

</details>

**Q92:** Explain MVCC in InnoDB. How does it implement snapshot isolation?

<details><summary>Answer</summary>

1. Each transaction gets a transaction ID (trx_id)
2. Each row stores `DB_TRX_ID` (last modifier) and `DB_ROLL_PTR` (undo log pointer)
3. At first read (REPEATABLE READ) or each statement (READ COMMITTED), InnoDB creates a **read view** — snapshot of active transaction IDs
4. When reading, InnoDB checks `DB_TRX_ID` against the read view:
   - If committed before snapshot → visible
   - If active at snapshot time → read from undo log
   - If started after snapshot → read from undo log

This creates **snapshot isolation**: readers never block writers, writers never block readers.

</details>

**Q93:** What are record locks, gap locks, and next-key locks?

<details><summary>Answer</summary>

- **Record lock**: locks a single index record
- **Gap lock**: locks the gap between index records
- **Next-key lock**: record lock + gap lock on the gap before the record (InnoDB's default at REPEATABLE READ)

```sql
-- Next-key lock on (10, 20] for id < 20 at REPEATABLE READ:
SELECT * FROM t WHERE id < 20 FOR UPDATE;
-- Blocks INSERT 5, 15, 25 (anything in the locked range)
```

**Trap:** `SELECT ... FOR UPDATE` on a non-indexed column locks **all rows** (full scan) plus all gaps!

</details>

**Q94:** What are intention locks (IS, IX)?

<details><summary>Answer</summary>

Table-level locks indicating intent to acquire row-level locks:
- **Intention Shared (IS)**: intends to set S locks on rows
- **Intention Exclusive (IX)**: intends to set X locks on rows

InnoDB always acquires IX on the table before row-level X locks.

**Trap:** IX locks can block DDL operations (which need X table lock).

</details>

**Q95:** What is a "deadlock"? How does InnoDB detect and handle them?

<details><summary>Answer</summary>

Deadlock: two or more transactions holding locks the other needs, creating a cycle. InnoDB uses a **wait-for graph** to detect deadlocks.

When detected, InnoDB rolls back the transaction that modified the fewest rows:
```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

**Trap:** For high-concurrency workloads (>1000 connections), deadlock detection can become expensive. Set `innodb_deadlock_detect = OFF` and rely on `innodb_lock_wait_timeout`.

</details>

**Q96:** What causes most deadlocks in InnoDB? How do you prevent them?

<details><summary>Answer</summary>

Common causes:
1. **Different lock order**: Tx1 locks A→B; Tx2 locks B→A
2. **Gap lock conflicts**: concurrent inserts in same index range
3. **AUTO-INC lock interactions**
4. **Foreign key enforcement**: cascading updates create hidden locks

Prevention:
1. **Consistent lock order**: always access tables/rows in same order
2. **Short transactions**: minimize lock hold time
3. **`SELECT ... FOR UPDATE NOWAIT`** (8.0+): fail fast
4. **`SELECT ... FOR UPDATE SKIP LOCKED`** (8.0+): skip locked rows
5. **Lower isolation level**: READ COMMITTED disables gap locks

</details>

**Q97:** What is `SELECT ... FOR UPDATE NOWAIT` vs `SKIP LOCKED`?

<details><summary>Answer</summary>

- **`NOWAIT`**: if any target row is locked, immediately returns error (no wait)
- **`SKIP LOCKED`**: skips locked rows, returns only unlocked rows

```sql
-- Fail fast if row is locked
SELECT * FROM inventory WHERE id = 1 FOR UPDATE NOWAIT;

-- Consumer queue pattern
SELECT * FROM job_queue ORDER BY priority LIMIT 1 FOR UPDATE SKIP LOCKED;
```

**Trap:** `SKIP LOCKED` returns inconsistent results (locked rows omitted).

</details>

**Q98:** What is `innodb_lock_wait_timeout`?

<details><summary>Answer</summary>

Maximum time a transaction waits for a row lock (default 50s). When exceeded:
```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

Recommended: OLTP 5-10s, batch 30-60s, reporting 10-30s.

**Trap:** This is row lock wait timeout only. Does NOT roll back the transaction — you must explicitly ROLLBACK or COMMIT.

</details>

**Q99:** How do you monitor locks and lock waits in MySQL 8.0?

<details><summary>Answer</summary>

```sql
SELECT * FROM performance_schema.data_locks\G
SELECT * FROM performance_schema.data_lock_waits\G

-- With human-readable info
SELECT r.trx_mysql_thread_id AS blocking_thread,
       b.trx_mysql_thread_id AS waiting_thread
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx r ON r.trx_id = w.BLOCKING_ENGINE_TRANSACTION_ID
JOIN information_schema.innodb_trx b ON b.trx_id = w.REQUESTING_ENGINE_TRANSACTION_ID;
```

Also: `SHOW ENGINE INNODB STATUS\G` shows latest deadlock.

</details>

**Q100:** What is the "phantom read" problem? How does InnoDB handle it?

<details><summary>Answer</summary>

Phantom read: re-executing a query sees new rows inserted by another transaction. At REPEATABLE READ, InnoDB prevents phantoms using **next-key locking** for locking reads. For non-locking reads, MVCC prevents phantoms (consistent read sees snapshot).

**Postgres comparison:** Postgres prevents phantoms using SSI without gap locks — allows more concurrent inserts but may return serialization failures.

</details>

**Q101:** What is a "lost update"? How do you prevent it?

<details><summary>Answer</summary>

Lost update: two transactions read the same value, then both write based on stale read. Prevention:

1. **`SELECT ... FOR UPDATE`** (pessimistic):
```sql
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
```

2. **Atomic UPDATE** (preferred):
```sql
UPDATE accounts SET balance = balance - 10 WHERE id = 1 AND balance >= 10;
```

3. **Optimistic locking** (version column):
```sql
UPDATE accounts SET balance = balance - 10, version = version + 1
WHERE id = 1 AND version = :read_version;
```

</details>

**Q102:** What is "read view" and "snapshot" in InnoDB MVCC?

<details><summary>Answer</summary>

A read view contains:
- `up_limit_id`: smallest active transaction ID at snapshot creation
- `low_limit_id`: largest transaction ID + 1
- `ids`: array of active transaction IDs

When reading, InnoDB checks `DB_TRX_ID`:
- `trx_id < up_limit_id`: visible
- `trx_id >= low_limit_id`: not visible
- `trx_id` in `ids`: not visible (read undo)
- Otherwise: visible

</details>

**Q103:** What happens when you run `UPDATE` without `WHERE` in a transaction?

<details><summary>Answer</summary>

At REPEATABLE READ:
1. Scans the clustered index (full table scan)
2. Acquires next-key locks on **every row** and all gaps
3. The entire table is effectively locked

At READ COMMITTED: gap locks disabled, but all rows are still locked.

**Trap:** Running UPDATE without WHERE in production can lock the table for minutes. MySQL doesn't have `statement_timeout` (Postgres does).

</details>

**Q104:** What is the difference between `FOR UPDATE` on an indexed vs non-indexed column?

<details><summary>Answer</summary>

- **Indexed column**: locks only matching rows + gaps
- **Non-indexed column**: locks ALL rows visited during full scan (the entire table) + all gaps

Always ensure columns used in `FOR UPDATE` WHERE clauses are indexed.

</details>

**Q105:** How do you detect which transaction is blocking others?

<details><summary>Answer</summary>

```sql
-- MySQL 8.0:
SELECT t1.PROCESSLIST_ID AS blocked_thread,
       t2.PROCESSLIST_ID AS blocking_thread
FROM performance_schema.threads t1
JOIN performance_schema.threads t2
JOIN performance_schema.data_lock_waits w
ON t1.THREAD_ID = w.REQUESTING_THREAD_ID
AND t2.THREAD_ID = w.BLOCKING_THREAD_ID;

-- Check for idle transactions holding locks:
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX WHERE trx_state = 'RUNNING';
```

</details>

**Q106:** What is the cost of transaction rollback in InnoDB?

<details><summary>Answer</summary>

Rolling back a large transaction is expensive because:
1. Must read undo log records and reverse each change
2. Undo application generates redo log
3. Other transactions may be waiting for locks
4. Rollback of millions of rows can take hours

**Best practice:** Batch large DMLs into manageable chunks (10,000-100,000 rows per commit).

</details>

**Q107:** What is `max_execution_time`?

<details><summary>Answer</summary>

```sql
-- Set for a single SELECT:
SELECT /*+ MAX_EXECUTION_TIME(5000) */ * FROM users WHERE ...;

-- Set globally:
SET GLOBAL max_execution_time = 5000;
```

Only applies to SELECT. For DML timeouts, use `innodb_lock_wait_timeout`.

</details>

**Q108:** How does InnoDB handle "write skew"?

<details><summary>Answer</summary>

Write skew: two concurrent transactions read overlapping data sets, then each writes to disjoint rows, creating an inconsistency. Example:

```sql
-- Both doctors check if at least one is on call:
SELECT COUNT(*) FROM doctors WHERE on_call = 1;
-- Both see 2 (so both go off call):
UPDATE doctors SET on_call = 0 WHERE id = 1;  -- Tx A
UPDATE doctors SET on_call = 0 WHERE id = 2;  -- Tx B
-- Both commit → 0 doctors on call (invariant violated)
```

**Fix:** Use `SELECT ... FOR UPDATE` to lock all matching rows, preventing concurrent modifications.

</details>

**Q109:** What are `NOWAIT` and `SKIP LOCKED` useful for?

<details><summary>Answer</summary>

- `NOWAIT`: optimistic retry pattern — fail fast instead of waiting for `innodb_lock_wait_timeout`
- `SKIP LOCKED`: work queues — multiple consumers each grab different rows

These were major additions in MySQL 8.0, enabling high-concurrency queue patterns without deadlocks or lock waits.

</details>

**Q110:** What is `innodb_autoinc_lock_mode = 2` risk?

<details><summary>Answer</summary>

Mode 2 (default in 8.0) provides highest concurrency but:
1. AUTO_INCREMENT values may have gaps (even without rollback)
2. With statement-based replication, IDs may differ on replica
3. Bulk inserts may interleave auto-inc values

**Always use ROW-based binlog when setting `innodb_autoinc_lock_mode = 2`.**

</details>

**Q111:** What are "phantom rows" and how do gap locks prevent them?

<details><summary>Answer</summary>

Phantoms: new rows inserted by another transaction that match your WHERE clause. Gap locks prevent inserts into the gaps between existing index records.

At READ COMMITTED (gap locks disabled), phantoms are possible even with `SELECT ... FOR UPDATE`. At REPEATABLE READ, gap locks prevent them for locking reads.

</details>

**Q112:** How do you safely retry a deadlock in application code?

<details><summary>Answer</summary>

```php
// Laravel example
$maxAttempts = 3;
$attempt = 0;

while ($attempt < $maxAttempts) {
    try {
        DB::transaction(function () use ($userId, $amount) {
            // your logic
        });
        break; // success
    } catch (\Illuminate\Database\QueryException $e) {
        if ($e->errorInfo[1] == 1213 && ++$attempt < $maxAttempts) {
            usleep(100000 * $attempt); // exponential backoff
            continue;
        }
        throw $e;
    }
}
```

**Trap:** Retrying too aggressively can make the problem worse. Always use backoff.

</details>

**Q113:** What is `innodb_deadlock_detect` and when would you disable it?

<details><summary>Answer</summary>

`innodb_deadlock_detect = ON` (default). InnoDB maintains a wait-for graph. At very high concurrency (>1000 connections), the deadlock detection overhead can become significant.

If deadlocks are extremely rare and you're willing to let transactions timeout instead, you can disable it: `innodb_deadlock_detect = OFF`. Rely on `innodb_lock_wait_timeout` instead.

**Trap:** With detection off, undetected deadlocks only surface via timeout (ERROR 1205). This can cause silent transaction accumulation.

</details>

**Q114:** How does `SELECT ... FOR UPDATE` interact with foreign keys?

<details><summary>Answer</summary>

When you lock a parent table row with `FOR UPDATE`, InnoDB may also acquire locks on child table rows (via the FK constraint). This is called **foreign key locking**: if another transaction tries to insert/update child rows referencing the locked parent, it will wait.

This can cause unexpected blocking and deadlocks. Be aware of FK-induced lock propagation.

</details>

**Q115:** What is the "next-key locking effect" and why does it matter for range queries?

<details><summary>Answer</summary>

Next-key locking applies not just to the rows that match the WHERE clause, but also to the gaps before them. A simple `SELECT * FROM t WHERE id > 10 FOR UPDATE` at REPEATABLE READ locks:
- All rows with id > 10
- The gap (10, ∞) — preventing inserts of id > 10 until the transaction commits

**This can block inserts that you didn't intend to block.** Use READ COMMITTED if you don't need gap locks.

---

## 5. Rapid-Fire: Replication (20 questions)

**Q116:** What are the three binary log formats?

<details><summary>Answer</summary>

- **STATEMENT (SBR)**: logs SQL statements. Compact but non-deterministic queries (NOW(), UUID()) cause replica divergence.
- **ROW (RBR)**: logs actual row changes. More space but deterministic. Required for GTID, Group Replication.
- **MIXED**: STATEMENT by default, switches to ROW for non-deterministic statements.

**Recommendation:** Use ROW (default in 8.0). Safest option.

**Trap:** Switching from STATEMENT to ROW can increase binlog size 5-10x. Validate disk space.

</details>

**Q117:** What is semi-synchronous replication?

<details><summary>Answer</summary>

Master waits for **at least one** replica to acknowledge receipt (write to relay log) before acknowledging commit to client.

Trade-off: reduces throughput (adds network RTT to commit latency). Falls back to async if no replica acknowledges within the timeout.

**Trap:** Only receipt is acknowledged, not apply. The replica may not have applied the transaction yet.

</details>

**Q118:** What is GTID? How does it simplify replication?

<details><summary>Answer</summary>

GTID = `source_id:transaction_id`. Benefits:
- **Auto-positioning**: no need for `MASTER_LOG_FILE`/`MASTER_LOG_POS`
- **Failover**: promote without hunting for binlog positions
- **Consistency**: each transaction executed exactly once

```sql
gtid_mode = ON
enforce_gtid_consistency = ON

CHANGE MASTER TO MASTER_HOST='...', MASTER_AUTO_POSITION = 1;
```

**Trap:** `gtid_mode` can only be changed in specific steps (OFF → OFF_PERMISSIVE → ON_PERMISSIVE → ON).

</details>

**Q119:** What are the two slave threads?

<details><summary>Answer</summary>

1. **I/O thread**: connects to master, reads binary log, writes to relay log. `Slave_IO_Running: Yes`
2. **SQL thread**: reads relay log, applies transactions. `Slave_SQL_Running: Yes`

```sql
SHOW SLAVE STATUS\G
-- Shows: Seconds_Behind_Master, Last_SQL_Error, etc.
```

If SQL thread is `No`, check `Last_SQL_Error` for the specific error.

</details>

**Q120:** What is parallel replication?

<details><summary>Answer</summary>

MySQL 8.0 can apply transactions in parallel using:
- `slave_parallel_workers` (number of applier threads)
- `slave_parallel_type`: `LOGICAL_CLOCK` (recommended)

The master groups transactions committed within the same window under the same `last_committed` value. Transactions with the same `last_committed` can be applied in parallel.

**Trap:** `LOGICAL_CLOCK` only works with ROW-based replication.

</details>

**Q121:** What causes replication lag? How do you mitigate it?

<details><summary>Answer</summary>

Causes:
1. Long-running writes on master
2. Replica under-provisioned (less CPU/RAM)
3. Single-threaded SQL thread
4. Lock contention on replica
5. DDL on replica blocking apply

Mitigation:
1. Enable parallel replication
2. Use semi-sync replication
3. Shard writes across multiple masters
4. Use ProxySQL with lag-aware read/write splitting

**Trap:** `Seconds_Behind_Master` can be misleading — calculated from timestamps and can be NULL or inaccurate.

</details>

**Q122:** How do you perform a master failover?

<details><summary>Answer</summary>

Planned switchover:
1. Stop writes on master (`SET GLOBAL read_only = ON`)
2. Wait for replicas to catch up
3. Promote replica: `STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only = OFF`
4. Point other replicas to new master

Unplanned failover:
1. Find replica with most advanced GTID
2. Promote it
3. Use Orchestrator or MHA to automate

**Trap:** Without semi-sync, transactions on dead master may not be on any replica. For zero RPO, use Group Replication.

</details>

**Q123:** What is the difference between `CHANGE MASTER TO` and `CHANGE REPLICATION SOURCE TO`?

<details><summary>Answer</summary>

MySQL 8.0.23+ introduced gender-neutral terminology:
- Old: `CHANGE MASTER TO`, `SHOW SLAVE STATUS`
- New: `CHANGE REPLICATION SOURCE TO`, `SHOW REPLICA STATUS`

Both work. New syntax preferred for future compatibility.

</details>

**Q124:** How do you skip a replication error?

<details><summary>Answer</summary>

```sql
-- Skip the next transaction (GTID):
SET GTID_NEXT = '<gtid_of_failing_tx>';
BEGIN; COMMIT;
SET GTID_NEXT = 'AUTOMATIC';

-- Skip N transactions (pre-GTID):
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = N;
```

**Trap:** Skipping an error accepts data divergence. Use only as temporary fix. After skipping, verify with `pt-table-checksum`.

</details>

**Q125:** What is `binlog_do_db` and `binlog_ignore_db`?

<details><summary>Answer</summary>

Controls which databases are logged to the binary log. **Pitfalls:**
1. Tests against the **current default database**, not the table's database
2. Cross-database queries can cause partial logging
3. **Do not use** — filter on the replica side instead

</details>

**Q126:** What is `expire_logs_days` / `binlog_expire_logs_seconds`?

<details><summary>Answer</summary>

Controls automatic purging of binary logs:
- `expire_logs_days` (deprecated in 8.0)
- `binlog_expire_logs_seconds` (8.0+): default 2592000 (30 days)

**Trap:** Binlog retention must cover your backup schedule. Also consider replica lag.

</details>

**Q127:** How do you monitor replication health?

<details><summary>Answer</summary>

Essential checks:
```sql
SHOW REPLICA STATUS\G
-- Must check: Replica_IO_Running: Yes, Replica_SQL_Running: Yes
-- Seconds_Behind_Master: low and stable

SELECT @@gtid_executed, @@gtid_purged;
SELECT * FROM sys.replication_status;
```

Monitoring checklist: I/O thread connected, SQL thread not stopped, error count = 0, lag not trending up.

</details>

**Q128:** What is `sync_binlog`?

<details><summary>Answer</summary>

Controls how often MySQL flushes the binary log to disk:
- `sync_binlog = 0`: OS controls — best performance, up to 1s of transaction loss on crash
- `sync_binlog = 1`: flush on every commit — safest, highest latency
- `sync_binlog = N`: flush every N commits

For safest settings: `sync_binlog = 1` AND `innodb_flush_log_at_trx_commit = 1` (double sync).

</details>

**Q129:** What is `binlog_row_image`?

<details><summary>Answer</summary>

Controls how much row data is written for ROW-based binlog:
- `full` (default): all columns. Maximum size.
- `minimal`: only PK and changed columns. Much smaller.
- `noblob`: like full but skips unchanged BLOB/TEXT.

**Recommendation:** Use `minimal` for production.

</details>

**Q130:** What is `binlog_checksum`?

<details><summary>Answer</summary>

`binlog_checksum = CRC32` (default in 8.0) adds a checksum to each binlog event. The replica validates checksums — detects corruption in transit or on disk.

**Trap:** On upgrade from 5.6 to 8.0, set `slave_sql_verify_checksum = OFF` temporarily.

</details>

**Q131:** What is delayed replication (`MASTER_DELAY`)?

<details><summary>Answer</summary>

Sets a time delay before the replica applies transactions:
```sql
CHANGE MASTER TO MASTER_DELAY = 3600;  -- 1 hour delay
```

Use for disaster recovery — if someone accidentally drops a table, you have 1 hour to stop replication before the DROP reaches the delayed replica.

</details>

**Q132:** What is `binlog_group_commit_sync_delay`?

<details><summary>Answer</summary>

Delays binary log flush by microseconds to allow more transactions to group commit:
```ini
binlog_group_commit_sync_delay = 100000  -- 100ms
binlog_group_commit_sync_no_delay_count = 10  -- flush after 10 transactions
```

Increases throughput by reducing fsyncs but adds latency per transaction.

</details>

**Q133:** What is the difference between `replicate_do_db` and `replicate_wild_do_table`?

<details><summary>Answer</summary>

- `replicate_do_db = sales`: only replicate when the **current default database** is `sales`
- `replicate_wild_do_table = sales.%`: replicate any table matching the pattern

`replicate_wild_*` is safer and more predictable.

</details>

**Q134:** How do you set up a replica from a backup?

<details><summary>Answer</summary>

1. Take backup with XtraBackup (includes binlog position):
```bash
xtrabackup --backup --slave-info --target-dir=/backup
xtrabackup --prepare --target-dir=/backup
```

2. Restore on replica:
```bash
xtrabackup --copy-back --target-dir=/backup
```

3. Start replica:
```sql
CHANGE REPLICATION SOURCE TO SOURCE_HOST='master', SOURCE_AUTO_POSITION=1;
START REPLICA;
```

</details>

**Q135:** How does `START REPLICA UNTIL` work?

<details><summary>Answer</summary>

Replicates up to a specific point, then stops:
```sql
START REPLICA UNTIL SOURCE_LOG_FILE = 'binlog.000042', SOURCE_LOG_POS = 12345;
-- Or with GTID:
START REPLICA UNTIL SQL_AFTER_GTIDS = '3E11FA47-71CA-11E1-9E33-C80AA9429562:23';
```

Used for PITR or testing replication up to a specific point.

---

## 6. Rapid-Fire: HA, Sharding & Operations (25 questions)

**Q136:** What is MySQL InnoDB Cluster?

<details><summary>Answer</summary>

Complete HA solution built on:
1. **Group Replication**: multi-master or single-primary with Paxos consensus
2. **MySQL Router**: automatic read/write splitting, failover routing
3. **MySQL Shell**: adminAPI for cluster management

Provides automatic failover (<10s), strong consistency, auto-provisioning.

**Trap:** Requires fast network (<1ms RTT), all tables must have PK, all nodes 8.0+.

**Postgres comparison:** Postgres has Patroni + etcd/Consul. MySQL's bundling is simpler to deploy.

</details>

**Q137:** What is Group Replication's certification phase?

<details><summary>Answer</summary>

Each transaction goes through:
1. Transaction executed locally, written to binlog
2. **Certification**: write set broadcast to group. All members vote to accept/reject
3. If accepted → commit on all members. If rejected → rollback

**Trap:** Write conflict rate increases with square of writing nodes. Use single-primary for write-heavy workloads.

</details>

**Q138:** What is Galera Cluster?

<details><summary>Answer</summary>

Synchronous multi-master cluster (Percona XtraDB Cluster / MariaDB Galera Cluster):
- All nodes writable
- Transactions committed on all nodes or none (quorum-based)
- Uses wsrep API with certification-based replication

**Trap:** Galera has "flow control" — if a node falls behind, it throttles the entire cluster. Nodes must be homogeneous.

</details>

**Q139:** What is ProxySQL?

<details><summary>Answer</summary>

High-performance MySQL proxy for:
- Read/write splitting
- Query routing (by user, schema, digest)
- Connection pooling
- Query caching
- Query rewriting
- Automatic failover

**Postgres comparison:** Postgres has PgBouncer (pooling), Pgpool-II (routing+pooling). ProxySQL is more feature-rich.

</details>

**Q140:** What is Vitess? When would you use it?

<details><summary>Answer</summary>

Vitess is a database clustering system for horizontal sharding:
- Auto-sharding with VSchema
- Query routing to the correct shard
- Connection pooling
- Online schema migration (gh-ost integration)

Use when you need horizontal sharding at scale (100+ nodes, TB+ data). Use ProxySQL for simpler routing/HA.

**Trap:** Vitess has significant operational complexity: etcd/Consul, vtgate, vtctld. Not for small deployments.

</details>

**Q141:** What is partitioning in MySQL?

<details><summary>Answer</summary>

Divides a table into physical sub-tables:
```sql
CREATE TABLE orders (
    id INT NOT NULL,
    created_at DATE NOT NULL
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

Types: RANGE, LIST, HASH, KEY, RANGE COLUMNS, LIST COLUMNS.

**Trap:** Partitioning does NOT automatically improve performance. Queries without partition key filter scan ALL partitions.

</details>

**Q142:** What is `pt-online-schema-change`?

<details><summary>Answer</summary>

Percona Toolkit's online schema change:
1. Creates shadow table with new schema
2. Creates triggers to capture DML changes
3. Copies data in chunks
4. `RENAME TABLE original TO old, shadow TO original`
5. Drops old table and triggers

**Limitations:** Trigger overhead during migration, requires PK, FKs complicate rename.

</details>

**Q143:** What is `gh-ost`? How is it better than pt-osc?

<details><summary>Answer</summary>

GitHub's triggerless online schema migration:
- Uses binary log stream instead of triggers
- Can read binlog from replica (reduces master load)
- More configurable throttling (replication lag, CPU)
- Pausable, testable (dry-run)

**Requirements:** ROW-based replication, PK on table, sufficient binlog retention.

</details>

**Q144:** What is the difference between mysqldump and XtraBackup?

<details><summary>Answer</summary>

| Aspect | mysqldump | XtraBackup |
|--------|-----------|------------|
| Type | Logical (SQL) | Physical (file copy) |
| Speed | Slow for large DB | Fast |
| Restore | Slow (re-execute SQL) | Fast (copy data + redo) |
| Locking | MVCC (`--single-transaction`) | Minimal (page copy) |
| PITR | Yes (with binlog pos) | Yes (with binlog pos) |

**Use mysqldump for:** small DBs (<5GB), schema-only, selective tables.

**Use XtraBackup for:** large DBs (100GB+), fast recovery, PITR, replica cloning.

</details>

**Q145:** What is Point-in-Time Recovery (PITR)?

<details><summary>Answer</summary>

Restore to an arbitrary point in time:
1. Restore full backup
2. Apply binary logs up to target time

```bash
mysqlbinlog --stop-datetime="2026-07-26 14:30:00" /var/log/mysql/binlog.* | mysql -u root
```

**Requirements:** binlogs enabled and retained, full backup at known binlog position.

</details>

**Q146:** How do you size the InnoDB buffer pool?

<details><summary>Answer</summary>

General rule: 60-80% of available RAM (after OS and other processes).

```ini
innodb_buffer_pool_size = 64G
innodb_buffer_pool_instances = 8  -- 1 instance per 8GB typically
```

Monitor:
- `Innodb_buffer_pool_read_requests` vs `Innodb_buffer_pool_reads` (hit rate > 99%)
- `SHOW ENGINE INNODB STATUS` → Buffer pool hit rate

</details>

**Q147:** How do you size the InnoDB redo log?

<details><summary>Answer</summary>

In 8.0.30+: `innodb_redo_log_capacity` (default 100MB). In older: `innodb_log_file_size` × `innodb_log_files_in_group`.

Guideline: set to 1-4x your buffer pool size for write-heavy workloads. Monitor with:
- `Log sequence number - Last checkpoint at` should not exceed 25% of redo log capacity

**Trap:** Too small → frequent flushes, write stalls. Too large → long crash recovery time.

</details>

**Q148:** What MySQL 8.0 features are most impactful for a senior engineer?

<details><summary>Answer</summary>

- **Atomic DDL**: DDL operations are crash-safe
- **Data dictionary**: transactional, in InnoDB (no .frm files)
- **JSON improvements**: binary JSON, partial updates, multi-value indexes
- **Windowing functions / CTEs** (8.0)
- **Hash joins** (8.0.18+)
- **Descending indexes** (8.0)
- **Invisible indexes** (8.0)
- **`EXPLAIN ANALYZE`** (8.0.18+)
- **`SKIP LOCKED` / `NOWAIT`** (8.0)
- **Instant ADD COLUMN** (8.0.12+)
- **Clone plugin** for provisioning replicas (8.0.17+)
- **Resource groups** for CPU/thread management
- **`SET_VAR` optimizer hint**
- **Roles** (RBAC)

</details>

**Q149:** What is the Clone Plugin?

<details><summary>Answer</summary>

MySQL 8.0.17+ Clone Plugin provisions replicas by cloning data from an existing MySQL instance:
```sql
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
CLONE INSTANCE FROM root@master_host:3306 IDENTIFIED BY 'password';
```

Faster than XtraBackup for provisioning because it's integrated. Supports both local and remote cloning.

</details>

**Q150:** What is MySQL Router?

<details><summary>Answer</summary>

MySQL Router routes client connections in InnoDB Cluster:
- Port 6446 for writes, 6447 for reads
- Automatic failover (detects cluster topology changes)
- Bootstrap: `mysqlrouter --bootstrap root@localhost:3306`

Sits between application and database (like ProxySQL but integrated with InnoDB Cluster).

</details>

**Q151:** What is Orchestrator?

<details><summary>Answer</summary>

Orchestrator is a replication topology manager:
- **Auto-discovery**: finds replication topologies
- **Failover**: detects master failure, promotes best replica
- **Recovery hooks**: runs scripts on failover
- **Web UI**: visual topology display
- Supports Consul, Zookeeper for consensus

</details>

**Q152:** How do you back up a large MySQL database efficiently?

<details><summary>Answer</summary>

For databases > 50GB:
```bash
# XtraBackup to S3
xtrabackup --backup --compress --stream=xbstream | mc pipe s3/bucket/

# Prepare
xtrabackup --prepare --apply-log-only --target-dir=/backup
```

For smaller DBs:
```bash
mysqldump --single-transaction --all-databases | gzip > backup.sql.gz
```

Best practice: combine full weekly backups with daily binlog backups for PITR.

</details>

**Q153:** What is `innodb_dedicated_server`?

<details><summary>Answer</summary>

MySQL 8.0 auto-tunes InnoDB parameters based on detected server memory:
```ini
innodb_dedicated_server = ON
```

Auto-sets: buffer pool size (1-4GB → 50%, 4GB+ → 70%), redo log size, flush method, buffer pool instances.

**Trap:** Only use on dedicated DB servers. On shared servers, it may over-allocate memory.

</details>

**Q154:** How do you handle slow queries in production?

<details><summary>Answer</summary>

1. **Enable slow query log** with low threshold:
```ini
long_query_time = 0.5
log_queries_not_using_indexes = 1
```

2. **Analyze with pt-query-digest** to identify top offenders

3. **For each slow query:**
   - Run `EXPLAIN ANALYZE`
   - Check if index exists and is being used
   - Look for filesort, temp tables, full scans
   - Consider rewriting (remove functions on indexed columns, add covering indexes)

4. **Short-term fix**: query hint (`MAX_EXECUTION_TIME`, `FORCE INDEX`)

5. **Long-term fix**: schema/index redesign, caching (Redis), query rewrite

</details>

**Q155:** What is the MySQL Resource Group feature?

<details><summary>Answer</summary>

MySQL 8.0 resource groups bind threads to specific CPU cores:
```sql
CREATE RESOURCE GROUP reporting_group
TYPE = USER
VCPU = 2-3
THREAD_PRIORITY = 5;

SET RESOURCE GROUP reporting_group FOR thread_id;
```

Use to isolate reporting queries from OLTP traffic at the OS level.

</details>

**Q156:** How do you handle MySQL security?

<details><summary>Answer</summary>

Key practices:
1. **Least privilege**: grant minimal required privileges
2. `mysql_secure_installation`: remove anonymous users, disable remote root
3. **Encrypted connections**: `require_secure_transport = ON`
4. **Password rotation**: `default_password_lifetime = 90`
5. **TDE**: `ENCRYPTION='Y'` for sensitive tablespaces
6. **Audit logging**: `audit_log` plugin

**Trap:** Never use `GRANT ALL ON *.* TO app_user`. Always scope to specific databases.

</details>

**Q157:** What is `innodb_print_all_deadlocks`?

<details><summary>Answer</summary>

When enabled, all deadlocks are logged (not just the latest):
```ini
innodb_print_all_deadlocks = ON
```

Each deadlock is written to the MySQL error log. Invaluable for debugging frequent deadlock issues.

**Trap:** On systems with many deadlocks, the error log can grow quickly.

</details>

**Q158:** How do you monitor MySQL in production?

<details><summary>Answer</summary>

Key metrics:
- **Availability**: `Uptime`, `Threads_connected`, connection errors
- **Performance**: QPS, TPS, buffer pool hit rate, redo log checkpoint age
- **Replication**: Seconds_Behind_Master, relay log space, IO/SQL thread status
- **Disk**: binlog growth rate, data size, pending writes

Tools: Prometheus + mysqld_exporter, Percona Monitoring & Management (PMM), Datadog, New Relic

</details>

**Q159:** What are MySQL optimizer hints?

<details><summary>Answer</summary>

```sql
-- Force index
SELECT * FROM t FORCE INDEX (idx_name) WHERE name = 'Alice';

-- Optimizer hints (8.0+):
SELECT /*+ BKA(t1) */ * FROM t1 JOIN t2 ON t1.id = t2.id;
SELECT /*+ SET_VAR(sort_buffer_size = 16M) */ * FROM t ORDER BY col;
SELECT /*+ MAX_EXECUTION_TIME(1000) */ * FROM t;
SELECT /*+ JOIN_ORDER(t1, t2, t3) */ * FROM t1 JOIN t2 JOIN t3;
```

Use hints sparingly — they mask underlying issues.

</details>

**Q160:** What is the MySQL 8.0 clone plugin's primary use case?

<details><summary>Answer</summary>

Provisioning new replicas faster than XtraBackup:
```sql
CLONE INSTANCE FROM user@host:port IDENTIFIED BY 'password';
```

Also supports local clone for backup:
```sql
CLONE LOCAL DATA DIRECTORY = '/backup/dir';
```

The process copies all data and redo log, applies redo, and the new instance is ready.

---

## 7. Code Puzzles

### Puzzle 1: Analyze EXPLAIN Output

Given this query and EXPLAIN output:

```sql
EXPLAIN SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id
ORDER BY order_count DESC
LIMIT 10;
```

EXPLAIN output:
```
id  select_type  table  type   key              rows  Extra
1   SIMPLE       u      range  idx_created_at   5000  Using where; Using index; Using temporary; Using filesort
1   SIMPLE       o      ref    idx_user_id      2     Using index
```

**Questions:**
1. What does `Using temporary; Using filesort` in the first row indicate?
2. How would you optimize this query?

<details><summary>Answer</summary>

1. `Using temporary; Using filesort`: MySQL needs a temp table to process the GROUP BY and a separate sort for the ORDER BY. This is because:
   - `GROUP BY u.id` must aggregate `o.id` (COUNT)
   - `ORDER BY order_count DESC` requires sorting the aggregated result
   - The index `idx_created_at` only covers the WHERE and range condition, not the GROUP BY

2. **Optimization:** Create a composite index on `users(created_at, id)`:
   ```sql
   CREATE INDEX idx_created_at_id ON users(created_at, id);
   ```
   This makes the index a covering index for the WHERE + GROUP BY on users. But the ORDER BY on an aggregate (`COUNT(o.id)`) can't use an index. Consider a summary table or application-level caching for the count.

**Follow-up:** For the `orders` table, `type: ref` with `rows: 2` means for each user, about 2 order rows are read. With 5000 users, that's ~10,000 rows examined — acceptable.

</details>

### Puzzle 2: Identify Lock Conflicts from SHOW ENGINE INNODB STATUS

Given this output:

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION 789456, ACTIVE 5 sec
LOCK WAIT 2 lock struct(s), 1 row lock(s)
MySQL thread id 42, query id 12345 updating
UPDATE inventory SET quantity = quantity - 1 WHERE id = 100 AND quantity > 0

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 10 page no 5 index PRIMARY lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
TRANSACTION 789457, ACTIVE 3 sec
3 lock struct(s), 2 row lock(s)
MySQL thread id 43, query id 12346 updating
UPDATE inventory SET quantity = quantity - 1 WHERE id = 200 AND quantity > 0

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 10 page no 5 index PRIMARY lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 10 page no 5 index PRIMARY lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (1)
```

**Questions:**
1. What is the deadlock? Which transactions are involved?
2. Why are these two updates deadlocking if they update different rows (id=100 and id=200)?

<details><summary>Answer</summary>

1. Both transactions are updating the `inventory` table. Transaction 1 (id=42) is updating id=100 and waiting for a lock. Transaction 2 (id=43) holds a lock and is also waiting. Transaction 1 was rolled back as the victim.

2. The key insight: **Both transactions hold and wait for locks on different rows on the same page**. Both rows (id=100 and id=200) are on **page no 5**. Transaction 2 has `3 lock struct(s), 2 row lock(s)` — this means the application is doing more than just the shown UPDATE:
   - There might be additional queries in each transaction (e.g., SELECT FOR UPDATE on other rows)
   - Or there's an FK constraint that locks additional rows

**Most likely cause:** The application does `SELECT ... FOR UPDATE` on additional rows or tables before the UPDATE, and the lock order differs between the two code paths.

</details>

### Puzzle 3: Determine ALTER TABLE Algorithm

For this table:
```sql
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    total DECIMAL(10,2) NOT NULL,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

Determine INSTANT/INPLACE/COPY for each:
a) `ALTER TABLE orders ADD COLUMN discount DECIMAL(5,2) DEFAULT 0.00;`
b) `ALTER TABLE orders DROP COLUMN notes;`
c) `ALTER TABLE orders ADD INDEX idx_created (created_at);`
d) `ALTER TABLE orders ADD COLUMN vip_status VARCHAR(10) AFTER status;`
e) `ALTER TABLE orders CHANGE COLUMN status status VARCHAR(50);`
f) `ALTER TABLE orders RENAME COLUMN notes TO comments;`

<details><summary>Answer</summary>

| Operation | Algorithm | Notes |
|-----------|-----------|-------|
| a) ADD COLUMN ... DEFAULT | **INSTANT** (8.0.12+) | Adding to end with DEFAULT |
| b) DROP COLUMN notes | **INPLACE** | Requires rebuild, allows concurrent DML |
| c) ADD INDEX | **INPLACE** | Secondary index creation online |
| d) ADD COLUMN AFTER status | **COPY** | Adding NOT at end requires COPY |
| e) CHANGE VARCHAR(20)→(50) | **INPLACE** | Increasing size within limits |
| f) RENAME COLUMN | **INSTANT** | Metadata-only |

**Key rules:**
- **INSTANT**: adding column to end, renaming, setting default
- **INPLACE**: adding/dropping indexes, adding/dropping columns (not repositioned)
- **COPY**: adding column not at end, changing PK, incompatible type changes

</details>

### Puzzle 4: Deadlock Prediction

**Session A:**
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
COMMIT;
```

**Session B:**
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;
```

**Questions:** Will these deadlock? Why? How to fix?

<details><summary>Answer</summary>

**Yes, classic deadlock:**
- Session A locks id=1, then tries id=2
- Session B locks id=2, then tries id=1
- Each waits for the other → deadlock detected

**Fix:** Always acquire locks in consistent order:
```sql
-- Both sessions lock by ascending ID:
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
```

**Laravel Eloquent fix:**
```php
$ids = [$from, $to];
sort($ids);
$accounts = Account::whereIn('id', $ids)->orderBy('id')->lockForUpdate()->get();
```

</details>

### Puzzle 5: Diagnose Replication Lag

Given this `SHOW REPLICA STATUS`:
```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Master: 8437
Read_Master_Log_Pos: 482349123
Exec_Master_Log_Pos: 312344512
Slave_SQL_Running_State: updating event
Retrieved_Gtid_Set: 3e11fa47-71ca-11e1-9e33-c80aa9429562:1-48231
Executed_Gtid_Set: 3e11fa47-71ca-11e1-9e33-c80aa9429562:1-42312
```

**Questions:**
1. What is the lag and how much data is backlogged?
2. What could be causing the SQL thread to be stuck?

<details><summary>Answer</summary>

1. **Lag:** 8437 seconds (~2.3 hours). Binlog position gap: 482349123 - 312344512 = ~170MB backlog. GTID gap: 48231 - 42312 = 5919 transactions behind.

2. **SQL thread state `updating event`** — it's currently executing a write. Possible causes:
   - A long-running UPDATE/DELETE on the replica (missing index)
   - Lock contention from reporting queries on the replica
   - The current binlog event is a large transaction (e.g., DELETE millions of rows)
   - Replica is under-provisioned (less CPU/memory than master)

**Next checks:**
- `SHOW FULL PROCESSLIST` on replica — what else is running?
- Is parallel replication enabled (`slave_parallel_workers`)?
- Are there long-running reporting queries blocking apply?

</details>

### Puzzle 6: SQL Query Optimization

Given this slow query on a 10M-row table:
```sql
SELECT u.id, u.name, o.total, o.created_at
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active'
  AND YEAR(o.created_at) = 2026
  AND o.total > 100
ORDER BY o.created_at DESC
LIMIT 50;
```

Current indexes: PK on both, `idx_user_status(users.status)`, `idx_order_user(orders.user_id)`, `idx_order_created(orders.created_at)`.

**Questions:**
1. What is the main performance issue?
2. How would you rewrite/optimize?

<details><summary>Answer</summary>

1. **Main issue:** `YEAR(o.created_at) = 2026` is a **function on the indexed column**. This prevents index usage on `created_at`. MySQL will apply YEAR() to every row, causing a full table/index scan.

2. **Rewrite to use range condition:**
```sql
SELECT u.id, u.name, o.total, o.created_at
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active'
  AND o.created_at >= '2026-01-01'
  AND o.created_at < '2027-01-01'
  AND o.total > 100
ORDER BY o.created_at DESC
LIMIT 50;
```

Now `idx_order_created` can be used for both the range condition and ORDER BY.

**Optimal indexes:**
```sql
CREATE INDEX idx_order_covering ON orders(created_at, total, user_id);
-- And ensure idx_user_status for the users filter
```

**Trap:** Never wrap indexed columns in functions in the WHERE clause.

</details>

### Puzzle 7: Backup Strategy Gap Analysis

A company's backup strategy:
- **Full backup**: XtraBackup every Sunday at 2am
- **Incremental**: XtraBackup nightly Mon-Sat at 2am
- **Binlog retention**: `binlog_expire_logs_seconds = 259200` (3 days)
- **Retention**: 30 days
- **Replication**: async, one replica (reporting)

**Questions:**
1. What are the gaps?
2. Maximum data loss in a disaster?
3. What improvements?

<details><summary>Answer</summary>

1. **Gaps:**
   - **Binlog retention (3 days) does NOT cover the backup interval**: Full backup is Sunday. By Friday, binlogs from before Tuesday have expired. If crash on Saturday, no binlogs for Mon-Tue → data loss.
   - **No off-site backup** — disaster at the datacenter destroys everything.
   - **No backup testing** — weekly full takes 4 hours; has it ever been restored?
   - **Single replica** — if master fails and replica has same corruption, total loss.

2. **Maximum data loss:** Up to **4 days** (crash on Saturday, last usable backup from Sunday, no binlogs for Mon-Thu).

3. **Improvements:**
   - Increase binlog retention to 8+ days: `SET GLOBAL binlog_expire_log_seconds = 691200;`
   - Add off-site backups (S3/cloud storage)
   - Test restores regularly (monthly)
   - Add a delayed replica (`MASTER_DELAY = 86400`)
   - Consider semi-sync replication

</details>

### Puzzle 8: Metadata Lock Analysis

Given `SHOW FULL PROCESSLIST`:
```
+-----+------+----------+------+---------------------------------+
| Id  | User | Command  | Time | State                           | Info
+-----+------+----------+------+---------------------------------+
| 42  | app  | Query    |  120 | Creating sort index             | SELECT * FROM orders ORDER BY total DESC
| 43  | app  | Query    |  115 | Sending data                    | SELECT o.*, u.name FROM orders o ...
| 44  | app  | Query    |   90 | Waiting for table metadata lock | ALTER TABLE orders ADD INDEX idx_total
| 45  | app  | Query    |   30 | Waiting for table metadata lock | INSERT INTO orders (...) VALUES (...)
| 46  | app  | Query    |   20 | Waiting for table metadata lock | UPDATE orders SET ... WHERE id = 100
```

**Questions:**
1. What is happening?
2. Which process is the blocker?
3. How to resolve without killing the ALTER?

<details><summary>Answer</summary>

1. **MDL chain:** The ALTER (id=44) needs exclusive MDL but is waiting because long-running SELECTs (42, 43) hold shared MDL. The INSERT (45) and UPDATE (46) are queued behind the ALTER waiting for MDL.

2. **Root cause:** Either session 42 (`Creating sort index` for 120s) or 43 (`Sending data` for 115s). They hold shared MDL.

3. **Resolution:** Kill the long-running SELECT that's blocking:
```sql
KILL 42;
KILL 43;
```
The ALTER will acquire MDL, proceed, and INSERTs/UPDATEs will resume.

**Better:** Use `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE, WAIT 10` to fail fast instead of blocking.

**Prevention:** Use `pt-online-schema-change` (acquires MDL only briefly), or schedule ALTERs during maintenance windows.

</details>

---

## 8. Debugging Scenarios

### Scenario 1: MySQL CPU 100%, All Queries in "Sending data"

**Situation:** Alert at 2pm — MySQL CPU 100%. `SHOW FULL PROCESSLIST` shows 50+ queries all in `Sending data` state on the `orders` table (5M rows). No recent code changes.

<details><summary>Answer</summary>

**Most likely cause:** Full table scans. All queries are scanning the entire table with no usable index, causing high CPU and buffer pool thrashing.

**Diagnosis:**
```sql
-- Check one query's plan:
EXPLAIN SELECT * FROM orders WHERE status = 'active' ORDER BY created_at DESC LIMIT 20;

-- Check buffer pool pressure:
SHOW STATUS LIKE 'Innodb_buffer_pool_read_%';
-- If Innodb_buffer_pool_reads (disk reads) is high, pool is thrashing.

-- Check handler counts:
FLUSH STATUS;
SELECT * FROM orders WHERE status = 'active' ORDER BY created_at DESC LIMIT 20;
SHOW SESSION STATUS LIKE 'Handler_read%';
-- High Handler_read_rnd_next vs Handler_read_key = full scan
```

**Immediate fix:**
1. Kill expensive queries: `KILL CONNECTION <id>;`
2. Add missing index:
```sql
ALTER TABLE orders ADD INDEX idx_status_created (status, created_at);
```
Use `gh-ost` or `pt-osc` to avoid blocking.

**Root cause:** The `status` column had no index (or a non-optimal one). The `ORDER BY created_at DESC LIMIT 20` caused a full scan + filesort. As traffic grew, queries piled up.

**Long-term prevention:**
- Enable slow query log with `log_queries_not_using_indexes = 1`
- Set `max_execution_time` as safety net
- Review all queries for index usage during code review

</details>

### Scenario 2: INSERTs Blocking on Metadata Lock

**Situation:** INSERTs into `orders` are timing out. `SHOW FULL PROCESSLIST`:
```
| Id | State                              | Info
| 10 | Waiting for table metadata lock     | INSERT INTO orders (...) VALUES (...)
| 11 | Waiting for table metadata lock     | INSERT INTO orders (...) VALUES (...)
| 12 | Sleep (600s)                        | (idle transaction)
| 13 | Waiting for table metadata lock     | ALTER TABLE orders ADD COLUMN ...
```

<details><summary>Answer</summary>

**Root cause:** Session 12 (idle for 10 minutes) has an open transaction holding a shared MDL on `orders`. The ALTER (13) waits for exclusive MDL. INSERTs queue behind the ALTER.

**Diagnosis:**
```sql
-- Find the blocking transaction:
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX
WHERE trx_state = 'RUNNING' AND trx_mysql_thread_id = 12;

-- Check metadata locks:
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_SCHEMA = 'sales' AND OBJECT_NAME = 'orders';
```

**Immediate fix:**
```sql
KILL CONNECTION 12;
```

**Root cause:** The application opened a transaction, ran some queries, but never committed/rolled back.

**Prevention:**
- Set `wait_timeout`/`interactive_timeout` to 300 seconds
- Monitor idle transactions: `SELECT * FROM PROCESSLIST WHERE COMMAND = 'Sleep' AND TIME > 300`
- Use `pt-online-schema-change` which minimizes MDL holding time
- Use `ALTER TABLE ... WAIT N` (8.0) to set max MDL wait time

</details>

### Scenario 3: Replication Lag Growing Continuously

**Situation:** Lag has been growing for 2 hours, now at 45 minutes.
```
Seconds_Behind_Master: 2700
Slave_SQL_Running_State: Reading event from the relay log
Relay_Log_Space: 2.5 GB (and growing)
```
Master CPU: 30%. Replica CPU: 80%. Master is executing a large DELETE.

<details><summary>Answer</summary>

**Root cause:** A large `DELETE FROM orders WHERE created_at < '2023-01-01'` on master generates millions of ROW binlog events. The replica applies these sequentially.

**Diagnosis:**
```sql
SHOW VARIABLES LIKE 'slave_parallel_workers';
SHOW VARIABLES LIKE 'slave_parallel_type';
SHOW FULL PROCESSLIST; -- check for reporting queries on replica
```

**Immediate fix:**
1. Pause/kill expensive reporting queries on the replica
2. Throttle the DELETE on master using pt-archiver:
```bash
pt-archiver --source h=master,D=sales,t=orders \
  --purge --where "created_at < '2023-01-01'" \
  --limit 10000 --sleep 1
```

**Long-term prevention:**
- Enable parallel replication: `slave_parallel_workers = 4` with `LOGICAL_CLOCK`
- Always batch large DELETEs (1000-10000 per batch with `LIMIT` and `SLEEP`)
- Monitor relay log space and lag trends with alerting

</details>

### Scenario 4: Frequent Deadlock Errors

**Situation:** Payment processing is getting frequent deadlock errors:
```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

Application code:
```php
DB::transaction(function () use ($userId, $amount) {
    $account = Account::lockForUpdate()->find($userId);
    $account->balance -= $amount;
    $account->save();

    Transaction::create([
        'user_id' => $userId,
        'amount' => $amount,
        'type' => 'debit'
    ]);
});
```

<details><summary>Answer</summary>

**Root cause:** The most likely deadlock scenario is **different lock order across code paths**. If there's another code path that locks accounts in a different order (or additional tables), deadlocks occur with concurrent requests.

**Fix — atomic update (no SELECT FOR UPDATE needed):**
```php
$affected = DB::update(
    "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?",
    [$amount, $userId, $amount]
);

if ($affected) {
    Transaction::create([
        'user_id' => $userId,
        'amount' => $amount,
        'type' => 'debit'
    ]);
}
```

**If you must use transactions:** Ensure consistent lock order and consider READ COMMITTED to disable gap locks.

**Additional checks:**
- Are there triggers/observers on accounts/transactions that acquire additional locks?
- Is `innodb_deadlock_detect = ON`?
- Consider lowering `innodb_lock_wait_timeout` to 5s for fast failure

</details>

### Scenario 5: MySQL Running Out of Memory

**Situation:** MySQL with 64GB RAM is killed by OOM killer. After restart, it runs for a few hours and dies again.

Config:
```ini
innodb_buffer_pool_size = 48G
innodb_log_file_size = 2G
max_connections = 500
query_cache_type = 0
table_open_cache = 4000
```

<details><summary>Answer</summary>

**Root cause:** 48G buffer pool + 2G×2 redo logs + per-connection buffers exceed 64GB.

**Memory calculation:**
- Buffer pool: 48G
- Redo logs: 2G × 2 = 4G
- Per-connection: ~1.2MB × 500 = 600MB (base)
- Temp tables (max 16MB each): up to 500 × 16MB = 8G
- Performance schema: ~500MB-1G
- Total: ~55-60G + OS overhead → OOM

**Immediate fix:**
1. Reduce `innodb_buffer_pool_size` to 32G
2. Reduce `max_connections` to 200 (use ProxySQL for pooling)
3. Enable `innodb_dedicated_server = ON`
4. Limit `tmp_table_size` and `max_heap_table_size`
5. Disable unnecessary Performance Schema instruments

**Long-term:**
- Use ProxySQL for connection pooling
- Monitor memory with `sys.memory_global_total`
- Use `cgroup`/`systemd` to limit MySQL memory as safety net

</details>

---

## 9. System Design Prompts

### Prompt 1: Multi-Tenant Inventory System Schema

**Context:** Design the DB schema for a multi-tenant inventory SaaS platform. Your existing experience is with PostgreSQL (Laravel, tenant_id on every table). How would this differ in MySQL?

<details><summary>Answer</summary>

**Approach 1: Shared tables with tenant_id column** (same as PostgreSQL)

```sql
CREATE TABLE inventory_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id INT NOT NULL,
    sku VARCHAR(50) NOT NULL,
    name VARCHAR(200) NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_tenant_sku (tenant_id, sku),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB;
```

**MySQL differences from PostgreSQL:**
- **Clustered index**: PK includes all row data. UUID as PK is expensive (stored in every secondary index). Use BIGINT AUTO_INCREMENT instead.
- **No partial indexes**: MySQL doesn't have `CREATE INDEX ... WHERE tenant_id = 1`. Use generated columns or separate tables.
- **No GIN/BRIN indexes**: Use B+Tree with composite indexes for multi-column queries.
- **ENUM vs lookup table**: Use ENUM for small fixed sets, lookup tables for dynamic values.
- **JSON**: MySQL JSON is functional but less powerful than Postgres JSONB for indexing.

**Recommendation:** Same shared-schema approach as Postgres (tenant_id on every table). Use partitioned tables per tenant only for very large tenants (>100M rows).

</details>

### Prompt 2: Sharding Strategy for E-Commerce Platform

**Context:** Design a sharding strategy for a rapidly growing e-commerce platform currently on a single MySQL 8.0 instance (2TB data, growing 100GB/month).

<details><summary>Answer</summary>

**Sharding key:** `customer_id` (or `tenant_id` for marketplace).

**Strategy:** Application-level sharding with Vitess, or ProxySQL + application routing.

**Vitess approach:**
```
Shard by customer_id % N (hash-based)
- Keyspace: orders, products, customers
- Each shard is a MySQL instance (or InnoDB Cluster)
- VSchema defines routing rules
- Resharding: double shards, migrate data online
```

**Application-level approach (simpler):**
```
customer_id hash → shard mapping in config/consul
Each shard has: master + 1-2 replicas
Cross-shard queries: scatter-gather via application
```

**Critical design decisions:**
1. **Cross-shard operations**: JOINs across shards are expensive. Prefer denormalization or CQRS.
2. **Global IDs**: Use Snowflake-style IDs (not AUTO_INCREMENT across shards):
   ```php
   // timestamp + shard_id + sequence
   ```
3. **Distributed transactions**: Avoid XA across shards. Use Saga pattern with compensating transactions.
4. **Shard rebalancing**: Plan for resharding (Vitess handles this online).

**MySQL-specific:**
- Each shard should be an InnoDB Cluster or async replication set
- Monitor per-shard load and rebalance hot shards
- Use ProxySQL for per-shard connection pooling

</details>

### Prompt 3: Zero-Downtime Schema Migration Strategy

**Context:** Design an online schema migration strategy for MySQL tables with millions of rows and 24/7 uptime requirement.

<details><summary>Answer</summary>

**Strategy:** Use `gh-ost` (GitHub's triggerless online schema migration):

```bash
gh-ost --alter="ADD COLUMN discount DECIMAL(5,2) DEFAULT 0.00" \
  --database="db" --table="orders" \
  --throttle-control-replicas="replica1:3306" \
  --max-load="Threads_running=25" \
  --chunk-size=1000 --execute
```

**Pipeline:**
1. **Prep:** Verify table has a PK, ROW-based replication is enabled, sufficient binlog retention
2. **Migration:** gh-ost creates a shadow table, streams binlog changes, copies data in chunks
3. **Cutover:** Switches tables atomically (RENAME)
4. **Validation:** Verify data consistency with `pt-table-checksum`
5. **Cleanup:** Drop old table after verification

**MySQL 8.0 features that help:**
- `ALGORITHM=INSTANT` for adding columns to end (8.0.12+)
- `ALGORITHM=INPLACE, LOCK=NONE` for index creation
- `ALTER TABLE ... WAIT N` for MDL timeout

**Fallback plan:** gh-ost can be cleanly canceled (pauses binlog replay, original table untouched).

**Monitoring:**
- Replication lag on all replicas
- Master threads_running and CPU
- gh-ost estimated completion time

</details>

### Prompt 4: MySQL HA with Zero RPO

**Context:** Design a high-availability setup for MySQL with zero data loss (RPO=0) and failover under 30 seconds (RTO<30s).

<details><summary>Answer</summary>

**Architecture:** MySQL InnoDB Cluster (Group Replication) with 3 nodes:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  MySQL 8.0  │     │  MySQL 8.0  │     │  MySQL 8.0  │
│  Primary    │←───→│  Secondary  │←───→│  Secondary  │
│  (writes)   │     │  (reads)    │     │  (reads)    │
└──────┬──────┘     └─────────────┘     └─────────────┘
       │
┌──────┴──────┐
│ MySQL Router│ ← Application
└─────────────┘
```

**Zero RPO requirements:**
- Group Replication with `group_replication_consistency = BEFORE_ON_PRIMARY_FAILOVER` (ensures data is on majority before ack)
- `group_replication_member_expel_timeout` set low for fast failover
- MySQL Router with `read_only` probe for automatic routing

**Alternative: Semi-sync replication + Orchestrator:**
- `rpl_semi_sync_master_enabled = 1`
- `rpl_semi_sync_master_timeout = 10000` (fall back to async after 10s)
- Orchestrator for failover with 3+ replicas, one delayed

**RTO<30s:**
- Pre-configured failover scripts (Orchestrator / MHA)
- ProxySQL for connection routing (detects master change in <5s)
- Application-side retry logic with idempotency keys

**Trade-off:** Group Replication requires fast network (<1ms RTT). Semi-sync + Orchestrator is simpler but may lose transactions on master crash.

</details>

### Prompt 5: MySQL Monitoring & Alerting at Scale

**Context:** Design a monitoring/alerting system for 50 MySQL instances (multi-shard, multi-region).

<details><summary>Answer</summary>

**Stack:** Prometheus + mysqld_exporter + Grafana + PagerDuty

**Metrics to collect (per instance):**

| Category | Key Metrics |
|----------|-------------|
| Availability | `Uptime`, `Threads_connected`, connection errors |
| Performance | QPS, TPS, buffer pool hit rate, `Innodb_rows_*` |
| Replication | Seconds_Behind_Master, GTID gap, relay log space |
| Locks | Lock waits, deadlocks, `Innodb_row_lock_current_waits` |
| Disk | Binlog growth, data size, disk usage |
| Memory | Buffer pool usage, memory allocated, OOM risk |

**Alerts (severity levels):**
```
CRITICAL (P1):
- Replica lag > 300s
- Instance down > 60s
- Disk usage > 90%
- Buffer pool hit rate < 95%

WARNING (P2):
- Replica lag > 30s
- Threads_connected > 80% of max_connections
- Deadlock rate > 10/min
- Slow queries > 100/min

INFO (P3):
- Disk usage > 75%
- Max_connections approaching limit
- Table fragmentation > 50%
```

**Per-region dashboards:**
- MySQL overview (all instances)
- Per-shard drill-down
- Replication topology view
- Query performance (Digest-based top N)

**Probes:**
- Synthetic queries every 10s (simple SELECT)
- Replication health check (GTID progress)
- Query response time percentile distribution

</details>

---

## 10. Questions to Ask the Interviewer

1. **What MySQL version are you currently running?** (Reveals migration posture, feature set available)

2. **What's your approach to zero-downtime schema migrations — do you use gh-ost, pt-osc, or something else?**

3. **How do you handle replication lag in read replicas when there's a write-heavy workload?**

4. **What's your backup and PITR strategy? How often do you test restores?**

5. **How do you monitor database performance? Prometheus/mysqld_exporter, PMM, or custom?**

6. **Have you encountered any MySQL-specific deadlock patterns in production? How were they resolved?**

7. **What's your disaster recovery plan for the database tier? Multi-region? Cross-region replicas?**

8. **How do you handle connections at scale — ProxySQL? MySQL Router? Direct connections?**

9. **What motivated you to choose MySQL over PostgreSQL for this use case?** (Or vice versa, if relevant)

10. **What's the biggest MySQL-related production incident you've handled, and what did the team learn from it?**

---

## 11. Red Flags to Avoid

1. **Claiming MySQL's `utf8` is real UTF-8.** It's `utf8mb3` — 3-byte only, no emoji support. Always use `utf8mb4`.

2. **Saying MyISAM is suitable for production.** No transactions, no crash recovery, table-level locking. Avoid in 8.0.

3. **Not knowing InnoDB's default isolation level** (REPEATABLE READ vs PostgreSQL's READ COMMITTED).

4. **Believing `SELECT COUNT(*)` is fast on InnoDB.** It's not — it scans an entire index.

5. **Saying MySQL replication is synchronous by default.** It's async by default. Semi-sync requires configuration.

6. **Not understanding the difference between `ibdata1` and `innodb_file_per_table`.** Shared tablespace never shrinks.

7. **Proposing `ALTER TABLE` in production without considering MDL.** It blocks all queries on the table until the metadata lock is acquired.

8. **Not knowing about gap locks at REPEATABLE READ.** A simple `SELECT ... FOR UPDATE` can lock more rows than expected.

9. **Saying MySQL 8.0 has a query cache.** It was removed. Don't rely on it (or suggest using it).

10. **Not understanding that `ONLY_FULL_GROUP_BY` is default in 5.7+.** Non-standard GROUP BY queries will break on upgrade.

---

> **End of Question Bank.**
>
> Next steps: Drill Rapid-Fire sections using `<details>` blocks. Work through Code Puzzles with actual `EXPLAIN ANALYZE` on a MySQL instance. Practice Debugging Scenarios aloud as if explaining to a team during an incident.
