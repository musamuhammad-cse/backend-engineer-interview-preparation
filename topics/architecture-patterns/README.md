# Architecture Patterns — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS (Laravel monolith → modular monolith), trading platform (event-driven microservices), Chronos (distributed system with Raft, leader election), Clean Architecture in Go/Laravel  
> **Note:** Architecture is THE senior signal. The differentiator is **trade-off analysis** — knowing WHEN to apply which pattern, not just WHAT they are. Interviewers want to see you weigh complexity vs benefit.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud | 25 min/section |
| 2 | Draw the architecture diagram on paper | 15 min/section |
| 3 | Answer the section's Q&A without looking | 25 min/section |
| 4 | Analyze your own projects against these patterns | 10 min |

**The senior signal is trade-off analysis.** Any junior can recite microservices advantages. A senior can articulate when a monolith is the right choice, when event-driven adds unnecessary complexity, and when CQRS is over-engineering.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | SOLID principles, coupling & cohesion, layered architecture, Clean Architecture, hexagonal architecture, dependency injection, separation of concerns | 5–7 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Microservices (characteristics, service granularity, API gateway, service discovery, circuit breaker), Event-Driven Architecture (events, commands, message broker patterns), DDD fundamentals (ubiquitous language, bounded context, aggregate, entity vs value object) | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | CQRS, Event Sourcing, Saga pattern (choreography vs orchestration), Strangler Fig pattern, Domain Events, Transactional Outbox, architecture decision records (ADRs), trade-off analysis, modular monolith vs microservices decision framework | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 130+ interview questions, architecture trade-off scenarios, system design prompts tied to your experience | Ongoing drill |

---

## Coverage map

### SOLID Principles
- Single Responsibility: one reason to change per class
- Open/Closed: open for extension, closed for modification
- Liskov Substitution: subtypes must be substitutable for base types
- Interface Segregation: don't force clients to depend on unused methods
- Dependency Inversion: depend on abstractions, not concretions

### Coupling & Cohesion
- Coupling: degree of dependency between modules (tight vs loose)
- Cohesion: degree of relatedness within a module (high vs low)
- Goal: High cohesion, low coupling

### Architectural Styles
- Layered architecture: presentation → application → domain → infrastructure
- Hexagonal (Ports & Adapters): core domain isolated from external concerns
- Clean Architecture: dependency rule — dependencies point inward
- Onion architecture: similar to Clean — domain at center

### Microservices
- Characteristics: bounded context, independent deploy, decentralized data, team alignment
- API Gateway: single entry point, routing, auth, rate limiting
- Service discovery: DNS-based (K8s), registry-based (Consul)
- Circuit breaker: fail fast, prevent cascading failures
- Saga: distributed transaction management

### Event-Driven Architecture
- Event: something that happened (fact, immutable)
- Command: request to do something (mutable, may fail)
- Message broker: Kafka, RabbitMQ, SQS/SNS
- Event sourcing: store events as source of truth
- CQRS: separate read and write models

### DDD (Domain-Driven Design)
- Ubiquitous language: shared terminology between devs and domain experts
- Bounded context: explicit boundary where a model applies
- Aggregate: cluster of domain objects treated as a unit
- Entity: object with identity (e.g., Order)
- Value Object: object without identity (e.g., Money, Address)
- Domain Event: something that happened in the domain
- Repository: collection-like access to aggregates
- Factory: complex aggregate creation

### CQRS
- Command side: write model (commands validate, mutate state)
- Query side: read model (queries return denormalized data)
- Separate models: may have different storage (SQL for writes, read-optimized for reads)
- When to use: complex domains with different read/write patterns

### Event Sourcing
- Store events, not current state
- Current state = fold over all events
- Benefits: audit trail, temporal queries, replay
- Challenges: event versioning, schema evolution, querying

### Saga Pattern
- Choreography: each service publishes events that trigger next step
- Orchestration: central coordinator tells services what to do
- Compensation: undo actions on failure
- Best for: long-running business transactions across services

### Strangler Fig Pattern
- Gradually replace a monolith incrementally
- Route new features to microservices
- Eventually strangle the monolith

### Trade-off Analysis
- Monolith vs microservices: complexity, team size, deployment frequency
- Event-driven vs request-driven: consistency, traceability, latency
- CQRS vs CRUD: read/write complexity, team maturity
- Synchronous vs async: coupling, latency, resilience

---

## Study order recommendation

Start with SOLID and Clean Architecture (foundation), then microservices patterns, then DDD/CQRS/Event Sourcing.

```
Week 1:  01-basic.md          + SOLID applied to your Laravel codebase
Week 2:  02-intermediate.md   + Microservices vs monolith trade-off
Week 3:  03-senior.md         + CQRS/Event Sourcing with Chronos
Week 4+: 04-question-bank.md daily drill + whiteboard practice
```

**Next topic in skill order:** Kafka, RabbitMQ, SQS, SNS.
