# Kafka, RabbitMQ, SQS & SNS — Senior Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Intermediate familiarity with Kafka, RabbitMQ, SQS, SNS  
> **Estimated time:** 10–12 hours

---

## Table of Contents

1. Kafka Performance Tuning
2. Kafka Exactly-Once Semantics
3. RabbitMQ Clustering and Quorum Queues
4. Message Broker Comparison
5. Schema Registry and Serialization
6. Advanced Consumer Patterns
7. Q&A

---

## 1. Kafka Performance Tuning

### Partition count

```
Too few partitions:
  - Limited parallelism (max consumers = partitions)
  - Each partition grows large (rebalance slow)

Too many partitions:
  - More files open on brokers
  - Rebalance takes longer
  - Higher memory usage
  - Client-side metadata requests grow

Rule of thumb: partitions = max throughput / throughput per partition
                OR partitions = max consumers × 2 (for headroom)
```

### Producer tuning

```go
writer := &kafka.Writer{
    Addr:     kafka.TCP(brokers...),
    Topic:    "orders",
    Balancer: &kafka.Hash{},

    // Batch size — larger = higher throughput, higher latency
    BatchSize:  100,         // messages per batch
    BatchBytes: 1_048_576,   // 1 MB max batch size
    BatchTimeout: 10 * time.Millisecond,

    // Compression — reduces network I/O, increases CPU
    Compression: kafka.Snappy,

    // ACKs:
    // 0 = fire-and-forget (fast, may lose messages)
    // 1 = leader acknowledges (default, good for most)
    // -1 = all ISR acknowledges (slowest, safest)
    RequiredAcks: kafka.RequiresAll,  // acks=-1
}
```

| `acks` setting | Durability | Throughput | Use case |
|----------------|------------|------------|----------|
| `0` | None | Highest | Monitoring, metrics (can lose some) |
| `1` | Leader only | High | Most applications |
| `-1` (all) | All ISR | Lower | Critical data (payments, inventory) |

### Consumer tuning

```go
reader := kafka.NewReader(kafka.ReaderConfig{
    GroupID:     "order-processor",
    MinBytes:    10_000,      // fetch at least 10KB
    MaxBytes:    10_000_000,  // fetch up to 10MB
    MaxWait:     1 * time.Second,
    HeartbeatInterval: 3 * time.Second,
    SessionTimeout:   45 * time.Second,
    RebalanceTimeout: 60 * time.Second,

    // Auto-commit vs manual commit
    CommitInterval: 0,  // 0 = manual commit (recommended for at-least-once)
})
```

| Consumer param | Description | Recommendation |
|---------------|-------------|----------------|
| `MinBytes` | Min data per fetch | 10KB–1MB (higher = better throughput) |
| `MaxWait` | Max wait if not enough data | 500ms–1s (higher = lower latency for low-throughput topics) |
| `HeartbeatInterval` | Heartbeat to coordinator | 3s (stay alive) |
| `SessionTimeout` | Detection of consumer failure | 45s (shorter = faster rebalance) |

### Rebalance tuning

Use **cooperative sticky** rebalancing (incremental rebalance):

```properties
# consumer.properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

Standard (eager) rebalancing: ALL consumers stop during rebalance.  
Cooperative: only affected consumers stop — others continue.

### Monitoring Kafka

| Metric | What it measures | Alert threshold |
|--------|-----------------|-----------------|
| `UnderReplicatedPartitions` | Partitions with fewer replicas than configured | > 0 |
| `ActiveControllerCount` | Should be 1 | != 1 |
| `RequestHandlerAvgIdlePercent` | Broker capacity | < 20% |
| `BytesInPerSec` / `BytesOutPerSec` | Throughput | Normal baseline |
| `TotalTimeMs` | End-to-end latency | Per-SLA |
| `OfflinePartitionsCount` | Partitions with no leader | > 0 (critical) |

---

## 2. Kafka Exactly-Once Semantics

### Idempotent producer

```go
writer := &kafka.Writer{
    Addr:         kafka.TCP(brokers...),
    Topic:        "orders",
    Idempotent:   true,   // ensures no duplicates
    RequiredAcks: kafka.RequiresAll,  // must be all
}
```

How it works:
- Producer assigns a `producer.id` (PID) and sequence number per partition
- Broker deduplicates messages with same PID + sequence number
- Retries automatically — identical messages are rejected as duplicates

### Transactions

For exactly-once across multiple partitions/topics:

```go
writer := &kafka.Writer{
    Addr:        kafka.TCP(brokers...),
    Transactional: true,
}

writer.BeginTransaction()
writer.WriteMessages(ctx, msg1, msg2)
writer.CommitTransaction(ctx)  // commit atomically

// On failure:
writer.AbortTransaction(ctx)
```

**Use case:** Read from Kafka → process → write to Kafka (exactly-once stream processing)

---

## 3. RabbitMQ Clustering and Quorum Queues

### Quorum queues (recommended)

Replaces classic mirrored queues. Based on Raft consensus.

```bash
# Declare quorum queue
rabbitmqadmin declare queue name=orders.queue durable=true \
  arguments='{"x-queue-type": "quorum"}'

# Or via API
curl -u guest:guest -X PUT \
  http://localhost:15672/api/queues/%2f/orders.queue \
  -d '{"durable": true, "arguments": {"x-queue-type": "quorum"}}'
```

| Feature | Classic mirrored | Quorum |
|---------|-----------------|--------|
| Consensus | Asynchronous | Raft (synchronous) |
| Data safety | May lose messages during failover | Stronger guarantees |
| Throughput | Higher | Slightly lower |
| Partition handling | Split-brain | Majority decision |
| Recommended | Legacy | New deployments |

### Publisher confirms with quorum queues

```php
$channel->confirm_select();

$channel->basic_publish(
    new AMQPMessage($body, ['delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT]),
    'exchange',
    'routing.key'
);

$channel->wait_for_pending_acks(5);  // wait up to 5 seconds
```

### Cluster architecture

```bash
# Join node to cluster
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

---

## 4. Message Broker Comparison

### Detailed comparison

| | Kafka | RabbitMQ | SQS | SNS |
|---|---|---|---|---|
| **Type** | Distributed log | Message broker | Managed queue | Managed pub/sub |
| **Model** | Pull | Push/Pull | Pull | Push |
| **Delivery** | At-least-once / Exactly-once | At-most-once / At-least-once | At-least-once (Std) / Exactly-once (FIFO) | At-least-once |
| **Ordering** | Per-partition | Per-queue | Best-effort (Std) / FIFO | None |
| **Throughput** | Millions/sec | Thousands/sec | Unlimited (Std) / 3K TPS (FIFO) | High |
| **Persistence** | Configurable retention (infinite) | Configurable | 14 days max | None |
| **Message size** | Default 1MB | Default 128MB | 256KB (extended via S3) | 256KB |
| **Consumer model** | Consumer groups | Competing consumers / Pub/sub | Competing consumers | Push to subscribers |
| **Replay** | Yes (by offset) | No (consumed = removed) | No (DLQ only) | No |
| **Operations** | High (ZooKeeper/KRaft) | Medium | None (managed) | None (managed) |
| **Cost** | Infrastructure | Infrastructure | Per-request | Per-request |
| **Best for** | Event streaming, log aggregation, high throughput | Complex routing, task queues | Simple queuing, AWS-native | Notifications, fan-out |

### When to use what

```
Use Kafka when:
  - High throughput (100K+ msg/sec)
  - Event streaming / log aggregation
  - Message replay needed
  - Multiple consumers with independent consumption
  - Long-term retention

Use RabbitMQ when:
  - Complex routing (topic/direct/fanout exchanges)
  - Flexible consumer patterns
  - Need for advanced broker features (dead-lettering, priority)
  - Moderate throughput (< 50K msg/sec)
  - You want to operate your own broker

Use SQS when:
  - Simple queue (no complex routing)
  - Fully managed (no ops)
  - AWS-native architecture
  - Need FIFO ordering
  - Throughput within limits (3K TPS for FIFO)

Use SNS when:
  - Pub/sub pattern (one event → many consumers)
  - Notifications (email, SMS, push)
  - Fan-out to multiple SQS queues
  - Event-driven architecture with minimal complexity
```

---

## 5. Schema Registry and Serialization

### Why schema registry

Without schema registry:
- Producer changes message format → consumer breaks
- No validation of message format
- Hard to evolve schema over time

With schema registry:
- Schemas stored centrally and versioned
- Producer validates against schema
- Consumer reads schema from registry (no need to know in advance)
- Schema evolution rules (backward/forward compatibility)

### Avro with Kafka

```go
// Producer with Avro
type Order struct {
    OrderID   string  `avro:"order_id"`
    CustomerID string `avro:"customer_id"`
    Total     float64 `avro:"total"`
}

schema := `{
  "type": "record",
  "name": "Order",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "total", "type": "double"}
  ]
}`

// Schema Registry stores the schema
// Messages include schema ID in the first bytes
// Consumers fetch schema by ID from registry
```

### Compatibility rules

| Rule | Old schema | New schema | Allowed? |
|------|-----------|------------|----------|
| Backward | Has field A | Field A optional with default | ✓ Consumer can read old data |
| Forward | Field A optional | Field A removed | ✓ Old consumer can read new data |
| Full | Both | Both | ✓ |
| None | Anything | Anything | ✗ May break |

---

## 6. Advanced Consumer Patterns

### Idempotent consumer

```php
class IdempotentOrderProcessor
{
    private Redis $processed;  // track processed message IDs

    public function __construct()
    {
        $this->processed = new Redis();  // set with TTL matching dedup window
    }

    public function handle(Message $msg): void
    {
        $messageId = $msg->getMessageId();

        // 1. Check if already processed (idempotency guard)
        if ($this->processed->exists($messageId)) {
            $msg->ack();  // already done — acknowledge and skip
            return;
        }

        // 2. Process (must be atomic with step 3)
        $this->processOrder($msg->getBody());

        // 3. Mark as processed
        $this->processed->setex($messageId, 3600, true);

        // 4. Acknowledge
        $msg->ack();
    }
}
```

### Competing consumers with Kafka

Use consumer groups for competing consumers pattern:

```go
// Multiple consumers in same group — each gets different partitions
reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers: []string{"broker:9092"},
    Topic:    "orders",
    GroupID:  "order-processor",  // same group = competing consumers
})
```

### Priority queue (RabbitMQ)

```bash
# Declare priority queue
rabbitmqadmin declare queue name=orders.priority durable=true \
  arguments='{"x-max-priority": 10}'
```

```php
// Producer — set priority
$msg = new AMQPMessage(json_encode($order), [
    'priority' => $order['is_premium'] ? 10 : 1,
]);
$channel->basic_publish($msg, 'exchange', 'routing.key');
```

### Delayed message (SQS delay queue)

```php
$sqs->sendMessage([
    'QueueUrl' => 'https://sqs.us-east-1.amazonaws.com/123/orders',
    'MessageBody' => json_encode($order),
    'DelaySeconds' => 300,  // message visible after 5 minutes
]);
```

Max 15 minutes. For longer delays, use a separate queue with Lambda to redrive.

---

## 7. Q&A

**Q: How do you choose the number of Kafka partitions?**
A: At least as many as max consumers in the group + headroom. More partitions = more parallelism but higher overhead. Rule of thumb: start with 3x broker count.

**Q: What's the difference between acks=1 and acks=-1 in Kafka?**
A: acks=1: leader acknowledges (if leader fails before replicating, message lost). acks=-1: all ISR replicas acknowledge (stronger durability, lower throughput).

**Q: What's a quorum queue in RabbitMQ?**
A: Raft-based replicated queue. Replaces classic mirrored queues. Stronger consistency, safe automatic failover.

**Q: When would you use SQS FIFO over Kafka?**
A: When you need strict FIFO ordering (not per-partition), fully managed infrastructure, and throughput under 3,000 TPS.

**Q: What's the Transactional Outbox pattern?**
A: Write to DB + outbox table in same transaction. Separate process (CDC or poller) publishes events to message broker. Ensures at-least-once delivery without dual-write issues.

**Q: How do you handle message schema evolution?**
A: Schema Registry with Avro/Protobuf. Backward compatibility for old consumers. Forward compatibility for old producers.

**Q: What's the difference between Kafka and Kinesis?**
A: Kafka: self-hosted or MSK, schema registry, Kafka Streams, broad ecosystem. Kinesis: AWS-managed, limited to AWS ecosystem, lower throughput per shard, shard management.

**Q: What's the idempotent consumer pattern?**
A: Store processed message IDs (Redis with TTL). On receiving a message, check if already processed. If yes, acknowledge without processing. Critical for at-least-once systems.

**Q: How do you debug a Kafka consumer lag?**
A: (1) Check consumer group offset lag (kafka-consumer-groups). (2) Check consumer processing time per message. (3) Check if consumers are stuck (rebalancing? dead?). (4) Add more consumers (max = partitions). (5) Optimize processing.

**Q: What's the difference between RabbitMQ topic exchange and Kafka topics?**
A: RabbitMQ topic: routing key pattern matching, push delivery. Kafka topic: partitioned log, pull delivery, offset tracking, replay capability.
