# PostgreSQL — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS (primary database — PostgreSQL), 88% query reduction (Postgres indexing & query optimization), 15M+ row zero-downtime migration (expand→migrate→contract, `NOT VALID`, `CREATE INDEX CONCURRENTLY`), 20K+ DAU trading platform (concurrency, isolation levels, `SKIP LOCKED`)

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**Every section ties to your real experience.** If you lived through the 88% query reduction, the 15M migration, the trading platform concurrency, and the multi-tenant SaaS — you should be able to speak to each of these from memory. The material here gives you the language and depth to frame those stories.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | PostgreSQL architecture (process model, shared buffers, WAL, vacuum), SQL data types, indexes (B+Tree, GiST, GIN, BRIN, SP-GiST), basic query patterns, MVCC, transaction isolation levels, roles & permissions | 6–8 hours |
| [`02-intermediate.md`](./02-intermediate.md) | `EXPLAIN ANALYZE` deep reading, indexing strategy (composite, partial, covering, expression, concurrent creation), query optimization patterns, locking (row-level, advisory, deadlocks, `FOR UPDATE`/`SKIP LOCKED`), replication (streaming, logical, cascading), connection pooling (PgBouncer), partitioning (declarative) | 10–12 hours |
| [`03-senior.md`](./03-senior.md) | Zero-downtime migrations (expand→migrate→contract, `NOT VALID`, `CREATE INDEX CONCURRENTLY`, backfill strategies), multi-tenancy indexing & query patterns, performance tuning (autovacuum tuning, WAL tuning, memory settings), high availability (Patroni, repmgr, streaming + WAL archiving, failover), backup & PITR (pgBackRest, pg_dump, pg_basebackup), advanced monitoring, Postgres vs MySQL deep comparison, extension ecosystem (PostGIS, pg_stat_statements, pg_cron, pg_partman) | 14–18 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 180+ interview questions, code puzzles, debugging scenarios tied to your actual incidents, system design prompts (multi-tenant schema, zero-downtime migration design, trading ledger, inventory system) | Ongoing drill |

---

## Coverage map

### PostgreSQL architecture
- Process model: postmaster (supervisor), backend processes (per connection), background workers (checkpointer, autovacuum, WAL writer, archiver, stats collector, logical replication launcher)
- Shared memory: shared buffers, WAL buffer, clog (commit log), lock space
- WAL (Write-Ahead Log): sequential writes, crash recovery, replication, PITR
- MVCC: tuple headers (xmin, xmax), visibility rules, tuple versions in heap blocks, HOT (Heap-Only Tuples) updates
- Vacuum: why it exists (dead tuples), autovacuum, `VACUUM FULL` vs `VACUUM`, freeze
- System catalogs: `pg_class`, `pg_attribute`, `pg_index`, `pg_stat_all_tables`, `pg_stat_statements`

### SQL & data modeling
- Data types: numeric (SMALLINT, INTEGER, BIGINT, DECIMAL/NUMERIC, REAL, DOUBLE, MONEY, SERIAL/BIGSERIAL), character (CHAR, VARCHAR, TEXT), binary (BYTEA), temporal (DATE, TIME, TIMESTAMP, TIMESTAMPTZ, INTERVAL), UUID, JSON/JSONB, arrays, hstore, network types (INET, CIDR, MACADDR), geometric, range types, composite types, custom types (CREATE DOMAIN, CREATE TYPE)
- JSONB: binary JSON, indexing with GIN, operators (@>, ?, ?|, ?&, ->, ->>, #>, #>>), performance vs JSON
- Array operations: ANY, ALL, unnest, array_agg
- Full-text search: tsvector, tsquery, GIN indexes, ranking (ts_rank), highlighting (ts_headline), dictionaries, configurations
- Constraints: NOT NULL, UNIQUE, PRIMARY KEY, FOREIGN KEY, CHECK, EXCLUSION
- Inheritance (table inheritance), partitioning (declarative)

### Indexing
- Index types: B+Tree (default, equality + range), GiST (full-text, geometry, range types), GIN (JSONB, arrays, full-text), BRIN (append-only, huge tables), SP-GiST (point clustering, network trees), Hash (equality only)
- B+Tree features: INCLUDE columns (covering), NULLS {FIRST|LAST}, DESC, partial indexes, concurrent creation
- Composite indexes: column order (equality, range, sort, include), left prefix rule, skip scan (PostgreSQL 15+)
- Partial indexes: `CREATE INDEX ... WHERE condition`, smaller and faster
- Covering indexes: `INCLUDE (col1, col2)` for index-only scans
- Expression indexes: `CREATE INDEX ON t (LOWER(name))`
- Operator classes: text_pattern_ops, varchar_pattern_ops for LIKE without prefix

### Query optimization
- `EXPLAIN` output: node types (Seq Scan, Index Scan, Index Only Scan, Bitmap Scan, Nested Loop, Hash Join, Merge Join, Sort, Aggregate), cost (startup, total), actual rows vs loops, buffers
- `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`
- Plan nodes: reading plans, identifying bottlenecks (row estimate mismatches, sequential scans on large tables, excessive loops, sort memory, join order)
- `pg_stat_statements`: query fingerprinting, total time, mean time, call count, rows
- Common optimizations: prevent Seq Scan on large tables, use covering indexes, optimize joins (Nested Loop vs Hash Join vs Merge Join), reduce sort memory, use `LIMIT` with proper index
- CTE optimization barrier (PostgreSQL 12+ optional materialization)

### Concurrency & locking
- Row-level locks: `FOR UPDATE` (exclusive), `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`
- Table-level locks: ACCESS SHARE, ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE
- Lock conflicts: X with Y (who blocks whom)
- Advisory locks: `pg_advisory_lock`, `pg_advisory_xact_lock`, `pg_try_advisory_lock`
- `SKIP LOCKED`: skip locked rows (queue workers, job claiming)
- `NOWAIT`: fail immediately if row is locked
- Deadlock detection: `deadlock_timeout`, `log_lock_waits`
- `idle_in_transaction_session_timeout`

### Replication & HA
- Streaming replication: WAL sender to WAL receiver, synchronous vs asynchronous, `synchronous_standby_names`
- Replication slots: physical slots, logical slots
- Cascading replication: standby replicating from another standby
- `hot_standby`: read-only queries on standby
- Logical replication: publish/subscribe, row-level, column selection, cross-version
- WAL archiving: `archive_mode`, `archive_command`, `restore_command`
- Failover: `pg_ctl promote`, Patroni (etcd/consul-based auto-failover), repmgr
- Connection pooling: PgBouncer (transaction pooling vs session pooling vs statement pooling), Pgpool-II
- Read-after-write consistency with replicas: `sticky = true` in Laravel

### Performance tuning
- Memory: `shared_buffers` (25% of RAM), `effective_cache_size` (50-75% of RAM), `work_mem` (per-sort/per-join), `maintenance_work_mem` (VACUUM, CREATE INDEX), `wal_buffers`
- Checkpoint tuning: `checkpoint_timeout`, `max_wal_size`, `min_wal_size`, checkpoint spreading
- Autovacuum tuning: `autovacuum_vacuum_threshold`, `autovacuum_vacuum_scale_factor`, `autovacuum_analyze_scale_factor`, per-table overrides, `vacuum_cost_limit`
- WAL tuning: `wal_level`, `wal_compression`, `wal_log_hints`, `full_page_writes`
- Query tuning: `random_page_cost` (default 4 — tune to 1.5 for SSD), `effective_io_concurrency`, `parallel_query_workers`
- `ALTER SYSTEM SET` vs `postgresql.conf`

### Extensions
- `pg_stat_statements`: query execution statistics (must-have)
- `pg_partman`: automated partition management
- `pg_cron`: scheduled job execution inside PostgreSQL
- `pgBackRest`: backup and restore tool
- `wal-g`: backup tool (used by TimescaleDB)
- `PostGIS`: spatial and geographic objects
- `uuid-ossp` / `pgcrypto`: UUID generation, cryptographic functions
- `pg_trgm`: trigram matching for fuzzy search and `ILIKE '%x%'`
- `btree_gin` / `btree_gist`: GIN/GiST support for B-Tree comparable types

---

## Study order recommendation

PostgreSQL is your primary database and the foundation of your most impressive stories. This should be one of your strongest interview topics.

```
Week 1:  01-basic.md              + Basic Q&A drill
Week 2:  02-intermediate.md       + Intermediate Q&A drill
Week 3:  03-senior.md (first half: migrations, HA, performance)
Week 4:  03-senior.md (second half: tuning, extensions, operations)
Week 5+: 04-question-bank.md daily drill
```

**Next topic in skill order:** MongoDB.
