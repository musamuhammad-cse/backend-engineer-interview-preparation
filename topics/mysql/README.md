# MySQL — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS (Laravel, PostgreSQL, Redis) — MySQL knowledge is essential for comparison and for roles where MySQL is the primary database. 88% query reduction, 15M+ row migration, 20K+ DAU trading platform.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**The senior signal is not knowing definitions.** It's knowing trade-offs, failure modes, and what you'd do at 3am when it breaks.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | MySQL architecture, InnoDB storage engine, SQL fundamentals vs MySQL extensions, data types & column types, indexes (B+Tree, primary/secondary, composite), `EXPLAIN`, basic query patterns, MySQL vs PostgreSQL comparison | 6–8 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Advanced indexing (covering, partial, full-text), query optimization & profiling, transactions & isolation levels (InnoDB specifics), locking (row-level, gap, next-key, table, metadata), deadlock diagnosis & prevention, replication (async, semi-sync, GTID), replication lag & consistency | 10–12 hours |
| [`03-senior.md`](./03-senior.md) | High availability (MHA, InnoDB Cluster, Group Replication, Orchestrator), sharding (Vitess, ProxySQL), partitioning (RANGE, LIST, HASH, KEY), performance tuning (buffer pool, redo log, query cache, thread pool), schema migration tools (pt-osc, gh-ost), backup & recovery (XtraBackup, mysqldump, PITR), migration from/to other databases, production incident runbooks | 12–14 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 160+ interview questions, code puzzles, debugging scenarios, system design prompts, performance optimization problems | Ongoing drill |

---

## Coverage map

### MySQL architecture
- Client/server protocol, connection handling, thread pool
- Query execution pipeline: parser → preprocessor → optimizer → executor → storage engine
- Storage engines: InnoDB (default), MyISAM, Memory, CSV, Archive — when each is appropriate
- InnoDB internals: B+Tree indexes, clustered indexes, secondary indexes, adaptive hash index, change buffer, doublewrite buffer, redo log, undo log, MVCC

### SQL & data modeling
- MySQL data types: numeric (TINYINT, INT, BIGINT, DECIMAL, FLOAT), string (CHAR, VARCHAR, TEXT, BLOB), temporal (DATE, DATETIME, TIMESTAMP, YEAR), JSON, spatial
- Column charset and collation (utf8mb4, utf8mb4_unicode_ci, performance implications)
- Normalization vs denormalization
- ENUM vs VARCHAR vs lookup tables
- Generated columns, CHECK constraints
- Foreign keys (InnoDB only)

### Indexing
- B+Tree structure, clustered index (primary key), secondary indexes
- Composite index column order (equality → range → sort)
- Covering indexes, index-only scans
- Cardinality and selectivity, `SHOW INDEX`
- `EXPLAIN` output: type (ALL, index, range, ref, eq_ref, const), key, rows, Extra (Using index, Using where, Using filesort, Using temporary)
- Full-text indexes (MATCH...AGAINST, boolean mode, query expansion)
- Spatial indexes (R-Tree for GIS data)

### Query optimization
- Slow query log, long_query_time, log_queries_not_using_indexes
- `EXPLAIN ANALYZE` (MySQL 8.0.18+)
- `SHOW PROFILE`, `SHOW PROFILES`
- `performance_schema`, `sys` schema
- Common optimizations: avoid SELECT *, use LIMIT, prefer JOIN over subqueries, use EXISTS vs IN, avoid functions on indexed columns, use covering indexes
- Query rewriting patterns

### Transactions & locking
- InnoDB row-level locking: shared (S) and exclusive (X) locks
- Intention locks (IS, IX)
- Record locks, gap locks, next-key locks
- Insert intention locks, AUTO-INC locks
- MVCC: undo log, read view, consistent non-locking reads
- Isolation levels: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ (default), SERIALIZABLE
- Phantom reads in InnoDB (prevented by next-key locking at REPEATABLE READ)
- Deadlock detection, `innodb_deadlock_detect`, `innodb_lock_wait_timeout`
- Transaction best practices

### Replication & high availability
- Binary log (statement-based, row-based, mixed)
- Asynchronous replication, semi-synchronous replication
- GTID-based replication (MySQL 5.6+)
- Replication lag: causes, monitoring, mitigation
- Group Replication (InnoDB Cluster)
- MySQL Router for connection routing
- Failover: MHA, Orchestrator, ProxySQL

### Performance & operations
- InnoDB buffer pool sizing and management
- InnoDB redo log sizing
- Query cache (deprecated in 8.0)
- Thread cache, table cache, table definition cache
- `innodb_flush_log_at_trx_commit` — durability vs performance trade-off
- `innodb_io_capacity`, `innodb_io_capacity_max`
- `max_connections`, connection pooling
- Schema migrations: pt-online-schema-change, gh-ost (triggerless)
- Backup: XtraBackup (physical), mysqldump (logical), PITR via binary logs
- Monitoring: Prometheus + mysqld_exporter, Percona Monitoring & Management (PMM)

---

## Study order recommendation

Even with PostgreSQL experience, MySQL has enough differences (default REPEATABLE READ, gap locking, different optimizer, different index structure) to warrant dedicated study.

```
Week 1:  01-basic.md         + Basic Q&A drill
Week 2:  02-intermediate.md  + Intermediate Q&A drill
Week 3:  03-senior.md        + Senior Q&A drill
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** PostgreSQL.
