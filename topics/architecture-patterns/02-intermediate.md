# Architecture Patterns — Intermediate Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** SOLID, Clean Architecture basics  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Microservices Architecture
2. API Gateway Pattern
3. Service Discovery
4. Circuit Breaker Pattern
5. Event-Driven Architecture
6. Message Broker Patterns
7. DDD Fundamentals
8. Q&A

---

## 1. Microservices Architecture

### Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Bounded context** | Each service owns a domain boundary |
| **Independent deploy** | Services can be deployed independently |
| **Decentralized data** | Each service owns its data store |
| **Team alignment** | Teams aligned to services (Conway's Law) |
| **Technology diversity** | Different languages/databases per service |
| **Failure isolation** | One service failure doesn't cascade |

### When to use microservices

```
Reasons TO use microservices:
  - Large team (50+ engineers)
  - Different deployment cadences per domain
  - Independent scaling needs
  - Polyglot technology requirements

Reasons NOT to use microservices:
  - Small team (< 10 engineers)
  - Simple domain (mostly CRUD)
  - Startup exploring product-market fit
  - No clear bounded contexts
```

### Monolith → Microservices decision framework

| Factor | Monolith | Microservices |
|--------|----------|---------------|
| Team size | < 10 | 50+ |
| Deployment frequency | Daily | Multiple/day |
| Domain complexity | Simple | Complex |
| Need for independent scaling | Low | High |
| Organizational alignment | Single team | Multiple teams |
| Operational maturity | Low | High |

### Service granularity

- **Domain-driven** — one service per bounded context (e.g., Order Service, Inventory Service)
- **Subdomain-driven** — one service per subdomain (e.g., Pricing Service, Shipping Service)
- **Too fine** — one service per entity (too many moving parts, high overhead)
- **Too coarse** — one service for everything (distributed monolith)

> **Trap:** "Microservices are small" — wrong. Services should be sized by BOUNDED CONTEXT, not by lines of code. A 5,000-line billing service is fine if it's a cohesive bounded context.

---

## 2. API Gateway Pattern

### Role of API Gateway

```
                    ┌──────────┐
                    │  Client  │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │   API    │
                    │  Gateway │
                    └────┬─────┘
                    ┌────┼────────────┐
                    │    │            │
               ┌────▼┐ ┌▼───┐   ┌────▼───┐
               │Order│ │User│   │Inventory│
               └─────┘ └────┘   └────────┘
```

| Responsibility | Description |
|---------------|-------------|
| **Routing** | Forward requests to appropriate services |
| **Authentication** | Validate JWT/OAuth before forwarding |
| **Rate limiting** | Throttle requests per client |
| **Request/response transformation** | Adapt between client and service formats |
| **Circuit breaking** | Fail fast when downstream is unhealthy |
| **Aggregation** | Combine responses from multiple services |
| **Caching** | Cache common responses |
| **Logging/monitoring** | Centralized request observability |

### Backend for Frontend (BFF)

```
┌──────────┐    ┌───────────┐    ┌──────────────┐
│  Mobile  │───►│ Mobile    │───►│ Order        │
│  Client  │    │ BFF       │    │ Service      │
└──────────┘    └───────────┘    ├──────────────┤
                                 │ User         │
┌──────────┐    ┌───────────┐    │ Service      │
│  Web     │───►│ Web       │───►├──────────────┤
│  Client  │    │ BFF       │    │ Inventory    │
└──────────┘    └───────────┘    │ Service      │
                                 └──────────────┘
```

Different clients get different API surfaces. Mobile BFF might return denormalized responses (fewer round trips). Web BFF might include full HTML.

### API Gateway vs Service Mesh

| Aspect | API Gateway | Service Mesh |
|--------|-------------|-------------|
| Location | Edge (external traffic) | Internal (service-to-service) |
| Functions | Auth, rate limit, routing | mTLS, circuit break, observability |
| Client | External (users, apps) | Internal (services) |
| Example | Kong, AWS API Gateway | Istio, Linkerd |

---

## 3. Service Discovery

### Client-side discovery

```
Service A ──► Service Registry (Consul, etcd)
                  │
Service A ────────┴──────────► Service B
```

Client queries the registry, gets instance IPs, load-balances.

### Server-side discovery

```
Service A ──► Load Balancer ──► Service B instances
                    │
               (via Service Registry)
```

Client hits a load balancer (ALB, K8s Service), which knows about healthy instances.

### Kubernetes service discovery

```yaml
# DNS-based: service-name.namespace.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 8080
```

Other services resolve `order-service:8080` via CoreDNS.

> **Trap:** Service discovery adds latency and complexity. K8s DNS-based discovery is usually sufficient without an external registry. External registries (Consul) add value for multi-cluster or hybrid deployments.

---

## 4. Circuit Breaker Pattern

### States

```
         ┌──────────────────┐
         │                  │
    ┌────▼────┐        ┌────┴─────┐
    │  CLOSED │        │  OPEN    │
    │ (normal)│        │ (failing)│
    └────┬────┘        └────┬─────┘
         │                  │
         │   failures >     │
         │   threshold      │
         └──────────────────►
                              │
                         ┌────▼──────┐
                         │ HALF-OPEN │
                         │ (probing) │
                         └────┬──────┘
                              │
                   success ───┴───► CLOSED
                   failure ───────► OPEN
```

### Implementation

```php
class CircuitBreaker
{
    private int $failureCount = 0;
    private int $threshold = 5;
    private int $timeout = 30; // seconds
    private ?int $openAt = null;

    public function call(callable $fn): mixed
    {
        if ($this->isOpen()) {
            if ($this->timeoutElapsed()) {
                $this->halfOpen();
            } else {
                throw new CircuitBreakerOpenException();
            }
        }

        try {
            $result = $fn();
            $this->success();
            return $result;
        } catch (Exception $e) {
            $this->failure();
            throw $e;
        }
    }

    private function isOpen(): bool
    {
        return $this->failureCount >= $this->threshold;
    }

    private function failure(): void
    {
        $this->failureCount++;
        if ($this->isOpen()) {
            $this->openAt = time();
        }
    }

    private function success(): void
    {
        $this->failureCount = 0;
        $this->openAt = null;
    }
}
```

### Bulkhead pattern

Isolate resources to prevent failure propagation:

```
┌───────────┐  ┌───────────┐  ┌───────────┐
│ Thread    │  │ Thread    │  │ Thread    │
│ Pool A    │  │ Pool B    │  │ Pool C    │
│ (Orders)  │  │ (Payments)│  │ (Shipping)│
├───────────┤  ├───────────┤  ├───────────┤
│ max: 10   │  │ max: 5    │  │ max: 5    │
└───────────┘  └───────────┘  └───────────┘
```

If payment API is slow, it only exhausts its own pool. Orders continue unaffected.

---

## 5. Event-Driven Architecture

### Events vs Commands

| | Event | Command |
|---|-------|---------|
| **What** | Something happened (fact) | Do something (request) |
| **Naming** | `OrderPlaced`, `PaymentReceived` | `PlaceOrder`, `ChargePayment` |
| **Mutability** | Immutable (history) | May change state |
| **Failure** | Consumer handles failure | May be rejected |
| **Publisher** | Doesn't expect a response | May receive error |

### Event types

```
Domain Event: Something that happened in the domain
  → OrderPlaced, InventoryReserved, PaymentFailed

Integration Event: Cross-service notification
  → OrderSubmitted (Order Service → Inventory Service)

Event Notification: Something happened (carries only ID)
  → "Order 123 was placed"

Event Carrying State: Full data for the consumer
  → Order with all items, prices, addresses
```

### Event-Driven vs Request-Driven

| Aspect | Request-Driven (REST) | Event-Driven |
|--------|----------------------|--------------|
| Coupling | Tight (knows about other services) | Loose (just publishes events) |
| Latency | Lower (synchronous) | Higher (async) |
| Consistency | Strong possible | Eventually consistent |
| Traceability | Easy (request-response) | Harder (event chain) |
| Resilience | Lower (cascading failures) | Higher (async queues) |

> **Trap:** Event-driven systems are harder to debug and reason about. You can't "call" a service and wait for a response. You need event tracing, correlation IDs, and monitoring.

---

## 6. Message Broker Patterns

### Publish-Subscribe (Pub/Sub)

```
Publisher ──► Topic ──┬──► Subscriber A
                      ├──► Subscriber B
                      └──► Subscriber C
```

Each subscriber gets every message. Used for broadcasting (SNS, Kafka consumer groups).

### Competing Consumers (Work Queue)

```
Producer ──► Queue ──┬──► Consumer 1
                     ├──► Consumer 2
                     └──► Consumer 3
```

Each message is consumed once. Used for workload distribution (SQS, RabbitMQ queues).

### Dead Letter Queue

```
Queue ──► maxReceiveCount exceeded ──► DLQ
```

Store failed messages for later inspection. Critical for production systems.

### Exactly-once vs At-least-once

| Delivery | Guarantee | Idempotent consumer needed? | Example |
|----------|-----------|----------------------------|---------|
| At-most-once | Message may not be delivered | No | Monitoring metrics |
| At-least-once | Message delivered but may be duplicated | Yes | SQS standard |
| Exactly-once | Message delivered exactly once | No (handled by broker) | SQS FIFO, Kafka transactions |

> **Trap:** Exactly-once delivery often comes at a cost (lower throughput, higher latency). Most systems can be designed with at-least-once + idempotent consumers.

---

## 7. DDD Fundamentals

### Ubiquitous Language

> A shared language between developers and domain experts. The same terms appear in code, conversations, and documentation.

```
Domain Expert: "When an order is placed, we check inventory and reserve stock."
Developer: class OrderPlaced { ... }
           class InventoryReserved { ... }
           class InventoryService { public function reserve(Order $order): void }
```

No translation layer between business and code.

### Bounded Context

> An explicit boundary where a domain model applies.

```
┌─────────────────────────┐   ┌─────────────────────────┐
│  Sales Context          │   │  Shipping Context        │
│                         │   │                         │
│  Customer has:          │   │  Customer has:           │
│  - name, email,         │   │  - name, address,        │
│    order history        │   │    shipping preferences  │
│                         │   │                         │
│  Product has:           │   │  Product has:            │
│  - name, price, stock   │   │  - weight, dimensions,   │
│                         │   │    fragility            │
└─────────────────────────┘   └─────────────────────────┘
```

Same term ("Customer") may have different meanings in different contexts. Sales cares about purchase behavior. Shipping cares about addresses.

### Aggregate

> A cluster of domain objects treated as a unit. Consistency boundary.

```php
class Order
{
    private OrderId $id;
    private OrderStatus $status;
    /** @var OrderLine[] */
    private array $lines;

    // Aggregate root — entry point for all operations
    public function addProduct(Product $product, Quantity $qty): void
    {
        // Business rules enforced inside the aggregate
        if ($this->status !== OrderStatus::Draft) {
            throw new DomainException('Can only add to draft orders');
        }
        $this->lines[] = new OrderLine($product, $qty);
    }

    public function place(): void
    {
        if (count($this->lines) === 0) {
            throw new DomainException('Cannot place empty order');
        }
        $this->status = OrderStatus::Placed;
    }
}
```

**Aggregate rules:**
- One aggregate per transaction boundary
- References to other aggregates via ID (not by object reference)
- Aggregate root is the only entry point for external access

### Entity vs Value Object

```php
// Entity — has identity
class OrderId
{
    public function __construct(public readonly Uuid $id) {}
}

class Order
{
    public function __construct(
        public readonly OrderId $id,            // identity
        public readonly CustomerId $customerId, // reference to another aggregate
        public OrderStatus $status,
    ) {}
}

// Value Object — equality by attributes, immutable
class Money
{
    public function __construct(
        public readonly int $amount,          // in smallest unit (cents)
        public readonly Currency $currency,
    ) {}

    public function add(Money $other): Money
    {
        if ($this->currency !== $other->currency) {
            throw new DomainException('Currency mismatch');
        }
        return new Money($this->amount + $other->amount, $this->currency);
    }

    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }
}
```

### Repository

Collection-like interface for aggregates:

```php
interface OrderRepository
{
    public function save(Order $order): void;
    public function find(OrderId $id): ?Order;
    public function delete(OrderId $id): void;
}
```

> **Trap:** A repository is NOT a DAO (Data Access Object). A repository operates on AGGREGATES (behavior + data). A DAO operates on individual database rows. Laravel's Eloquent acts as both — know the difference.

---

## 8. Q&A

**Q: When should you use an API Gateway?**
A: When you have multiple microservices and need centralized auth, rate limiting, routing, or request transformation. For simple architectures, a reverse proxy (Nginx) is sufficient.

**Q: What's the difference between a circuit breaker and a retry?**
A: Circuit breaker prevents calls to a failing service (fail fast). Retry attempts the same call again. They work together: retry a few times, then circuit break.

**Q: What's a bounded context?**
A: A boundary where a domain model applies. Different contexts may have different definitions of the same concept (e.g., "Customer" in Sales vs Shipping).

**Q: What's an aggregate in DDD?**
A: A cluster of domain objects treated as a unit. The aggregate root is the entry point. Consistency rules are enforced within the aggregate.

**Q: What's the difference between an entity and a value object?**
A: Entity has identity (two entities are different even with same attributes). Value object has no identity (two VOs are equal if attributes match).

**Q: What's an integration event vs a domain event?**
A: Domain event = something that happened within a bounded context. Integration event = event published across service boundaries for other contexts to consume.

**Q: What's event-driven architecture good for?**
A: Loose coupling, asynchronous workflows, multiple subscribers, audit trails, and systems where strong consistency isn't required.

**Q: What's event-driven architecture bad for?**
A: Systems requiring strong consistency, simple CRUD, real-time request-response, small teams, and developers unfamiliar with async debugging.
