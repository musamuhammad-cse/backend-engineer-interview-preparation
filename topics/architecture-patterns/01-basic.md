# Architecture Patterns — Basic Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — foundational software design concepts  
> **Estimated time:** 5–7 hours

---

## Table of Contents

1. SOLID Principles
2. Coupling and Cohesion
3. Layered Architecture
4. Clean Architecture
5. Hexagonal Architecture (Ports & Adapters)
6. Dependency Injection
7. Separation of Concerns
8. Q&A

---

## 1. SOLID Principles

### SRP — Single Responsibility Principle

> A class should have ONE reason to change.

```php
// BAD: OrderProcessor handles validation, pricing, persistence, notification
class OrderProcessor
{
    public function process(array $data): void
    {
        $this->validate($data);
        $this->calculateTotal($data);
        $this->save($data);
        $this->notify($data);
    }
}

// GOOD: Each class has one responsibility
class OrderValidator { public function validate(array $data): void {} }
class OrderPricing { public function calculate(array $data): Money {} }
class OrderRepository { public function save(Order $order): void {} }
class OrderNotifier { public function notify(Order $order): void {} }
```

> **Trap:** SRP does NOT mean "one method per class." It means one reason to change. If both validation rules AND pricing logic change for different reasons, they belong in different classes.

### OCP — Open/Closed Principle

> Open for extension, closed for modification.

```php
// BAD: adding a new shipping method requires modifying this class
class ShippingCalculator
{
    public function calculate(Order $order, string $method): Money
    {
        return match ($method) {
            'standard' => $order->weight * 2,
            'express'  => $order->weight * 5,
            'overnight' => $order->weight * 10,
        };
    }
}

// GOOD: new methods are added via new classes
interface ShippingRate
{
    public function calculate(Weight $weight): Money;
}

class StandardRate implements ShippingRate { ... }
class ExpressRate implements ShippingRate { ... }
class OvernightRate implements ShippingRate { ... }
```

### LSP — Liskov Substitution Principle

> Subtypes must be substitutable for their base types without breaking the program.

```php
// BAD: Square extends Rectangle but violates behavior
class Rectangle
{
    public function setWidth(int $w): void { $this->width = $w; }
    public function setHeight(int $h): void { $this->height = $h; }
    public function getArea(): int { return $this->width * $this->height; }
}

class Square extends Rectangle
{
    public function setWidth(int $w): void { $this->width = $w; $this->height = $w; }
    public function setHeight(int $h): void { $this->width = $h; $this->height = $h; }
}

// Client code expects Rectangle behavior
function printArea(Rectangle $rect): void
{
    $rect->setWidth(5);
    $rect->setHeight(10);
    echo $rect->getArea();  // Rectangle: 50, Square: 100 — WRONG!
}
```

> **Trap:** The Rectangle-Square problem is the classic LSP violation. A `Square` is NOT a substitute for `Rectangle` if the client relies on independent width/height.

### ISP — Interface Segregation Principle

> Don't force clients to depend on interfaces they don't use.

```php
// BAD: fat interface
interface Worker
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

// GOOD: segregated interfaces
interface Workable { public function work(): void; }
interface Eatable { public function eat(): void; }
interface Sleepable { public function sleep(): void; }

class Human implements Workable, Eatable, Sleepable { ... }
class Robot implements Workable { ... }  // doesn't need eat/sleep
```

### DIP — Dependency Inversion Principle

> Depend on abstractions, not concretions.

```php
// BAD: high-level module depends on low-level module
class OrderService
{
    private MySQLRepository $repo;  // depends on concrete class

    public function __construct()
    {
        $this->repo = new MySQLRepository();  // tightly coupled
    }
}

// GOOD: both depend on abstraction
interface OrderRepository
{
    public function save(Order $order): void;
    public function find(OrderId $id): ?Order;
}

class OrderService
{
    public function __construct(
        private OrderRepository $repo  // depends on abstraction
    ) {}
}

class MySQLRepository implements OrderRepository { ... }
class PostgresRepository implements OrderRepository { ... }
```

---

## 2. Coupling and Cohesion

### Coupling

Degree of dependency between modules. Lower is better.

| Type | Description | Severity |
|------|-------------|----------|
| Content coupling | Module modifies internal data of another | Worst |
| Common coupling | Modules share global data | Bad |
| Control coupling | One module passes control flag to another | Moderate |
| Stamp coupling | Modules share a composite data structure | Moderate |
| Data coupling | Modules share only parameters (primitive types) | Best |

```php
// TIGHT coupling — class creates its own dependencies
class ReportGenerator
{
    private Database $db;
    public function __construct()
    {
        $this->db = new Database('mysql:host=localhost');  // hard-coded
    }
}

// LOOSE coupling — dependencies injected
class ReportGenerator
{
    public function __construct(private Database $db) {}  // injected
}
```

### Cohesion

Degree to which elements within a module belong together. Higher is better.

| Type | Description |
|------|-------------|
| Functional | Elements contribute to a single function (best) |
| Sequential | Output of one is input to another |
| Communicational | Share same data but unrelated functions |
| Temporal | Related only by time (e.g., startup) |
| Procedural | Related by sequence of operations |
| Logical | Related by category (e.g., all input handling) |
| Coincidental | No meaningful relationship (worst) |

### Goal: High cohesion, low coupling

```
High Cohesion ────► Module focused on one thing
Low Coupling ──────► Module doesn't depend on internals of others
```

---

## 3. Layered Architecture

### Traditional layers

```
┌──────────────────────────────────┐
│     Presentation Layer (UI)      │
│   Controllers, Views, DTOs      │
└──────────┬───────────────────────┘
           │
┌──────────▼───────────────────────┐
│     Application Layer            │
│   Use Cases, Application Services│
└──────────┬───────────────────────┘
           │
┌──────────▼───────────────────────┐
│     Domain Layer (Business)      │
│   Entities, Value Objects,       │
│   Domain Services, Repositories  │
└──────────┬───────────────────────┘
           │
┌──────────▼───────────────────────┐
│     Infrastructure Layer         │
│   DB, Cache, Queue, External API │
└──────────────────────────────────┘
```

**Dependency rule:** Upper layers depend on lower layers. Never the reverse.

### Common pitfall: "Anemic Domain Model"

```php
// ANEMIC: Domain objects are just data bags — all logic in services
class Order
{
    public string $status;       // public, no behavior
    public array $items;
}

class OrderService
{
    public function placeOrder(Order $order): void
    {
        // All logic here — procedural, not OOP
        $order->status = 'pending';
        // ...
    }
}

// RICH: Domain objects contain behavior
class Order
{
    public function __construct(
        private readonly OrderId $id,
        private OrderStatus $status = OrderStatus::Draft,
        private array $items = [],
    ) {}

    public function place(): void
    {
        if ($this->items === []) {
            throw new DomainException('Cannot place empty order');
        }
        $this->status = OrderStatus::Pending;
    }
}
```

> **Trap:** Laravel Eloquent models often become anemic (all logic in services/controllers). Push business rules into domain objects (DTOs, Action classes, or dedicated Domain models separate from Eloquent).

---

## 4. Clean Architecture

### Dependency Rule

> Source code dependencies point INWARD. Nothing in an inner circle can know about an outer circle.

```
┌─────────────────────────────────────┐
│  Frameworks & Drivers               │
│  (Web, DB, UI, External APIs)       │  ▲
│                                     │  │
│  ┌─────────────────────────────┐    │  │
│  │  Interface Adapters         │    │  │
│  │  (Controllers, Presenters,  │    │  │
│  │   Gateways)                 │    │  │
│  │                             │    │  │
│  │  ┌─────────────────────┐    │    │  │
│  │  │  Application        │    │    │  │
│  │  │  (Use Cases)        │    │    │  │
│  │  │                     │    │    │  │
│  │  │  ┌─────────────┐    │    │    │  │
│  │  │  │  Domain     │    │    │    │  │
│  │  │  │  Entities   │    │    │    │  │
│  │  │  └─────────────┘    │    │    │  │
│  │  └─────────────────────┘    │    │  │
│  └─────────────────────────────┘    │  │
└─────────────────────────────────────┘  │
                 Dependencies point inward
```

### Clean Architecture in Go (Chronos example)

```go
// Domain layer — entities and business rules
type Job struct {
    ID        JobID
    Schedule  string
    Status    JobStatus
    CreatedAt time.Time
}

type JobService interface {
    Schedule(job *Job) error
    Cancel(id JobID) error
    Run(id JobID) error
}

// Application layer — use cases
type Scheduler struct {
    jobs    JobService
    clock   Clock
}

func (s *Scheduler) CreateJob(name string, schedule string) (*Job, error) {
    // Validate schedule format (domain rule)
    if !validCron(schedule) {
        return nil, ErrInvalidSchedule
    }
    // ...
    return s.jobs.Schedule(job)
}

// Infrastructure layer — concrete implementations
type PostgresJobService struct {
    db *sql.DB
}

func (p *PostgresJobService) Schedule(job *Job) error {
    // actual DB persistence
}
```

### Clean Architecture in Laravel

```
app/
├── Domain/
│   ├── Models/              # Rich domain models (not Eloquent!)
│   │   └── Order.php
│   ├── ValueObjects/
│   │   └── Money.php
│   └── Services/
│       └── PricingService.php
├── Application/
│   └── UseCases/
│       ├── PlaceOrderUseCase.php
│       └── CancelOrderUseCase.php
├── Infrastructure/
│   ├── Persistence/
│   │   └── EloquentOrderRepository.php
│   └── Queue/
│       └── LaravelEventBus.php
└── Interface/
    ├── Controllers/
    │   └── OrderController.php
    └── Requests/
        └── PlaceOrderRequest.php
```

> **Trap:** Clean Architecture adds indirection. For CRUD-heavy apps without complex business rules, it's over-engineering. Apply it selectively to the complex parts of your system.

---

## 5. Hexagonal Architecture (Ports & Adapters)

```
                    ┌──────────────────┐
                    │     Adapter      │
                    │  (Web Controller)│
                    └───────┬──────────┘
                            │ Port (Interface)
                    ┌───────▼──────────┐
                    │                  │
┌──────────┐ Port   │    Application   │   Port   ┌──────────┐
│ Adapter  │◄───────┤      Core        ├─────────►│ Adapter  │
│ (MySQL)  │        │                  │          │ (SQS)    │
└──────────┘        └──────────────────┘          └──────────┘
                    │                  │
                    └───────┬──────────┘
                            │ Port (Interface)
                    ┌───────▼──────────┐
                    │     Adapter      │
                    │  (Redis)         │
                    └──────────────────┘
```

**Ports** = interfaces (contracts). **Adapters** = implementations (DB, web, queue, etc.).

The core application knows nothing about the outside world. It only knows ports. Adapters plug into ports.

---

## 6. Dependency Injection

### Without DI (tight coupling)

```php
class OrderController
{
    public function place(Request $request): Response
    {
        $service = new OrderService(              // Controller creates dependencies
            new MySQLOrderRepository(new DB()),    // Chain of new's
            new EmailNotifier()
        );
        return $service->place($request);
    }
}
```

### With DI (loose coupling)

```php
class OrderController
{
    public function __construct(
        private PlaceOrderUseCase $useCase,       // Injected via constructor
    ) {}

    public function place(Request $request): Response
    {
        return $this->useCase->execute($request);
    }
}
```

**Benefits:**
- Swap implementations (test double vs real)
- Change behavior without modifying code
- Clear dependency graph

**Laravel's service container:**
```php
// AppServiceProvider.php
public function registers(): void
{
    $this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
    $this->app->bind(EventBus::class, RabbitMQEventBus::class);
}
```

---

## 7. Separation of Concerns

### Principles

| Principle | Description |
|-----------|-------------|
| **SoC** | Separate distinct concerns into distinct modules |
| **DRY** | Don't repeat yourself |
| **YAGNI** | You ain't gonna need it |
| **KISS** | Keep it simple, stupid |
| **Law of Demeter** | Talk only to your immediate friends |

### Law of Demeter example

```php
// VIOLATION: chaining through multiple objects
$order->getCustomer()->getAddress()->getCity();

// FOLLOWS: direct method on the object that matters
$order->getCustomerCity();
```

### DRY vs premature abstraction

```php
// DRY taken too far — two unrelated things forced into one abstraction
class EntityRepository  // Repository for ALL entities
{
    public function save(string $type, array $data): void { ... }
    public function find(string $type, int $id): array { ... }
}

// BETTER: Separate repositories per aggregate
class OrderRepository { ... }
class ProductRepository { ... }
```

> **Trap:** DRY is good for BUSINESS LOGIC, not for coincidental code similarity. Don't abstract two things that just happen to have similar code but serve different purposes. The abstraction will break when they diverge.

---

## 8. Q&A

**Q: What's the difference between Clean Architecture and Hexagonal Architecture?**
A: Both aim to isolate domain logic. Clean Architecture emphasizes the dependency rule with concentric circles. Hexagonal emphasizes ports and adapters at the boundary. They're complementary — Clean describes the overall structure, Hexagonal describes the boundary interaction.

**Q: What's the most important SOLID principle for a senior engineer?**
A: Dependency Inversion. It enables testability, flexibility, and is the foundation for Clean/Hexagonal architecture.

**Q: What's tight coupling and why is it bad?**
A: When a module depends heavily on the internal details of another. Changes propagate, testing is hard, swapping implementations requires code changes.

**Q: What's the dependency rule in Clean Architecture?**
A: Dependencies point inward. Outer layers (frameworks, DB, UI) depend on inner layers (domain, use cases). Inner layers NEVER depend on outer layers.

**Q: What's an anemic domain model?**
A: Domain objects with no behavior — just getters/setters. All logic is in services. Anti-pattern — violates encapsulation.

**Q: When should you NOT use Clean Architecture?**
A: Simple CRUD apps, prototypes, small projects with simple business rules. The indirection adds unnecessary complexity. Apply it to complex domains only.

**Q: What's the difference between entity and value object?**
A: Entity: has identity (two entities are different even if same attributes). Value Object: defined by attributes (two VOs with same values are equal). Money($100, USD) is a VO. Order ID is an entity.
