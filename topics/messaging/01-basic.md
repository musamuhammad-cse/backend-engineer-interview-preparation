# Kafka, RabbitMQ, SQS & SNS — Basic Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — ground-level messaging concepts  
> **Estimated time:** 4–6 hours

---

## Table of Contents

1. Messaging Fundamentals
2. Delivery Guarantees
3. Push vs Pull
4. Pub/Sub vs Competing Consumers
5. Message Ordering
6. Dead-Letter Queues
7. Q&A

---

## 1. Messaging Fundamentals

### Core concepts

| Term | Description |
|------|-------------|
| **Message** | Unit of data exchanged between services |
| **Producer** | Service that sends messages |
| **Consumer** | Service that receives messages |
| **Broker** | Intermediate service that stores and forwards messages |
| **Queue** | Point-to-point channel (one consumer processes each message) |
| **Topic** | Pub/sub channel (all subscribers receive each message) |
| **Exchange** | Routing layer (RabbitMQ) — receives messages and routes to queues |

### Architecture

```
Producer ──► Broker ──► Consumer
               │
               ├── Store (disk/memory)
               ├── Route (to correct queue)
               └── Deliver (push or wait for pull)
```

### Why message queues?

| Reason | Problem it solves |
|--------|-------------------|
| **Decoupling** | Producer doesn't need to know about consumers |
| **Load leveling** | Buffer spikes (queue absorbs bursts) |
| **Async processing** | Don't block the request while processing |
| **Reliability** | Messages survive producer/consumer crashes |
| **Scalability** | Add more consumers to increase throughput |

### When to use a message queue

- Background job processing (sending emails, processing images)
- Cross-service communication in microservices
- Event-driven workflows (order → payment → shipping)
- Buffering high-throughput data (metrics, logs, clickstreams)

> **Trap:** Don't add a message broker for simple request-response. If the consumer needs to reply synchronously, use HTTP/gRPC. Message queues add latency and complexity.

---

## 2. Delivery Guarantees

### At-most-once

Message is delivered zero or one time (may be lost). **Highest throughput, lowest reliability.**

```
Producer → Broker → Consumer
          ↑
     Message may be lost before consumer gets it
```

**Use case:** Monitoring metrics, logging — losing one data point is acceptable.

### At-least-once

Message is delivered one or more times (may be duplicated). **Consumer must be idempotent.**

```
Producer → Broker → Consumer (ACK)
          ↑           ↓ failure
          └───────────┘ retry
```

**Use case:** Payment processing (you can handle duplicates with idempotency keys). Most common choice.

### Exactly-once

Message is delivered exactly once. **Hardest to achieve, lowest throughput.**

```
Producer → Broker → Consumer
          │
     Idempotent producer + transactional broker + idempotent consumer
```

**Use case:** Financial transactions, inventory counts. Often not worth the cost — at-least-once + idempotent consumer is good enough.

---

## 3. Push vs Pull

### Push (SNS, RabbitMQ)

Broker sends messages to consumers as soon as they arrive.

```
Broker ──────────► Consumer 1
       ──────────► Consumer 2
```

**Pros:** Low latency (message delivered immediately)  
**Cons:** Consumer can be overwhelmed (backpressure), broker must track consumer state

### Pull (SQS, Kafka)

Consumer requests messages from broker.

```
Consumer 1 ──► Broker (poll)
Consumer 2 ──► Broker (poll)
```

**Pros:** Consumer controls processing rate, easier backpressure  
**Cons:** Higher latency (polling interval), empty polls waste resources

### Comparison

| Aspect | Push | Pull |
|--------|------|------|
| Latency | Lower | Higher (poll interval) |
| Backpressure | Hard (need credits/backoff) | Natural (just don't poll) |
| Broker complexity | Higher (tracks consumers) | Lower |
| Consumer complexity | Lower | Higher |
| Example | SNS, RabbitMQ | SQS, Kafka |

---

## 4. Pub/Sub vs Competing Consumers

### Pub/Sub (Topic)

```
Producer ──► Topic ──┬──► Consumer A (gets all messages)
                     ├──► Consumer B (gets all messages)
                     └──► Consumer C (gets all messages)
```

**Each consumer receives every message.** Use for broadcasting.

**Examples:** SNS topic, Kafka topic (different consumer groups), RabbitMQ topic exchange

### Competing Consumers (Queue)

```
Producer ──► Queue ──┬──► Worker 1
                     ├──► Worker 2
                     └──► Worker 3
```

**Each message is consumed once.** Use for workload distribution.

**Examples:** SQS queue, RabbitMQ queue, Kafka consumer group

### Choosing between them

| Pattern | Use when |
|---------|----------|
| Pub/Sub | Multiple independent consumers need every event |
| Competing consumers | Scale processing of a single workload |

---

## 5. Message Ordering

### Types of ordering

| Ordering | Description | Example |
|----------|-------------|---------|
| **No ordering** | Any order | SQS standard |
| **Best-effort** | Usually in order but not guaranteed | SQS standard |
| **Per-partition order** | Ordered within a partition (Kafka) | Kafka |
| **Strict FIFO** | Global ordering for entire queue | SQS FIFO, RabbitMQ single queue |

### Partition-based ordering (Kafka)

```
Topic "orders" with 3 partitions:
┌──────────────┐
│ Partition 0  │  ← Order 1, Order 4, Order 7
├──────────────┤
│ Partition 1  │  ← Order 2, Order 5, Order 8
├──────────────┤
│ Partition 2  │  ← Order 3, Order 6, Order 9
└──────────────┘

Ordering guaranteed WITHIN a partition, NOT across partitions.
Messages with same key go to same partition (e.g., key=customer_id).
```

### FIFO (SQS FIFO)

```php
// All messages in FIFO queue are strictly ordered
// Per-message-group ordering
$sqs->sendMessage([
    'QueueUrl' => 'https://sqs.us-east-1.amazonaws.com/123/orders.fifo',
    'MessageBody' => json_encode($order),
    'MessageGroupId' => (string) $order['customer_id'],  // per-group ordering
    'MessageDeduplicationId' => $order['order_id'],       // exactly-once
]);
```

---

## 6. Dead-Letter Queues

### Concept

A DLQ stores messages that cannot be processed successfully:

```
Main Queue ──► (maxReceiveCount exceeded) ──► DLQ
                    │
               Message processed? ──► Delete from main queue
                    │ No
               retry ──► maxReceiveCount exceeded ──► DLQ
```

### When messages end up in DLQ

- Consumer throws exception repeatedly
- Message processing times out (visibility timeout exceeded)
- Message is malformed (poison pill)
- External dependency unavailable

### DLQ management

```
DLQ Monitoring ──► Alert (CloudWatch alarm on DLQ depth)
                       │
                  Manual inspection ──► Fix bug ──► Redrive to main queue
                       │
                  Auto-redrive (after fix deployed)
```

> **Trap:** DLQ is not automatic cleanup. Messages sit in DLQ indefinitely unless you:
> - Set retention period on DLQ
> - Build a redrive mechanism
> - Monitor and process manually

---

## 7. Q&A

**Q: What's the difference between a queue and a topic?**
A: Queue = point-to-point (one consumer per message). Topic = pub/sub (all subscribers get all messages).

**Q: What delivery guarantee does SQS standard provide?**
A: At-least-once. Messages may be delivered more than once. Consumer must be idempotent.

**Q: What's the difference between at-least-once and exactly-once?**
A: At-least-once: message may be duplicated. Exactly-once: message delivered exactly once. Exactly-once is harder to achieve and has lower throughput.

**Q: What's the difference between push and pull messaging?**
A: Push: broker sends to consumer (SNS, RabbitMQ). Pull: consumer requests from broker (SQS, Kafka).

**Q: What's a dead-letter queue?**
A: A queue that stores messages that repeatedly fail processing. For inspection and reprocessing.

**Q: What's message ordering?**
A: Guarantee that messages are processed in the order they were sent. SQS FIFO = strict global order. Kafka = per-partition order. SQS standard = best-effort.

**Q: What's a consumer group in Kafka?**
A: A group of consumers that cooperate to consume from a topic. Each partition is assigned to one consumer in the group.

**Q: What's the difference between competing consumers and pub/sub?**
A: Competing consumers: each message consumed once (work queue). Pub/sub: all consumers get all messages (broadcast).
