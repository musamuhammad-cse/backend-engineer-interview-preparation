# DynamoDB — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** AWS ecosystem (SQS, SNS, Lambda, API Gateway — DynamoDB is the default NoSQL choice on AWS), multi-tenant SaaS, trading platform, Chronos  
> **Note:** DynamoDB is the most common NoSQL database in FAANG interviews (especially Amazon). The senior signal is **single-table design**, **access pattern-driven schema**, and **partition/key distribution**.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what kills senior loops | 5 min |

**The senior signal is single-table design and access pattern analysis.** Unlike PostgreSQL where you design schema first, DynamoDB requires you to model based on access patterns. FAANG interviews heavily test this.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Tables, items, attributes, primary key (partition key vs partition+sort key), CRUD (PutItem, GetItem, UpdateItem, DeleteItem, BatchGetItem, BatchWriteItem), Query vs Scan, RCU/WCU (provisioned vs on-demand), basic indexes (LSI, GSI), pagination, conditional expressions, DynamoDB Local | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | LSI vs GSI deep dive (vs projected attributes, throttling), consistency models (eventual vs strong), DynamoDB Streams (records, consumers with Lambda), DAX (caching layer), TTL (auto-expire), transactions (transactGet/transactWrite), optimistic locking with version numbers, PartiQL, DynamoDB Fluent API | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Single-table design (access pattern-driven schema, adjacency list pattern, composite sort keys), hot key mitigation, partition splitting (how partitions scale), auto-scaling vs on-demand, DynamoDB Accelerator (DAX) deep dive, global tables (multi-region replication), backup and restore (PITR, on-demand), DynamoDB + Lambda + API Gateway (serverless stack), DynamoDB vs DynamoDB alternatives (ScyllaDB, Cassandra) | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 140+ interview questions, code puzzles (GSI design, query patterns, single-table model), debugging scenarios (throttling, hot keys, high latency), system design prompts | Ongoing drill |

---

## Coverage map

### DynamoDB fundamentals
- Table, item, attribute (document + key-value data model)
- Primary key: partition key only (hash) vs partition key + sort key (hash + range)
- Secondary indexes: Local Secondary Index (LSI) vs Global Secondary Index (GSI)
- CRUD API: PutItem, GetItem, UpdateItem, DeleteItem, BatchGetItem, BatchWriteItem
- Query (by partition key + optional sort key conditions)
- Scan (full table scan — expensive)
- Pagination: LastEvaluatedKey, ExclusiveStartKey
- Conditional expressions (condition check before write)
- ReturnValues (NONE, ALL_OLD, UPDATED_OLD, ALL_NEW, UPDATED_NEW)
- Item size limit: 400 KB (including attribute names)

### Capacity and pricing
- Read Capacity Unit (RCU): 1 strongly consistent read of 4 KB/s, 2 eventually consistent reads of 4 KB/s
- Write Capacity Unit (WCU): 1 write of 1 KB/s
- Provisioned mode: specify RCU/WCU, auto-scaling possible
- On-demand mode: pay per request (unlimited throughput)
- Burst capacity: unused RCU/WCU accumulated for up to 5 minutes
- Throttling: ProvisionedThroughputExceededException

### Indexes
- LSI: Same partition key, different sort key. Created at table creation. Max 5 LSIs per table.
- GSI: Different partition key + sort key. Can be created after table creation. Max 20 GSIs per table.
- Projected attributes: KEYS_ONLY, INCLUDE, ALL
- GSI write capacity is separate from base table
- LSI shares throughput with base table
- Throttling on indexes affects base table and vice versa

### Consistency and transactions
- Eventually consistent reads (default): ~1 second lag, half the cost
- Strongly consistent reads: latest data, higher latency/cost
- ACID transactions (transactGet, transactWrite): 3x cost in capacity
- Transactional capacity: 2 RCU/WCU per item per transaction
- Optimistic locking: conditional expression on version attribute

### Streams and change data capture
- DynamoDB Streams: 24-hour retention, ordered per partition, Kinesis-compatible record format
- Stream records: INSERT, MODIFY, REMOVE
- View types: KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES
- Lambda triggers: process stream records in near-real-time
- Kinesis Data Streams for DynamoDB (2023+): longer retention, higher throughput

### Advanced features
- DAX (DynamoDB Accelerator): in-memory cache, microsecond latency, write-through
- TTL: auto-delete items after timestamp, no cost, eventual deletion (within 48 hours)
- Global Tables: multi-region active-active replication
- Point-in-Time Recovery (PITR): 35-day continuous backup
- On-demand backup: full table backup
- PartiQL: SQL-compatible query language for DynamoDB
- DynamoDB Fluent API / DocumentClient (DynamoDBMapper in Java)

### Single-table design
- One table per application (not per entity)
- Access pattern-driven schema: identify all queries first, then design
- Composite sort key: `ENTITY#ID` for hierarchical sorting
- Adjacency list pattern: store edges as items with partition key = from, sort key = to
- Overloaded attributes: same attribute means different things for different entities
- GSIs for alternative access patterns
- Sparse indexes: GSIs only index items that have the GSI key attribute

### DynamoDB vs alternatives
- DynamoDB vs Cassandra/ScyllaDB: managed vs self-managed, consistency model, single-table design culture
- DynamoDB vs MongoDB: consistency, scalability, query flexibility, pricing
- DynamoDB vs PostgreSQL: NoSQL vs relational, access pattern-first vs schema-first

---

## Study order recommendation

DynamoDB is a major topic in FAANG interviews (especially Amazon). Focus on single-table design as the key differentiator.

```
Week 1:  01-basic.md          + Basic Q&A drill
Week 2:  02-intermediate.md   + Intermediate Q&A drill (GSI design)
Week 3:  03-senior.md         + Senior Q&A drill (single-table modeling)
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** AWS services.
