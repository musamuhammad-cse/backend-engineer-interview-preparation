# Kafka, RabbitMQ, SQS & SNS — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** SQS for async jobs in Laravel inventory SaaS, SNS for notifications, RabbitMQ experience from prior stacks, SQS FIFO for trading platform order processing  
> **Note:** Messaging is core to distributed systems. The senior signal is **comparing message brokers based on throughput, ordering, durability, and operational complexity** — knowing WHEN to use Kafka vs RabbitMQ vs SQS vs SNS, not just how to use one.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud | 20 min/section |
| 2 | Design a messaging architecture for a distributed system | 20 min/section |
| 3 | Answer the section's Q&A without looking, then diff | 20 min/section |
| 4 | Write a producer/consumer for each broker from memory | 15 min |

**The senior signal is broker comparison and trade-off analysis.** Knowing Kafka topic config is table stakes; knowing when Kafka is overkill and RabbitMQ is a better fit is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Message queue concepts (producer, consumer, broker, queue, topic), delivery guarantees (at-most-once, at-least-once, exactly-once), push vs pull, pub/sub vs competing consumers, dead-letter queues, message ordering | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | RabbitMQ (exchanges, bindings, queues, ACK/NACK, publisher confirms), Kafka (topics, partitions, offsets, consumer groups, retention, replication), SQS (standard vs FIFO, visibility timeout, DLQ, long polling), SNS (topics, subscriptions, message filtering, fan-out) | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Kafka performance tuning (partition count, batch size, acks, min.insync.replicas), Kafka exactly-once semantics, RabbitMQ clustering/quorum queues, message broker comparison (Kafka vs RabbitMQ vs SQS vs SNS vs Pulsar), schema registry, message serialization (Avro, Protobuf), advanced patterns (competing consumers with Kafka, priority queues, idempotent consumers) | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 140+ interview questions, architecture scenarios, debugging, STAR templates | Ongoing drill |

---

## Coverage map

### Core concepts
- Message, producer, consumer, broker, queue, topic
- Delivery guarantees: at-most-once, at-least-once, exactly-once
- Push (SNS, RabbitMQ) vs Pull (SQS, Kafka)
- Pub/sub vs point-to-point (competing consumers)
- Message ordering: global order (FIFO), per-partition order (Kafka), no order (standard)
- Dead-letter queues (DLQ)
- Message persistence (disk vs memory)
- Consumer acknowledgements (auto vs manual)

### RabbitMQ
- Exchanges: direct, topic, fanout, headers
- Bindings: routing key pattern matching
- Queues: durable vs transient, auto-delete, exclusive
- Publisher confirms, consumer ACK/NACK, prefetch
- Dead letter exchange (DLX)
- Clustering: traditional HA queues vs quorum queues
- Shovel, federation (cross-cluster replication)

### Kafka
- Topic, partition, offset, consumer group
- Brokers, controller, ZooKeeper / KRaft
- Replication: leader, followers, ISR (in-sync replicas)
- Retention: time-based, size-based, compaction
- acks: 0, 1, all (acks=-1 with min.insync.replicas)
- Consumer group rebalancing
- Exactly-once semantics (EOS): idempotent producer, transactions
- Schema Registry with Avro/Protobuf
- Kafka Streams, KSQL, Connect

### SQS
- Standard (at-least-once, best-effort order, unlimited TPS)
- FIFO (exactly-once, strict order, 300/3,000 TPS)
- Visibility timeout, delay queues, dead-letter queues
- Long polling vs short polling
- Message batching (up to 10 messages)
- SQS Extended Client (large messages via S3)

### SNS
- Topic: standard vs FIFO
- Subscription: SQS, Lambda, email, SMS, HTTP/S, mobile push
- Message filtering (attributes-based)
- Fan-out pattern (SNS → multiple SQS queues)
- Message durability vs SQS (SNS has no built-in persistence)

### Comparison

| Broker | Delivery | Ordering | Throughput | Persistence | Operational complexity |
|--------|----------|----------|------------|-------------|----------------------|
| Kafka | At-least-once / Exactly-once | Per-partition | Highest | Configurable retention | High |
| RabbitMQ | At-most-once / At-least-once | Per-queue | Medium | Configurable | Medium |
| SQS Standard | At-least-once | Best-effort | Unlimited (nearly) | 14 days max | None (managed) |
| SQS FIFO | Exactly-once | Strict FIFO | 3,000 TPS | 14 days max | None (managed) |
| SNS | At-least-once | None (parallel) | High | None (pass-through) | None (managed) |

### Patterns
- Idempotent consumers (handle duplicates)
- Competing consumers (scalable processing)
- Dead-letter queue + reprocessing
- Fan-out (SNS → SQS, Kafka consumer groups)
- Priority queue
- Request-response over message queue
- Claim check (large message → S3 + reference in queue)
- Transactional outbox

---

## Study order recommendation

Focus on SQS/SNS (your daily tools), then Kafka (senior signal), then RabbitMQ for comparison.

```
Week 1:  01-basic.md          + SQS producer/consumer with Laravel
Week 2:  02-intermediate.md   + Kafka producer/consumer in Go
Week 3:  03-senior.md         + Kafka tuning, schema registry, broker comparison
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** OAuth/JWT/OWASP.
