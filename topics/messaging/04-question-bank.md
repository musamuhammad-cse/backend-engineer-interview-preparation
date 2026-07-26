# Kafka, RabbitMQ, SQS & SNS — Question Bank

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Type:** 140+ rapid-fire Q&A, architecture scenarios, debugging, STAR templates

---

## Table of Contents

1. Rapid-Fire Q&A (140+ questions)
2. Architecture Scenarios
3. Debugging Scenarios
4. STAR Templates

---

## 1. Rapid-Fire Q&A

### Core Concepts

**Q: What's a message broker?**
A: Middleware that receives, stores, and forwards messages between producers and consumers.

**Q: What's the difference between a queue and a topic?**
A: Queue = one-to-one (competing consumers). Topic = one-to-many (pub/sub).

**Q: What delivery guarantee is most common?**
A: At-least-once. Every message is delivered at least once (may duplicate). Consumer must be idempotent.

**Q: What's at-most-once delivery?**
A: Message may be lost but never duplicated. Highest throughput, low reliability.

**Q: What's exactly-once delivery?**
A: Message delivered exactly once. Hardest to achieve, lowest throughput.

**Q: What's the difference between push and pull models?**
A: Push: broker sends to consumer (SNS, RabbitMQ). Pull: consumer requests from broker (SQS, Kafka).

**Q: What's a dead-letter queue?**
A: Queue for messages that repeatedly fail processing. For inspection and reprocessing.

**Q: What's a poison pill message?**
A: Message that cannot be processed (malformed, wrong format). Goes to DLQ.

**Q: What's message ordering?**
A: Guarantee messages are processed in order. Types: per-partition, FIFO, best-effort.

### Kafka

**Q: What's a Kafka topic?**
A: Named channel for messages. Divided into partitions.

**Q: What's a Kafka partition?**
A: Ordered, immutable log within a topic. Unit of parallelism.

**Q: How does Kafka ordering work?**
A: Within a partition only. Messages with same key go to same partition.

**Q: What's a consumer group?**
A: Group of consumers sharing a topic. Each partition assigned to one consumer.

**Q: What happens if you have more consumers than partitions?**
A: Some consumers are idle (no partition assigned).

**Q: What's an offset?**
A: Position of a message within a partition. Consumers track offsets.

**Q: What's the difference between auto-commit and manual commit?**
A: Auto-commit = periodic commit (may lose messages). Manual = commit after processing (at-least-once).

**Q: What's a Kafka broker?**
A: Server that stores data and serves clients.

**Q: What's a Kafka cluster?**
A: Group of brokers.

**Q: What's the controller broker?**
A: One broker elected as controller — manages partition leaders.

**Q: What's ZooKeeper in Kafka?**
A: Manages cluster metadata (deprecated in KRaft mode).

**Q: What's KRaft?**
A: Kafka Raft — Kafka's own consensus protocol replacing ZooKeeper.

**Q: What's replication factor?**
A: Number of copies of each partition across brokers.

**Q: What's an ISR?**
A: In-Sync Replicas — replicas fully caught up with the leader.

**Q: What happens when a leader fails?**
A: ISR elects a new leader.

**Q: What's acks=0?**
A: Fire-and-forget — no acknowledgement from broker. Fast, can lose messages.

**Q: What's acks=1?**
A: Leader acknowledges. If leader crashes before replicating, message lost.

**Q: What's acks=-1 (all)?**
A: All ISR replicas acknowledge. Strongest durability, lowest throughput.

**Q: What's min.insync.replicas?**
A: Minimum ISR that must acknowledge for acks=-1. Prevents writes to degraded clusters.

**Q: What's Kafka retention?**
A: How long messages are kept. Time-based (default 7 days) or size-based.

**Q: What's log compaction?**
A: Retain only the latest message per key. For key-value state restore.

**Q: What's rebalancing in Kafka?**
A: When a consumer joins or leaves, partitions reassigned. Causes temporary unavailability.

**Q: What's cooperative sticky rebalancing?**
A: Incremental rebalance — only affected consumers rebalance, others continue.

**Q: What's Kafka Connect?**
A: Tool for streaming data between Kafka and other systems (DB, S3, Elasticsearch).

**Q: What's Kafka Streams?**
A: Stream processing library for Kafka (stateless and stateful operations).

**Q: What's KSQL?**
A: SQL interface for stream processing on Kafka.

**Q: What's a schema registry?**
A: Central store for message schemas. Ensures producer/consumer compatibility.

**Q: What serialization formats work with schema registry?**
A: Avro, Protobuf, JSON Schema.

**Q: What's the difference between Kafka and Kinesis?**
A: Kafka: self/managed, schema registry, Kafka Streams. Kinesis: AWS-managed, limited ecosystem.

### RabbitMQ

**Q: What's an exchange in RabbitMQ?**
A: Routing layer. Receives messages from producers, routes to queues based on exchange type and bindings.

**Q: What exchange types does RabbitMQ support?**
A: Direct, Topic, Fanout, Headers.

**Q: What's a binding?**
A: Link between an exchange and a queue with a routing key.

**Q: What's the difference between direct and topic exchanges?**
A: Direct = exact routing key match. Topic = pattern match with `*` and `#` wildcards.

**Q: What's a quorum queue?**
A: Raft-based replicated queue. Safe, consistent, recommended over mirrored queues.

**Q: What's publisher confirm?**
A: Broker acknowledges message receipt. Producer knows message isn't lost.

**Q: What's consumer ACK?**
A: Consumer acknowledges processing. NACK with requeue = retry later.

**Q: What's prefetch count?**
A: Max unacknowledged messages sent to a consumer. Controls flow.

**Q: What's a dead letter exchange (DLX)?**
A: Exchange where messages are routed after exceeding max retries or TTL.

**Q: What's RabbitMQ federation?**
A: Cross-cluster message replication (separate clusters, WAN).

**Q: What's RabbitMQ shovel?**
A: Moves messages from one broker to another (queue-to-queue).

**Q: What's the default exchange?**
A: Direct exchange with empty name. Routes to queue with same name as routing key.

**Q: Can you use RabbitMQ for pub/sub?**
A: Yes. Fanout exchange → multiple queues → multiple consumers.

**Q: What's TTL in RabbitMQ?**
A: Time-to-live on messages or queues. Expired messages are dropped or sent to DLX.

### SQS

**Q: What's the difference between SQS standard and FIFO?**
A: Standard: at-least-once, best-effort order, unlimited TPS. FIFO: exactly-once, strict order, 3,000 TPS.

**Q: What's visibility timeout in SQS?**
A: Time a message is hidden after being received. Default 30s, max 12 hours.

**Q: What's long polling vs short polling?**
A: Long polling: wait up to 20s for messages (fewer empty responses, less cost). Short polling: returns immediately.

**Q: What's a dead-letter queue in SQS?**
A: Messages that exceed maxReceiveCount go here. Retention up to 14 days.

**Q: What's the max message size in SQS?**
A: 256 KB. Use SQS Extended Client for larger messages (store in S3, reference in SQS).

**Q: How does SQS handle duplicates in FIFO?**
A: MessageDeduplicationId — identical IDs within 5-minute window are deduplicated.

**Q: What's a delay queue?**
A: Messages are hidden for a configurable delay (0–900 seconds). Not available for FIFO.

**Q: What's message batching in SQS?**
A: Send/receive/delete up to 10 messages in one API call. Reduces cost.

**Q: How does SQS guarantee exactly-once in FIFO?**
A: MessageDeduplicationId + MessageGroupId. Dedup window = 5 minutes.

**Q: What's a message group in SQS FIFO?**
A: Ensures ordered delivery within a group. Different groups can be processed in parallel.

### SNS

**Q: What's an SNS topic?**
A: Pub/sub channel. Publishers send to topic, subscribers receive.

**Q: What subscription protocols does SNS support?**
A: SQS, Lambda, email, SMS, HTTP/HTTPS, mobile push.

**Q: What's the fan-out pattern?**
A: One SNS topic → multiple SQS queues. Each queue independently consumed.

**Q: Does SNS provide message durability?**
A: No — SNS is pass-through. Durability depends on subscriber (SQS stores up to 14 days).

**Q: What's message filtering in SNS?**
A: Subscription filter policy. Only matching messages delivered to that subscriber.

**Q: What's the difference between SNS and SQS?**
A: SNS = push pub/sub (no persistence). SQS = pull queue (persistence up to 14 days).

**Q: Can SNS send to FIFO queues?**
A: Yes, using FIFO topic + FIFO SQS subscriber.

**Q: What's SNS raw message delivery?**
A: Sends message body directly (not wrapped in SNS JSON envelope). For SQS/Lambda.

### Comparison

**Q: When would you use SQS over Kafka?**
A: Lower throughput, fully managed, simpler, FIFO needed, AWS-native.

**Q: When would you use Kafka over SQS?**
A: Higher throughput, message replay, long retention, streaming, multiple consumer groups.

**Q: When would you use RabbitMQ over SQS?**
A: Complex routing (topic/direct/fanout exchanges), need to operate own broker.

**Q: When would you use SNS over Kafka?**
A: Simple pub/sub, AWS-native, push notifications, email/SMS.

**Q: What's the main trade-off between RabbitMQ and Kafka?**
A: RabbitMQ: flexible routing, push/pull, lower throughput. Kafka: high throughput, partitioned, replay, streaming.

### Advanced

**Q: What's the Transactional Outbox pattern?**
A: Write DB + outbox in same transaction, then publish events. Avoids dual-write problem.

**Q: What's the idempotent consumer pattern?**
A: Track processed message IDs. Skip duplicates. Required for at-least-once systems.

**Q: What's the claim check pattern?**
A: Store large message in S3, put reference in queue. Consumer fetches from S3.

**Q: What's the competing consumers pattern?**
A: Multiple consumers process from same queue. Scales processing horizontally.

**Q: What's priority queue?**
A: Higher priority messages processed first. Supported in RabbitMQ, not SQS.

**Q: What's a delayed message?**
A: Message visible after a delay. SQS delay queue (max 15 min). RabbitMQ TTL + DLX.

**Q: What's message deduplication in Kafka?**
A: Idempotent producer uses PID + sequence number. Broker rejects duplicates.

**Q: How do you monitor Kafka consumer lag?**
A: `kafka-consumer-groups --bootstrap-server --group --describe`. Shows current-lag per partition.

**Q: How do you reprocess messages from DLQ?**
A: SQS: redrive to source queue. RabbitMQ: republish from DLX. Kafka: reset consumer offset.

**Q: What's SLIs for messaging systems?**
A: Throughput (msg/sec), latency (p99 end-to-end), consumer lag, error rate, DLQ depth.

---

## 2. Architecture Scenarios

### Scenario 1: Trading platform event processing

**Context:** 20K+ DAU trading platform. Orders, trades, market data.

**Requirements:**
- Order execution: < 100ms, exactly-once processing
- Market data: real-time broadcast to all users (WebSocket)
- Trade history: durable, auditable
- Analytics: batch processing of daily trades

**Design:**
- **Order execution:** SQS FIFO (per-customer ordering, exactly-once)
- **Market data:** Kafka (high throughput, multiple consumer groups: WebSocket service, analytics, archive)
- **Trade history:** Kafka with infinite retention (compacted topic per instrument)
- **Analytics:** Kafka → Spark/Flink → S3 (Parquet)
- **Fan-out for notifications:** SNS → SQS (email, SMS)

### Scenario 2: Inventory SaaS async processing

**Context:** Multi-tenant Laravel inventory SaaS. S3 uploads, CSV processing, email reports, stock sync.

**Requirements:**
- File uploads (S3 → SQS → Lambda for validation)
- CSV processing (SQS → ECS task)
- Email reports (SQS → Laravel queue worker)
- Multi-channel stock sync (SNS → SQS per channel)

**Design:**
- **File uploads:** S3 event → SQS → Lambda (validate format/schema)
- **Large file processing:** SQS → ECS RunTask (Fargate, heavy processing)
- **Async jobs:** Laravel queue (SQS driver) for emails, report generation
- **Stock sync fan-out:** SNS topic → per-channel SQS queues (Amazon, eBay, web store)
- **Dead letters:** DLQ per queue → CloudWatch alarm → manual inspection → redrive

### Scenario 3: Kafka for microservices event bus

**Context:** 20 microservices, event-driven architecture.

**Requirements:**
- Cross-service events (`OrderPlaced`, `PaymentReceived`)
- Event replay for new services (catch-up)
- Schema registry for contract enforcement
- Event sourcing for some services

**Design:**
- **Event bus:** Kafka (6 brokers, replication=3, 24 partitions per topic)
- **Schema:** Avro with Schema Registry (backward compatibility)
- **Services:** Each service is a consumer group
- **Replay:** Reset offset to replay events for new consumers
- **Event sourcing:** Separate compacted topics per aggregate
- **Monitoring:** Confluent Control Center / Prometheus + Grafana

### Scenario 4: SNS fan-out for notification system

**Context:** Multi-channel notification system (email, SMS, push, Slack).

**Requirements:**
- One event → multiple channels
- Each channel independently scalable
- Filter messages per channel

**Design:**
```
Order Service → SNS Topic (order-events)
                    │
          ┌─────────┼──────────┐
          │         │          │
    ┌─────▼──┐ ┌───▼────┐ ┌───▼─────┐
    │ SQS    │ │ SQS    │ │ SQS     │
    │ Email  │ │ SMS    │ │ Slack   │
    └────┬───┘ └───┬────┘ └────┬────┘
         │         │           │
    ┌────▼──┐ ┌───▼────┐ ┌───▼─────┐
    │ Email │ │ SMS    │ │ Slack   │
    │ Worker│ │ Worker │ │ Worker  │
    └───────┘ └────────┘ └─────────┘
```

**Filtering:** SMS subscription only receives high-priority events. Email receives all. Slack receives order-related events.

---

## 3. Debugging Scenarios

### Scenario 1: Kafka consumer lag growing

**Symptom:** Consumer lag increases over time. Messages are produced faster than consumed.

**Debug:**
1. Check consumer processing time per message (add metrics)
2. Check if consumer has enough partitions (max consumers = partitions)
3. Check consumer health (stuck? rebalancing? crashed?)
4. Check downstream dependencies (DB slow? API rate limiting?)
5. Check consumer configuration (batch size, max.poll.records)

**Fix:** Add partitions → add consumers. Or optimize processing. Or both.

### Scenario 2: SQS messages not being processed

**Symptom:** Queue depth growing. Lambda doesn't process messages.

**Debug:**
1. Check Lambda event source mapping — enabled? correct queue?
2. Check Lambda concurrency — at limit? set to 0?
3. Check Lambda execution role — has SQS permissions?
4. Check Lambda logs — any errors?
5. Check DLQ — messages moved to DLQ?
6. Check visibility timeout — longer than Lambda timeout?

**Root cause:** Reserved concurrency set to 0 (previous deployment misconfiguration).

### Scenario 3: RabbitMQ message loss

**Symptom:** Under high load, some messages are not delivered.

**Debug:**
1. Check if publisher confirms enabled (producer)
2. Check queue durability (survives broker restart?)
3. Check delivery_mode (2 = persistent?)
4. Check consumer ACK (manual ACK with requeue on failure?)
5. Check disk space (broker may have stopped accepting messages)

**Root cause:** `delivery_mode = 1` (non-persistent) on production queue. Messages lost on broker restart.

### Scenario 4: Duplicate processing

**Symptom:** Some orders are processed twice.

**Debug:**
1. Check if consumer is idempotent (does it check processed IDs?)
2. Check if SQS FIFO is used (exactly-once) vs standard (at-least-once)
3. Check Lambda timeout vs visibility timeout (Lambda times out → message reappears)
4. Check consumer ACK logic (ack before or after processing?)
5. Check retry logic (consumer retries internally + broker retries)

**Root cause:** Standard SQS (at-least-once) + consumer without idempotency guard. Consumer crashed after processing but before deleting message.

### Scenario 5: RabbitMQ cluster partition

**Symptom:** Some queues unavailable. Clients intermittently fail to connect.

**Debug:**
1. Check cluster status: `rabbitmqctl cluster_status`
2. Check network between nodes (partition detected?)
3. Check queue leaders — all on one node?
4. Check quorum queue majority (minority partition disconnects)

**Root cause:** Network partition separated 2 of 3 cluster nodes. Quorum queue lost majority.

---

## 4. STAR Templates

### Template 1: Kafka consumer lag incident

**Situation:** During flash sale on trading platform, Kafka consumer lag grew to 500K messages. Order processing delay increased from 50ms to 5 minutes.

**Task:** Reduce lag and prevent recurrence.

**Action:**
```
1. Immediate:
   - Added 6 more partitions (16 → 22), redeployed consumers (increased parallelism)
   - Lag dropped 500K → 50K in 10 minutes

2. Root cause:
   - Consumer batch size was too small (10 messages per fetch)
   - Processing had a slow DB query (500ms per message)
   - Partitions were under-provisioned for peak traffic

3. Fixes:
   - Increased batch size: 10 → 500 messages
   - Optimized DB query: added index, reduced to 20ms
   - Partition calculator: peak throughput / per-partition throughput × 2
   - Added auto-scaling for consumers based on lag

4. Monitoring:
   - Added lag alert at 10K messages
   - Dashboard for consumer group lag per partition
```

**Result:** Lag never exceeds 1K messages during subsequent flash sales. Processing latency < 100ms p99.

---

### Template 2: SQS DLQ overflow

**Situation:** SQS DLQ accumulated 50K messages overnight. Orders were failing silently.

**Task:** Identify root cause and fix.

**Action:**
```
1. Inspect DLQ messages: all from same service (payment processing)
2. Check logs: payment API was returning 503 errors (maintenance window)
3. Check retry: maxReceiveCount = 3, but payment API was down for 2 hours
4. Fix: Implemented circuit breaker for payment API
   - After 5 consecutive failures → circuit open (fast fail)
   - After 30s → half-open (try one request)
   - Success → close circuit
5. Redrive: Reprocessed all 50K messages from DLQ to main queue
6. Prevention:
   - Added DLQ depth alarm at 100 messages
   - Circuit breaker prevents mass DLQ fills
   - Payment API status page integration
```

**Result:** Zero data loss. All 50K orders reprocessed. Circuit breaker prevented future mass failures.

---

### Template 3: RabbitMQ cluster migration

**Situation:** Classic mirrored RabbitMQ cluster had split-brain issues during network partitions. Needed to migrate to quorum queues.

**Task:** Zero-downtime migration from classic mirrored to quorum queues.

**Action:**
```
1. Create new quorum queues alongside existing classic queues
2. Set up shovel to forward messages from classic → quorum
3. Point consumers to quorum queues
4. Run dual consumers for a week (validate consistency)
5. Remove shovel and classic queues
6. Update IaC to use quorum queues for all new deployments
```

**Result:** Zero-downtime migration. No split-brain incidents after migration. Quorum queues provided automatic failover.
