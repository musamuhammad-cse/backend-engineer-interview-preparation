# Kafka, RabbitMQ, SQS & SNS — Intermediate Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Messaging fundamentals (queues, topics, delivery guarantees)  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. RabbitMQ Deep Dive
2. Kafka Deep Dive
3. SQS Deep Dive
4. SNS Deep Dive
5. Q&A

---

## 1. RabbitMQ Deep Dive

### Architecture

```
Producer ──► Exchange ──► (routing) ──► Queue ──► Consumer
                 │                          │
            Binding rules               ACK/NACK
```

### Exchange types

| Type | Routing | Example |
|------|---------|---------|
| **Direct** | Exact routing key | `routing_key = "order.created"` → queue bound with `order.created` |
| **Topic** | Pattern matching | `order.*` matches `order.created`, `order.cancelled` |
| **Fanout** | Broadcast to all bound queues | No routing key, all queues receive |
| **Headers** | Match on message headers | Routing based on header values (rare) |

### Topic exchange example

```php
// Producer
$channel->basic_publish(
    new AMQPMessage(json_encode($order)),
    'orders_exchange',      // exchange name
    'order.created'          // routing key
);

// Consumer binding
$channel->queue_bind('email_queue', 'orders_exchange', 'order.created');
$channel->queue_bind('sms_queue', 'orders_exchange', 'order.*');   // wildcard
$channel->queue_bind('all_queue', 'orders_exchange', '#');         // match all
```

**Wildcards:**
- `*` — matches exactly one word (e.g., `order.*` matches `order.created`)
- `#` — matches zero or more words (e.g., `order.#` matches `order.created.success`)

### Publisher confirms

```php
$channel->set_ack_handler(function (AMQPMessage $msg) {
    echo "Message confirmed: " . $msg->getBody() . PHP_EOL;
});
$channel->set_nack_handler(function (AMQPMessage $msg) {
    echo "Message lost: " . $msg->getBody() . PHP_EOL;
});
$channel->confirm_select();  // enable publisher confirms

$channel->basic_publish($msg, 'exchange', 'routing.key');
$channel->wait_for_pending_acks();  // block until broker confirms
```

### Consumer ACK/NACK

```php
$channel->basic_consume('order_queue', '', false, false, false, false, function ($msg) {
    try {
        processOrder($msg->body);
        $msg->ack();  // acknowledge — message removed from queue
    } catch (Exception $e) {
        $msg->nack(false, true);  // reject and requeue
    }
});

// prefetch — process only N messages at a time
$channel->basic_qos(null, 5, null);
```

### Durability

| Setting | Behavior |
|---------|----------|
| Queue durable = true | Queue survives broker restart |
| Message delivery_mode = 2 | Message persisted to disk |
| Queue auto-delete = true | Queue deleted when last consumer unsubscribes |
| Queue exclusive = true | Queue belongs to one connection, deleted when connection closes |

---

## 2. Kafka Deep Dive

### Architecture

```
                    ┌──────────────────┐
                    │  ZooKeeper/KRaft  │
                    │  (cluster state)  │
                    └────────┬─────────┘
                             │
Producer ──► Broker 1 ──► ──┼── ► Partition 0 (Leader)
           ┌────────┐       │    ┌─ Replica on Broker 2
           │ Broker 2│       │    └─ Replica on Broker 3
           ├────────┤   Consumer Group A
           │ Broker 3│   ┌──────────┐
           └────────┘   │Consumer 1│──► Partition 0
                        └──────────┘
                        ┌──────────┐
                        │Consumer 2│──► Partition 1
                        └──────────┘
```

### Topics and partitions

```
Topic "orders" — 3 partitions:
  Partition 0: [offset 0][offset 1][offset 2]...
  Partition 1: [offset 0][offset 1][offset 2]...
  Partition 2: [offset 0][offset 1][offset 2]...
```

- Partitions are the unit of parallelism
- Messages with the same key go to the same partition
- Ordering guaranteed within a partition

### Basic producer

```go
package main

import (
    "github.com/segmentio/kafka-go"
)

func produce() {
    writer := &kafka.Writer{
        Addr:     kafka.TCP("localhost:9092"),
        Topic:    "orders",
        Balancer: &kafka.Hash{},
        BatchSize: 100,
    }
    defer writer.Close()

    writer.WriteMessages(ctx, []kafka.Message{
        {
            Key:   []byte("customer-123"),
            Value: []byte(`{"orderId": "abc", "amount": 100}`),
        },
    })
}
```

### Consumer group

```go
func consume() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:   []string{"localhost:9092"},
        Topic:     "orders",
        GroupID:   "order-processor",
        MinBytes:  10e3,    // 10KB
        MaxBytes:  10e6,    // 10MB
        MaxWait:   1 * time.Second,
    })

    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            log.Fatal(err)
        }
        processOrder(msg.Value)
        // Commit offset automatically (auto-commit default)
    }
}
```

### Consumer group rebalancing

When a consumer joins/leaves a group, partitions are reassigned:

```
Group with 3 consumers, 6 partitions:
  Consumer A: Partition 0, 1
  Consumer B: Partition 2, 3
  Consumer C: Partition 4, 5

Consumer C crashes:
  Rebalance → Partition 4, 5 reassigned to A and B
  Consumer A: Partition 0, 1, 4
  Consumer B: Partition 2, 3, 5
```

**Impact:** During rebalance, partitions are unavailable. In some configurations, all partitions stop processing. Use `cooperative-sticky` rebalancing for incremental rebalance.

### Retention

| Strategy | Description |
|----------|-------------|
| **Time-based** | Delete messages older than N days (default 7 days) |
| **Size-based** | Delete oldest messages when topic exceeds N GB |
| **Compact** | Keep only the latest message per key (for keyed topics — useful for state restore) |

```bash
# Compacted topic — only keeps latest value per key
kafka-topics --create --topic user-profiles \
  --config cleanup.policy=compact \
  --config delete.retention.ms=86400000 \
  --bootstrap-server localhost:9092
```

### Replication

| Setting | Description |
|---------|-------------|
| `replication.factor` | Number of copies (typical: 3) |
| `min.insync.replicas` | Minimum replicas that must acknowledge writes (typical: 2) |
| `leader` | Handles reads/writes for a partition |
| **ISR** (In-Sync Replicas) | Replicas that are fully caught up with the leader |

```
Broker 1: Partition 0 (LEADER)  ──► Partition 0 (FOLLOWER): Broker 2
                                  ──► Partition 0 (FOLLOWER): Broker 3

ISR: [Broker 1, Broker 2, Broker 3]
Leader failure → ISR elects new leader from in-sync replicas
```

---

## 3. SQS Deep Dive

### Standard vs FIFO

```php
// Standard — at-least-once, best-effort order, unlimited TPS
$sqs->sendMessage([
    'QueueUrl' => 'https://sqs.us-east-1.amazonaws.com/123/orders',
    'MessageBody' => json_encode($order),
]);

// FIFO — exactly-once, strict order, 300/3,000 TPS
$sqs->sendMessage([
    'QueueUrl' => 'https://sqs.us-east-1.amazonaws.com/123/orders.fifo',
    'MessageBody' => json_encode($order),
    'MessageGroupId' => (string) $order['customer_id'],  // per-customer ordering
    'MessageDeduplicationId' => $order['order_id'],       // dedup within 5-min window
]);
```

### Receiving and processing

```php
// Receive messages (long polling)
$result = $sqs->receiveMessage([
    'QueueUrl' => 'https://sqs.us-east-1.amazonaws.com/123/orders',
    'MaxNumberOfMessages' => 10,
    'WaitTimeSeconds' => 20,        // long polling (1–20s)
    'VisibilityTimeout' => 30,      // hide from other consumers
]);

foreach ($result['Messages'] as $message) {
    try {
        processMessage($message['Body']);
        $sqs->deleteMessage([
            'QueueUrl' => 'https://sqs.us-east-1.amazonaws.com/123/orders',
            'ReceiptHandle' => $message['ReceiptHandle'],
        ]);
    } catch (RetryableException $e) {
        // Don't delete — message becomes visible again after visibility timeout
    } catch (PermanentFailure $e) {
        // Optionally: call ChangeMessageVisibility to send to DLQ faster
    }
}
```

### Visibility timeout

```
Consumer receives message → message hidden (default 30s)
  ├── Consumer calls DeleteMessage → message deleted
  ├── Consumer crashes → message reappears after 30s
  └── Consumer calls ChangeMessageVisibility → extends timeout
```

**How visibility timeout works with Lambda:**

```
Lambda receives message → visibility timeout starts
  ├── Lambda succeeds → SQS deletes message
  └── Lambda fails/times out → message becomes visible again (retry)
```

> **Trap:** If your Lambda takes longer than the visibility timeout, the message becomes visible again while Lambda is still processing. Set visibility timeout >= Lambda timeout + 30s buffer.

### Dead-letter queue

```yaml
# Terraform — SQS with DLQ
resource "aws_sqs_queue" "orders_dlq" {
  name = "orders-dlq"
  message_retention_seconds = 1209600  # 14 days
}

resource "aws_sqs_queue" "orders" {
  name = "orders-queue"

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 5
  })
}
```

---

## 4. SNS Deep Dive

### SNS Topic

```php
// Create topic
$result = $sns->createTopic(['Name' => 'order-events']);
$topicArn = $result['TopicArn'];

// Subscribe SQS queue to topic
$sns->subscribe([
    'TopicArn' => $topicArn,
    'Protocol' => 'sqs',
    'Endpoint' => 'arn:aws:sqs:us-east-1:123456789012:order-processor',
]);

// Subscribe email
$sns->subscribe([
    'TopicArn' => $topicArn,
    'Protocol' => 'email',
    'Endpoint' => 'admin@example.com',
]);

// Publish to topic
$sns->publish([
    'TopicArn' => $topicArn,
    'Message' => json_encode($order),
    'Subject' => 'New Order',
    'MessageAttributes' => [
        'event_type' => [
            'DataType' => 'String',
            'StringValue' => 'order_created',
        ],
        'priority' => [
            'DataType' => 'Number',
            'StringValue' => '1',
        ],
    ],
]);
```

### Fan-out pattern

```
Order Service publishes to SNS Topic
                │
            SNS Topic
          ┌─────┼─────┐
          │     │     │
    ┌─────▼┐ ┌─▼──┐ ┌▼────┐
    │ SQS  │ │SQS │ │SQS  │
    │Order │ │Email│ │Anal │
    │Svc   │ │Svc  │ │Svc  │
    └──────┘ └────┘ └─────┘
```

Each consumer has its own queue. The Order Service publishes once; all three queues receive independently.

### Message filtering

```json
// SQS subscription with filter policy
{
  "event_type": ["order_created", "order_shipped"],
  "priority": [{"numeric": [">=", 1]}]
}
```

Messages that don't match the filter are NOT delivered to that subscription.

---

## 5. Q&A

**Q: What's an exchange in RabbitMQ?**
A: Routing layer — receives messages from producers and routes to queues based on exchange type and binding rules.

**Q: What exchange types does RabbitMQ support?**
A: Direct (exact routing key), Topic (pattern matching), Fanout (broadcast), Headers (header values).

**Q: What's a Kafka partition?**
A: Unit of parallelism within a topic. Each partition is an ordered, immutable log. Partitions are distributed across brokers.

**Q: How does Kafka guarantee ordering?**
A: Within a partition. Messages with the same key go to the same partition. No ordering guarantee across partitions.

**Q: What's a consumer group in Kafka?**
A: Group of consumers sharing work. Each partition is assigned to one consumer in the group. Enables horizontal scaling.

**Q: What's the difference between SQS standard and FIFO?**
A: Standard: at-least-once, best-effort order, unlimited TPS. FIFO: exactly-once, strict order, 3,000 TPS max.

**Q: What's visibility timeout in SQS?**
A: Time a message is hidden from other consumers after being received. Used to prevent duplicate processing.

**Q: What's the fan-out pattern in SNS?**
A: One SNS topic publishes to multiple SQS queues. Each consumer gets every message independently.

**Q: What's the difference between Kafka and SQS?**
A: Kafka: pull-based, high throughput, retention-based, partitioned, needs operational management. SQS: fully managed, limited retention (14 days), scalable without tuning.

**Q: What's publisher confirm in RabbitMQ?**
A: Broker confirms receipt of a message. Producer waits for confirmation — ensures message wasn't lost before reaching broker.
