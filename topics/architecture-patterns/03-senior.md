# Architecture Patterns — Senior Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Microservices, Event-Driven, DDD fundamentals  
> **Estimated time:** 10–12 hours

---

## Table of Contents

1. CQRS (Command Query Responsibility Segregation)
2. Event Sourcing
3. Saga Pattern
4. Strangler Fig Pattern
5. Domain Events
6. Transactional Outbox Pattern
7. Architecture Decision Records
8. Trade-Off Analysis
9. Q&A

---

## 1. CQRS (Command Query Responsibility Segregation)

### Concept

Separate read and write models:

```
┌──────────────────────────────────────────────────────┐
│  Write Side (Commands)     │  Read Side (Queries)     │
│                            │                          │
│  ┌──────────────────┐     │  ┌──────────────────┐     │
│  │ Command Handler  │     │  │ Query Handler    │     │
│  │ (validates,      │     │  │ (returns data,   │     │
│  │  mutates state)  │     │  │  no side effects)│     │
│  └────────┬─────────┘     │  └────────┬─────────┘     │
│           │               │           │               │
│  ┌────────▼─────────┐     │  ┌────────▼─────────┐     │
│  │ Write DB         │     │  │ Read DB           │     │
│  │ (normalized)     │─────│──► (denormalized)    │     │
│  └──────────────────┘     │  └──────────────────┘     │
└──────────────────────────────────────────────────────┘
```

### When to use CQRS

| Use CQRS when | DON'T use CQRS when |
|---------------|---------------------|
| Different read/write patterns | Simple CRUD (same model works) |
| Complex business logic on writes | Write model is straightforward |
| Denormalized read models needed | Read model matches write model |
| High read throughput requirements | Low traffic |
| Multiple read representations | One representation |

### CQRS without Event Sourcing (simple form)

```php
// Command side
class PlaceOrderCommand
{
    public function __construct(
        public readonly string $customerId,
        public readonly array $items,
    ) {}
}

class PlaceOrderHandler
{
    public function __construct(
        private OrderRepository $orders,
        private EventBus $events,
    ) {}

    public function handle(PlaceOrderCommand $command): void
    {
        $order = new Order(CustomerId::fromString($command->customerId));

        foreach ($command->items as $item) {
            $order->addProduct(
                ProductId::fromString($item['productId']),
                Quantity::fromInt($item['quantity']),
            );
        }

        $order->place();

        $this->orders->save($order);
        $this->events->publish(new OrderPlaced($order->id));
    }
}

// Query side
class OrderProjection
{
    public function __construct(
        private DB $readDb,  // denormalized table
    ) {}

    public function onOrderPlaced(OrderPlaced $event): void
    {
        $this->readDb->insert('order_summaries', [
            'id' => $event->orderId,
            'status' => 'placed',
            'total' => $event->total,
            'created_at' => now(),
        ]);
    }
}

class GetOrderSummaryQuery
{
    public function __construct(private DB $readDb) {}

    public function handle(string $orderId): array
    {
        return $this->readDb->fetch(
            'SELECT * FROM order_summaries WHERE id = ?',
            [$orderId]
        );
    }
}
```

### CQRS challenges

- **Eventual consistency** — writes are immediately visible in write model but read model lags
- **Complexity** — two models to maintain, projection logic, event handling
- **Operational overhead** — two databases, ETL-like projection processes

> **Trap:** CQRS adds significant complexity. Don't apply it unless you have clear different read/write patterns. Many teams use CQRS for small sections of a large system (e.g., reporting read models) rather than the entire system.

---

## 2. Event Sourcing

### Concept

Store events as the source of truth. Current state is derived by replaying events.

```
Events (append-only log):
  ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────┐
  │Order   │ │Order   │ │Inventory │ │Payment │
  │Placed  │ │Paid    │ │Reserved  │ │Captured│
  └────────┘ └────────┘ └──────────┘ └────────┘
                  │
                  ▼ replay
          ┌──────────────┐
          │ Current      │
          │ State (Order)│
          │ status=paid  │
          │ total=$100   │
          └──────────────┘
```

### Implementation

```php
// Event
class OrderPlaced
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $customerId,
        public readonly array $items,
        public readonly int $total,
        public readonly DateTimeImmutable $occurredAt,
    ) {}
}

// Aggregate that rebuilds from events
class Order
{
    private string $id;
    private string $status = 'draft';
    private array $items = [];
    private int $total = 0;

    // Command handler — validates, creates events
    public static function place(string $customerId, array $items): self
    {
        $order = new self();
        $order->apply(new OrderPlaced(
            orderId: Uuid::generate(),
            customerId: $customerId,
            items: $items,
            total: self::calculateTotal($items),
            occurredAt: new DateTimeImmutable(),
        ));
        return $order;
    }

    // Apply an event (state change)
    public function apply(object $event): void
    {
        match ($event::class) {
            OrderPlaced::class => $this->applyOrderPlaced($event),
            OrderPaid::class => $this->applyOrderPaid($event),
            default => throw new DomainException('Unknown event: ' . $event::class),
        };
    }

    private function applyOrderPlaced(OrderPlaced $event): void
    {
        $this->id = $event->orderId;
        $this->status = 'placed';
        $this->items = $event->items;
        $this->total = $event->total;
    }

    private function applyOrderPaid(OrderPaid $event): void
    {
        if ($this->status !== 'placed') {
            throw new DomainException('Can only pay placed orders');
        }
        $this->status = 'paid';
    }
}

// Event store
interface EventStore
{
    public function save(string $aggregateId, int $expectedVersion, array $events): void;
    public function getEvents(string $aggregateId): array;
}

// Rebuilding aggregate from events
class OrderRepository
{
    public function __construct(private EventStore $events) {}

    public function find(string $orderId): Order
    {
        $order = new Order();
        $orderEvents = $this->events->getEvents($orderId);

        foreach ($orderEvents as $event) {
            $order->apply($event);
        }

        return $order;
    }
}
```

### Event versioning

Events live forever. Schema must evolve:

```php
// Version 1
class OrderPlacedV1
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $customerId,
        public readonly string $productId,  // single product
        public readonly int $quantity,
    ) {}
}

// Version 2 — multiple items
class OrderPlacedV2
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $customerId,
        public readonly array $items,  // replaces productId + quantity
    ) {}
}

// Upcaster — converts V1 to V2 on read
class OrderPlacedUpcaster
{
    public function upcast(array $eventData): array
    {
        if (!isset($eventData['items'])) {
            // Convert V1 format to V2
            $eventData['items'] = [[
                'productId' => $eventData['productId'],
                'quantity' => $eventData['quantity'],
            ]];
            unset($eventData['productId'], $eventData['quantity']);
        }
        return $eventData;
    }
}
```

### Benefits and drawbacks

| Benefit | Drawback |
|---------|----------|
| Complete audit trail | Event versioning complexity |
| Temporal queries (state at any point in time) | Querying current state requires replay |
 | Event-driven integrations | Event store operational complexity |
| Strong consistency within aggregate | Snapshot maintenance for performance |
| No object-relational impedance mismatch | Learning curve |

---

## 3. Saga Pattern

### Choreography Saga

Each service publishes events that trigger the next step.

```
Order Service ──► OrderCreated ──► Payment Service
                                      │
                                  PaymentSuccess
                                      │
                                      ▼
Inventory Service ◄── PaymentSuccess
        │
  InventoryReserved
        │
        ▼
Shipping Service ◄── InventoryReserved
```

**Compensation on failure:**

```
Payment Service ──► PaymentFailed ──► Order Service
                                      │
                                  OrderCancelled
                                      │
                                      ▼
                              Notification Service
```

**Pros:** Simple, no central coordinator. **Cons:** Hard to trace, logic distributed across services.

### Orchestration Saga

A central coordinator tells each service what to do.

```
┌────────────────────────────────────────────┐
│           Saga Orchestrator                │
│                                            │
│  1. CreateOrder   ──► Order Service        │
│  2. ProcessPayment ──► Payment Service      │
│  3. ReserveInventory ─► Inventory Service   │
│  4. ShipOrder     ──► Shipping Service      │
│                                            │
│  On failure: call compensating actions      │
│  3. CancelInventoryReservation              │
│  2. RefundPayment                           │
│  1. MarkOrderAsFailed                       │
└────────────────────────────────────────────┘
```

### Implementation

```php
class CreateOrderSaga
{
    public function __construct(
        private CommandBus $commands,
        private EventBus $events,
    ) {}

    #[Asynchronous]
    public function handle(OrderCreated $event): void
    {
        $state = new SagaState($event->orderId);

        try {
            // Step 1: Process payment
            $this->commands->dispatch(
                new ProcessPayment($event->orderId, $event->total)
            );
            $state->paymentProcessed = true;

            // Step 2: Reserve inventory
            $this->commands->dispatch(
                new ReserveInventory($event->orderId, $event->items)
            );
            $state->inventoryReserved = true;

            // Step 3: Schedule shipping
            $this->commands->dispatch(
                new ScheduleShipping($event->orderId, $event->customerId)
            );

            // Step 4: Confirm order
            $this->events->publish(new OrderConfirmed($event->orderId));
        } catch (Exception $e) {
            // Compensate in reverse order
            if ($state->inventoryReserved) {
                $this->commands->dispatch(
                    new CancelInventoryReservation($event->orderId)
                );
            }
            if ($state->paymentProcessed) {
                $this->commands->dispatch(
                    new RefundPayment($event->orderId)
                );
            }
            $this->events->publish(new OrderFailed($event->orderId, $e->getMessage()));
        }
    }
}
```

### Saga comparison

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| Coupling | Lower (events only) | Higher (knows coordinator) |
| Traceability | Harder | Easier (central coordinator) |
| Complexity for small flows | Simple | Over-engineered |
| Complexity for complex flows | Hard to reason about | Organized |
| Failure handling | Distributed (each service compensates) | Centralized (coordinator) |

---

## 4. Strangler Fig Pattern

### Concept

Gradually replace a monolithic application with microservices. Route new features to new services. Eventually, all functionality is migrated.

```
Phase 1:                    Phase 2:                    Phase 3:
┌──────────┐                ┌──────────┐                ┌──────────┐
│ Monolith │                │ Monolith │                │          │
│ (100%)   │                │ (70%)    │                │ (0%)     │
└──────────┘                └─────┬────┘                └──────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
              ┌─────▼──┐   ┌─────▼──┐   ┌─────▼──┐
              │ New     │   │ New    │   │ New     │
              │ Service │   │ Service│   │ Service │
              │ A       │   │ B      │   │ C       │
              └─────────┘   └────────┘   └────────┘
```

### Strategy

1. **Identify bounded contexts** in the monolith
2. **Extract one context** at a time
3. **Route new functionality** to the new service
4. **Gradually migrate old functionality** via strangler
5. **Decommission old code** when migration complete

### Implementation with API Gateway

```nginx
# Nginx — route certain paths to new services
location /api/v2/orders {
    proxy_pass http://order-service:8080;
}

location /api/v1/orders {
    # Old path — still handled by monolith
    proxy_pass http://monolith:8080;
}

# Feature flag via header
location /api/orders {
    if ($http_x_api_version = "v2") {
        proxy_pass http://order-service:8080;
    }
    proxy_pass http://monolith:8080;
}
```

### Risks

- **Increased latency** — calls may go through monolith + new service
- **Data duplication** — old and new systems may have overlapping data
- **Transaction complexity** — distributed transactions across old + new
- **Migration fatigue** — teams lose motivation before completion

> **Trap:** The Strangler Fig is slow. It may take months or years. Each extracted service adds operational overhead. Only extract if there's clear benefit (independent scaling, team alignment, deployment frequency).

---

## 5. Domain Events

### What they are

> Something that happened in the domain that domain experts care about.

```php
// Domain event — past tense, immutable
class OrderPlaced
{
    public function __construct(
        public readonly OrderId $orderId,
        public readonly CustomerId $customerId,
        public readonly Money $total,
        public readonly DateTimeImmutable $occurredAt,
    ) {}
}
```

### When to publish

Inside the aggregate, AFTER state change:

```php
class Order
{
    private array $domainEvents = [];

    public function place(): void
    {
        if ($this->status !== OrderStatus::Draft) {
            throw new DomainException('Can only place draft orders');
        }
        $this->status = OrderStatus::Placed;

        $this->domainEvents[] = new OrderPlaced(
            orderId: $this->id,
            customerId: $this->customerId,
            total: $this->total,
            occurredAt: new DateTimeImmutable(),
        );
    }

    public function releaseEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

### Use cases

- **Trigger side effects** — send email, update read model
- **Integrate with other contexts** — publish to message broker
- **Audit logging** — store events for compliance
- **Event Sourcing** — events are the source of truth

---

## 6. Transactional Outbox Pattern

### Problem

When you need to atomically update the database AND publish an event:

```php
// Problem: DB succeeds, but event publish fails → inconsistency
$order->place();
$orderRepo->save($order);
$this->eventBus->publish(new OrderPlaced(...));  // What if this fails?
```

### Solution: Outbox table

```
┌─────────────────────────────────────────┐
│  Transaction                            │
│                                         │
│  1. Save Order to orders table          │
│  2. Insert event to outbox table        │
│  ────────────────── ATOMIC ──────────►  │
│                                         │
└─────────────────────────────────────────┘
                                          │
                Outbox Poller ──────────────┘
                    │
                    ▼
            Publish to Event Bus
                    │
                Delete from outbox
```

### Implementation

```php
class PlaceOrderHandler
{
    public function __construct(
        private DB $db,
    ) {}

    public function handle(PlaceOrderCommand $command): void
    {
        $this->db->transaction(function () use ($command) {
            // 1. Domain logic
            $order = Order::place($command->customerId, $command->items);

            // 2. Save aggregate
            $this->db->insert('orders', [
                'id' => $order->id,
                'status' => $order->status,
                'total' => $order->total,
            ]);

            // 3. Save event to outbox (same transaction!)
            foreach ($order->releaseEvents() as $event) {
                $this->db->insert('outbox', [
                    'id' => Uuid::generate(),
                    'type' => $event::class,
                    'payload' => json_encode($event),
                    'created_at' => now(),
                ]);
            }
        });

        // Outbox poller (separate process) publishes events
    }
}
```

### Alternatives

| Pattern | Description |
|---------|-------------|
| **Transactional Outbox** | Insert event in same DB transaction, poller publishes |
| **Change Data Capture (CDC)** | Read DB transaction log (WAL), publish events (Debezium) |
| **2PC (Two-Phase Commit)** | Distributed transaction across DB and message broker (complex, not recommended) |
| **Kafka Connect** | CDC for Kafka + PostgreSQL |

---

## 7. Architecture Decision Records

### What are ADRs?

A document that captures an important architecture decision, including context, options, decision, and consequences.

### ADR format

```markdown
# ADR-001: Use RabbitMQ for async event processing

## Status
Accepted

## Context
We need async communication between Order Service and Inventory Service.
Options considered: SQS, RabbitMQ, Kafka, direct HTTP.

## Decision
Use RabbitMQ with the following configuration:
- Exchange type: topic
- Producer confirms: enabled
- Consumer: manual ACK with retry

## Consequences
Positive:
- Low operational complexity (familiar with RabbitMQ)
- Good developer experience (PHP-friendly)

Negative:
- Not as scalable as Kafka for high-throughput event streaming
- Requires self-hosting (vs SQS managed service)

## Trade-offs
- Chose RabbitMQ over SQS because we need pub/sub with multiple consumers
- Chose over Kafka because throughput doesn't justify operational complexity
```

### When to write ADRs

- Major architecture decisions (service boundaries, event bus, DB choice)
- Decisions with significant trade-offs
- Decisions that affect multiple teams
- Decisions you may revisit later

---

## 8. Trade-Off Analysis

### Microservices vs Monolith

```
Monolith:
  + Simple development (single codebase, single deploy)
  + Easy to refactor (IDE tools work well)
  + Low latency (in-process calls)
  + Simple transactions (ACID)
  + Easy testing (end-to-end in one process)
  - Hard to scale independently
  - Team coupling (merge conflicts, shared deploy)
  - Technology lock-in

Microservices:
  + Independent scaling, deployment, teams
  + Technology diversity
  + Failure isolation
  + Organizational alignment
  - Network latency, fault tolerance
  - Distributed transactions (sagas)
  - Testing at boundaries (contract tests)
  - Operational complexity (observability, deployment)

Decision: Start with monolith → extract when proven boundaries
```

### Event-Driven vs Request-Driven

```
Request-Driven (REST/gRPC):
  + Lower latency (sync)
  + Simpler debugging (request → response)
  + Strong consistency (wait for response)
  - Tighter coupling
  - Cascading failures
  - Lower throughput (blocking)

Event-Driven:
  + Loose coupling
  + Higher resilience (async)
  + Multiple consumers
  + Audit trail
  - Higher latency (eventually consistent)
  - Harder debugging (event chains)
  - More infrastructure (message broker)

Decision: Request-driven for commands that need immediate confirmation.
Event-driven for cross-service notifications and workflows.
```

### CQRS vs CRUD

```
CRUD:
  + Simple — one model for everything
  + Easy to implement
  + Strong consistency
  - Read/write patterns coupled
  - Hard to optimize reads without affecting writes

CQRS:
  + Optimized read and write models
  + Scalable (independent read/write scaling)
  + Different storage per model
  - Eventual consistency
  - More code to maintain
  - Projection logic (ETL-like)

Decision: Use CQRS selectively — complex domains only, not entire system.
```

### Synchronous vs Asynchronous Communication

```
Synchronous (HTTP/gRPC):
  + Immediate response
  + Easier to debug
  + Strong consistency
  - Tight coupling
  - Cascading failures
  - Lower resilience

Asynchronous (Message Broker):
  + Loose coupling
  + Higher resilience
  + Load leveling (queues)
  - Eventual consistency
  - Harder debugging
  - Dead letter handling

Decision: Default to synchronous for internal service calls (gRPC within a cluster).
Use async for cross-service events and workflows.
```

---

## 9. Q&A

**Q: What's CQRS and when should you use it?**
A: Separate read and write models. Use when read and write patterns differ significantly (e.g., complex business logic for writes, denormalized views for reads). Don't use for simple CRUD.

**Q: What's Event Sourcing and what are its trade-offs?**
A: Store events as source of truth, derive state by replaying. Benefits: audit trail, temporal queries. Drawbacks: event versioning complexity, querying requires replay, operational overhead.

**Q: What's the difference between choreography and orchestration sagas?**
A: Choreography: services react to events (decentralized). Orchestration: central coordinator manages steps. Choreography → loose coupling. Orchestration → better traceability.

**Q: What's the Transactional Outbox pattern?**
A: Insert event into database in the same transaction as the aggregate. A separate poller reads the outbox and publishes events. Ensures  atomicity between state change and event publication.

**Q: When would you use the Strangler Fig pattern?**
A: When migrating a monolith to microservices incrementally. Replace functionality piece by piece, routing new features to new services while old code still runs.

**Q: What's a Saga?**
A: A sequence of local transactions where each step publishes an event or is orchestrated. On failure, compensating transactions undo completed steps. Used for distributed transactions across services.

**Q: What's the difference between event sourcing and just logging events?**
A: Event sourcing stores events as the primary data store. State is derived from events. Logging is a secondary concern. In ES, you CANNOT recover state except by replaying events.

**Q: How do you handle event schema evolution in event sourcing?**
A: Upcasting: on read, transform events from old schema to new schema. Events are never modified — only the upcasting logic changes.

**Q: What's an ADR and why is it important?**
A: Architecture Decision Record. Documents context, options, decision, and consequences. Important for team alignment, onboarding, and revisiting decisions later.

**Q: Should you start with microservices?**
A: No. Start with a modular monolith. Extract services when you understand the bounded contexts and need independent scaling/deployment. Premature microservices lead to distributed monoliths.
