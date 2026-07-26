# MySQL — Tier 1: Basic (Architecture, Storage Engine & SQL)

This guide covers MySQL from the perspective of a Senior Backend Engineer with PostgreSQL experience. Each section highlights MySQL–PostgreSQL differences, common traps, and likely follow-up questions.

> **Primary DB experience:** PostgreSQL → MySQL differences are called out throughout.

---

## Table of Contents

1. [MySQL Architecture Overview](#1-mysql-architecture-overview)
2. [InnoDB Storage Engine Deep Dive](#2-innodb-storage-engine-deep-dive)
3. [Data Types — Complete Reference](#3-data-types--complete-reference)
4. [Indexes — B+Tree Foundation](#4-indexes--btree-foundation)
5. [Creating & Managing Tables](#5-creating--managing-tables)
6. [Querying with MySQL](#6-querying-with-mysql)
7. [INSERT, UPDATE, DELETE Patterns](#7-insert-update-delete-patterns)
8. [MySQL vs PostgreSQL — Key Differences](#8-mysql-vs-postgresql--key-differences)
9. [Tier 1 Q&A Drill](#9-tier-1-qa-drill)

---

## 1. MySQL Architecture Overview

### Client/Server Protocol

MySQL uses a custom client/server protocol (not HTTP). Communication happens over TCP port 3306 by default, or a Unix socket on localhost. The protocol is binary after an initial handshake phase.

```sql
-- Connect via TCP
mysql -h hostname -u user -p -P 3306

-- Connect via Unix socket (faster on same host)
mysql -u user -p -S /var/run/mysqld/mysqld.sock
```

### Connection Handling

MySQL uses a **thread-per-connection** model (default). Each client connection gets its own OS thread. This differs from PostgreSQL's **process-per-connection** model.

| MySQL | PostgreSQL |
|---|---|
| Thread-per-connection | Process-per-connection |
| Lower memory overhead per connection (~256 KB per thread) | Higher overhead per connection (~5-10 MB per process) |
| All threads share the same address space (one crash can bring down all) | Processes are isolated (one crash kills one connection only) |
| `max_connections` default: 151 | `max_connections` default: 100 |

MySQL 8.0 introduced an optional **Thread Pool** plugin (in MySQL Enterprise Edition / Percona Server) that reuses threads to handle many connections efficiently.

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';
```

### Query Execution Pipeline

The journey of a query through MySQL:

```
SQL Text → Parser → Preprocessor → Optimizer → Executor → Storage Engine API
```

1. **Parser** — Tokenizes SQL, builds a parse tree, validates syntax
2. **Preprocessor** — Resolves table/column names, validates grants, expands `*`
3. **Optimizer** — Generates execution plans, evaluates costs, chooses indexes, applies transformations (e.g., semi-join, materialization)
4. **Executor** — Executes the plan, calls the storage engine API to read/write rows
5. **Storage Engine API** — Abstract interface (handler API) that all storage engines implement

```sql
-- See the execution plan
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'test@example.com';

-- See optimizer trace (MySQL 8.0)
SET optimizer_trace="enabled=on";
SELECT * FROM users WHERE email = 'test@example.com';
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

### Storage Engine Pluggable Architecture

MySQL's unique feature: **pluggable storage engines**. The server layer handles connections, parsing, optimization — storage engines handle data storage, indexing, transactions, locking.

```sql
-- Check available engines
SHOW ENGINES;

-- Check default engine
SHOW VARIABLES LIKE 'default_storage_engine';

-- Set engine per table
CREATE TABLE t (id INT PRIMARY KEY) ENGINE=InnoDB;
```

| Engine | Transactions | Row-level locks | Foreign Keys | Persists data | Use case |
|---|---|---|---|---|---|
| **InnoDB** | Yes | Yes | Yes | Yes | **Default** — general purpose |
| MyISAM | No | Table-level | No | Yes | Deprecated, read-only archives |
| Memory (HEAP) | No | Table-level | No | No (volatile) | Temp tables, caching |
| CSV | No | Table-level | No | Yes (CSV files) | Data exchange |
| Archive | No | Row-level | No | Yes (compressed) | Log storage, append-only |
| NDB (Cluster) | Yes | Row-level | Yes | Yes (distributed) | High-availability clustering |

### Trap: MyISAM in Production

**Trap**: Thinking MyISAM is acceptable for production workloads.

MyISAM has no transactions, no row-level locking, and is **crash-unsafe** (crashes can corrupt tables). If the server crashes during a write, MyISAM tables may require repair. MyISAM only supports table-level locks — any write blocks all reads to the table.

```sql
-- MyISAM table repair — not crash-safe
CHECK TABLE my_table;
REPAIR TABLE my_table;
```

**Follow-up**: "When would you ever use MyISAM today?" — Legacy read-only data warehouses where you don't care about crash safety and want compressed MyISAM tables. Even then, InnoDB is almost always better.

### Data Directory Structure

The data directory (`datadir`) contains all database files:

```
/var/lib/mysql/
├── ibdata1              # System tablespace (data dictionary, undo logs, doublewrite buffer)
├── ib_logfile0          # Redo log (circular, sequential write)
├── ib_logfile1          # Redo log (second file)
├── mysql/               # mysql system database
│   ├── user.ibd         # User accounts and privileges
│   ├── db.ibd           # Database-level privileges
│   ├── tables_priv.ibd  # Table-level privileges
│   ├── columns_priv.ibd # Column-level privileges
│   ├── procs_priv.ibd   # Stored procedure privileges
│   └── ...
├── sys/                 # sys schema (MySQL 8.0+)
├── performance_schema/  # Performance statistics
├── my_database/         # Per-database directory
│   ├── my_table.ibd     # Per-table tablespace (if innodb_file_per_table=ON)
│   └── ...
└── undo_001             # Undo tablespace (MySQL 8.0+)
```

```sql
-- Find datadir
SHOW VARIABLES LIKE 'datadir';
```

### Trap: Not Understanding `datadir` for Backups

**Trap**: Assuming you can just `cp -r` the `datadir` for backups.

- `ibdata1` and `ib_logfile*` must be backed up consistently (use `FLUSH TABLES WITH READ LOCK` or a snapshot)
- Unflushed changes in the redo log will be lost if you copy files while the server is running
- Use `mysqldump` or `mysqlbackup` (or snapshots + binary logs) for proper backups

```sql
-- Consistent backup with lock
FLUSH TABLES WITH READ LOCK;
-- Copy files
UNLOCK TABLES;

-- Better: use mysqldump
mysqldump --single-transaction --all-databases > backup.sql
```

### `mysql` System Database

The `mysql` schema holds MySQL's privilege and metadata tables:

```sql
USE mysql;
SELECT User, Host, plugin FROM user;
SELECT * FROM db WHERE Db = 'mydb';
SELECT * FROM tables_priv WHERE Table_name = 'users';
```

In MySQL 8.0+, privilege tables use the InnoDB engine (previously MyISAM), making grant operations transactional.

**PG comparison**: PostgreSQL stores roles/privileges in system catalogs (`pg_authid`, `pg_auth_members`). MySQL's privilege system is more granular per object but historically had no row-level security (RLS added in 8.0 via `CREATE ROLE` and partial RLS).

---

## 2. InnoDB Storage Engine Deep Dive

### Clustered Index

InnoDB stores rows in a **clustered index** (a B+Tree where the leaf pages contain the actual row data). The primary key IS the clustered index.

- Rows are physically ordered by the primary key
- Accessing a row by PK is the fastest path (one B+Tree traversal finds the row)
- The clustered index is the table — there is no separate heap structure like PostgreSQL

```sql
-- PK lookup: single B+Tree traversal
SELECT * FROM users WHERE id = 42;
```

**PG comparison**: PostgreSQL uses a heap for row storage. Indexes store (ctid) pointers to heap tuples. MVCC creates new row versions in the heap. MySQL's clustered index stores rows in the index itself — no separate heap.

### Secondary Indexes

Secondary indexes (non-clustered indexes) store the **primary key value** in the leaf, not a row pointer or CTID.

```sql
CREATE INDEX idx_email ON users(email);

-- Lookup: traverse idx_email B+Tree → find PK value → traverse clustered index
SELECT * FROM users WHERE email = 'test@example.com';
```

This means:
- Lookup requires **two** B+Tree traversals (index → PK → row)
- A large PK (e.g., UUID, composite) is stored in EVERY secondary index → bloat

```sql
-- Bad: UUID PK bloats all secondary indexes
CREATE TABLE bad (
    id CHAR(36) PRIMARY KEY,   -- 16 bytes but stored as CHAR(36) → 36 bytes × 2 (secondary index leaf) = 72 bytes
    email VARCHAR(255),
    INDEX idx_email(email)      -- leaf stores: email + 36-byte PK
);

-- Good: integer PK
CREATE TABLE good (
    id INT AUTO_INCREMENT PRIMARY KEY,  -- 4 bytes
    uuid CHAR(36) NOT NULL,
    email VARCHAR(255),
    UNIQUE INDEX idx_uuid(uuid),        -- leaf stores: uuid + 4-byte PK
    INDEX idx_email(email)              -- leaf stores: email + 4-byte PK
);
```

### Trap: Tables Without Explicit PK

**Trap**: Creating a table without an explicit primary key.

InnoDB will:
1. Look for a `NOT NULL` unique key as the clustered index
2. If none found, add a hidden **6-byte `ROW_ID`** as an auto-incrementing clustered index

Problems:
- The `ROW_ID` is global across all tables without PKs (6-byte counter shared)
- Every secondary index references this hidden 6-byte row ID
- You cannot control or query by it
- Replication relies on primary keys for row-based replication (RBR) — missing PK causes full table scans for UPDATE/DELETE

```sql
CREATE TABLE no_pk (
    name VARCHAR(100),
    INDEX idx_name(name)
);
-- 👆 InnoDB adds hidden row_id → secondary index references 6-byte invisible PK

-- Always add an explicit PK
CREATE TABLE with_pk (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    INDEX idx_name(name)
);
```

**Follow-up**: "What if I use a UUID as PK?" — Insert performance suffers because UUIDs are random and cause page splits in the clustered index. Use `UUID_TO_BIN()` in MySQL 8.0 or a sequential UUID (e.g., ULID, Snowflake).

### MVCC (Multi-Version Concurrency Control)

InnoDB implements MVCC to allow concurrent reads and writes without blocking each other.

Key components:

#### Undo Log

- Stores old versions of rows (before-images)
- Located in the system tablespace (`ibdata1`) or separate undo tablespaces (MySQL 8.0+)
- Used for: consistent reads (SELECT), rollback of transactions, purging old versions

```sql
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
SHOW VARIABLES LIKE 'innodb_undo_log_truncate';
```

**PG comparison**: PostgreSQL stores old row versions in the heap (dead tuples). InnoDB stores them in undo logs, keeping the current row in the clustered index. PG requires VACUUM to reclaim dead tuple space; InnoDB requires purge (automatic background process).

#### Read View

Each transaction gets a consistent snapshot of the database at the time the first read occurs (for REPEATABLE READ) or at each statement (for READ COMMITTED). The read view tracks which transactions were active at that moment.

```sql
-- REPEATABLE READ: consistent view for entire transaction
-- READ COMMITTED: consistent view per statement
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT * FROM users WHERE id = 1;  -- snapshot taken here
-- ...
COMMIT;
```

### Redo Log (`ib_logfile`)

The redo log is a **write-ahead log** for crash recovery:

- Sequential writes to `ib_logfile0`, `ib_logfile1` (circular buffer)
- Every modification is written to redo log BEFORE the data file (WAL — Write-Ahead Logging)
- On crash, InnoDB replays the redo log to restore committed transactions

```sql
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_log_files_in_group';
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

`innodb_flush_log_at_trx_commit`:
- `1` (default) — flush to disk on every commit (durable but slower)
- `2` — flush to OS cache on commit, flush to disk per second (loses 1s of data on crash)
- `0` — write to log + flush to disk per second (loses 1s of data, fastest)

**PG comparison**: PostgreSQL uses WAL (Write-Ahead Log) — similar concept but different implementation. PG's WAL is more feature-rich: supports physical + logical replication, archiving, and point-in-time recovery out of the box. MySQL's redo log is primarily for crash recovery; binary log handles replication and PITR.

### Doublewrite Buffer

InnoDB writes pages in two places: the doublewrite buffer first, then the actual data file. This prevents **partial page writes** (when 16 KB page is only partially written due to power failure, leaving corrupt data).

```sql
SHOW VARIABLES LIKE 'innodb_doublewrite';
```

**Trap**: Doublewrite on systems with atomic write size (ZFS, some SSDs with 16 KB atomic writes).

On ZFS or Fusion-IO devices where 16 KB writes are atomic, the doublewrite buffer is unnecessary overhead (doubles I/O). You can disable it safely.

```sql
-- For systems with atomic write size (ZFS)
SET GLOBAL innodb_doublewrite = 0;
```

### Change Buffer

Caches changes to **secondary index entries** when the page is not in the buffer pool. Instead of immediately reading the page from disk to apply the change, InnoDB buffers it and merges later.

```sql
SHOW VARIABLES LIKE 'innodb_change_buffer_max_size';  -- % of buffer pool
SHOW VARIABLES LIKE 'innodb_change_buffering';         -- all, none, inserts, etc.
```

Most beneficial for: workloads with many secondary indexes and non-unique indexes (unique indexes must check for duplicates immediately).

**PG comparison**: PostgreSQL doesn't have a direct equivalent. PG's B+Tree (actually B-Link-Tree) implementation handles concurrent index inserts differently — no change buffer concept.

### Adaptive Hash Index (AHI)

InnoDB builds an in-memory hash index on frequently accessed pages in the buffer pool. This is automatic and transparent.

```sql
SHOW VARIABLES LIKE 'innodb_adaptive_hash_index';
SHOW STATUS LIKE '%adaptive_hash%';
```

Can be disabled if hash index contention is detected (mutex contention on AHI).

### Buffer Pool

The buffer pool is InnoDB's main memory cache — stores data pages, index pages, change buffer, AHI, row locks, etc.

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_buffer_pool_instances';  -- Multiple instances reduce contention
```

- Default in MySQL 8.0: 128 MB (always too small for production)
- Recommended: 60-80% of available RAM for a dedicated DB server
- LRU-based eviction with midpoint insertion strategy (prevents large scans from flushing hot data)

```sql
SHOW ENGINE INNODB STATUS\G
-- Look at BUFFER POOL AND MEMORY section
-- Shows: total pages, free pages, database pages, hit rate
-- Target: >99% hit rate on read-heavy workloads
```

**Trap**: Buffer pool too small.

Symptoms: disk thrashing, high read I/O, `Innodb_buffer_pool_reads` (reads from disk) >> `Innodb_buffer_pool_read_requests`.

```sql
-- Calculate buffer pool hit rate
SELECT (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_rate
FROM performance_schema.global_status
WHERE Innodb_buffer_pool_read_requests > 0;
```

---

## 3. Data Types — Complete Reference

### Numeric Types

| Type | Storage | Range (Signed) | Range (Unsigned) |
|---|---|---|---|
| `TINYINT` | 1 byte | -128 to 127 | 0 to 255 |
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | 0 to 65,535 |
| `MEDIUMINT` | 3 bytes | -8,388,608 to 8,388,607 | 0 to 16,777,215 |
| `INT` | 4 bytes | -2,147,483,648 to 2,147,483,647 | 0 to 4,294,967,295 |
| `BIGINT` | 8 bytes | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 | 0 to 18,446,744,073,709,551,615 |

```sql
CREATE TABLE numeric_example (
    age TINYINT UNSIGNED,           -- 0-255, good for age
    score SMALLINT,                 -- -32768 to 32767
    population MEDIUMINT UNSIGNED,  -- up to 16M
    id INT UNSIGNED AUTO_INCREMENT, -- 0 to 4B
    big BIGINT                      -- large counters
);

-- INT(11) is misleading — display width is deprecated in 8.0.19+
CREATE TABLE int_width (
    id INT(11)   -- Same as INT. Display width means nothing in 8.0+
);
```

#### DECIMAL

For exact precision — use for money, financial calculations.

```sql
CREATE TABLE pricing (
    price DECIMAL(10,2),  -- 10 total digits, 2 after decimal: 99999999.99
    tax_rate DECIMAL(5,4) -- 0.0000 to 9.9999
);
```

Storage: approx 4 bytes per 9 digits. DECIMAL(10,2) → 5 bytes.

**PG comparison**: `DECIMAL` is `NUMERIC` in PostgreSQL. Same behavior. MySQL also supports `NUMERIC` as a synonym.

#### FLOAT / DOUBLE

For approximate numeric values. Never use for money.

```sql
CREATE TABLE scientific (
    measurement FLOAT(7,4),   -- 4 bytes, ~7 significant digits
    precise DOUBLE(16,4)      -- 8 bytes, ~15 significant digits
);
```

### Trap: INT(11) Display Width

**Trap**: Thinking `INT(11)` limits the value to 11 digits.

```sql
CREATE TABLE myth (
    id INT(11)
);

INSERT INTO myth VALUES (123456789012);  -- OK, INT can hold up to 2147483647
-- Wait — it can't. INT(11) is still INT (4 bytes, max ~2.1B signed)
```

`INT(11)` is a deprecated display width (removed in MySQL 8.0.19). Only affects padding with `ZEROFILL`. It does NOT restrict values.

### String Types

| Type | Max Length | Storage | Notes |
|---|---|---|---|
| `CHAR(N)` | 255 chars | Fixed (N × charset bytes) | Padded to N, good for fixed-length codes |
| `VARCHAR(N)` | 65,535 chars | Variable (length + data) | 1-2 byte length prefix + actual data |
| `TINYTEXT` | 255 bytes | Variable | No length prefix shown to user |
| `TEXT` | 65,535 bytes | Variable | |
| `MEDIUMTEXT` | 16,777,215 bytes | Variable | |
| `LONGTEXT` | 4,294,967,295 bytes | Variable | |

```sql
CREATE TABLE strings (
    code CHAR(3),              -- Fixed: 'ABC' → 3 bytes (latin1)
    email VARCHAR(255),        -- Variable: 1 or 2 byte length prefix + data
    bio TEXT,                  -- Stored off-page if > 767 bytes (compact row format)
    big_content MEDIUMTEXT     -- Up to 16 MB
);
```

**PG comparison**: MySQL's `VARCHAR(N)` caps at 65,535 characters. PostgreSQL's `VARCHAR(N)` has no practical limit (up to 1 GB in a single field with `TOAST`). MySQL's `TEXT` is stored off-page for large values; PG's `TOAST` automatically compresses and stores large values out of line.

### Charset & Collation

```sql
-- Check server defaults
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';

-- Set per table
CREATE TABLE utf8_table (
    name VARCHAR(100)
) DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Set per column
CREATE TABLE mixed (
    name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
    code VARCHAR(10) CHARACTER SET latin1 COLLATE latin1_general_ci
);
```

#### utf8mb4 vs utf8mb3

| Charset | Max bytes per char | Emoji support | Notes |
|---|---|---|---|
| `utf8mb3` | 3 bytes | No ❌ | MySQL's "utf8" is actually utf8mb3 |
| `utf8mb4` | 4 bytes | Yes ✅ | True UTF-8, use this |

```sql
-- This can silently truncate emoji
CREATE TABLE bad (
    comment VARCHAR(255) CHARSET utf8   -- actually utf8mb3
);

INSERT INTO bad VALUES ('Hello 😊');  -- 😊 truncated or error

-- Correct
CREATE TABLE good (
    comment VARCHAR(255) CHARSET utf8mb4
);
```

**Trap**: `utf8` in MySQL is NOT UTF-8. It's utf8mb3 (max 3 bytes). Always use `utf8mb4`.

#### Collation Options

```sql
SHOW COLLATION WHERE Charset = 'utf8mb4';
```

| Collation | Performance | Accuracy | Notes |
|---|---|---|---|
| `utf8mb4_general_ci` | Fast | Less accurate | Unicode 4.0, simpler comparison rules |
| `utf8mb4_unicode_ci` | Slower | More accurate | Unicode 4.0, standard algorithm |
| `utf8mb4_unicode_520_ci` | Slower | More accurate | Unicode 5.20 |
| `utf8mb4_0900_ai_ci` | Fast | Most accurate | Unicode 9.0, accent-insensitive, MySQL 8.0 default |

**Recommendation**: Use `utf8mb4_unicode_ci` for 5.7, `utf8mb4_0900_ai_ci` for 8.0+.

### Temporal Types

| Type | Storage | Range | Timezone |
|---|---|---|---|
| `DATE` | 3 bytes | `1000-01-01` to `9999-12-31` | No |
| `DATETIME` | 5-8 bytes | `1000-01-01 00:00:00` to `9999-12-31 23:59:59` | No |
| `TIMESTAMP` | 4 bytes | `1970-01-01 00:00:01` UTC to `2038-01-19 03:14:07` UTC | Yes (converts to session tz) |
| `YEAR` | 1 byte | `1901` to `2155` (or `0000` to `9999` in 2-digit mode) | No |
| `TIME` | 3 bytes | `-838:59:59` to `838:59:59` | No |

```sql
CREATE TABLE temporal (
    birth_date DATE,
    created_at DATETIME(3),       -- microsecond precision
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    event_year YEAR,
    duration TIME
);

-- TIMESTAMP auto-converts to session timezone
SET time_zone = '+00:00';
INSERT INTO temporal (created_at, updated_at) VALUES (NOW(), NOW());

SET time_zone = '+05:30';
SELECT updated_at FROM temporal;  -- Shows +05:30 time
```

### Trap: TIMESTAMP 2038 Problem

**Trap**: Using `TIMESTAMP` for dates beyond 2038.

```sql
CREATE TABLE future_planning (
    event_date TIMESTAMP  -- max: 2038-01-19 03:14:07 UTC
);

INSERT INTO future_planning VALUES ('2050-01-01');  -- ERROR or truncation
```

Use `DATETIME` for dates beyond 2038. `DATETIME` has no timezone conversion but can store dates up to 9999.

**Follow-up**: "Which type should you use for `created_at`/`updated_at`?" — `TIMESTAMP` is conventional (auto-conversion, less storage), but `DATETIME` is safer for long-range dates. Many teams use `BIGINT` (Unix epoch in milliseconds) for portability.

### JSON Type

Native JSON type (MySQL 5.7+) stores JSON in an optimized binary format (similar to PostgreSQL's JSONB).

```sql
CREATE TABLE json_demo (
    id INT PRIMARY KEY,
    data JSON
);

INSERT INTO json_demo VALUES (1, '{"name": "Alice", "age": 30, "tags": ["admin", "user"]}');

-- JSON functions
SELECT JSON_EXTRACT(data, '$.name') FROM json_demo;           -- "Alice"
SELECT data->'$.name' FROM json_demo;                          -- "Alice" (-> operator)
SELECT data->>'$.name' FROM json_demo;                         -- Alice (unquoted)
SELECT JSON_UNQUOTE(JSON_EXTRACT(data, '$.name')) FROM json_demo; -- Alice

-- JSON_TABLE (MySQL 8.0+)
SELECT jt.*
FROM json_demo,
     JSON_TABLE(data, '$' COLUMNS (
         name VARCHAR(100) PATH '$.name',
         age INT PATH '$.age'
     )) AS jt;

-- Multi-value index (MySQL 8.0.17+) — index into JSON array
CREATE INDEX idx_tags ON json_demo((CAST(data->'$.tags' AS UNSIGNED ARRAY)));
```

**PG comparison**: MySQL's `JSON` type is similar to PostgreSQL's `JSONB` (binary, optimized). MySQL 8.0 has multi-value indexes for JSON arrays. PG has hash indexes for JSONB and more expression index flexibility. Both support JSON path expressions (MySQL 8.0.17+).

### ENUM

A string object with a fixed set of values, stored internally as an integer (1-2 bytes).

```sql
CREATE TABLE enum_demo (
    status ENUM('pending', 'active', 'suspended', 'deleted')  -- stored as 1,2,3,4
);

INSERT INTO enum_demo VALUES ('active');  -- stored as 2
SELECT status + 0 FROM enum_demo;         -- returns 2 (integer value)
```

- Max 65,535 distinct values
- ORDER BY on ENUM sorts by **index** (insertion order), not alphabetically
- Adding new values requires ALTER TABLE (metadata lock)

### Trap: ENUM Ordering Surprises

**Trap**: Assuming ENUM sorts alphabetically.

```sql
CREATE TABLE colors (
    name ENUM('red', 'blue', 'green')
);

INSERT INTO colors VALUES ('red'), ('blue'), ('green');
SELECT * FROM colors ORDER BY name;
-- Result: red, blue, green (ordered by index 1,2,3, not alphabetically)
```

To sort alphabetically, cast to string: `ORDER BY CAST(name AS CHAR)` or use `VARCHAR`.

### SET Type

Like ENUM but can store multiple values (bitmask):

```sql
CREATE TABLE permissions (
    perms SET('read', 'write', 'execute', 'admin')
);

INSERT INTO permissions VALUES ('read,write');
```

Max 64 distinct values. Stored as a bitmask.

### Trap: Non-Standard GROUP BY

**Trap**: MySQL's default `sql_mode` in older versions allowed selecting columns not in GROUP BY without aggregate functions.

```sql
-- MySQL 5.7 default (sql_mode includes ONLY_FULL_GROUP_BY in 8.0)
SET sql_mode = '';
SELECT name, email, COUNT(*) FROM users GROUP BY name;  -- Works! Returns arbitrary email

-- MySQL 8.0.1+ default
SET sql_mode = 'ONLY_FULL_GROUP_BY';
SELECT name, email, COUNT(*) FROM users GROUP BY name;  -- ERROR: not in GROUP BY
```

**Follow-up**: "How do you handle columns not in GROUP BY?" — Use `ANY_VALUE()` (MySQL 5.7+) to explicitly suppress the error when you know any value is acceptable, or better: restructure the query to be standards-compliant.

---

## 4. Indexes — B+Tree Foundation

### B+Tree Structure

MySQL uses B+Tree indexes (InnoDB default). Key properties:

- All data is in leaf nodes (clustered index: row data; secondary index: PK value)
- Internal nodes store only keys (pointers to children)
- Leaf nodes are linked in a doubly-linked list for efficient range scans
- Height is typically 3-4 for millions of rows

```
             [50]
            /    \
        [20]      [80]
       /    \     /    \
     [10]  [30] [60]  [90]    ← Leaf nodes contain data
      ↓-----↓-----↓-----↓     ← Leaf linked list for range scan
```

```sql
-- Range scan uses leaf node linked list
SELECT * FROM users WHERE id BETWEEN 100 AND 200;
-- Only traverses tree once, then walks leaf nodes sequentially
```

### Clustered vs Secondary Indexes

| | Clustered Index | Secondary Index |
|---|---|---|
| Leaf contains | Full row data | PK value |
| One per table | Yes (1) | Many |
| Primary key | Yes (implicit) | No |
| Lookup | Single B+Tree traversal | Two traversals |
| Row order | PK order | Index key order |

```sql
-- Primary key = clustered index
CREATE TABLE users (
    id INT PRIMARY KEY,        -- Clustered index
    email VARCHAR(255),
    name VARCHAR(100),
    INDEX idx_email(email),    -- Secondary index
    INDEX idx_name(name)       -- Another secondary index
);

-- Query plan
EXPLAIN SELECT * FROM users WHERE id = 1;       -- type: const, single traversal
EXPLAIN SELECT * FROM users WHERE email = 'a';  -- type: ref, two traversals
```

### Trap: Large PK Bloating Secondary Indexes

**Trap**: Using a wide primary key (e.g., UUID, multi-column) without considering secondary index bloat.

Every secondary index stores the PK value in its leaf nodes. If PK is 100 bytes, every secondary index entry has 100+ bytes overhead.

```sql
-- BAD: UUID CHAR(36) PK → every secondary index has 36 byte overhead
CREATE TABLE bad (
    id CHAR(36) PRIMARY KEY,
    email VARCHAR(255),
    INDEX idx_email(email)  -- leaf: email + 36-byte PK
);

-- GOOD: Integer surrogate PK
CREATE TABLE good (
    id INT AUTO_INCREMENT PRIMARY KEY,
    uuid BINARY(16) NOT NULL UNIQUE,  -- Business UUID separate
    email VARCHAR(255),
    INDEX idx_email(email)  -- leaf: email + 4-byte PK
);
```

**Follow-up**: "How do you handle UUIDs?" — Store as `BINARY(16)` using `UUID_TO_BIN()` / `BIN_TO_UUID()` in MySQL 8.0, or use a sequential key generator (Snowflake, ULID) for clustered index performance.

### Composite Indexes

Indexes on multiple columns. The **leftmost prefix rule** applies: queries must use columns from the left of the index definition.

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT NOT NULL,
    status VARCHAR(20),
    created_at DATETIME,
    INDEX idx_user_status_date (user_id, status, created_at)
);

-- Uses index fully
SELECT * FROM orders WHERE user_id = 42 AND status = 'active' AND created_at > '2024-01-01';

-- Uses index partially (user_id for lookup, then filters in memory or uses index for sort)
SELECT * FROM orders WHERE user_id = 42 AND created_at > '2024-01-01';
-- Uses user_id from index, created_at cannot use index because status was skipped

-- Does NOT use the index
SELECT * FROM orders WHERE status = 'active';  -- Can't use idx — leftmost column missing
```

**Column order rules:**
1. **Equality columns first** — `WHERE col1 = 'x' AND col2 = 'y'` → put both in index
2. **Range columns last** — `WHERE col1 = 'x' AND col2 > 10` → `INDEX(col1, col2)`, col2 can be range but stops further columns
3. **Sort columns** — Can be satisfied by index if in order matching ORDER BY

```sql
-- Index can satisfy ORDER BY if leading columns match
CREATE INDEX idx_user_date ON orders(user_id, created_at);

SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at;
-- No filesort needed — index already provides order
```

### Cardinality and Selectivity

- **Cardinality**: Number of unique values in an index
- **Selectivity**: Cardinality / total rows (high = good)
- MySQL's optimizer uses cardinality estimates to decide whether to use an index

```sql
-- Show cardinality
SHOW INDEX FROM orders;
-- Cardinality column shows estimated distinct values

-- Update statistics
ANALYZE TABLE orders;

-- Check index selectivity
SELECT COUNT(DISTINCT user_id) AS cardinality,
       COUNT(*) AS total,
       COUNT(DISTINCT user_id) / COUNT(*) AS selectivity
FROM orders;
```

### Using `SHOW INDEX` / `EXPLAIN`

```sql
-- Show all indexes on a table
SHOW INDEX FROM users\G
-- Output columns: Table, Non_unique, Key_name, Seq_in_index, Column_name, Collation, Cardinality, Sub_part, Packed, Null, Index_type, Comment, Index_comment

-- EXPLAIN basics
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

#### EXPLAIN Output

| Column | Meaning |
|---|---|
| `id` | SELECT identifier (larger = earlier execution) |
| `select_type` | SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION, etc. |
| `table` | Table name |
| `type` | **Access method** — most important column |
| `possible_keys` | Indexes MySQL could use |
| `key` | Index MySQL chose |
| `key_len` | Bytes of the index used |
| `ref` | Columns/constants compared to index |
| `rows` | Estimated rows examined |
| `Extra` | Additional information (Using index, Using where, Using filesort, etc.) |

#### `type` Access Methods — Best to Worst

| type | Meaning | Example |
|---|---|---|
| `system` | Table has 0 or 1 row | Trivial |
| `const` | Primary key or unique index lookup | `WHERE id = 1` |
| `eq_ref` | Join on PK/unique index, one row per join | `JOIN ... ON t1.id = t2.user_id` |
| `ref` | Non-unique index lookup, multiple rows | `WHERE email = 'test@example.com'` |
| `range` | Index range scan | `WHERE id BETWEEN 10 AND 20` |
| `index` | Full index scan (all leaf pages read) | No useful filter, but index covers query |
| `ALL` | Full table scan (worst) | No index used |

```sql
-- const
EXPLAIN SELECT * FROM users WHERE id = 1;                        -- type: const

-- ref
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';    -- type: ref

-- range
EXPLAIN SELECT * FROM users WHERE id BETWEEN 100 AND 200;        -- type: range

-- ALL (bad)
EXPLAIN SELECT * FROM users WHERE name LIKE '%search%';          -- type: ALL
```

### Trap: One Index Per Table Per Query

**Trap**: Assuming MySQL uses multiple indexes in a single query.

MySQL typically uses **one index per table** per query (the optimizer picks the most selective one). Exceptions: **index merge** optimization (MySQL 5.0+), which can combine results from multiple indexes.

```sql
CREATE TABLE t (
    a INT,
    b INT,
    INDEX idx_a(a),
    INDEX idx_b(b)
);

-- MySQL picks ONE index (whichever it estimates is more selective)
EXPLAIN SELECT * FROM t WHERE a = 1 OR b = 2;
-- Possible: index_merge (using union(idx_a, idx_b))
-- Possible: full table scan (if OR covers too many rows)

-- Index merge is less efficient than a composite index
CREATE INDEX idx_a_b ON t(a, b);  -- Better for most queries
```

**Follow-up**: "When does index merge help?" — When you have two highly selective predicates on different columns and you can't change the schema. But a composite index is almost always better.

### Trap: `key_len` Calculation

`key_len` tells you how many bytes of the index MySQL is using. Understanding it helps debug whether your composite index is used fully or partially.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'x' AND name = 'y';
-- key_len = 767 means only email part used (if email is VARCHAR(255) utf8mb4)
-- key_len = 1534 means both parts used
```

**Calculation rules:**
- `INT` = 4 bytes, `BIGINT` = 8 bytes, `SMALLINT` = 2 bytes
- `VARCHAR(N)` = `N × charset_bytes + 2` (length prefix) + nullable flag (1 byte if nullable)
- `CHAR(N)` = `N × charset_bytes`
- `utf8mb4` uses 4 bytes per character

```sql
-- For: VARCHAR(255) utf8mb4, NOT NULL
-- key_len = 255 × 4 + 2 = 1022

-- For: VARCHAR(255) utf8mb4, NULL
-- key_len = 255 × 4 + 2 + 1 = 1023

CREATE TABLE example (
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    INDEX idx_email_name(email, name)
);

EXPLAIN SELECT * FROM example WHERE email = 'test@example.com';
-- key_len = 1022 (only email used)

EXPLAIN SELECT * FROM example WHERE email = 'test@example.com' AND name = 'Alice';
-- key_len = 1022 + 402 = 1424 (both parts used: 100*4 + 2 for name if nullable)
```

### Summary of Index Guidelines

1. **Always have an explicit PK** — preferably small (INT/BIGINT)
2. **Keep PK narrow** — every secondary index includes it
3. **Composite index column order** — equality columns first, then range columns, then sort columns
4. **Monitor `key_len`** in EXPLAIN — if it's shorter than expected, your index isn't fully used
5. **Avoid too many indexes** — each index slows writes
6. **Limit index on low-cardinality columns** — `INDEX(is_active)` on a boolean is usually useless (unless 99% are one value and 1% is the other)
7. **Prefix indexes for long strings** — `INDEX(email(10))` indexes first 10 chars only

---

## 5. Creating & Managing Tables

### CREATE TABLE Syntax

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    status ENUM('active', 'inactive') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX idx_email(email),
    INDEX idx_name(name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

Key clauses:
- `ENGINE=InnoDB` — storage engine (default in 8.0)
- `DEFAULT CHARSET=utf8mb4` — character set
- `COLLATE=utf8mb4_unicode_ci` — collation
- `AUTO_INCREMENT=1000` — starting auto-increment value

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id INT UNSIGNED NOT NULL,
    total DECIMAL(12,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at DATETIME(3),
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB;
```

### ALTER TABLE

```sql
-- Add column
ALTER TABLE users ADD COLUMN phone VARCHAR(20) AFTER email;

-- Drop column
ALTER TABLE users DROP COLUMN phone;

-- Modify column (type change, often rebuilds table)
ALTER TABLE users MODIFY COLUMN name VARCHAR(200) NOT NULL;

-- Rename column (MySQL 8.0+)
ALTER TABLE users RENAME COLUMN name TO full_name;

-- Add index
ALTER TABLE users ADD INDEX idx_email(email);

-- Drop index
ALTER TABLE users DROP INDEX idx_name;

-- Rename table
ALTER TABLE users RENAME TO customers;
```

### Trap: ALTER TABLE Blocks and Rebuilds

**Trap**: Running `ALTER TABLE` on a large production table without understanding the implications.

Many `ALTER TABLE` operations rebuild the table by copying data to a temp table (blocking writes). Some operations also block reads during different phases.

```sql
-- This rebuilds the entire table → blocks writes during copy phase
ALTER TABLE large_table MODIFY COLUMN name VARCHAR(500);

-- Some operations are "instant" in MySQL 8.0 (instant ADD COLUMN)
-- Types of ALTER: COPY (worst), INPLACE (better), INSTANT (best, 8.0.12+)
```

| Operation | Algorithm | Blocks writes | Blocks reads |
|---|---|---|---|
| ADD COLUMN (non-last) | INSTANT (8.0.12+) | No | No |
| ADD COLUMN (last position) | INSTANT (8.0.12+) | No | No |
| DROP COLUMN | INPLACE | Yes (brief) | No |
| ADD INDEX | INPLACE | No | No |
| DROP INDEX | INPLACE | No | No |
| MODIFY COLUMN type | COPY | Yes | Yes (during copy) |
| CHANGE COLUMN name | INPLACE | No | No |
| SET DEFAULT | INPLACE | No | No |
| RENAME TABLE | — | No | No |

For operations that COPY: use **pt-online-schema-change** (Percona Toolkit) or **gh-ost** (GitHub) to run online migrations without downtime.

```sql
-- Check algorithm for your ALTER
ALTER TABLE t ADD COLUMN x INT, ALGORITHM=INSTANT, LOCK=NONE;
-- If INSTANT not supported, falls back to INPLACE or COPY
```

### Generated Columns

MySQL 8.0 supports generated (computed) columns:

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    price DECIMAL(10,2),
    tax DECIMAL(5,2),
    total DECIMAL(10,2) GENERATED ALWAYS AS (price + (price * tax)) STORED
);

-- VIRTUAL: not stored, computed on read (no space, slightly slower reads)
-- STORED: stored physically (takes space, faster reads)

CREATE TABLE virtual_example (
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    full_name VARCHAR(101) GENERATED ALWAYS AS (CONCAT(first_name, ' ', last_name)) VIRTUAL
);
```

**Indexing generated columns**:

MySQL 8.0.13+ allows indexing VIRTUAL columns.

```sql
-- Can index VIRTUAL column (MySQL 8.0.13+)
CREATE TABLE users (
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    full_name VARCHAR(101) GENERATED ALWAYS AS (CONCAT(first_name, ' ', last_name)) VIRTUAL,
    INDEX idx_full_name(full_name)
);

SELECT * FROM users WHERE full_name = 'Alice Smith';  -- Uses index
```

### Foreign Keys

Only supported by InnoDB. Enforce referential integrity with cascading actions.

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE        -- Delete orders when user is deleted
        ON UPDATE CASCADE        -- Update user_id in orders if PK changes
);

CREATE TABLE profiles (
    id INT PRIMARY KEY,
    user_id INT NOT NULL UNIQUE,
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE
);
```

InnoDB automatically creates an index on the foreign key column (unlike PostgreSQL where you must create the index manually).

### Trap: Foreign Key Performance Overhead

**Trap**: Using foreign keys on write-heavy tables without considering the overhead.

Every INSERT/UPDATE/DELETE on the child table checks the parent table for referential integrity. Each check involves a lookup on the parent's index. For high-traffic tables (e.g., logging millions of rows/second), this overhead can be significant.

```sql
-- Without FK: fast inserts
CREATE TABLE audit_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,       -- No FK constraint
    action VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- With FK: each insert checks users(id) for existence
CREATE TABLE audit_log_fk (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    action VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)  -- Overhead on every insert
);
```

**Follow-up**: "When would you skip foreign keys?" — High-throughput event/audit logging, data warehouse loads where integrity is enforced application-side, or batch import jobs where FK checks slow things down significantly.

### CHECK Constraints

MySQL 8.0.16+ enforces `CHECK` constraints:

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    salary DECIMAL(10,2),
    age INT,
    CHECK (salary > 0),
    CHECK (age >= 18 AND age <= 120)
);

-- Named constraint
CREATE TABLE products (
    id INT PRIMARY KEY,
    price DECIMAL(10,2) CONSTRAINT positive_price CHECK (price > 0)
);

-- Constraint name shows in errors
INSERT INTO products VALUES (1, -10);
-- ERROR: Check constraint 'positive_price' is violated
```

### Trap: NOT NULL with No Default on ALTER

**Trap**: Adding a NOT NULL column without a default to a table that already has rows.

```sql
-- Fails on a table with existing rows
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
-- ERROR 1364: Field 'phone' doesn't have a default value

-- Correct: add with a default first, then modify
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL DEFAULT '';
-- Or: add as nullable, fill data, then make NOT NULL
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
UPDATE users SET phone = '';
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;
```

---

## 6. Querying with MySQL

### SELECT Basics

```sql
SELECT * FROM users;
SELECT id, email, name FROM users WHERE status = 'active' ORDER BY created_at DESC LIMIT 10;
SELECT COUNT(*) FROM users;
SELECT DISTINCT status FROM users;
```

### JOIN Types

```sql
-- INNER JOIN
SELECT u.name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id;

-- LEFT JOIN
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;

-- RIGHT JOIN (rare — usually rewritten as LEFT JOIN)
SELECT u.name, o.total
FROM orders o
RIGHT JOIN users u ON o.user_id = u.id;  -- Returns all users even without orders

-- FULL OUTER JOIN via UNION (MySQL lacks FULL JOIN)
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
UNION
SELECT u.name, o.total
FROM users u
RIGHT JOIN orders o ON o.user_id = u.id;
```

### Subqueries

```sql
-- Uncorrelated subquery
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- Correlated subquery (evaluated per row)
SELECT u.*,
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;

-- EXISTS (often faster than IN for correlated subqueries)
SELECT u.*
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 100);

-- Derived table (subquery in FROM)
SELECT dt.status, COUNT(*)
FROM (
    SELECT id, CASE WHEN total > 100 THEN 'high' ELSE 'low' END AS status
    FROM orders
) dt
GROUP BY dt.status;
```

**MySQL 8.0 optimizer improvements**: Subquery materialization, semi-join transformations, and better EXISTS/IN optimizations. MySQL 5.7 already improved significantly over 5.6/5.5.

### Trap: Subquery Performance in MySQL 5.7+

**Trap**: Assuming subqueries are always slower than joins in MySQL.

MySQL 5.7+ and 8.0 optimize many subqueries into semi-joins automatically. Test with EXPLAIN rather than assuming.

```sql
-- These two may have the same execution plan in MySQL 8.0
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);
SELECT DISTINCT u.* FROM users u JOIN orders o ON o.user_id = u.id;
```

### Common Table Expressions (CTE)

MySQL 8.0+ supports CTEs with `WITH`:

```sql
-- Simple CTE
WITH active_users AS (
    SELECT id, email, name
    FROM users
    WHERE status = 'active' AND last_login > '2024-01-01'
)
SELECT * FROM active_users;

-- Recursive CTE (e.g., hierarchy)
WITH RECURSIVE org_tree AS (
    SELECT id, name, parent_id, 1 AS depth
    FROM employees
    WHERE parent_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.parent_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON ot.id = e.parent_id
)
SELECT * FROM org_tree;
```

### Window Functions

MySQL 8.0+ supports window functions (similar to PostgreSQL):

```sql
-- ROW_NUMBER
SELECT name, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank
FROM employees;

-- RANK vs DENSE_RANK
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
-- RANK: 1, 2, 2, 4 (skips 3)
-- DENSE_RANK: 1, 2, 2, 3 (no gaps)

-- LAG / LEAD
SELECT created_at,
       LAG(created_at) OVER (ORDER BY created_at) AS prev_order,
       TIMESTAMPDIFF(SECOND, LAG(created_at) OVER (ORDER BY created_at), created_at) AS seconds_since_last
FROM orders
WHERE user_id = 42;

-- Aggregation with OVER
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;

-- Window frame (running total)
SELECT created_at, total,
       SUM(total) OVER (ORDER BY created_at ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;
```

### UNION vs UNION ALL

```sql
-- UNION: deduplicates (slower, sorts internally)
SELECT name FROM active_users
UNION
SELECT name FROM archived_users;

-- UNION ALL: no dedup (faster)
SELECT name FROM active_users
UNION ALL
SELECT name FROM archived_users;
```

### GROUP BY with WITH ROLLUP

```sql
-- WITH ROLLUP adds subtotal and grand total rows
SELECT status, YEAR(created_at) AS year, COUNT(*) AS count
FROM orders
GROUP BY status, YEAR(created_at) WITH ROLLUP;
-- Returns: per-status per-year, per-status total, grand total
```

### HAVING

```sql
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING order_count > 10;
```

### LIMIT with OFFSET

```sql
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 0;  -- Page 1
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 10; -- Page 2
```

### Trap: LIMIT Without ORDER BY

**Trap**: Using `LIMIT` without `ORDER BY` — result set is non-deterministic.

```sql
-- Which 10 rows do you get? Unpredictable.
SELECT * FROM users LIMIT 10;

-- Always add ORDER BY
SELECT * FROM users ORDER BY id LIMIT 10;
```

**Follow-up**: "Why is LIMIT without ORDER BY non-deterministic?" — Because MySQL returns rows in whatever order the storage engine provides them, which can change based on the execution plan, index used, or data distribution. Without ORDER BY, the set is undefined.

### Trap: Large OFFSET Performance

`LIMIT 1000000, 10` still reads all previous 1,000,000 rows and discards them.

```sql
-- Bad: reads 1,000,010 rows, discards 1,000,000
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 1000000;

-- Better: keyset pagination (remember last id)
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;

-- Alternative: seek method
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-01', 1000000)
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

---

## 7. INSERT, UPDATE, DELETE Patterns

### INSERT

```sql
-- Standard INSERT
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice');

-- Multiple rows
INSERT INTO users (email, name) VALUES
    ('bob@example.com', 'Bob'),
    ('charlie@example.com', 'Charlie');

-- INSERT ... SET (MySQL-specific syntax)
INSERT INTO users SET email = 'dave@example.com', name = 'Dave';

-- INSERT ... SELECT
INSERT INTO user_backup (id, email, name)
SELECT id, email, name FROM users WHERE deleted_at IS NOT NULL;
```

#### INSERT IGNORE

Silently ignores errors (duplicate key, constraint violations):

```sql
INSERT IGNORE INTO users (id, email) VALUES (1, 'exists@example.com');
-- If id=1 exists, silently skipped (warning generated, not error)
```

Use with caution — it hides legitimate errors. Use `ON DUPLICATE KEY UPDATE` when you want upsert behavior explicitly.

#### INSERT ... ON DUPLICATE KEY UPDATE (UPSERT)

```sql
INSERT INTO users (id, email, name) VALUES (1, 'alice@example.com', 'Alice')
ON DUPLICATE KEY UPDATE
    email = 'alice@example.com',
    name = 'Alice',
    updated_at = NOW();

-- With VALUES() for reference (deprecated in MySQL 8.0.20+)
INSERT INTO users (id, email, name) VALUES (1, 'alice@example.com', 'Alice')
ON DUPLICATE KEY UPDATE
    name = VALUES(name);  -- Deprecated, use aliases instead

-- Modern syntax (MySQL 8.0.20+)
INSERT INTO users (id, email, name) VALUES (1, 'alice@example.com', 'Alice') AS new
ON DUPLICATE KEY UPDATE
    name = new.name,
    updated_at = NOW();
```

#### REPLACE

DELETE + INSERT (not true UPSERT):

```sql
REPLACE INTO users (id, email, name) VALUES (1, 'alice@example.com', 'Alice');
-- If id=1 exists: DELETEs row, then INSERTs new row
```

### Trap: REPLACE vs ON DUPLICATE KEY UPDATE

**Trap**: Thinking `REPLACE` is the same as `INSERT ... ON DUPLICATE KEY UPDATE`.

| | `REPLACE` | `ON DUPLICATE KEY UPDATE` |
|---|---|---|
| Action if exists | DELETE + INSERT (two operations) | UPDATE (single operation) |
| Auto-increment | Increments (even if DELETE+INSERT same id) | Increments only on INSERT |
| Triggers | Fires DELETE + INSERT triggers | Fires UPDATE trigger (or INSERT on new) |
| Foreign keys | Can fail if child rows reference the PK | Updates in-place |
| Cost | Higher (delete + insert + index maintenance) | Lower (update only) |

```sql
-- REPLACE increments auto_increment even for existing rows
CREATE TABLE t (id INT AUTO_INCREMENT PRIMARY KEY, val INT);
INSERT INTO t (val) VALUES (1);  -- id=1
REPLACE INTO t (id, val) VALUES (1, 2);
-- id becomes 2! Because: DELETE id=1, INSERT id=2

-- ON DUPLICATE KEY UPDATE preserves auto_increment
INSERT INTO t (id, val) VALUES (1, 2) ON DUPLICATE KEY UPDATE val = 2;
-- id stays 1
```

### UPDATE

```sql
-- Single table
UPDATE users SET name = 'Alice Smith' WHERE id = 1;

-- Multi-table UPDATE (MySQL-specific syntax)
UPDATE orders o
JOIN users u ON o.user_id = u.id
SET o.status = 'cancelled'
WHERE u.status = 'suspended';

-- With ORDER BY and LIMIT
UPDATE queue SET picked = 1
WHERE id IN (
    SELECT id FROM queue WHERE picked = 0 ORDER BY priority LIMIT 10
);
```

### DELETE

```sql
-- Single table
DELETE FROM users WHERE id = 1;

-- Multi-table DELETE
DELETE u, o
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.status = 'deleted';
-- Deletes user row AND matching order rows

-- With ORDER BY and LIMIT (useful for batch deletes)
DELETE FROM logs WHERE created_at < '2023-01-01' ORDER BY id LIMIT 1000;
```

### TRUNCATE

DDL operation — removes all rows, resets auto-increment counter.

```sql
TRUNCATE TABLE temp_logs;
-- Drops and recreates table (DDL) unlike DELETE (DML)
```

| | `DELETE` | `TRUNCATE` |
|---|---|---|
| Type | DML | DDL |
| Rollback | Yes (if within transaction) | No (in most cases) |
| Auto-increment | Does not reset | Resets to 0 |
| Triggers | Fires per row | Does not fire |
| Speed | Slow (log per row) | Very fast (deallocate pages) |
| WHERE | Supported | Not supported |

### LOAD DATA INFILE

Bulk import — much faster than INSERT:

```sql
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(email, name, @created_at)
SET created_at = STR_TO_DATE(@created_at, '%Y-%m-%d');
```

### Trap: LOAD DATA INFILE Bypasses

**Trap**: `LOAD DATA INFILE` can bypass triggers and foreign key checks.

```sql
-- This can bypass triggers and FK constraints
LOAD DATA INFILE '/tmp/data.csv' INTO TABLE orders
FIELDS TERMINATED BY ',';

-- To be safe, use INSERT ... SELECT instead if you need triggers
-- Or re-enable FK checks manually after (not recommended: may corrupt data)
SET FOREIGN_KEY_CHECKS = 0;
LOAD DATA INFILE '/tmp/data.csv' INTO TABLE orders;
SET FOREIGN_KEY_CHECKS = 1;
```

---

## 8. MySQL vs PostgreSQL — Key Differences

### Comparison Table

| Feature | MySQL | PostgreSQL |
|---|---|---|
| **Storage engine** | Pluggable (InnoDB default) | Integrated (heap-based MVCC) |
| **Connection model** | Thread-per-connection | Process-per-connection |
| **Default isolation** | REPEATABLE READ | READ COMMITTED |
| **MVCC** | Undo log (old versions in undo tablespace) | Old tuples in heap (dead tuples) |
| **Vacuum** | No vacuum — purge process (automatic, less overhead) | Autovacuum required (can cause bloat if misconfigured) |
| **Clustered index** | Yes (PK is clustered) | No (heap-based) |
| **Index types** | B+Tree, Fulltext, Spatial (R-Tree), Hash (Memory engine only) | B+Tree, GiST, GIN, BRIN, SP-GiST, Hash |
| **Partial indexes** | No (MySQL 8.0+) | Yes (`CREATE INDEX ... WHERE condition`) |
| **Concurrent indexes** | INPLACE index creation (non-blocking) | `CREATE INDEX CONCURRENTLY` |
| **JSON** | JSON type (binary, optimized) | JSONB (decomposed binary) |
| **Replication** | Async, semi-sync, Group Replication (InnoDB Cluster) | Streaming WAL, physical + logical replication |
| **Full-text search** | Built-in (InnoDB, MyISAM) | Built-in (tsvector/tsquery, more powerful) |
| **GEO/PostGIS** | Spatial (limited) | PostGIS (industry standard) |
| **Extensions** | Plugins (limited ecosystem) | Rich extension ecosystem |
| **CHECK constraints** | Enforced since 8.0.16 | Fully supported |
| **Generated columns** | VIRTUAL / STORED | VIRTUAL / STORED (similar) |
| **LIMIT/OFFSET** | Supported | Supported |
| **GROUP BY** | Non-standard by default (set `ONLY_FULL_GROUP_BY`) | Standard (all non-aggregated cols must be in GROUP BY) |
| **Administration** | `mysql` system database, `SHOW` commands | System catalogs (`pg_catalog`), `\d` commands |
| **Backup** | `mysqldump`, binary log for PITR | `pg_dump`, WAL archiving for PITR |

### Detailed Differences

#### MVCC Implementation

**MySQL (InnoDB)**:
- Old row versions stored in **undo log**
- Current row version is always in the clustered index
- Purge process (automatic, low overhead) cleans old versions
- No vacuum needed

**PostgreSQL**:
- Old row versions stored in **heap** as dead tuples
- VACUUM must reclaim dead tuple space (autovacuum)
- Table bloat if VACUUM can't keep up
- HOT (Heap-Only Tuples) updates help with bloat when no index columns change

#### Locking

**MySQL (InnoDB)**:
- Row-level locks + gap locks (in REPEATABLE READ)
- Gap locks prevent phantoms (records appearing in range scans)
- Can cause deadlocks more frequently than PG
- `SHOW ENGINE INNODB STATUS\G` for lock diagnostics

**PostgreSQL**:
- Row-level locks (no gap locks)
- Snapshot isolation prevents phantoms without gap locks
- Lower deadlock rate
- `pg_locks` for lock diagnostics

#### Full-Text Search

```sql
-- MySQL full-text
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('database' IN NATURAL LANGUAGE MODE);
```

```sql
-- PostgreSQL full-text
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'database');
```

PostgreSQL's full-text is more feature-rich (ranking, dictionaries, thesauri, stemming, multiple languages). MySQL's is adequate for basic search.

### Trap: Assuming MySQL and PostgreSQL Are Interchangeable

**Trap**: Porting queries between MySQL and PostgreSQL without testing.

Critical differences:
- **GROUP BY** behavior differs (MySQL may accept non-standard queries)
- **LIMIT/OFFSET** syntax is the same, but optimization differs
- **Indexing** strategies differ (partial indexes, covering indexes, index types)
- **Replication** architecture is fundamentally different
- **Memory management** differs (buffer pool vs shared_buffers + OS cache)
- **ALTER TABLE** locking behavior differs significantly (MySQL can block; PG uses MVCC for DDL)

```sql
-- This works in MySQL but NOT in PostgreSQL
SELECT name, email, COUNT(*) FROM users GROUP BY name;
-- PostgreSQL: ERROR — email must be in GROUP BY or aggregate

-- PostgreSQL alternative
SELECT name, ANY_VALUE(email), COUNT(*) FROM users GROUP BY name;
-- Note: MySQL 8.0+ also has ANY_VALUE()
```

**Follow-up**: "When is MySQL faster than PostgreSQL?" — Simple read queries with good indexes, especially with InnoDB's clustered index for PK lookups. High-volume INSERT workloads with simple schema (InnoDB's change buffer helps). Read-replica scaling (MySQL has more mature read-replica tooling for many years).

**Follow-up**: "When is PostgreSQL faster?" — Complex queries with many joins (PG's optimizer is more sophisticated), analytical queries (parallel query execution, BRIN indexes), workloads requiring concurrent index creation, geospatial queries (PostGIS), workloads needing partial indexes.

---

## 9. Tier 1 Q&A Drill

### Q1: What's the difference between MySQL's thread-per-connection and PostgreSQL's process-per-connection?

**A**: MySQL uses a thread-per-connection model — each client gets a lightweight thread within the mysqld process. This uses less memory per connection (~256 KB vs ~5-10 MB per PG process) but a crash in one thread can affect others since they share address space. PostgreSQL uses separate OS processes, providing better isolation (one crash kills one connection) at higher memory overhead. MySQL 8.0 Enterprise offers a Thread Pool plugin to reuse threads for many connections.

### Q2: Explain how InnoDB's clustered index works and why it matters.

**A**: InnoDB stores row data in the leaf nodes of the primary key's B+Tree. This means:
- PK lookups are extremely fast (single tree traversal directly to the row data)
- Rows are physically ordered by PK order (inserts near the end for auto-increment PKs)
- Every secondary index stores the PK value as a row pointer (requires two lookups: secondary index → PK → clustered index)
- A wide PK bloats ALL secondary indexes
- **Best practice**: Use a small, monotonically increasing PK (INT/BIGINT AUTO_INCREMENT)

### Q3: What happens if you create a table without a primary key in InnoDB?

**A**: InnoDB will:
1. Look for a NOT NULL unique key to use as the clustered index
2. If none found, add a hidden 6-byte `ROW_ID` as an auto-incrementing clustered index (global counter across all tables without PKs)
3. This ROW_ID is invisible and not queryable
4. Every secondary index references this hidden ROW_ID
5. Replication (row-based) performs poorly for UPDATE/DELETE without PK
6. Always add an explicit primary key — every table should have one

### Q4: Compare MySQL's utf8mb4 with PostgreSQL's UTF-8 support.

**A**: MySQL's `utf8` (utf8mb3) only supports 3-byte UTF-8, meaning emoji (which need 4 bytes) will be truncated. Always use `utf8mb4` for full Unicode support. PostgreSQL's UTF-8 always supports 4-byte characters. In MySQL, charset and collation are set at server, database, table, and column levels — providing granular control but also complexity. PostgreSQL uses database-wide encoding (usually UTF-8) and per-column collations.

### Q5: What is the TIMESTAMP 2038 problem and how do you avoid it?

**A**: MySQL's `TIMESTAMP` type stores 4 bytes internally (Unix epoch), limiting its range to `1970-01-01` to `2038-01-19 03:14:07 UTC`. After that date, TIMESTAMP values overflow. To avoid:
- Use `DATETIME` (5-8 bytes, supports up to year 9999) for dates beyond 2038
- Alternative: store as `BIGINT` with Unix epoch in milliseconds
- Note: TIMESTAMP auto-converts to session timezone; DATETIME does not — consider this when choosing

### Q6: Explain the MySQL query execution pipeline.

**A**: The pipeline: **Parser** (tokenize SQL, build parse tree) → **Preprocessor** (resolve tables/columns, validate grants) → **Optimizer** (generate execution plans, evaluate costs, choose indexes, apply semi-join/materialization transformations) → **Executor** (execute the plan, call storage engine API) → **Storage Engine API** (abstract handler interface for InnoDB, MyISAM, etc.). Use `EXPLAIN FORMAT=JSON` and `OPTIMIZER_TRACE` to understand optimizer decisions.

### Q7: What's the difference between CHAR, VARCHAR, and TEXT in MySQL?

| Type | Storage | Max Length | Behavior |
|---|---|---|---|
| `CHAR(N)` | Fixed, N × bytes per char | 255 chars | Padded with spaces, good for fixed codes |
| `VARCHAR(N)` | Variable, 1-2 byte prefix + data | 65,535 chars | Most common for strings |
| `TEXT` | Variable, off-page for large values | 65,535 bytes | No inline length prefix, needed for > 65k chars |

Key: `VARCHAR(N)` caps at 65,535 **characters** (not bytes, depends on charset). `TEXT` is stored off-page for large values (row format dependent). PostgreSQL's VARCHAR has no practical limit (TOAST handles large values).

### Q8: How does MySQL's REPEATABLE READ isolation level handle phantom reads compared to PostgreSQL?

**A**: InnoDB's REPEATABLE READ prevents phantom reads using **gap locks** (locking gaps between index records). This prevents new rows from being inserted into a range being read. However, gap locks increase deadlock probability. PostgreSQL's REPEATABLE READ uses **snapshot isolation** — each transaction sees a consistent snapshot of the database at the start of the transaction. Phantoms are prevented by versioning, not locking. PostgreSQL's approach has lower contention and fewer deadlocks but uses more storage for old row versions.

### Q9: What are the key differences between REPLACE and INSERT ... ON DUPLICATE KEY UPDATE?

**A**: `REPLACE` does DELETE + INSERT when a duplicate key is found — it increments auto-increment ID, fires DELETE and INSERT triggers, and may fail if foreign key constraints reference the row. `INSERT ... ON DUPLICATE KEY UPDATE` performs an UPDATE in-place — preserves the auto-increment value, fires UPDATE trigger, and respects foreign keys. Use ON DUPLICATE KEY UPDATE for true upsert; REPLACE is usually wrong for production use.

### Q10: What is the doublewrite buffer and when can you disable it?

**A**: The doublewrite buffer protects against partial page writes (when a 16 KB page is only partially written during a crash, leaving it corrupt). InnoDB writes pages to the doublewrite buffer before writing to the actual data file. You can disable it (`innodb_doublewrite=0`) on systems with atomic write sizes at the storage layer, such as ZFS or some SSDs with 16 KB+ atomic write capabilities. Disabling it on traditional storage risks data corruption on crashes.

---

> **Next**: [Tier 2 — Performance](/topics/mysql/02-performance.md)
