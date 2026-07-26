# MySQL — Tier 3: Senior (High Availability, Performance & Operations)

This guide covers MySQL-specific operational patterns for senior engineers coming from a PostgreSQL background. PostgreSQL and MySQL share much SQL syntax, but their operational characteristics — replication, clustering, storage engines, backup tooling, and performance tuning — differ significantly. This reference targets the MySQL 8.0 feature set.

---

## Table of Contents

1. [High Availability Architectures](#1-high-availability-architectures)
2. [Sharding](#2-sharding)
3. [Partitioning](#3-partitioning)
4. [InnoDB Performance Tuning](#4-innodb-performance-tuning)
5. [Schema Migrations at Scale](#5-schema-migrations-at-scale)
6. [Backup & Recovery](#6-backup--recovery)
7. [Performance Troubleshooting Runbook](#7-performance-troubleshooting-runbook)
8. [Native MySQL Features in 8.0+](#8-native-mysql-features-in-80)
9. [Tools Ecosystem](#9-tools-ecosystem)
10. [MySQL 8.0 vs 5.7 Migration](#10-mysql-80-vs-57-migration)
11. [MySQL in the Cloud](#11-mysql-in-the-cloud)
12. [Tier 3 Q&A Drill](#12-tier-3-qa-drill)

---

## 1. High Availability Architectures

PostgreSQL's HA story revolves around streaming replication, `pg_rewind`, Patroni/etcd, and `pg_auto_failover`. MySQL has multiple competing HA approaches with different trade-offs.

### Master-Slave with Manual Failover

**MHA (Master High Availability)**

MHA is a Perl-based tool that monitors the master. On failure it promotes the best slave (based on binlog position) and re-points remaining slaves. It requires:

- SSH access between all nodes
- Semi-synchronous or asynchronous replication
- A candidate slave for promotion

MHA workflow: health check → master dead → identify latest slave → apply relay logs → promote → repoint slaves → notify.

```sql
-- Check binlog positions for promotion ordering (MHA does this automatically)
SHOW MASTER STATUS;
SHOW SLAVE STATUS\G
```

**Orchestrator**

Orchestrator (GitHub) manages MySQL replication topologies. It provides:

- Topology discovery and visualization (web UI)
- Automated failover with hooks
- Recovery validation
- `orchestrator-client` CLI for manual interventions

Orchestrator stores topology state in its own backend (MySQL or SQLite). It can recover from master failure, fix replication issues, and adjust topology.

> **Trap:** MHA requires SSH key-based authentication between all nodes — if SSH breaks, failover fails. Orchestrator can split-brain without proper `RaftConsistent` configuration or a shared backend; use the Raft consensus mode in production. Group Replication demands strict network latency (< 5ms recommended); higher latency causes performance collapse.

> **Follow-up:** *How would you choose between MHA and Orchestrator?* — MHA is simpler for basic failover, Orchestrator provides richer topology management and a recovery API. In modern deployments, InnoDB Cluster is preferred for new builds.

### InnoDB Cluster

InnoDB Cluster = Group Replication + MySQL Router + MySQL Shell.

**Group Replication** (built-in since 5.7.17) provides multi-master or single-master replication with automatic conflict detection. It uses a distributed consensus protocol (Paxos-like) via a group communication system (XCom).

- **Single-primary mode:** one node accepts writes; others are read-only. On primary failure, a new primary is automatically elected.
- **Multi-primary mode:** all nodes accept writes. Conflicts are detected at commit time — the transaction that commits first wins, others roll back.

```sql
-- Check Group Replication members
SELECT * FROM performance_schema.replication_group_members;

-- Check group replication status
SELECT * FROM performance_schema.replication_group_member_stats;
```

**MySQL Router** sits between applications and cluster nodes. It provides:
- Automatic routing: read-write to primary, read-only to secondaries
- Caching of metadata from Group Replication
- Port-based routing (e.g., 6446 for RW, 6447 for RO)

**MySQL Shell** provides admin API:
```js
// MySQL Shell JavaScript/Python API
var cluster = dba.createCluster('myCluster');
cluster.addInstance('user@host2:3306');
cluster.status();
```

> **Trap:** Multi-primary Group Replication has a high conflict rate under write-heavy workloads — prefer single-primary. All Group Replication traffic must be on a trusted network (no encryption by default, no built-in firewall integration). Network partitions can break the group's quorum. Cross-AZ writes add latency proportional to the round trip.

> **Follow-up:** *Why would you pick InnoDB Cluster over Patroni for MySQL?* — InnoDB Cluster is natively integrated into MySQL (no external DCS like etcd required), but Patroni is more battle-tested for large-scale deployments and supports async replication topologies without strict latency requirements.

### Galera Cluster / Percona XtraDB Cluster (PXC)

Galera implements synchronous virtually-synchronous replication (SST/IST). PXC bundles Galera wsrep provider with Percona Server.

- **Synchronous certification:** all nodes must agree a transaction can commit. No data loss on failover.
- **Write throughput degrades** with node count (every node certifies every write).
- **Flow control:** if a node falls behind, the cluster throttles writes.
- **State transfer:** SST (Snapshot State Transfer via xtrabackup/rsync/mysqldump) for new nodes; IST (Incremental State Transfer) for lagging nodes.

```sql
-- Check wsrep status
SHOW STATUS LIKE 'wsrep_%';

-- Cluster size
SHOW STATUS LIKE 'wsrep_cluster_size';

-- Flow control paused
SHOW STATUS LIKE 'wsrep_flow_control_paused';
```

> **Trap:** Galera requires all tables to have a PRIMARY KEY (otherwise replication breaks). Write throughput is bounded by the slowest node. Network latency directly impacts write latency (certification phase blocks). SST can saturate the network and cause timeouts.

> **Follow-up:** *When would you use Galera over InnoDB Cluster?* — Galera for synchronous multi-master with zero data loss guarantee (finance, telco). InnoDB Cluster for simpler topology and MySQL-native integration.

### NDB Cluster

NDB Cluster uses a shared-nothing architecture with in-memory storage. Data is automatically partitioned across data nodes. It provides:

- High write throughput (data node partitioning)
- 99.999% uptime design goals
- In-memory by default (disk-based optional)

| Feature | InnoDB Cluster | Galera/PXC | NDB Cluster |
|---------|---------------|------------|-------------|
| Replication | Semi-sync (Group Replication) | Synchronous (virtually) | Synchronous with 2PC |
| Write scaling | Primary limits writes | All nodes certifying | Partitioned writes |
| Node count limit | 9 (GR limit) | 8-16 practical | 48+ data nodes |
| Latency sensitivity | Very high (<5ms) | High | Medium |
| Storage | Disk (InnoDB) | Disk (InnoDB/XtraDB) | In-memory |

> **Follow-up:** *Has NDB Cluster seen wide adoption?* — It's niche, used by telecoms (high availability, low latency) and some financial systems. Most web applications use InnoDB-based solutions.

---

## 2. Sharding

PostgreSQL sharding typically involves `postgres_fdw`, Citus (now Azure Cosmos DB), or application-level partitioning. MySQL sharding is more fragmented.

### Why Shard?

- Data volume exceeds single server capacity (multi-TB)
- Write throughput exceeds single server IOPS/CPU capacity
- Geographic distribution requirements

### Vertical vs Horizontal

**Vertical sharding:** split by domain (users DB, orders DB, inventory DB). This is the simplest form but doesn't solve single-table growth.

**Horizontal sharding:** split the same table across servers by a shard key. Each shard holds a subset of rows.

### Shard Key Selection

A good shard key has:
- **High cardinality** (many distinct values)
- **Even distribution** (no hot shards)
- **Query alignment** (most queries filter on the shard key)

```sql
-- Bad: shard by country
-- Most data goes to US/China shards → hotspots
-- Good: shard by user_id (high cardinality, even distribution)
-- Shard = abs(crc32(user_id)) % number_of_shards
```

### Sharding Solutions

**Vitess** (YouTube-originated, now CNCF graduated):
- `VTGate`: proxy layer that parses queries and routes to correct shards
- `VTTablet`: per-shard agent managing MySQL instances
- Topology service (etcd/Consul/ZooKeeper)
- Transparent sharding: application doesn't know about shards
- VSchema defines how tables are sharded

```
Application → VTGate → VTTablet → MySQL (shard-0)
                      → VTTablet → MySQL (shard-1)
                      → VTTablet → MySQL (shard-2)
```

**ProxySQL:**
- Query routing, read/write splitting, connection pooling
- NOT a sharding solution — routes to backends but doesn't split data
- Can be used as a frontend to sharded clusters with multiple hostgroups

**Application-level sharding:**
- Routing logic in application code or ORM
- Simple to understand, harder to maintain
- Consistent hashing for resharding

```php
// Application-level shard routing
function getShardConnection(int $userId): PDO {
    $shardId = $userId % $config['shard_count'];
    return new PDO($config['shards'][$shardId]['dsn']);
}
```

### Sharding Challenges

| Challenge | Mitigation |
|-----------|-----------|
| Cross-shard joins | Avoid — denormalize or fan-out queries to all shards |
| Distributed transactions | 2PC is slow. Design for single-shard transactions |
| Global auto-increment | Use UUID, Snowflake, or Vitess sequence tables |
| Resharding | Consistent hashing, Vitess `Reshard` workflow |
| Backup and restore | Backup each shard independently; test cluster restore |

```sql
-- Vitess sequence table pattern (if not using UUIDs)
CREATE TABLE user_seq (
    id INT NOT NULL,
    next_id BIGINT NOT NULL,
    cache BIGINT NOT NULL,
    PRIMARY KEY (id)
) ENGINE = InnoDB;
```

> **Trap:** Choosing a bad shard key creates hotspots that kill performance. Sharding adds significant complexity — always try read replicas, better indexing, connection pooling, and caching first. Cross-shard queries are slow and complex; design queries to touch a single shard. Resharding without downtime is very hard — use consistent hashing or Vitess's built-in reshard.

> **Follow-up:** *How would you choose between Vitess and application-level sharding?* — Vitess for transparent sharding with resharding support at scale (50+ nodes). Application-level for simpler setups (under 10 shards) where team can own the routing logic.

---

## 3. Partitioning

MySQL partitioning is an **InnoDB table-level** feature (not to be confused with sharding). A single partitioned table maps to multiple physical `.ibd` files.

### Partition Types

```sql
-- RANGE: partition by contiguous ranges
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    created_at DATE NOT NULL,
    data TEXT
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2019 VALUES LESS THAN (2020),
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- LIST: partition by discrete values
CREATE TABLE orders_by_status (
    id INT NOT NULL,
    status ENUM('pending','shipped','delivered','cancelled') NOT NULL
)
PARTITION BY LIST (status) (
    PARTITION p_pending VALUES IN ('pending'),
    PARTITION p_active VALUES IN ('shipped','delivered'),
    PARTITION p_cancelled VALUES IN ('cancelled')
);

-- HASH: hash function on a column
CREATE TABLE logs (
    id INT NOT NULL,
    created_at DATETIME NOT NULL
)
PARTITION BY HASH (id) PARTITIONS 8;

-- KEY: MySQL's internal hash function
CREATE TABLE sessions (
    session_id CHAR(32) NOT NULL,
    data TEXT
)
PARTITION BY KEY (session_id) PARTITIONS 16;

-- RANGE COLUMNS: range based on multiple columns
CREATE TABLE metrics (
    id INT NOT NULL,
    server_id INT NOT NULL,
    recorded_at DATETIME NOT NULL
)
PARTITION BY RANGE COLUMNS (server_id, recorded_at) (
    PARTITION p1 VALUES LESS THAN (100, '2025-01-01'),
    PARTITION p2 VALUES LESS THAN (200, '2025-01-01')
);
```

### Partition Pruning

MySQL scans only relevant partitions when the WHERE clause filters on the partition key.

```sql
-- Only scans p2021 and p2022 (prunes other partitions)
SELECT * FROM orders WHERE created_at BETWEEN '2021-06-01' AND '2022-06-01';
```

Check whether pruning is working:
```sql
EXPLAIN SELECT * FROM orders WHERE created_at = '2021-06-01';
-- Look for partitions: p2021 in EXPLAIN output
```

### Partition Management

```sql
-- ADD PARTITION
ALTER TABLE orders ADD PARTITION (
    PARTITION p2023 VALUES LESS THAN (2024)
);

-- DROP PARTITION (instant — removes partition + data)
ALTER TABLE orders DROP PARTITION p2019;

-- TRUNCATE PARTITION
ALTER TABLE orders TRUNCATE PARTITION p2020;

-- REORGANIZE PARTITION (split or merge)
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- EXCHANGE PARTITION (instant partition swap)
CREATE TABLE orders_archive LIKE orders;
ALTER TABLE orders EXCHANGE PARTITION p2019 WITH TABLE orders_archive;
```

### Partitioning vs Sharding

| Aspect | Partitioning | Sharding |
|--------|-------------|----------|
| Scope | Single server | Multiple servers |
| Complexity | Low (built-in) | High (application/DBA complexity) |
| Scaling | Does NOT scale writes beyond one server | Scales reads AND writes |
| Cross-partition queries | Possible within the table | Cross-shard queries are expensive |
| Management | MySQL handles transparently | Manual or via Vitess/ProxySQL |

### Limitations

- **Unique key must include all partition columns** (unless using KEY partitioning)
- **Foreign keys are not supported** on partitioned tables
- **No full-text indexes** on partitioned tables
- **ALTER TABLE on partitioned tables** can be expensive (rebuilds all partitions)
- **Partition count matters:** > 50 partitions degrades performance (metadata overhead, too many open files)

```sql
-- This FAILS: unique key doesn't include partition column
CREATE TABLE users (
    id INT NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at DATE NOT NULL,
    UNIQUE KEY (email)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022)
);

-- Correct: include partition column in unique key
CREATE TABLE users (
    id INT NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at DATE NOT NULL,
    UNIQUE KEY (email, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022)
);
```

> **Trap:** Partitioning without partition pruning is worse than no partitioning — queries that don't filter on the partition key scan ALL partitions (more `.ibd` files to open = slower). Unique constraints on non-partitioned columns are impossible (all unique keys must include partitioning columns). EXCHANGE PARTITION doesn't validate data compatibility — you must have a CHECK constraint on the table or validate manually. > 50 partitions can actually hurt performance.

> **Follow-up:** *When would you use partitioning in MySQL vs PostgreSQL declarative partitioning?* — MySQL and PG partitioning are conceptually similar. MySQL lacks foreign key support on partitioned tables (PG supports it as of PG12). MySQL subpartitioning is limited (only HASH/KEY subpartitions of RANGE/LIST). Both support pruning similarly.

---

## 4. InnoDB Performance Tuning

PostgreSQL tuners focus on `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size`, `wal_buffers`, and `checkpoint_completion_target`. InnoDB has a different set of levers.

### Buffer Pool (`innodb_buffer_pool_size`)

The buffer pool caches data pages and index pages in memory. PostgreSQL equivalent: `shared_buffers`.

- **Setting:** 70-80% of available RAM on a dedicated MySQL server.
- **Oversized:** swap, OOM killer, degraded performance.
- **Undersized:** high disk I/O, more page evictions.

```sql
-- Current buffer pool size
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Buffer pool hit ratio (should be > 99%)
SELECT (1 - (
    SELECT SUM(innodb_buffer_pool_reads)
    FROM performance_schema.global_status
    WHERE variable_name = 'Innodb_buffer_pool_reads'
) / (
    SELECT SUM(innodb_buffer_pool_read_requests)
    FROM performance_schema.global_status
    WHERE variable_name = 'Innodb_buffer_pool_read_requests'
)) * 100 AS buffer_pool_hit_ratio;
```

**Multiple buffer pool instances** (`innodb_buffer_pool_instances`): reduces contention when the buffer pool is large. Default: 1 for < 1GB, 8 for 8GB+. Set to number of CPU cores typically.

**Buffer pool warmup:** after restart, the buffer pool is cold. Warm it up:
```sql
-- Manual warmup (older approach)
SELECT COUNT(*) FROM large_table;

-- 8.0 autowarmup
SET GLOBAL innodb_buffer_pool_dump_at_shutdown = ON;
SET GLOBAL innodb_buffer_pool_load_at_startup = ON;
```

### Redo Log (`innodb_log_file_size`)

PostgreSQL equivalent: `wal_size` / `wal_segment_size`. The redo log absorbs writes before they are flushed to tablespace files.

- **Should be large enough** to absorb at least an hour of peak writes.
- **Small log** → frequent checkpoints → disk thrashing.
- **8.0+:** `innodb_redo_log_capacity` replaces `innodb_log_file_size` (dynamic variable).

```sql
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';
-- or for 5.7
SHOW VARIABLES LIKE 'innodb_log_file_size';
```

Estimate required redo log size:
```sql
-- Monitor redo log throughput
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN (
    'Innodb_os_log_written',
    'Innodb_redo_log_bytes_used'
);
```

### Undo Log

- `innodb_undo_tablespaces`: store undo logs in separate tablespaces (reduces ibdata1 bloat). Default 2 in 8.0.
- `innodb_max_undo_log_size`: cap undo space growth (8.0+).
- `innodb_undo_log_truncate`: automatically truncate undo tablespaces (8.0+).

### Flushing Configuration

```sql
-- Safe: fsync on every commit (maximum durability)
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
SET GLOBAL sync_binlog = 1;

-- Faster: write to OS cache only (crash unsafe — loses up to 1 second of data)
SET GLOBAL innodb_flush_log_at_trx_commit = 2;

-- Compromise: full fsync on commits, batch binlog syncs
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
SET GLOBAL sync_binlog = 0;
```

| `innodb_flush_log_at_trx_commit` | Behavior | Durability | Performance |
|--------------------------------|----------|------------|-------------|
| 1 | fsync on each commit | Full ACID | Slowest |
| 2 | write to OS cache, flush every 1s | Lose 1s on crash | ~2x faster |
| 0 | write to log every 1s | Lose 1s on crash | ~2x faster |

**`innodb_io_capacity`** and **`innodb_io_capacity_max`**: limit background flushing based on your I/O subsystem capability.

- `innodb_io_capacity = 200` for HDD, `= 2000` for SSD, `= 50000+` for NVMe.

### Adaptive Hash Index

`innodb_adaptive_hash_index` is ON by default. It builds a hash index on frequently accessed pages.

- **Benefit:** faster point lookups.
- **Problem:** under high contention workloads, AHI can cause latch contention (btr_search_ latch).
- **Fix:** disable if `SHOW ENGINE INNODB STATUS` shows high rw-lock wait on `btr_search_latch`.

### Thread Concurrency

`innodb_thread_concurrency`: limits the number of concurrent threads inside InnoDB.

- `0` = no limit (default).
- Set to `2 * CPU cores` for high-concurrency systems.

### Performance Schema

MySQL 8.0 Performance Schema is stable and low-overhead (vs. 5.7 where it could add 10-20% overhead).

```sql
-- Find queries waiting on I/O
SELECT event_name, count_star, sum_timer_wait
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE event_name LIKE 'wait/io/file/innodb%'
ORDER BY sum_timer_wait DESC
LIMIT 10;
```

> **Trap:** Setting buffer pool too large causes swap and OOM killer (leave memory for OS, connections, temp tables). Redo log too small causes frequent checkpoints and performance thrashing. Using `innodb_flush_log_at_trx_commit=2` without understanding crash safety means losing up to 1 second of committed transactions. Not reserving RAM for OS (at least 2-4GB) and connection memory can lead to OOM.

> **Follow-up:** *In PostgreSQL you tune shared_buffers + effective_cache_size + work_mem. What's the MySQL equivalent strategy?* — In MySQL you tune: `innodb_buffer_pool_size` (primary cache), `innodb_log_file_size` (WAL-equivalent), `tmp_table_size`/`max_heap_table_size` (work_mem equivalent for temp tables), and `sort_buffer_size`/`join_buffer_size` per connection. Connection memory adds up fast; keep `sort_buffer_size` conservative (default 256KB).

---

## 5. Schema Migrations at Scale

This is where MySQL shows the most painful difference from PostgreSQL. PG can `ALTER TABLE` on large tables with lock-free operations (`CREATE INDEX CONCURRENTLY`, `ALTER TABLE ... ADD COLUMN ... DEFAULT (expr)` with no rewrite as of PG11+). MySQL `ALTER TABLE` on InnoDB typically copies the table lock.

### Problem: ALTER TABLE Copies the Table

In MySQL (pre-8.0), even adding a column can copy the entire table and block writes for the duration. For a 500GB table, this is hours of write blocking.

```sql
-- This blocks writes to the table for the entire copy duration (could be hours)
ALTER TABLE large_table ADD COLUMN new_col INT NOT NULL DEFAULT 0;
```

### Online Schema Change Tools

**pt-online-schema-change** (Percona Toolkit):

1. Creates a shadow table with the new schema
2. Adds triggers to the original table (AFTER INSERT, UPDATE, DELETE) to sync changes
3. Copies rows in chunks from original to shadow
4. Tables remain consistent via triggers
5. `RENAME TABLE` swap at the end

```bash
pt-online-schema-change \
  --alter "ADD COLUMN new_col INT NOT NULL DEFAULT 0" \
  D=database,t=large_table \
  --chunk-time=0.5 \
  --max-lag=5 \
  --execute
```

**gh-ost** (GitHub's triggerless online schema migration):

1. Creates a shadow table
2. Reads binary log events to track changes (no triggers)
3. Copies rows in chunks from original to shadow
4. Applies binlog events to catch up
5. Atomic cut-over

```bash
gh-ost \
  --alter="ADD COLUMN new_col INT NOT NULL DEFAULT 0" \
  --database="database" \
  --table="large_table" \
  --execute
```

### pt-osc vs gh-ost

| Feature | pt-online-schema-change | gh-ost |
|---------|------------------------|--------|
| Mechanism | Triggers on original table | Binary log stream |
| Write overhead | Trigger overhead on every write | Minimal (via slave binlog stream) |
| Controllability | Limited (pausing not clean) | Pausable, testable, throttle-able |
| Requirements | Full table access | Binary log in ROW mode, full schema copy |
| Auditability | Triggers are invisible | Dry-run mode, --test-on-replica |

```bash
# gh-ost dry run on replica first
gh-ost \
  --alter="DROP COLUMN old_col" \
  --database="database" \
  --table="large_table" \
  --test-on-replica \
  --execute
```

### MySQL 8.0 `ALGORITHM` Improvements

MySQL 8.0.29+ supports `ALGORITHM=INSTANT` for some DDL operations:

```sql
-- Instant ADD COLUMN (8.0+)
ALTER TABLE t ADD COLUMN c INT, ALGORITHM=INSTANT;

-- Constraints for INSTANT:
-- - Column must be added at the end (not FIRST, not AFTER)
-- - No DEFAULT expression (literal only)
-- - No NOT NULL without DEFAULT (would fail on existing rows)
-- - Not for PK columns
-- - Not for virtual generated columns

-- ALGORITHM=INPLACE (rebuilds table, allows concurrent DML)
ALTER TABLE t DROP COLUMN c, ALGORITHM=INPLACE, LOCK=NONE;

-- ALGORITHM=COPY (disables concurrent DML — old behavior)
ALTER TABLE t MODIFY COLUMN c VARCHAR(200), ALGORITHM=COPY;
```

| Operation | ALGORITHM | Allows concurrent DML |
|-----------|-----------|----------------------|
| ADD COLUMN (simple, last position) | INSTANT | Yes |
| ADD COLUMN (not simple) | INPLACE | Yes (LOCK=NONE) |
| ADD INDEX | INPLACE | Yes (LOCK=NONE) |
| DROP COLUMN | INPLACE | Yes (LOCK=NONE) |
| CHANGE COLUMN data type | COPY | No |
| ADD PRIMARY KEY | INPLACE | Yes |
| DROP PRIMARY KEY | COPY | No |
| MODIFY COLUMN charset | COPY | No |

> **Trap:** Never assume ALTER TABLE on MySQL is instant — even adding an index can take hours on a large table. `LOCK=NONE` doesn't mean zero impact; triggers from pt-osc add overhead, and binlog events from gh-ost consume disk space. gh-ost requires binary log in ROW mode and creates a full copy of the table — ensure enough disk space (double the table size). pt-online-schema-change triggers can cause deadlocks under high write load.

> **Follow-up:** *How does PostgreSQL handle large schema migrations differently?* — PostgreSQL can create indexes concurrently (`CREATE INDEX CONCURRENTLY`) without blocking writes. `ALTER TABLE ADD COLUMN` with a non-volatile DEFAULT is metadata-only (no table copy) as of PG11. `pg_repack` can rebuild tables with minimal locking. MySQL's online DDL tools are third-party (gh-ost, pt-osc) or only recently natively instantaneous.

---

## 6. Backup & Recovery

PostgreSQL uses `pg_dump` (logical) and `pg_basebackup` + WAL archiving (physical/PITR). MySQL's equivalents differ significantly.

### Logical Backup: mysqldump

```bash
# InnoDB consistent backup with binlog position
mysqldump \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --master-data=2 \
  --all-databases \
  > full_backup.sql
```

| Option | Purpose |
|--------|---------|
| `--single-transaction` | Consistent read (InnoDB only) — does NOT lock tables |
| `--master-data=2` | Records binlog file and position (for PITR) |
| `--routines`, `--triggers`, `--events` | Include stored procedures, triggers, events |
| `--all-databases` | All databases (mysql, sys, etc.) |

```bash
# Restore logical backup
mysql -u root -p < full_backup.sql
```

### Physical Backup: Percona XtraBackup

`pg_basebackup` directly copies WAL files. XtraBackup copies InnoDB data files at the filesystem level, then applies the redo log for consistency.

```bash
# Full backup
xtrabackup --backup --target-dir=/backups/mysql/full-$(date +%Y%m%d)

# Prepare (apply redo log to make it consistent)
xtrabackup --prepare --target-dir=/backups/mysql/full-20261201

# Incremental backup (based on LSN)
xtrabackup --backup --target-dir=/backups/mysql/inc1 \
  --incremental-basedir=/backups/mysql/full-20261201

# Prepare incremental (apply base + inc)
xtrabackup --prepare --target-dir=/backups/mysql/full-20261201 \
  --incremental-dir=/backups/mysql/inc1

# Restore (move prepared backup to datadir)
xtrabackup --copy-back --target-dir=/backups/mysql/full-20261201
```

### Point-in-Time Recovery (PITR)

PostgreSQL PITR: base backup + WAL segments + `recovery.conf`/`recovery.signal` + restore_command.
MySQL PITR: full backup + binary logs + mysqlbinlog.

```bash
# 1. Restore full XtraBackup
xtrabackup --copy-back --target-dir=/backups/mysql/full-20261201

# 2. Get binlog position from backup
cat /backups/mysql/full-20261201/xtrabackup_binlog_info
# mysql-bin.000123  456789

# 3. Apply binary logs from backup position to target position
mysqlbinlog \
  --start-position=456789 \
  --stop-datetime="2026-12-01 14:30:00" \
  /var/log/mysql/mysql-bin.000123 \
  /var/log/mysql/mysql-bin.000124 \
  | mysql -u root -p
```

### Backup Validation

```bash
# Periodically restore to a test environment
xtrabackup --copy-back \
  --target-dir=/backups/mysql/full-20261201 \
  --datadir=/var/mysql-test

# Verify schema and data
pt-table-checksum h=localhost,u=root --databases=test_db
```

### Backup Strategy

| Frequency | Type | Retention |
|-----------|------|-----------|
| Daily | Full XtraBackup | 30 days |
| Hourly | Incremental | 7 days |
| Continuous | Binary logs | Based on disk/archival |

```bash
# Automate binary log archival to S3
mysqlbinlog --read-from-remote-server \
  --host=localhost --user=replication \
  /var/log/mysql/mysql-bin.* \
  | gzip \
  | aws s3 cp - s3://backup-bucket/binlogs/$(date +%Y%m%d-%H%M).gz
```

> **Trap:** `mysqldump` without `--single-transaction` acquires table locks and blocks writes. XtraBackup *must* be prepared before restore; copying the raw backup directory gives an inconsistent state. Not testing backups periodically guarantees you'll discover they're broken during an actual disaster. Restoring with a different major MySQL version can fail due to data dictionary incompatibility.

> **Follow-up:** *How does XtraBackup compare to pg_basebackup?* — `pg_basebackup` copies the entire data directory (including WAL). XtraBackup copies the InnoDB files then applies the redo log (prepare phase). `pg_basebackup` has no "prepare" step, but requires a running server. Both support incremental backups.

---

## 7. Performance Troubleshooting Runbook

### Slow Queries

Enable slow query log:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.1;  -- 100ms threshold
SET GLOBAL log_queries_not_using_indexes = ON;
SET GLOBAL min_examined_row_limit = 100;
```

```bash
# Analyze slow queries
pt-query-digest /var/log/mysql/slow-query.log

# With limits
pt-query-digest /var/log/mysql/slow-query.log \
  --limit=10 \
  --output=slowlog
```

### Find Current Running Queries

```sql
SHOW FULL PROCESSLIST;

-- Master view (8.0+)
SELECT * FROM performance_schema.processlist;

-- Find locked queries
SELECT * FROM performance_schema.data_locks;
```

```bash
# Kill a query
mysqladmin kill <thread_id>
```

### High CPU

1. Check processlist for long-running queries
2. Check `SHOW ENGINE INNODB STATUS` for contention
3. Look for table scans (`SELECT` without `WHERE`, no index)
4. Check for `Using temporary; Using filesort` in `EXPLAIN`

```sql
-- Find queries using temp tables/filesort
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
SHOW GLOBAL STATUS LIKE 'Sort_%';
```

### High I/O

```bash
iostat -x 1
# Check await, %util
```

```sql
-- InnoDB I/O
SHOW ENGINE INNODB STATUS\G
-- Section: FILE I/O

-- Check pages read/written
SELECT * FROM performance_schema.global_status
WHERE variable_name LIKE 'Innodb_pages_%';
```

### Deadlocks

```sql
-- Enable deadlock logging
SET GLOBAL innodb_print_all_deadlocks = ON;

-- Check recent deadlocks (non-dynamic status output)
SHOW ENGINE INNODB STATUS\G
-- Look for "LATEST DETECTED DEADLOCK"
```

```sql
-- Analyze deadlock victims and transactions
SELECT * FROM performance_schema.data_locks
WHERE ENGINE_TRANSACTION_ID IN (
    SELECT ENGINE_TRANSACTION_ID
    FROM performance_schema.data_lock_waits
);
```

### Replication Lag

```sql
SHOW SLAVE STATUS\G
-- Check: Seconds_Behind_Master

-- Use pt-heartbeat for precise lag (fractional seconds)
-- On master:
pt-heartbeat --interval=0.1 --update --create-table --daemonize

-- On slave:
pt-heartbeat --check --master-server-id=1
```

### Connection Spikes

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Connection_errors_max_connections';

-- Current connection usage
SELECT COUNT(*) FROM performance_schema.threads;
```

```bash
# ProxySQL connection pooling (recommended)
# Adds connection multiplexing to absorb spikes
mysql -h proxy_host -P 6033 -u monitor -p
```

### Out of Memory Diagnosis

```sql
-- Per-session memory buffers (adds up!)
SHOW VARIABLES LIKE 'sort_buffer_size';
SHOW VARIABLES LIKE 'join_buffer_size';
SHOW VARIABLES LIKE 'tmp_table_size';
SHOW VARIABLES LIKE 'binlog_cache_size';

-- Buffer pool
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

> **Trap:** Setting `long_query_time` too high (e.g., 2s) misses slow queries. Use `log_slow_rate_limit` for sampling on very busy servers. `SHOW FULL PROCESSLIST` in InnoDB shows current transaction info including `trx_state` and `trx_mysql_thread_id` via `INFORMATION_SCHEMA.INNODB_TRX`. Not profiling queries from all sources (including monitoring tools, cron jobs, replication threads) leads to blind optimization.

> **Follow-up:** *Your first action when MySQL becomes unresponsive?* — Check `SHOW FULL PROCESSLIST` for massive query pileup, then `SHOW ENGINE INNODB STATUS` for deadlocks or I/O issues, then OS-level `iostat`, `vmstat`, `top`. Kill the identified problematic query with `KILL QUERY <thread_id>`.

---

## 8. Native MySQL Features in 8.0+

### CTEs

```sql
-- Simple CTE
WITH users_orders AS (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
)
SELECT u.name, uo.order_count
FROM users u
JOIN users_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 10;

-- Recursive CTE (e.g., tree traversal)
WITH RECURSIVE category_path AS (
    SELECT id, name, parent_id, 1 AS level
    FROM categories
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, cp.level + 1
    FROM categories c
    JOIN category_path cp ON c.parent_id = cp.id
)
SELECT * FROM category_path ORDER BY level, name;
```

### Window Functions

```sql
-- ROW_NUMBER: deduplication
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM sessions
)
SELECT * FROM ranked WHERE rn = 1;

-- RANK / DENSE_RANK
SELECT
    product_id,
    SUM(amount) AS total_sales,
    RANK() OVER (ORDER BY SUM(amount) DESC) AS sales_rank,
    DENSE_RANK() OVER (ORDER BY SUM(amount) DESC) AS dense_rank
FROM order_items
GROUP BY product_id;

-- LEAD / LAG (compare with previous row)
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS daily_change,
    LEAD(revenue, 1) OVER (ORDER BY date) AS next_day_revenue
FROM daily_revenue;

-- NTILE (quantile/bucketing)
SELECT
    user_id,
    lifetime_value,
    NTILE(4) OVER (ORDER BY lifetime_value DESC) AS quartile
FROM users;

-- FIRST_VALUE / LAST_VALUE / NTH_VALUE
SELECT
    date,
    revenue,
    FIRST_VALUE(revenue) OVER (ORDER BY date) AS first_revenue,
    LAST_VALUE(revenue) OVER (
        ORDER BY date
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_revenue
FROM daily_revenue;
```

### LATERAL Derived Tables

```sql
-- LATERAL allows referencing preceding FROM items
SELECT u.name, o.*
FROM users u
JOIN LATERAL (
    SELECT * FROM orders
    WHERE user_id = u.id
    ORDER BY created_at DESC
    LIMIT 5
) o ON true;
```

### NOWAIT and SKIP LOCKED

```sql
-- SKIP LOCKED: skip rows locked by other transactions (queue worker pattern)
-- PostgreSQL equivalent: SELECT ... FOR UPDATE SKIP LOCKED
START TRANSACTION;
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY priority DESC
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- NOWAIT: fail immediately if row is locked
-- PostgreSQL equivalent: SELECT ... FOR UPDATE NOWAIT
SELECT * FROM inventory
WHERE product_id = 42
FOR UPDATE NOWAIT;
```

### Functional Indexes

```sql
-- Index on a JSON field expression
CREATE INDEX idx_email_domain ON users ((JSON_EXTRACT(email_meta, '$.domain')));

-- Index on expression
CREATE INDEX idx_upper_name ON users ((UPPER(name)));

-- These use hidden virtual columns internally (adds storage overhead)
```

### Invisible Indexes

```sql
-- Make an index invisible (test dropping without actually dropping)
ALTER TABLE users ALTER INDEX idx_email INVISIBLE;

-- Monitor if it's used
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'mydb' AND OBJECT_NAME = 'users'
AND INDEX_NAME = 'idx_email';

-- Make it visible again
ALTER TABLE users ALTER INDEX idx_email VISIBLE;
```

### Descending Indexes

```sql
-- MySQL 8.0 supports descending order in indexes (previously ignored)
CREATE INDEX idx_created_at_desc ON orders (created_at DESC);
```

### Resource Groups

```sql
-- Create resource group for reporting queries
CREATE RESOURCE GROUP reporting_group
    TYPE = USER
    VCPU = 0-1  -- CPU affinity
    THREAD_PRIORITY = 5;

-- Lower priority for batch jobs
SET RESOURCE GROUP reporting_group;

SELECT * FROM huge_report_table;
```

### Optimizer Hints

```sql
-- Force index usage
SELECT /*+ INDEX(t idx_name) */ * FROM t WHERE name = 'test';

-- Set variable for query execution
SELECT /*+ SET_VAR(sort_buffer_size=10M) */
    * FROM large_table ORDER BY name;

-- Limit execution time (fail if exceeds)
SELECT /*+ MAX_EXECUTION_TIME(1000) */
    * FROM large_table ORDER BY name;

-- Join order hints
SELECT /*+ BKA(t1) */ *
FROM t1 JOIN t2 ON t1.id = t2.ref;
```

### Default Character Set

```sql
-- MySQL 8.0 defaults to utf8mb4 with utf8mb4_0900_ai_ci
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';
-- utf8mb4 / utf8mb4_0900_ai_ci

-- utf8mb3 in 5.7 (alias for utf8, which is a subset — no emoji)
```

> **Trap:** Not leveraging window functions leads to complex self-joins. `NOWAIT` and `SKIP LOCKED` are InnoDB-only (won't work on MyISAM). Functional indexes use hidden virtual columns with associated storage overhead — they're not zero-cost. Resource groups require CAP_SYS_NICE capability (may need Linux capability configuration).

> **Follow-up:** *Window functions were added in MySQL 8.0 (2018). What was the common pattern before?* — User-defined variables (`@var`), correlated subqueries, and complex self-joins. These patterns are error-prone and should be migrated to window functions.

---

## 9. Tools Ecosystem

### Percona Toolkit

The Swiss Army Knife for MySQL operations.

| Tool | Purpose |
|------|---------|
| `pt-query-digest` | Analyze slow query logs, general logs, and tcpdump |
| `pt-online-schema-change` | Online schema migrations (with triggers) |
| `pt-heartbeat` | Measure real replication lag (sub-second) |
| `pt-kill` | Kill queries matching conditions (user, time, db, query pattern) |
| `pt-index-usage` | Find unused or duplicate indexes |
| `pt-archiver` | Archive rows from large tables without impact |
| `pt-stalk` | Collect diagnostic data before a problem (watch for spikes) |
| `pt-mysql-summary` | Summarize MySQL server configuration and status |
| `pt-config-diff` | Diff MySQL configuration files (validates consistency) |
| `pt-table-checksum` | Check table consistency across replication |

```bash
# Analyze slow query log
pt-query-digest /var/log/mysql/mysql-slow.log

# Kill queries running longer than 60 seconds
pt-kill --busy-time=60 --kill \
  --match-db=mydb \
  --victims=all \
  --daemonize

# Archive old orders (moves data, no locking)
pt-archiver \
  --source h=localhost,D=db,t=orders \
  --where "created_at < NOW() - INTERVAL 1 YEAR" \
  --limit 1000 \
  --commit-each \
  --purge

# Check replication lag precisely
pt-heartbeat --update --interval=0.1 --daemonize
pt-heartbeat --check --master-server-id=1
```

### ProxySQL

ProxySQL is a transparent proxy for MySQL that sits between application and database.

```
Application → ProxySQL (6033) → MySQL backends
```

Key features:

- **Query routing** — read/write splitting via query rules (regex-based)
- **Connection pooling** — multiplexes backend connections (handles connection spikes)
- **Query caching** — in-memory query result cache
- **Query rewriting** — rewrite queries on the fly
- **Firewall** — block queries by pattern or user database
- **Metrics** — `stats_mysql_query_digest` tables

```sql
-- Admin interface (6032)
mysql -h 127.0.0.1 -P 6032 -u admin -padmin

-- Add backends
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (0, '10.0.0.1', 3306);
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

-- Query rule: SELECTs → read replicas (hostgroup 1), writes → master (hostgroup 0)
INSERT INTO mysql_query_rules
  (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES
  (1, 1, '^SELECT', 1, 1),
  (2, 1, '^INSERT|UPDATE|DELETE', 0, 1);
LOAD MYSQL QUERY RULES TO RUNTIME;

-- Monitor query statistics
SELECT * FROM stats_mysql_query_digest ORDER BY sum_time DESC LIMIT 10;
```

### MySQL Shell

MySQL Shell provides admin API for InnoDB Cluster and cluster management.

```js
// Check instance configuration
dba.checkInstanceConfiguration('root@host:3306');

// Create cluster
var cluster = dba.createCluster('prod_cluster');

// Add instances
cluster.addInstance('root@host2:3306');
cluster.addInstance('root@host3:3306');

// Status
cluster.status({extended: 1});

// Switchover (manual primary change)
cluster.setPrimaryInstance('host2:3306');

// Rejoin a failed instance
cluster.rejoinInstance('host3:3306');
```

### MySQL Router

```bash
# Configure Router for InnoDB Cluster
mysqlrouter --bootstrap root@host:3306 --user=mysqlrouter

# Router endpoints
# Port 6446: read-write to primary
# Port 6447: read-only to secondaries
```

### Percona Monitoring & Management (PMM)

PMM provides Grafana dashboards, Prometheus metrics, and Query Analytics (QAN) for MySQL.

```bash
# Run PMM server via Docker
docker run --name pmm-server -p 443:443 percona/pmm-server:2

# Add client
pmm-admin add mysql --query-source=perfschema
```

### mysqld_exporter

```bash
# Prometheus exporter for MySQL
mysqld_exporter \
  --config.my-cnf=/etc/mysql/client.cnf \
  --collect.info_schema.processlist \
  --collect.slave_status
```

> **Trap:** Not using `pt-query-digest` means you're optimizing blindly. ProxySQL's query rules can be confusing and misconfigured rules can break routing (write queries hitting replicas cause errors). PMM collects a lot of data and requires dedicated resources (minimum 4GB RAM for server). Never run `pt-online-schema-change` without testing on a replica first.

> **Follow-up:** *How do you choose between ProxySQL and HAProxy for MySQL?* — ProxySQL understands MySQL protocol (query routing, read/write splitting, query caching). HAProxy is a TCP load balancer — it doesn't understand MySQL queries, so it can't split read/write traffic.

---

## 10. MySQL 8.0 vs 5.7 Migration

### Key Compatibility Changes

| Area | 5.7 | 8.0 | Impact |
|------|-----|-----|--------|
| Data Dictionary | `.frm`, `.par`, `.opt` files on disk | InnoDB transactional DD (no files) | Backup/restore tools must be updated; direct file-level operations break |
| Default charset | utf8mb3 (utf8 alias) | utf8mb4 | Storage size increase (3 bytes → 4 bytes per char). Index length limits affected |
| Default auth plugin | `mysql_native_password` | `caching_sha2_password` | Old clients can't authenticate |
| `GROUP BY` | `sql_mode` may not include `ONLY_FULL_GROUP_BY` | Default includes `ONLY_FULL_GROUP_BY` | Queries with non-aggregated non-GROUP-BY columns fail |
| Query Cache | Available (deprecated in 5.7) | **Removed** | Applications relying on QC see performance drop |
| `PASSWORD()` | Available | **Removed** | Authentication code breaks |
| CTE / Window Functions | Not available | Available | Opportunity to refactor |
| `NOWAIT` / `SKIP LOCKED` | Not available | Available | New locking patterns |
| Invisible Indexes | Not available | Available | Safer index management |
| Descending Indexes | DESC on index was ignored | DESC order stored | Can optimize `ORDER BY DESC` |
| `innodb_undo_tablespaces` | Modify requires restart | Dynamic | Rolling upgrades need attention |
| `innodb_large_prefix` | Existed (default ON) | Removed (always ON) | Remove from config |
| `innodb_file_format` | Existed | Removed (always Barracuda) | Remove from config |
| `query_cache_type` | Existed | Removed | Remove from config |

### Upgrade Procedure

1. **Check compatibility**:

```bash
mysqlsh -- util checkForServerUpgrade --user=root --host=localhost
```

```sql
-- Check tables for strict mode issues
SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE, ROW_FORMAT
FROM INFORMATION_SCHEMA.TABLES
WHERE ENGINE = 'MyISAM' OR ROW_FORMAT NOT IN ('Dynamic', 'Compressed');
```

2. **Fix sql_mode issues**:

```sql
SELECT @@sql_mode;
-- 8.0: ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE,
--       ERROR_FOR_DIVISION_BY_ZERO, NO_ENGINE_SUBSTITUTION
```

3. **Update authentication plugin** (if needed):

```ini
# my.cnf: fallback for old clients
default_authentication_plugin=mysql_native_password
```

```sql
-- Or migrate specific users
ALTER USER 'app_user'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
```

4. **Remove deprecated variables** from my.cnf:

```
# Remove these from my.cnf before 8.0 upgrade:
# query_cache_type=0
# query_cache_size=0
# innodb_large_prefix=ON
# innodb_file_format=Barracuda
# log_warnings=2        → replaced by log_error_verbosity
# sql_mode=...          → will be overwritten by 8.0 default + system vars
```

5. **Check binary log format**:

```sql
SHOW VARIABLES LIKE 'binlog_format';
-- ROW is recommended (required for gh-ost, Group Replication)
```

6. **Upgrade in-place**:

```bash
# MySQL 5.7 → 8.0 in-place upgrade
mysqlsh -- upgrade --user=root
# Or traditional:
mysql_upgrade -u root -p
# mysql_upgrade is no longer needed in 8.0.16+ (server auto-runs upgrade on data dictionary startup)
```

### PHP-Specific Migration Issues

```php
// 5.7: mysql_native_password works with ext-mysqli
// 8.0: caching_sha2_password may not work with old mysqlnd

// Fix: Update to PHP 7.4+ which supports caching_sha2_password
// Or configure MySQL to use mysql_native_password:
// SET GLOBAL default_authentication_plugin = mysql_native_password;
```

| PHP Version | caching_sha2_password support |
|-------------|-------------------------------|
| PHP 7.0-7.3 | Not supported (use mysql_native_password) |
| PHP 7.4 | Supported (via mysqlnd) |
| PHP 8.0+ | Fully supported |

### Performance Regression Risks

| Feature removed | Replacement |
|----------------|-------------|
| Query Cache | External cache (Redis/Memcached), ProxySQL query caching |
| MyISAM engine | InnoDB (almost everything). Consider memory engine for temp tables |

> **Trap:** Not testing client library compatibility — PHP's `ext-mysql` was removed in PHP 7.0, `ext-mysqli` needs updates for `caching_sha2_password`. Query cache removal causes severe performance regression if the application relied on QC — migrate to Redis/Memcached/ProxySQL query caching. Old monitoring tools (pre-8.0) can't read the new data dictionary and return no results. Direct file-level operations (copying `.ibd` files between servers) may fail due to DD changes.

> **Follow-up:** *What are the most common migration failures from 5.7 to 8.0?* — Client authentication failures (caching_sha2_password), sql_mode changes breaking GROUP BY queries, query cache removal degrading read performance, and `PASSWORD()` function calls in application code breaking.

---

## 11. MySQL in the Cloud

### RDS MySQL

AWS RDS is the most common MySQL cloud deployment.

**Backups:**
- Automated daily snapshots (retention up to 35 days)
- Transaction log backups every 5 minutes (PITR capability)
- Backups consume I/O credits on gp2 volumes (can throttle OLTP)

**Read Replicas:**
- Up to 15 read replicas per master
- Cross-region replication support
- Read replicas can be promoted to master
- Replication is async (some lag expected)

**Multi-AZ:**
- Synchronous standby in another AZ
- Automatic failover on AZ failure
- **Standby is NOT accessible** for reads (unlike Aurora)
- Failover causes a CNAME switch (30-60s connection interruption)

**Scaling:**
- Storage auto-scaling (10GB increments when usage > 90%)
- Instance resize (downtime: 5-10 minutes)
- No storage scaling down

**Limitations:**
- `max_connections` = `LEAST({DBInstanceClassMemory/9531392}, 5000)` — cannot override
- No SUPER privilege (some DBA operations need workarounds)
- No binary log direct access (use Kinesis/DMS for streaming)
- No `SELECT ... INTO OUTFILE` (use `SELECT ... INTO OUTFILE S3`)

```sql
-- RDS max_connections formula
SHOW VARIABLES LIKE 'max_connections';
-- db.r5.large (16GB): 16000/9531392 ≈ 1716
-- Cannot increase beyond formula-derived value
```

**Parameter Groups:**
- Dynamic parameters apply immediately
- Static parameters require instance reboot
- Must use custom parameter group to tune

### Aurora MySQL

Aurora MySQL is Amazon's MySQL-compatible engine with a distributed storage layer.

**Architecture:**
- Separate compute (DB instances) and storage (distributed 6TB-128TB)
- Storage: 6-way replication across 3 AZs (tertiary striping)
- Compute: up to 15 reader endpoints + 1 writer

**Key Differences from RDS MySQL:**

| Feature | RDS MySQL | Aurora MySQL |
|---------|-----------|--------------|
| Storage | EBS volumes (gp2/io1) | Distributed storage (log-structured) |
| Max storage | 64TB (io1) | 128TB |
| Read replicas | 15 (async) | 15 (minimal lag — shared storage) |
| Replica lag | Variable | < 100ms typically |
| Crash recovery | Minutes (redo log apply) | < 60s (shared storage handles recovery) |
| Backups | Snapshots + binlog | Snapshots + continuous backup to S3 |
| Failover | 30-60s DNS change | < 30s (faster with Aurora auto) |
| Storage auto-scaling | Yes (limited) | Yes (automatic, up to 128TB) |

**Aurora MySQL Compatibility:**
- MySQL 5.7 (Aurora 2.x) and MySQL 8.0 (Aurora 3.x)
- **Not 100% compatible:** no Group Replication, no InnoDB Cluster, no XA transactions in some versions, no async replication in same manner as RDS, no `ALTER TABLE ... INSTANT` in older versions
- Performance schema is limited (can't query some tables)
- Binlog is optional (conflicts with fast write performance)

```sql
-- Aurora-specific variables
SELECT @@aurora_version;
SHOW VARIABLES LIKE 'aurora_%';

-- Check reader/writer status
SELECT @@innodb_read_only;
-- 0 = writer, 1 = reader
```

### Cloud SQL for MySQL (GCP)

Similar to RDS, with automated backups, read replicas, and point-in-time recovery.

- Failover replica (regional, not accessible for reads)
- No multi-AZ writes (single-zone primary + failover replica)
- `max_connections` similarly tied to instance memory

### Migration to Cloud

**AWS DMS (Database Migration Service):**

```bash
# DMS tasks support:
# - Full load + CDC (continuous replication)
# - Schema conversion (minimal for MySQL → MySQL)
# - Validation of migrated data
# - No downtime during migration
```

Steps:
1. Create DMS replication instance
2. Source endpoint (MySQL on-prem)
3. Target endpoint (RDS/Aurora MySQL)
4. Migration task: full load + CDC
5. Validation
6. Cut-over (stop app, catch up CDC, switch DNS)

### RDS vs Self-Managed on EC2

| Factor | RDS | Self-managed EC2 |
|--------|-----|-----------------|
| Operational overhead | Minimal | Full DBA ownership |
| Customization | Can't install Percona Toolkit or custom plugins | Full control |
| Performance | gp3/io2 Block Express EBS | Local NVMe instances (i3/i4i) — better IOPS |
| Monitoring | CloudWatch, Performance Insights, enhanced monitoring | PMM, mt-query-digest, Prometheus |
| Cost | Higher (managed service premium) | Lower (EC2 + EBS) |
| HA | Multi-AZ (managed failover) | Orchestrator, MHA, InnoDB Cluster (you manage) |

> **Trap:** RDS Multi-AZ is NOT a read replica — the standby is not accessible, and Multi-AZ doesn't scale reads. Aurora MySQL is not 100% MySQL compatible — some features (Group Replication, XA transactions) are missing or behave differently. RDS backups consume I/O credits on gp2 volumes — with low credits, performance degrades. RDS `max_connections` is tied to instance memory and can't be overridden.

> **Follow-up:** *Why might you choose self-managed Percona Server over RDS?* — Percona Server has diagnostic features (logical backup locks, unquoted PK optimization, `SHOW ALL PROFILES`), custom plugins (MyRocks for compression), and you can install Percona Toolkit, ProxySQL, Orchestrator without restrictions. RDS abstracts away the OS and limits some DBA operations.

---

## 12. Tier 3 Q&A Drill

**Q1: You're designing a multi-region HA MySQL setup. Compare InnoDB Cluster across 3 AZs vs async replication with Orchestrator failover. Which would you choose and why?**

**A:** InnoDB Cluster across AZs requires < 5ms latency between nodes — risky across regions. Use async replication with Orchestrator for cross-region, and InnoDB Cluster (single-primary) within a region for automatic failover. For active-active multi-region, use Vitess or application-level multi-master with conflict resolution.

**Q2: Your 500GB orders table has become unmanageable. Walk through your approach before deciding to shard.**

**A:** 1) Check if indexing is optimal (missing indexes, redundant indexes). 2) Consider partitioning by date (RANGE on `created_at`) for time-series data with partition pruning. 3) Archiving old data with pt-archiver. 4) Read replicas for read scaling. 5) Buffer pool size optimization. Only after exhausting these, consider sharding with Vitess or application-level sharding.

**Q3: Explain the difference between `innodb_flush_log_at_trx_commit=1` and `=2`. When would you accept the risk of `=2`?**

**A:** `=1` fsyncs the redo log on every commit (full ACID durability). `=2` writes to OS cache only (fsync every 1s). `=2` loses up to 1 second of committed transactions on a crash. Acceptable for: analytics/staging/reporting servers, or when slave can take over quickly and data loss is acceptable.

**Q4: Your slave is showing 5 seconds of replication lag that spikes to 30 seconds during peak hours. What's your diagnostic process?**

**A:** 1) Check `SHOW SLAVE STATUS` for `Seconds_Behind_Master` and `SQL_Remaining_Delay`. 2) Use `pt-heartbeat` for precise measurement. 3) Check slave hardware (CPU, I/O). 4) Check if slave is running long queries blocking replication (`SHOW PROCESSLIST`). 5) Check `innodb_flush_log_at_trx_commit` on slave (set to 2 if safe). 6) Consider: parallel replication (`slave_parallel_workers`), better slave hardware, split read traffic across more replicas.

**Q5: What happens when you run `ALTER TABLE` on a 100GB InnoDB table without any online tool?**

**A:** MySQL creates a temporary table with the new schema, copies all rows from the original (blocked for writes during copy), creates indexes, then drops the original and renames. The table is locked (LOCK=SHARED by default, or LOCK=EXCLUSIVE). This blocks all writes for potentially hours. Use pt-online-schema-change or gh-ost, or use `ALGORITHM=INSTANT` in 8.0 if applicable.

**Q6: You need to recover to a specific point in time, but your last full backup was 5 days ago. Walk through the recovery steps.**

**A:** 1) Restore the full backup (Percona XtraBackup prepare + copy-back). 2) Get binlog position from backup (`xtrabackup_binlog_info`). 3) Apply all binary logs from that position to the target time using `mysqlbinlog --stop-datetime="..."`. 4) Verify data consistency. 5) If binary logs are corrupted, recovery is limited to the last good log position.

**Q7: Design a sharding strategy for a social media application's user feed. Consider: 100M users, 1B posts, read-heavy, geo-distributed.**

**A:** Shard by `user_id` (high cardinality, even distribution, aligns with feed queries). Use Vitess for transparent routing. Each shard holds a subset of users' data. Feed queries are single-shard (`WHERE user_id = ?`). Cross-shard operations (e.g., global search) use Vitess's scatter-gather. Reshard: Vitess `Reshard` workflow with consistent hashing. Geo: route users to nearest shard region.

**Q8: MySQL 8.0 removed the query cache. Your application relied on it for frequently read-but-rarely-changed reference data. What's your migration strategy?**

**A:** 1) Identify the queries: enable slow log, use pt-query-digest. 2) Implement application-level caching (Redis/Memcached) for the read-heavy data. 3) Or use ProxySQL query caching at the proxy layer. 4) Or use MySQL Router's caching if on InnoDB Cluster. 5) Measure the performance improvement after migration.

**Q9: Explain the difference between MySQL's `PARTITION BY RANGE` and PostgreSQL's declarative partitioning. Can you migrate directly?**

**A:** Both support RANGE, LIST, HASH. MySQL requires unique keys to include partition columns; PG supports unique indexes without partition columns (but global indexes don't exist — uniqueness is per-partition). MySQL doesn't support foreign keys on partitioned tables; PG does (PG12+). Migration: convert `PARTITION BY RANGE` syntax, handle unique key changes, drop/add foreign keys if needed. Most partitioning constructs map directly.

**Q10: What's the difference between `EXPLAIN` in MySQL vs PostgreSQL? How do you identify a bad query plan in MySQL?**

**A:** MySQL `EXPLAIN` shows: id, select_type, table, type (const/ref/range/index/ALL — ALL is bad), possible_keys, key, key_len, ref, rows, Extra (Using where, Using index, Using temporary, Using filesort). PostgreSQL `EXPLAIN ANALYZE` shows actual execution costs vs estimates, loops, and buffers. In MySQL, look for: table scan (ALL), Using filesort, Using temporary, no key used, rows estimate significantly off.

**Q11: Your MySQL 5.7 to 8.0 migration plan. What are the showstoppers?**

**A:** 1) `PASSWORD()` calls in application code (removed). 2) `sql_mode` changes breaking GROUP BY queries. 3) Query cache removal (performance regression). 4) `caching_sha2_password` causing authentication failures from old PHP clients. 5) Custom monitoring tools that read `.frm` files directly. 6) MyISAM tables not fully compatible with 8.0 features. 7) `innodb_large_prefix`, `innodb_file_format`, `query_cache_*` in my.cnf must be removed.

**Q12: Write a query to find the top 5 products by revenue for each category, using MySQL 8.0 features.**

```sql
SELECT category_id, product_id, total_revenue, rank_no
FROM (
    SELECT
        p.category_id,
        p.product_id,
        SUM(oi.quantity * oi.unit_price) AS total_revenue,
        RANK() OVER (
            PARTITION BY p.category_id
            ORDER BY SUM(oi.quantity * oi.unit_price) DESC
        ) AS rank_no
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category_id, p.product_id
) ranked
WHERE rank_no <= 5
ORDER BY category_id, rank_no;
```

**Q13: Your monitoring shows high `innodb_buffer_pool_reads` (disk reads) despite the buffer pool being 80% of RAM. Diagnose.**

**A:** 1) Check buffer pool hit ratio — should be > 99%. If low: insufficient size or working set doesn't fit. 2) Check if MySQL has enough RAM (avoid swap). 3) Check for large table scans (queries not using indexes). 4) Check if buffer pool is fragmented (check `innodb_buffer_pool_pages_data` vs `innodb_buffer_pool_pages_total`). 5) Consider multiple buffer pool instances for contention. 6) Use Buffer Pool LRU dump/load for warmup after restart.

**Q14: How do you handle deadlocks in a high-concurrency transactional system?**

**A:** 1) Enable `innodb_print_all_deadlocks = ON`. 2) Analyze latest deadlock in `SHOW ENGINE INNODB STATUS`. 3) Fix: ensure consistent lock order across all transactions (e.g., always update accounts in the same order). 4) Reduce transaction size/duration. 5) Use READ COMMITTED instead of REPEATABLE READ (reduces gap locking). 6) Consider `innodb_deadlock_detect=OFF` in very high contention (but handle with lock wait timeout). 7) Implement retry logic in application.

**Q15: Compare MySQL Group Replication with PostgreSQL synchronous replication.**

**A:** MySQL Group Replication uses group communication (Paxos-like) across a set of nodes. All nodes commit independently but must certify transactions — a transaction isn't committed until a majority agrees. PostgreSQL sync replication: one or more standbys must acknowledge the WAL flush. GR handles multi-node failover automatically; PG sync replication requires external tools (Patroni, repmgr) for automated failover. GR is sensitive to latency (< 5ms); PG sync replication can handle higher latency with `synchronous_commit = remote_write` (lighter durability guarantee).
