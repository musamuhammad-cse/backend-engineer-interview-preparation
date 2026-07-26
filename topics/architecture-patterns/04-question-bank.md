# Architecture Patterns — Question Bank

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Type:** 130+ rapid-fire Q&A, architecture trade-off scenarios, system design prompts tied to your experience

---

## Table of Contents

1. Rapid-Fire Q&A (130+ questions)
2. Architecture Trade-Off Scenarios
3. System Design Prompts
4. Whiteboard Practice

---

## 1. Rapid-Fire Q&A

### SOLID & Clean Architecture

**Q: What does SOLID stand for?**
A: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

**Q: What's the most important SOLID principle for microservices?**
A: Dependency Inversion — high-level modules shouldn't depend on low-level modules. Both depend on abstractions.

**Q: What's the dependency rule in Clean Architecture?**
A: Dependencies point inward. Outer layers depend on inner layers. Nothing in an inner circle depends on an outer circle.

**Q: What's the difference between Clean Architecture and Hexagonal Architecture?**
A: Clean organizes by layers (concentric circles). Hexagonal focuses on ports and adapters at boundaries. Complementary.

**Q: What's an anemic domain model?**
A: Domain objects with no behavior — all logic in services. Anti-pattern.

**Q: What's the Law of Demeter?**
A: "Don't talk to strangers." A method should only call methods on: itself, its parameters, objects it creates, its direct fields.

**Q: What's the difference between coupling and cohesion?**
A: Coupling = dependency between modules (low is good). Cohesion = relatedness WITHIN a module (high is good).

### Microservices

**Q: What defines a microservice?**
A: Independent deployable, bounded context, owns its data, team-aligned, failure-isolated.

**Q: When should you NOT use microservices?**
A: Small team, simple domain, no clear bounded contexts, low operational maturity.

**Q: What's the difference between an API Gateway and a reverse proxy?**
A: API Gateway adds auth, rate limiting, request transformation, aggregation. Reverse proxy (Nginx) does basic routing and caching.

**Q: What's a BFF?**
A: Backend For Frontend — a separate API gateway per client type (mobile, web, IoT). Each BFF optimizes for its client.

**Q: What's service discovery?**
A: Mechanism for services to find each other's network locations. DNS-based (K8s) or registry-based (Consul).

**Q: What's a circuit breaker?**
A: Prevents calls to a failing service. States: CLOSED (normal), OPEN (failing), HALF-OPEN (probing).

**Q: What's a bulkhead pattern?**
A: Isolate resources into separate pools so failure in one doesn't affect others.

**Q: What's a distributed monolith?**
A: Microservices that are tightly coupled — must be deployed together, share databases, hard to change independently. Anti-pattern.

**Q: How do microservices communicate?**
A: Synchronous (HTTP/gRPC) or asynchronous (message broker).

### Event-Driven Architecture

**Q: What's the difference between an event and a command?**
A: Event = something happened (fact, immutable). Command = do something (request, may fail).

**Q: What's pub/sub vs competing consumers?**
A: Pub/sub: each subscriber gets all messages. Competing consumers: each message consumed once (queue).

**Q: What's a dead letter queue?**
A: Queue for messages that repeatedly fail processing. For inspection and reprocessing.

**Q: What's at-least-once delivery?**
A: Message delivered at least once (may be duplicated). Consumer must be idempotent.

**Q: What's exactly-once delivery?**
A: Message delivered exactly once. Higher cost, lower throughput.

**Q: What's event-driven architecture good for?**
A: Loose coupling, async workflows, multiple consumers, audit trails.

**Q: What's event-driven architecture bad for?**
A: Strong consistency requirements, simple CRUD, request-response patterns.

### DDD

**Q: What's ubiquitous language?**
A: Shared terminology between developers and domain experts. Same terms in code, docs, conversations.

**Q: What's a bounded context?**
A: Explicit boundary where a domain model applies. Different contexts may have different definitions.

**Q: What's an aggregate?**
A: Cluster of domain objects treated as a unit. Consistency boundary. Accessed only via aggregate root.

**Q: What's an aggregate root?**
A: The root entity of an aggregate. External access only through the root.

**Q: What's an entity vs value object?**
A: Entity: identity (two entities differ even if same attributes). Value object: equality by attributes (immutable).

**Q: What's a repository?**
A: Collection-like interface for aggregates. Not a DAO — operates on aggregates, not rows.

**Q: What's a domain event?**
A: Something that happened in the domain that domain experts care about. Past tense.

**Q: What's a domain service?**
A: Stateless service that implements domain logic that doesn't fit in an entity or value object.

**Q: What's a factory in DDD?**
A: Encapsulates complex aggregate creation logic.

### CQRS & Event Sourcing

**Q: What's CQRS?**
A: Separate read and write models. Commands mutate state. Queries return data.

**Q: When should you use CQRS?**
A: Different read/write patterns, complex writes, denormalized reads needed.

**Q: When should you NOT use CQRS?**
A: Simple CRUD, same model for read/write, low complexity.

**Q: What's Event Sourcing?**
A: Store events as source of truth. Current state = replay all events.

**Q: What are the benefits of Event Sourcing?**
A: Audit trail, temporal queries, no O/R mismatch, event-driven integrations.

**Q: What are the drawbacks of Event Sourcing?**
A: Event versioning, query complexity, snapshot management, learning curve.

**Q: What's an upcaster?**
A: Transforms old-format events to current schema on read. Enables event schema evolution.

**Q: What's the difference between event sourcing and logging?**
A: ES stores events as PRIMARY data. State is derived from events. Logging is secondary.

### Saga & Distributed Transactions

**Q: What's a saga?**
A: Sequence of local transactions with compensating actions on failure.

**Q: What's the difference between choreography and orchestration sagas?**
A: Choreography: decentralized, services react to events. Orchestration: central coordinator.

**Q: When would you use choreography?**
A: Simple flows with few services, lower coupling needed.

**Q: When would you use orchestration?**
A: Complex flows, easier traceability needed, new team members.

**Q: What's a compensating transaction?**
A: Undoes a previously completed step in a saga (e.g., refund payment, release inventory).

**Q: What's a two-phase commit (2PC)?**
A: Distributed transaction protocol. Not recommended for microservices (blocking, not fault-tolerant).

### Patterns & Anti-Patterns

**Q: What's the Strangler Fig pattern?**
A: Gradually replace a monolith by routing new functionality to microservices.

**Q: What's the Transactional Outbox pattern?**
A: Publish events via outbox table in same DB transaction. Prevents dual-write problem.

**Q: What's an ADR?**
A: Architecture Decision Record. Documents context, options, decision, consequences.

**Q: What's a distributed monolith?**
A: Microservices that are coupled via shared DB, sync calls, coupled deployment.

**Q: What's the difference between orchestration and choreography in event-driven?**
A: Orchestration = central coordinator. Choreography = services dance to events.

**Q: What's a sidecar?**
A: Helper container in the same pod for cross-cutting concerns (logging, proxy, monitoring).

**Q: What's Conway's Law?**
A: Organizations design systems that mirror their communication structure.

**Q: What's the Inverse Conway Maneuver?**
A: Reorganize teams to match desired architecture.

### General

**Q: What architecture would you use for a simple CRUD app?**
A: Layered architecture with a monolith. Clean Architecture is overkill.

**Q: What architecture would you use for a complex domain with many business rules?**
A: Clean Architecture with DDD. Consider CQRS for complex read/write differences.

**Q: What architecture would you use for 100 microservices?**
A: API Gateway, service mesh (Istio), event-driven communication, GitOps.

**Q: How do you decide between monolith and microservices?**
A: Start monolith. Extract when: team grows, bounded contexts emerge, independent scaling needed.

**Q: What's the most important quality for a senior architect?**
A: Trade-off analysis — knowing when a simpler solution is better than the "perfect" architecture.

### SOLID specific

**Q: Give an example of SRP violation in a Laravel controller.**
A: Controller handling validation, domain logic, persistence, and response formatting.

**Q: Give an example of DIP in Laravel.**
A: Injecting interfaces in constructors, binding concrete implementations in service providers.

**Q: What's the LSP violation with the Rectangle/Square example?**
A: Square extends Rectangle but breaks width/height independence. Subtypes must maintain parent behavior.

**Q: How does ISP relate to interface design in microservices?**
A: Service interfaces should be client-specific, not fat contracts. Each service exposes only what its consumers need.

---

## 2. Architecture Trade-Off Scenarios

### Scenario 1: Monolith or microservices for your inventory SaaS?

**Context:** Your multi-tenant inventory management SaaS (Laravel, PostgreSQL). Currently a monolith. 5,000+ tenants, 20K+ users.

**Factors:**
- Team: 8 engineers
- Deployment frequency: 1–2x per week
- Scaling: DB is bottleneck, app scales fine
- Domain: Inventory, orders, customers, reports

**Analysis:**
- Team is 8 — microservices would spread too thin
- No clear bounded contexts (inventory/orders are tightly coupled)
- Deployment frequency isn't constrained by monolith
- Main bottleneck is DB, not app

**Decision:** Keep monolith. Focus on modular monolith: separate packages for inventory, orders, reports. Extract to microservices only when a bounded context is proven and team grows.

### Scenario 2: Event-driven for the trading platform

**Context:** Your 20K+ DAU trading platform. Real-time order execution, price feeds, trade history.

**Factors:**
- Low latency requirement (order execution < 100ms)
- Event-driven pattern (market data is naturally event-based)
- Multiple consumers (frontend, analytics, compliance)

**Decision:**
- Use event-driven for MARKET DATA (price updates → WebSocket to clients)
- Use request-driven for ORDER EXECUTION (low latency, strong consistency needed)
- Hybrid: synchronous for commands, async for events
- Message broker: Kafka for market data (high throughput, replay), SQS for internal events

### Scenario 3: CQRS for Chronos reports

**Context:** Chronos distributed scheduler. Users create jobs, view execution history, generate reports.

**Factors:**
- Write model: schedule jobs, state transitions (Raft)
- Read model: job history (millions of records), run reports, aggregate stats
- Different access patterns: complex queries on read, simple state changes on write

**Analysis:**
- Different read/write patterns ✓
- Complex read model (reports, aggregations) ✓
- Current read model is slow (same DB as writes)

**Decision:** Apply CQRS for the reports/analytics portion only:
- Write side: Event Sourcing for job state changes
- Read side: Project to separate read DB (TimescaleDB for time-series job events)
- Reports query read DB — no impact on write performance

### Scenario 4: Saga for order processing

**Context:** E-commerce order flow: place order → reserve inventory → charge payment → ship → notify.

**Factors:**
- Multiple services involved
- Must handle failures (payment fails → release inventory)
- Async processing acceptable (order confirmation can be delayed)

**Decision:**
- Orchestration saga (central coordinator)
- Steps: PlaceOrder → ReserveInventory → ProcessPayment → ScheduleShipping → Notify
- Compensation: PaymentFailed → ReleaseInventory → MarkOrderFailed
- Use Temporal.io or AWS Step Functions for orchestration
- Each step idempotent
- Timeout per step (payment: 5 min, inventory: 1 min)

### Scenario 5: Strangler Fig for legacy ERP migration

**Context:** Legacy ERP monolith (PHP, MySQL). 10 years old, 500K lines. Extracting to microservices.

**Factors:**
- Can't rewrite — too large
- Some modules change frequently (pricing, inventory), others stable (accounting)
- Database is shared (one MySQL DB)

**Strategy:**
1. Identify most volatile bounded context: pricing
2. Extract pricing: read from legacy DB (for backward compatibility) + write to new DB
3. API Gateway: route `/api/v2/pricing/*` to new service, everything else to monolith
4. Sync data: legacy DB → pricing service (CDC with Debezium)
5. Repeat for next bounded context: inventory
6. Eventually: monolith only serves stable modules

**Risks:** Data sync complexity, increased latency for straddled calls, migration fatigue.

---

## 3. System Design Prompts

### Prompt 1: Multi-tenant SaaS architecture

**Design a scalable architecture for your inventory SaaS.**

Consider:
- Tenant isolation (shared DB vs per-tenant DB vs hybrid)
- Authentication (Passport/OAuth per tenant)
- Data access (row-level with `organization_id`)
- Caching (per-tenant Redis keys)
- Background jobs (SQS per tenant or shared)
- Scaling: DB bottleneck, app scaling

**Key decisions:**
- Shared DB with row-level isolation (simplest, most cost-effective)
- Per-tenant cache namespaces: `org:{id}:*`
- Read replicas for reporting-heavy tenants
- Feature flags per tenant

### Prompt 2: Event-driven inventory sync

**Design an event-driven system for inventory synchronization across channels (web store, marketplace, warehouse).**

Components:
- Product service: manages catalog
- Inventory service: manages stock levels
- Order service: places orders
- Channel service: syncs to external marketplaces (Amazon, eBay)

**Events:**
- `ProductCreated` → Channel service creates listing
- `StockUpdated` → Channel service updates marketplace stock levels
- `OrderPlaced` → Inventory service reserves stock
- `OrderShipped` → Inventory service decrements stock

**Trade-offs:**
- Eventual consistency between channels (minutes delay)
- At-least-once delivery (idempotent handlers)
- DLQ for failed channel syncs

### Prompt 3: CQRS for reporting

**Design a CQRS-based reporting system for your trading platform.**

Requirements:
- Real-time trade execution reports
- Daily P&L summaries
- Historical trade search

**Write side:**
- Trade events stored in Event Store (Kafka)
- Each trade is an event: TradeExecuted, TradeCancelled, TradeSettled

**Read side:**
- Real-time report: stream processor updates Redis sorted sets
- Daily P&L: batch job runs nightly, stores in PostgreSQL
- Historical search: Elasticsearch indexed from Kafka

### Prompt 4: Saga for payment flow

**Design a saga for multi-step payment processing.**

Flow:
1. Customer submits payment
2. Authorization (hold funds)
3. Inventory reservation
4. Shipping label creation
5. Capture payment
6. Send confirmation

**Orchestrator:** Step Functions
**Compensation:**
- Auth fails → cancel (no compensation needed)
- Inventory fails → release auth hold
- Shipping fails → release auth hold, release inventory
- Capture fails → (rare, but retry + manual intervention)

**Error handling:**
- Retry transient errors (3 attempts with backoff)
- DLQ for permanent failures
- Manual reconciliation for edge cases

### Prompt 5: Modular monolith migration

**Design the structure for a modular monolith in Laravel.**

```
app/
├── Domains/
│   ├── Inventory/
│   │   ├── Models/                    # Domain models (not Eloquent)
│   │   ├── Repositories/              # Contracts + implementations
│   │   ├── Services/                  # Business logic
│   │   ├── Events/                    # Domain events
│   │   └── Http/
│   │       ├── Controllers/
│   │       └── Requests/
│   ├── Orders/
│   │   └── (same structure)
│   └── Customers/
│       └── (same structure)
├── Infrastructure/
│   ├── Laravel/                       # Framework-specific adapters
│   └── Bus/                           # Event/message bus
└── Shared/
    ├── ValueObjects/
    └── Kernel/                        # HTTP console, exceptions
```

**Communication between domains:**
- In-process: Domain events via Laravel's event system
- Out-of-process: When extracted to microservice, swap to message broker

---

## 4. Whiteboard Practice

### Exercise 1: Draw Clean Architecture

Take 5 minutes to draw Clean Architecture on a whiteboard:
- Label the 4 layers (Frameworks, Interface Adapters, Application, Domain)
- Draw arrows showing dependency direction
- Label where each belongs: Controller, Use Case, Entity, Repository, DB

### Exercise 2: Design a microservice boundary

Given the inventory domain, draw bounded contexts:
- Sales context: Product, Price, Discount
- Inventory context: Stock, Warehouse, Movement
- Order context: Cart, Order, Payment
- Shipping context: Package, Shipment, Tracking

Draw arrows between contexts showing communication method (sync/async).

### Exercise 3: Event storming

Walk through the "Place Order" flow:
1. Customer places order → `OrderPlaced` event
2. Inventory checks stock → `StockReserved` or `OutOfStock` event
3. Payment processes → `PaymentSucceeded` or `PaymentFailed`
4. On success: `OrderConfirmed`, notification sent
5. On failure: compensation events

### Exercise 4: Trade-off matrix

| Decision | Option A | Option B | Your choice |
|----------|----------|----------|-------------|
| DB for reporting | Same transactional DB | CQRS + read replica | ? |
| Service communication | REST (sync) | Events (async) | ? |
| Deployment strategy | Rolling | Blue-green | ? |
| Data isolation per tenant | Shared DB row-level | Per-tenant DB | ? |

For each option, explain the trade-off.

---

> **Next topic in skill order:** Kafka, RabbitMQ, SQS, SNS.
