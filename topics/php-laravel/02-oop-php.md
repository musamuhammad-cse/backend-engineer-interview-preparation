# PHP / Laravel — Tier 2: Object-Oriented Design

> **This file is the canonical source for SOLID and design patterns in this study set.** It builds on the language-level OOP mechanics in [`01-basic.md`](./01-basic.md) §8 (visibility, traits, late static binding, magic methods) and goes into design: SOLID applied to real Laravel code, patterns worth knowing, composition over inheritance, value objects, and the anti-patterns interviewers probe for.
>
> Senior interviewers test whether you can *apply* these to a real problem and say when **not** to — not whether you can recite definitions. Every pattern below has a "when this is the wrong choice" note for that reason.

---

## Table of Contents

1. [SOLID Principles with Laravel Code](#1-solid-principles-with-laravel-code)
2. [Covariance & Contravariance — the Mechanics of LSP](#2-covariance--contravariance--the-mechanics-of-lsp)
3. [Design Patterns in the Wild](#3-design-patterns-in-the-wild)
4. [Composition vs Inheritance — The Laravel Way](#4-composition-vs-inheritance--the-laravel-way)
5. [Dependency Injection & SOLID](#5-dependency-injection--solid)
6. [Interfaces vs Abstract Classes — Deep Comparison](#6-interfaces-vs-abstract-classes--deep-comparison)
7. [Traits — Advanced Patterns & Pitfalls](#7-traits--advanced-patterns--pitfalls)
8. [Immutability & Value Objects](#8-immutability--value-objects)
9. [Design by Contract](#9-design-by-contract)
10. [PHP 8.x OOP Features](#10-php-8x-oop-features)
11. [Anti-Patterns & Code Smells](#11-anti-patterns--code-smells)
12. [Tier 2 Q&A Drill](#12-tier-2-qa-drill)

---

## 1. SOLID Principles with Laravel Code

### Single Responsibility Principle (SRP)

> A class should have one, and only one, reason to change.

```php
// BAD — this class does three things
class InventoryItem extends Model
{
    public function syncToSearchIndex(): void
    {
        // 1. Business logic
        $doc = ['id' => $this->id, 'sku' => $this->sku, 'qty' => $this->quantity];
        
        // 2. Infrastructure
        Http::withHeaders(['X-API-Key' => config('services.elasticsearch.key')])
            ->put("https://es:9200/items/{$this->id}", $doc);
        
        // 3. Notification
        Mail::to('admin@example.com')->send(new SearchIndexUpdated($this));
    }
}

// GOOD — each class has one reason to change
class SyncItemToSearchIndex
{
    public function __construct(
        private readonly SearchClient $search,
        private readonly Dispatcher $events,
    ) {}

    public function execute(InventoryItem $item): void
    {
        $this->search->index($item->toSearchDocument());
        $this->events->dispatch(new ItemIndexed($item->id));
    }
}

class NotifyAdminOnIndexFailure
{
    public function handle(ItemIndexFailed $event): void
    {
        Mail::to(config('admin.email'))->send(new SearchIndexFailureAlert($event->item));
    }
}
```

> **Trap:** Models are the most common SRP violation in Laravel. A model with 500+ lines of scopes, accessors, observers, and business logic has five reasons to change. Extract domain logic into Action classes and keep models focused on attributes, relationships, and scopes.

> **Follow-up:** *How do you know when to extract?* When a method references infrastructure (HTTP, mail, queue, cache) from a model, or when a model method exceeds ~20 lines. Also: when you can't test a business rule without booting the entire Laravel stack.

---

### Open/Closed Principle (OCP)

> Open for extension, closed for modification.

```php
// BAD — adding a channel means modifying the dispatcher
class NotificationService
{
    public function send(Notification $notification, User $user): void
    {
        if ($notification->channel === 'email') {
            Mail::to($user)->send(new NotificationMail($notification));
        } elseif ($notification->channel === 'sms') {
            SMS::send($user->phone, $notification->body);
        } elseif ($notification->channel === 'slack') {
            Slack::message($user->slack_id, $notification->body);
        }
        // ... modification for every new channel
    }
}

// GOOD — new channels are new classes, no modification
interface NotificationChannel
{
    public function send(Notification $notification, User $user): void;
}

final class EmailChannel implements NotificationChannel
{
    public function send(Notification $notification, User $user): void
    {
        Mail::to($user)->send(new NotificationMail($notification));
    }
}

final class SmsChannel implements NotificationChannel
{
    public function send(Notification $notification, User $user): void
    {
        SMS::send($user->phone, $notification->body);
    }
}

// ⚠️ This binding is NOT open/closed — the match arm grows with every new channel.
// It moves the modification out of the dispatcher, but it doesn't eliminate it.
$this->app->bind(NotificationChannel::class, fn ($app) => match (config('notification.default')) {
    'email' => $app->make(EmailChannel::class),
    'sms'   => $app->make(SmsChannel::class),
    'slack' => $app->make(SlackChannel::class),
});

// ✅ Genuinely open/closed — tag the implementations and let each declare what it supports.
// Adding WebhookChannel means writing one class and adding it to the tag list. Nothing else changes.
$this->app->tag([
    EmailChannel::class,
    SmsChannel::class,
    SlackChannel::class,
], 'notification.channels');

$this->app->bind(
    ChannelDispatcher::class,
    fn ($app) => new ChannelDispatcher($app->tagged('notification.channels'))
);
```

```php
interface NotificationChannel
{
    public function supports(Organization $org): bool;
    public function send(Notification $notification, User $user): void;
}

final class ChannelDispatcher
{
    /** @param iterable<NotificationChannel> $channels */
    public function __construct(private readonly iterable $channels) {}

    public function dispatch(Notification $notification, User $user): void
    {
        foreach ($this->channels as $channel) {
            if ($channel->supports($user->organization)) {
                $channel->send($notification, $user);
            }
        }
    }
}
```

> **The distinction that matters in an interview:** moving a conditional from a service into a container binding is *not* OCP — you've just relocated the modification. True OCP means the extension point is registration, and the dispatcher never learns that a new implementation exists. Say that explicitly; most candidates present the `match` version and call it done.

> **Real-world tie-in:** In your multi-tenant SaaS, different organizations enable different channels. With the tagged version, each channel answers `supports()` from the org's own config, so per-tenant channel selection needs no dispatcher changes at all.

> **When OCP is the wrong call:** if there will only ever be two implementations and they never change, an `if`/`match` is more readable than an interface, three classes, and a container tag. Abstraction you don't need is a cost, not an investment.

---

### Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types without altering correctness.

```php
// BAD — Square breaks Rectangle's contract
class Rectangle
{
    public function __construct(
        protected int $width,
        protected int $height,
    ) {}

    public function setWidth(int $width): void { $this->width = $width; }
    public function setHeight(int $height): void { $this->height = $height; }
    public function area(): int { return $this->width * $this->height; }
}

class Square extends Rectangle
{
    public function setWidth(int $width): void
    {
        $this->width = $width;
        $this->height = $width;  // surprise! changing width also changes height
    }
}

function printArea(Rectangle $r): void
{
    $r->setWidth(5);
    $r->setHeight(4);
    echo $r->area();  // expect 20, get 16 if $r is a Square
}

// GOOD — use composition or separate hierarchies
interface Shape
{
    public function area(): int;
}

final readonly class Rectangle implements Shape
{
    public function __construct(public int $width, public int $height) {}
    public function area(): int { return $this->width * $this->height; }
}

final readonly class Square implements Shape
{
    public function __construct(public int $side) {}
    public function area(): int { return $this->side * $this->side; }
}
```

### The four ways to break LSP

A subtype must not:

| Violation | Example |
|-----------|---------|
| **Strengthen a precondition** | Parent accepts any `int`; child throws unless it's positive |
| **Weaken a postcondition** | Parent guarantees a sorted result; child returns unsorted |
| **Throw a new exception type** | Parent's contract says "returns null if missing"; child throws |
| **Change observable side effects** | `Square::setWidth()` also mutating height |

```php
// Real Laravel example — a repository that silently changes the contract
interface StockRepository
{
    /** @throws ModelNotFoundException when the item does not exist */
    public function findOrFail(int $id, int $orgId): InventoryItem;
}

final class CachingStockRepository implements StockRepository
{
    public function findOrFail(int $id, int $orgId): InventoryItem
    {
        // ❌ LSP violation: caller catches ModelNotFoundException and gets null instead
        return $this->cache->get("item:{$orgId}:{$id}") ?? $this->inner->findOrFail($id, $orgId);
    }
}
```

The bug: if the cache holds `null` for a miss, `??` falls through correctly — but if the cache legitimately stores a "known absent" marker, the decorator returns something the interface says is impossible. Decorators are the most common place LSP breaks in Laravel codebases, because it's tempting to "improve" behavior while wrapping.

> **Trap:** LSP is about the *contract*, not the type signature. PHP's type system will happily let you violate it — throwing a new exception type or returning an empty collection where the parent returned a populated one both compile fine. Only tests and documentation catch it.

> **Follow-up:** *How do you enforce LSP?* Write the test suite against the **interface**, then run it against every implementation. Pest datasets make this cheap:

```php
it('honours the StockRepository contract', function (string $implementation) {
    $repo = app($implementation);

    expect(fn () => $repo->findOrFail(999_999, 1))
        ->toThrow(ModelNotFoundException::class);
})->with([
    EloquentStockRepository::class,
    CachingStockRepository::class,
    InMemoryStockRepository::class,
]);
```

---

### Interface Segregation Principle (ISP)

> No client should be forced to depend on methods it does not use.

```php
// BAD — fat interface forces implementations to stub unused methods
interface InventoryItemInterface
{
    public function deductStock(int $amount): void;
    public function addSupplier(Supplier $supplier): void;
    public function generateReport(): Report;
    public function syncToSearchIndex(): void;
    public function calculateShipping(): Money;
}

// GOOD — focused interfaces
interface StockAdjustable
{
    public function deductStock(int $amount): void;
    public function addStock(int $amount): void;
}

interface Searchable
{
    public function toSearchDocument(): array;
    public function indexedAt(): ?Carbon;
}

interface Reportable
{
    public function generateReport(): Report;
}

// Laravel itself follows this — look at the Contracts directory:
// Illuminate\Contracts\Queue\ShouldQueue
// Illuminate\Contracts\Auth\Authenticatable
// Illuminate\Contracts\Cache\Repository
// Illuminate\Contracts\Container\Container
```

---

### Dependency Inversion Principle (DIP)

> Depend on abstractions, not concretions. High-level modules should not depend on low-level modules; both should depend on abstractions.

```php
// BAD — high-level domain depends directly on low-level infrastructure
class InventoryService
{
    public function syncToElasticsearch(InventoryItem $item): void
    {
        $client = \Elastic\Elasticsearch\ClientBuilder::create()
            ->setHosts(['https://es:9200'])
            ->build();
        
        $client->index([
            'index' => 'inventory',
            'id'    => $item->id,
            'body'  => $item->toArray(),
        ]);
    }
}

// GOOD — domain depends on abstraction; infrastructure is injected
interface ItemIndexer
{
    public function index(InventoryItem $item): void;
    public function remove(int $itemId): void;
}

final class ElasticsearchItemIndexer implements ItemIndexer
{
    public function __construct(
        private readonly Client $client,
        private readonly string $index,
    ) {}

    public function index(InventoryItem $item): void
    {
        $this->client->index([
            'index' => $this->index,
            'id'    => $item->id,
            'body'  => $item->toSearchDocument(),
        ]);
    }
}

final class InventoryService
{
    public function __construct(
        private readonly ItemIndexer $indexer,
    ) {}

    public function setStockLevel(InventoryItem $item, int $newQuantity, int $expectedVersion): void
    {
        // Optimistic lock — a blind read-modify-write here would be a lost update.
        // See 04-senior.md §4 for the full treatment.
        $affected = InventoryItem::query()
            ->whereKey($item->getKey())
            ->where('lock_version', $expectedVersion)
            ->update([
                'quantity'     => $newQuantity,
                'lock_version' => $expectedVersion + 1,
            ]);

        if ($affected === 0) {
            throw new ConcurrentModificationException($item->getKey());
        }

        $this->indexer->index($item->refresh());
    }
}
```

> **The test argument:** With the abstracted version, you can test `InventoryService` by injecting a `FakeItemIndexer` that records calls. With the concrete version, you're testing Elasticsearch's client library, not your business logic.

> **Trap:** DIP is not "put an interface in front of everything." The test is whether the *dependency direction* is inverted — does the domain define the contract, or does infrastructure? `ItemIndexer` lives in your domain namespace and is written in your domain's language (`index(InventoryItem)`); `ElasticsearchItemIndexer` lives in infrastructure and depends on the domain. If you'd created `interface ElasticsearchClientInterface { public function bulk(array $params); }`, you'd have added an interface without inverting anything — the domain would still be thinking in Elasticsearch terms.

---

## 2. Covariance & Contravariance — the Mechanics of LSP

This is the PHP-specific machinery behind LSP, and the most likely follow-up after "explain Liskov." PHP has supported full variance since 7.4.

### Covariant return types — narrowing is safe

A child may return a **more specific** type than the parent declares.

```php
abstract class RepositoryFactory
{
    abstract public function make(): Repository;
}

final class StockRepositoryFactory extends RepositoryFactory
{
    public function make(): StockRepository   // ✅ narrower than Repository — allowed
    {
        return new EloquentStockRepository();
    }
}
```

**Why it's safe:** every caller expects *at least* a `Repository`. Getting a `StockRepository` satisfies that and more. Nothing can break.

`static` return types are the everyday case — that's why fluent builders work in subclasses:

```php
class Builder
{
    public function where(string $column): static { return $this; }
}
```

### Contravariant parameter types — widening is safe

A child may accept a **more general** type than the parent declares.

```php
class Logger
{
    public function log(RuntimeException $e): void {}
}

final class VerboseLogger extends Logger
{
    public function log(Throwable $e): void {}   // ✅ wider than RuntimeException — allowed
}
```

**Why it's safe:** every caller passes at most a `RuntimeException`. A method accepting any `Throwable` handles that fine.

### The two that are forbidden

```php
class Base
{
    public function handle(Throwable $e): Throwable {}
}

final class Broken extends Base
{
    // ❌ Fatal: narrowing a parameter. A caller passing a TypeError would break.
    public function handle(RuntimeException $e): Throwable {}
}

final class AlsoBroken extends Base
{
    // ❌ Fatal: widening a return. A caller expecting Throwable might get a string.
    public function handle(Throwable $e): mixed {}
}
```

### The memory aid

> **Parameters widen, returns narrow.** Or: *be liberal in what you accept, conservative in what you return* — Postel's law, enforced by the type system.

### Property types are invariant

```php
class Base { public Throwable $error; }

final class Child extends Base
{
    public RuntimeException $error;   // ❌ Fatal: property types must match exactly
}
```

**Why:** a property is both read and written. Narrowing would break writers; widening would break readers. Since it does both, only exact match is safe. PHP 8.4's asymmetric visibility doesn't change this — but property *hooks* let you get a similar effect by making the property virtual.

### `#[Override]` (PHP 8.3) — catching the silent LSP break

```php
class BaseObserver
{
    public function updated(Model $model): void {}
}

final class ItemObserver extends BaseObserver
{
    #[Override]
    public function updated(Model $model): void {}   // ✅ verified at compile time

    #[Override]
    public function updatedd(Model $model): void {}  // ❌ Fatal: nothing to override
}
```

> **Why this matters practically:** a typo in an overridden method name means your override silently never runs — the parent's version executes instead and the bug is invisible. `#[Override]` turns that into a compile-time error. Worth mentioning as a small habit that catches a real class of bug.

> **Follow-up:** *Does Laravel use variance anywhere you'd notice?* Yes — `Model::newCollection()` returning a narrowed collection type, `static` returns throughout the query builder, and `newEloquentBuilder()` returning a custom builder subclass. All three depend on covariant returns.

---

## 3. Design Patterns in the Wild

### Strategy Pattern

Used when you need to swap algorithms at runtime based on configuration or input.

```php
// The strategy interface
interface PricingStrategy
{
    public function calculatePrice(OrderItem $item, Organization $org): Money;
}

// Concrete strategies
final class StandardPricing implements PricingStrategy
{
    public function calculatePrice(OrderItem $item, Organization $org): Money
    {
        return $item->unitPrice->multiply($item->quantity);
    }
}

final class BulkPricing implements PricingStrategy
{
    public function calculatePrice(OrderItem $item, Organization $org): Money
    {
        $base = $item->unitPrice->multiply($item->quantity);
        
        if ($item->quantity >= 100) {
            return $base->multiply(0.8);  // 20% discount
        }
        
        if ($item->quantity >= 50) {
            return $base->multiply(0.9);  // 10% discount
        }
        
        return $base;
    }
}

final class TieredPricing implements PricingStrategy
{
    public function calculatePrice(OrderItem $item, Organization $org): Money
    {
        $tiers = $org->pricingTiers->sortByDesc('min_quantity');
        
        foreach ($tiers as $tier) {
            if ($item->quantity >= $tier->min_quantity) {
                return $item->unitPrice->multiply($item->quantity)->multiply($tier->multiplier);
            }
        }
        
        return $item->unitPrice->multiply($item->quantity);
    }
}

// Context
final class OrderPricingService
{
    public function __construct(
        /** @var array<string, PricingStrategy> keyed by Organization::$pricing_model */
        private readonly array $strategies,
    ) {}

    public function calculateLineTotal(OrderItem $item, Organization $org): Money
    {
        $strategy = $this->resolveStrategy($org);
        return $strategy->calculatePrice($item, $org);
    }

    private function resolveStrategy(Organization $org): PricingStrategy
    {
        return $this->strategies[$org->pricing_model] ?? new StandardPricing();
    }
}

// Container binding
$this->app->singleton(OrderPricingService::class, function ($app) {
    return new OrderPricingService([
        'standard' => new StandardPricing(),
        'bulk'     => new BulkPricing(),
        'tiered'   => new TieredPricing(),
    ]);
});
```

> **Real-world tie-in:** Different tenants in your SaaS might have different pricing models. Strategy pattern lets you add new models without touching the order processing code — just add a new class and a config entry.

> **When Strategy is the wrong call:** if the "strategies" differ by one number, you want a config value, not a class hierarchy. `BulkPricing` and `TieredPricing` above are arguably the same strategy with different tier tables — three classes where one class plus a `pricing_tiers` row would do. The test: if adding a variant means writing a class whose body is a different constant, it's data, not behaviour.

> **Follow-up:** *How do you pick the strategy without a `match` that grows forever?* Either let each strategy declare what it handles (`supports()`, as in the OCP section), or key a map by an enum so exhaustiveness is checked. Reaching for a growing `match` is the thing OCP is trying to prevent.

---

### Factory Pattern

```php
// Simple Factory
final class NotificationFactory
{
    public function __construct(
        private readonly array $channels,
    ) {}

    public function create(string $channel): NotificationChannel
    {
        return $this->channels[$channel]
            ?? throw new InvalidArgumentException("Unknown channel: {$channel}");
    }
}

// Abstract Factory — creates families of related objects
interface WarehouseDocumentFactory
{
    public function createPickList(Order $order): PickList;
    public function createPackingSlip(Order $order): PackingSlip;
    public function createShippingLabel(Shipment $shipment): ShippingLabel;
}

final class DomesticDocumentFactory implements WarehouseDocumentFactory
{
    public function createPickList(Order $order): PickList
    {
        return new DomesticPickList($order);
    }

    public function createPackingSlip(Order $order): PackingSlip
    {
        return new DomesticPackingSlip($order);
    }

    public function createShippingLabel(Shipment $shipment): ShippingLabel
    {
        return new DomesticShippingLabel($shipment);
    }
}

final class InternationalDocumentFactory implements WarehouseDocumentFactory
{
    public function createPickList(Order $order): PickList
    {
        return new InternationalPickList($order);  // includes customs info
    }

    public function createPackingSlip(Order $order): PackingSlip
    {
        return new InternationalPackingSlip($order);
    }

    public function createShippingLabel(Shipment $shipment): ShippingLabel
    {
        return new InternationalShippingLabel($shipment);  // includes customs declaration
    }
}
```

> **What Abstract Factory buys over three separate factories:** it guarantees the family is *consistent*. You cannot accidentally pair a domestic packing slip with an international shipping label, because one factory produces the whole set. That consistency guarantee is the entire justification — if the products don't need to match each other, you want simple factories, not an abstract one.

> **The Laravel angle worth naming:** `Illuminate\Support\Manager` is the framework's own factory base class, and it's what `Cache`, `Queue`, `Mail`, `Session`, and `Auth` all extend. It resolves a driver by name from config, memoises it, and lets you register your own with `extend()`:
>
> ```php
> Cache::extend('tenant-redis', fn ($app, $config) => Cache::repository(
>     new TenantAwareRedisStore($app['redis'], $config['prefix'])
> ));
> ```
>
> Saying "Laravel's driver system is the Factory pattern via `Manager`, and `extend()` is the registration hook" is much stronger than reciting the GoF definition.

> **Terminology trap:** `Model::factory()` is *not* this pattern — that's a test data builder (Factory Boy / fixture replacement). Interviewers occasionally ask "you use factories in Laravel, right?" to see whether you distinguish the two.

---

### Observer Pattern

```php
// Domain event (the "subject" notification)
final class StockLevelChanged
{
    public function __construct(
        public readonly int $itemId,
        public readonly int $organizationId,
        public readonly int $oldQuantity,
        public readonly int $newQuantity,
    ) {}
}

// Observer interface
interface StockLevelObserver
{
    public function handle(StockLevelChanged $event): void;
}

// Concrete observers
final class ReorderNotifier implements StockLevelObserver
{
    public function __construct(
        private readonly ReorderThreshold $threshold,
        private readonly NotificationService $notifications,
    ) {}

    public function handle(StockLevelChanged $event): void
    {
        if ($event->newQuantity <= $this->threshold->forItem($event->itemId)) {
            $this->notifications->notifyLowStock($event->itemId, $event->organizationId);
        }
    }
}

final class SearchIndexUpdater implements StockLevelObserver
{
    public function handle(StockLevelChanged $event): void
    {
        UpdateSearchIndex::dispatch($event->itemId, $event->organizationId);
    }
}

final class AuditLogger implements StockLevelObserver
{
    public function handle(StockLevelChanged $event): void
    {
        AuditLog::create([
            'organization_id' => $event->organizationId,
            'event'           => 'stock_level_changed',
            'item_id'         => $event->itemId,
            'old_quantity'    => $event->oldQuantity,
            'new_quantity'    => $event->newQuantity,
        ]);
    }
}

// In Laravel, use events/listeners — this is the same pattern.
// The "subject" is the Dispatcher; the registry is $listen (or auto-discovery in L11+).
final class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        StockLevelChanged::class => [
            ReorderNotifier::class,
            SearchIndexUpdater::class,
            AuditLogger::class,
        ],
    ];
}
```

> **Two distinctions the interviewer is listening for:**
>
> **Eloquent observers vs domain events.** `ItemObserver::updated()` fires on a *persistence* event and knows nothing about intent — it can't tell a stock deduction from a typo correction in the name, and it fires inside the caller's transaction. `StockLevelChanged` is a *domain* event: it's dispatched deliberately, carries meaning, and can be deferred past commit. Use observers for mechanical concerns (slugs, UUIDs, cache busting) and domain events for anything a business person would recognise as a thing that happened.
>
> **`ShouldQueue` and `afterCommit`.** A synchronous listener runs inside the transaction, so a slow one (an HTTP call to Elasticsearch) holds row locks open for its entire duration — this is a classic source of lock contention under load. A queued listener dispatched *before* commit can be picked up by a worker that then reads a row the transaction hasn't committed yet, so the job fails on a record that "doesn't exist." Both problems, one answer:
>
> ```php
> final class SearchIndexUpdater implements ShouldQueue
> {
>     public $afterCommit = true;   // or set 'after_commit' => true on the queue connection
> }
> ```

> **When Observer is the wrong call:** when the reaction is mandatory and the caller must know whether it succeeded. Events are fire-and-forget by design — the dispatcher gets no meaningful result and a listener throwing may or may not surface. If "deduct stock" *must* also "reserve the pick slot" or the whole thing rolls back, that's a direct call inside the transaction, not an event. Overusing events makes control flow invisible: a new engineer reading `deductStock()` cannot tell that seven other things happen.

---

### Repository Pattern

```php
// Contract
interface StockRepository
{
    public function findForOrganization(int $id, int $orgId): ?InventoryItem;
    public function save(InventoryItem $item): void;
    public function decrementStock(int $itemId, int $orgId, int $amount): int;
    public function lowStock(int $orgId, int $threshold = 10): Collection;
}

// Implementation
final class EloquentStockRepository implements StockRepository
{
    public function findForOrganization(int $id, int $orgId): ?InventoryItem
    {
        return InventoryItem::where('id', $id)
            ->where('organization_id', $orgId)
            ->first();
    }

    public function save(InventoryItem $item): void
    {
        $item->save();
    }

    public function decrementStock(int $itemId, int $orgId, int $amount): int
    {
        return InventoryItem::query()
            ->where('id', $itemId)
            ->where('organization_id', $orgId)
            ->where('quantity', '>=', $amount)
            ->decrement('quantity', $amount);
    }

    public function lowStock(int $orgId, int $threshold = 10): Collection
    {
        return InventoryItem::where('organization_id', $orgId)
            ->whereColumn('quantity', '<=', 'reorder_point')
            ->where('quantity', '<=', $threshold)
            ->get();
    }
}

// Test double
final class InMemoryStockRepository implements StockRepository
{
    private array $items = [];

    public function findForOrganization(int $id, int $orgId): ?InventoryItem
    {
        return $this->items["{$orgId}:{$id}"] ?? null;
    }

    public function save(InventoryItem $item): void
    {
        $this->items["{$item->organization_id}:{$item->id}"] = $item;
    }

    public function decrementStock(int $itemId, int $orgId, int $amount): int
    {
        $key = "{$orgId}:{$itemId}";
        
        if (! isset($this->items[$key]) || $this->items[$key]->quantity < $amount) {
            return 0;
        }

        $this->items[$key]->quantity -= $amount;
        return 1;
    }

    public function lowStock(int $orgId, int $threshold = 10): Collection
    {
        return collect($this->items)
            ->filter(fn ($item) => $item->organization_id === $orgId && $item->quantity <= $threshold)
            ->values();
    }
}
```

> **When NOT to use Repository:** When your interface just proxies Eloquent methods one-to-one (`find`, `save`, `delete`), you're adding indirection without benefit. Use repositories when the domain needs to be decoupled from Eloquent (e.g., testing without a database, swapping to an API client, or when the query logic is complex and reusable).

---

### Decorator Pattern

```php
// Base interface
interface InventoryResolver
{
    public function resolve(int $itemId, int $orgId): InventoryItem;
}

// Concrete implementation
final class DatabaseInventoryResolver implements InventoryResolver
{
    /** @param class-string<InventoryItem> $itemClass */
    public function __construct(
        private readonly string $itemClass = InventoryItem::class,
    ) {}

    public function resolve(int $itemId, int $orgId): InventoryItem
    {
        return $this->itemClass::query()
            ->where('id', $itemId)
            ->where('organization_id', $orgId)
            ->firstOrFail();
    }
}

// Decorator — adds caching
final class CachingInventoryResolver implements InventoryResolver
{
    public function __construct(
        private readonly InventoryResolver $inner,
        private readonly Repository $cache,
        private readonly int $ttl = 300,
    ) {}

    public function resolve(int $itemId, int $orgId): InventoryItem
    {
        return $this->cache->remember(
            "org:{$orgId}:item:{$itemId}",
            $this->ttl,
            fn () => $this->inner->resolve($itemId, $orgId),
        );
    }
}

// Decorator — adds metrics
final class MeasuredInventoryResolver implements InventoryResolver
{
    public function __construct(
        private readonly InventoryResolver $inner,
        private readonly Metrics $metrics,
    ) {}

    public function resolve(int $itemId, int $orgId): InventoryItem
    {
        $start = microtime(true);
        
        try {
            return $this->inner->resolve($itemId, $orgId);
        } finally {
            $this->metrics->histogram('inventory.resolve_ms', (microtime(true) - $start) * 1000);
        }
    }
}

// Composition
$resolver = new MeasuredInventoryResolver(
    new CachingInventoryResolver(
        new DatabaseInventoryResolver(),
        $cache,
        ttl: 300,
    ),
    $metrics,
);
```

> **Follow-up:** *How does `$app->extend()` relate to Decorator?* It's Laravel's built-in way to wrap any binding:
> ```php
> $this->app->extend(InventoryResolver::class, function (InventoryResolver $inner, Application $app) {
>     return new CachingInventoryResolver($inner, $app->make(Repository::class));
> });
> ```
> This is the most practical application of Decorator in Laravel — zero changes to the original class, fully configurable via the container.

> **Trap in the example above:** caching a hydrated Eloquent model is risky. The model serializes with its loaded relations, its `$exists` flag, and whatever `$appends` were set at write time; on read you get a stale object graph that still responds to `save()`. Cache the primitive payload (an array or a DTO) and rehydrate, or cache only the ID and re-fetch. Interviewers who have been burned by this will ask.

> **When Decorator is the wrong call:** if every decorator in the stack is always applied in the same order and never varies, you've built a fixed pipeline with extra indirection — put the behavior in one class. Decorator earns its keep when the composition is *configurable* (cache in prod, no cache in tests, metrics only when sampling is on).

---

### Command Pattern

```php
// The command
final readonly class DeductStockCommand
{
    public function __construct(
        public int $itemId,
        public int $orgId,
        public int $amount,
        public string $reason,
        public string $idempotencyKey,
    ) {}
}

// The handler
final class DeductStockHandler
{
    public function __construct(
        private readonly StockRepository $repo,
        private readonly Dispatcher $events,
    ) {}

    public function handle(DeductStockCommand $command): InventoryItem
    {
        $affected = $this->repo->decrementStock(
            $command->itemId,
            $command->orgId,
            $command->amount,
        );

        if ($affected === 0) {
            $item = $this->repo->findForOrganization($command->itemId, $command->orgId);
            
            throw $item === null
                ? new ModelNotFoundException()
                : new InsufficientStockException($command->itemId, $command->amount, $item->quantity);
        }

        $item = $this->repo->findForOrganization($command->itemId, $command->orgId);

        $this->events->dispatch(new StockDeducted(
            itemId: $item->id,
            organizationId: $command->orgId,
            amount: $command->amount,
            remaining: $item->quantity,
            reason: $command->reason,
        ));

        return $item;
    }
}

// Invoker
final class StockController
{
    public function __construct(
        private readonly DeductStockHandler $handler,
    ) {}

    public function __invoke(DeductStockRequest $request, InventoryItem $item): JsonResponse
    {
        $result = $this->handler->handle(new DeductStockCommand(
            itemId: $item->id,
            orgId: $item->organization_id,
            amount: $request->integer('amount'),
            reason: $request->string('reason'),
            idempotencyKey: $request->header('Idempotency-Key'),
        ));

        return response()->json([
            'item_id' => $result->id,
            'remaining' => $result->quantity,
        ]);
    }
}
```

---

### Adapter Pattern

Wraps a third-party interface you don't control so your domain sees the shape it wants. Distinct from Decorator: a Decorator implements the *same* interface it wraps; an Adapter *converts* one interface to another.

```php
// Your domain's language — you own this
interface ShippingRateProvider
{
    public function quote(Shipment $shipment): Money;
}

// The vendor's language — you don't own this, and it's ugly
// $fedex->getRates(['wt' => 12.5, 'unit' => 'LB', 'dest_zip' => '94103']) => ['total' => '18.40']

final class FedExRateAdapter implements ShippingRateProvider
{
    public function __construct(private readonly FedExClient $fedex) {}

    public function quote(Shipment $shipment): Money
    {
        $response = $this->fedex->getRates([
            'wt'       => $shipment->weightKg * 2.20462,
            'unit'     => 'LB',
            'dest_zip' => $shipment->destination->postalCode,
        ]);

        return new Money((int) round($response['total'] * 100), 'USD');
    }
}

final class DhlRateAdapter implements ShippingRateProvider
{
    public function quote(Shipment $shipment): Money
    {
        // Completely different API shape, same domain interface
        $grams = (int) ($shipment->weightKg * 1000);
        $result = $this->dhl->calculate($grams, $shipment->destination->countryCode);

        return new Money($result->getPriceInMinorUnits(), $result->getCurrency());
    }
}
```

> **Why this is the highest-value pattern in practice:** it's your anti-corruption layer. Vendor SDKs change, get deprecated, and leak their idioms (arrays of magic strings, pounds instead of kilos) throughout your codebase if you let them. One adapter class per vendor means a breaking SDK upgrade touches exactly one file. It also makes vendor swaps a config change, and it makes your domain testable with a `FakeRateProvider`.

> **The distinction interviewers check:** Decorator = same interface in and out, adds behaviour. Adapter = different interface in and out, adds translation. Both wrap. Confusing them is common.

---

### Template Method Pattern

The base class owns the *sequence*; subclasses fill in the *steps*. This is the one legitimate use of inheritance in an otherwise composition-first design, because the algorithm's shape genuinely is shared.

```php
abstract class InventoryImporter
{
    /** The template method — final, so the sequence cannot be broken. */
    final public function import(UploadedFile $file, int $orgId): ImportResult
    {
        $rows = $this->parse($file);

        $this->validateHeaders(array_keys($rows[0] ?? []));

        $result = new ImportResult();

        DB::transaction(function () use ($rows, $orgId, $result): void {
            foreach ($rows as $index => $row) {
                try {
                    $this->importRow($row, $orgId);
                    $result->succeeded++;
                } catch (ValidationException $e) {
                    $result->addFailure($index, $e->getMessage());
                }
            }
        });

        $this->afterImport($result, $orgId);   // hook, optional

        return $result;
    }

    abstract protected function parse(UploadedFile $file): array;
    abstract protected function requiredHeaders(): array;
    abstract protected function importRow(array $row, int $orgId): void;

    /** Hook with a default — subclasses override only if they care. */
    protected function afterImport(ImportResult $result, int $orgId): void
    {
    }

    private function validateHeaders(array $headers): void
    {
        $missing = array_diff($this->requiredHeaders(), $headers);

        if ($missing !== []) {
            throw new MissingHeadersException($missing);
        }
    }
}

final class CsvInventoryImporter extends InventoryImporter
{
    protected function parse(UploadedFile $file): array
    {
        return array_map('str_getcsv', file($file->getRealPath()));
    }

    protected function requiredHeaders(): array
    {
        return ['sku', 'name', 'quantity'];
    }

    protected function importRow(array $row, int $orgId): void
    {
        InventoryItem::updateOrCreate(
            ['organization_id' => $orgId, 'sku' => $row['sku']],
            ['name' => $row['name'], 'quantity' => (int) $row['quantity']],
        );
    }

    protected function afterImport(ImportResult $result, int $orgId): void
    {
        RecalculateStockLevels::dispatch($orgId);
    }
}
```

> **Note `final` on the template method.** Without it a subclass can override `import()` and skip the transaction, the header validation, or the error collection — which defeats the entire point. Marking the template `final` and the steps `abstract`/`protected` is what makes this pattern safe. Interviewers who know the pattern look for that keyword.

> **Template Method vs Strategy:** same goal (vary part of an algorithm), opposite mechanism. Template Method uses **inheritance** and binds at compile time — one subclass per variant, and a variant can't change at runtime. Strategy uses **composition** and binds at runtime — you can swap the strategy per request or per tenant. Prefer Strategy when the variation is a runtime choice, Template Method when it's a fixed taxonomy and the shared sequence is substantial. Laravel's `Illuminate\Console\Command` (`handle()` filled in by you, lifecycle owned by the base) is Template Method.

---

### Null Object Pattern

Replaces `null` checks with an object that implements the interface and does nothing harmless.

```php
interface AuditLogger
{
    public function record(string $event, array $context): void;
}

final class DatabaseAuditLogger implements AuditLogger
{
    public function record(string $event, array $context): void
    {
        AuditLog::create(['event' => $event, 'context' => $context]);
    }
}

final class NullAuditLogger implements AuditLogger
{
    public function record(string $event, array $context): void
    {
        // intentionally does nothing
    }
}
```

```php
// Before — every call site pays the null tax, and one missed check is a fatal
$this->logger?->record('stock.deducted', $context);

// After — the dependency is never null, so there is nothing to check
$this->logger->record('stock.deducted', $context);
```

```php
// Bind per environment; no conditional logic anywhere in the domain
$this->app->bind(
    AuditLogger::class,
    fn () => config('audit.enabled') ? DatabaseAuditLogger::class : NullAuditLogger::class,
);
```

You already use this without naming it: Laravel's `null` cache driver, `array`/`log` mail drivers, and `sync` queue driver are all Null Objects (or close cousins). Naming the pattern when describing your test setup is a cheap, credible signal.

> **When it's the wrong call:** when the absence is *meaningful* and the caller must react differently. A `NullUser` that returns `false` from every permission check is fine; a `NullOrder` that silently swallows `markAsPaid()` hides a real bug. Use Null Object for **optional collaborators**, not for **missing domain entities**.

---

### Builder Pattern

For objects with many optional parameters, where a constructor would be an unreadable positional list.

```php
final class ReportQueryBuilder
{
    private array $filters = [];
    private array $groupBy = [];
    private ?DateRange $period = null;
    private int $limit = 100;

    public function __construct(private readonly int $organizationId) {}

    public function forPeriod(DateRange $period): self
    {
        $clone = clone $this;
        $clone->period = $period;

        return $clone;
    }

    public function filterBy(string $field, mixed $value): self
    {
        $clone = clone $this;
        $clone->filters[$field] = $value;

        return $clone;
    }

    public function groupBy(string ...$fields): self
    {
        $clone = clone $this;
        $clone->groupBy = [...$clone->groupBy, ...$fields];

        return $clone;
    }

    public function build(): ReportQuery
    {
        return new ReportQuery(
            organizationId: $this->organizationId,
            period: $this->period ?? DateRange::lastThirtyDays(),
            filters: $this->filters,
            groupBy: $this->groupBy,
            limit: $this->limit,
        );
    }
}

$query = (new ReportQueryBuilder($orgId))
    ->forPeriod(DateRange::thisQuarter())
    ->filterBy('warehouse_id', 3)
    ->groupBy('category', 'supplier')
    ->build();
```

> **Note the `clone` in every method.** A mutable builder returning `$this` is the common implementation, but it means a "partially configured" builder shared between call sites gets mutated by whoever touches it next. Returning a clone makes the builder immutable and safe to store as a base configuration — the same reasoning behind PSR-7's wither methods. Mentioning that trade-off unprompted is the senior detail here; Laravel's own query builder is *mutable*, which is why `$base = User::where(...)` followed by two different `->where()` chains surprises people.

---

### Specification Pattern

Turns a business rule into a first-class, composable, testable object.

```php
interface Specification
{
    public function isSatisfiedBy(InventoryItem $item): bool;
    public function toQuery(Builder $query): Builder;
}

final class BelowReorderPoint implements Specification
{
    public function isSatisfiedBy(InventoryItem $item): bool
    {
        return $item->quantity <= $item->reorder_point;
    }

    public function toQuery(Builder $query): Builder
    {
        return $query->whereColumn('quantity', '<=', 'reorder_point');
    }
}

final class HasActiveSupplier implements Specification
{
    public function isSatisfiedBy(InventoryItem $item): bool
    {
        return $item->supplier?->is_active === true;
    }

    public function toQuery(Builder $query): Builder
    {
        return $query->whereHas('supplier', fn (Builder $q) => $q->where('is_active', true));
    }
}

final class AndSpecification implements Specification
{
    /** @var list<Specification> */
    private array $specs;

    public function __construct(Specification ...$specs)
    {
        $this->specs = $specs;
    }

    public function isSatisfiedBy(InventoryItem $item): bool
    {
        foreach ($this->specs as $spec) {
            if (! $spec->isSatisfiedBy($item)) {
                return false;
            }
        }

        return true;
    }

    public function toQuery(Builder $query): Builder
    {
        foreach ($this->specs as $spec) {
            $query = $spec->toQuery($query);
        }

        return $query;
    }
}
```

```php
$needsReorder = new AndSpecification(
    new BelowReorderPoint(),
    new HasActiveSupplier(),
);

// Same rule, two execution contexts — and they cannot drift apart
$candidates = $needsReorder->toQuery(InventoryItem::query())->get();   // in SQL
$shouldFlag = $needsReorder->isSatisfiedBy($item);                     // in PHP
```

> **The problem it solves, and why it's worth mentioning for your inventory SaaS:** the same business rule usually ends up written twice — once as an Eloquent scope for the list view, once as an `if` in the service that acts on a single item. They drift, and then the dashboard shows 40 items needing reorder while the job processes 37. Forcing both readings through one object makes divergence impossible.

> **When it's the wrong call:** for a single simple rule, an Eloquent scope is clearer and shorter. Specification earns its keep when rules **compose** (`and`/`or`/`not` combinations chosen at runtime, e.g. user-defined alert conditions) or when the same rule genuinely must run in both SQL and memory.

---

## 4. Composition vs Inheritance — The Laravel Way

### The problem with deep inheritance

```php
// BAD — fragile base class problem
class BaseModel extends Model
{
    public function scopeActive($query)
    {
        return $query->where('active', true);
    }

    public function scopeForOrganization($query, int $orgId)
    {
        return $query->where('organization_id', $orgId);
    }

    public function toArray(): array
    {
        $array = parent::toArray();
        $array['formatted_created_at'] = $this->created_at?->format('M j, Y');
        return $array;
    }
}

class InventoryItem extends BaseModel {}  // inherits everything, whether needed or not
class User extends BaseModel {}          // User doesn't need scopeActive
class Order extends BaseModel {}         // Order has different toArray needs
```

### Traits for cross-cutting concerns

```php
// GOOD — traits compose behavior selectively
trait HasOrganization
{
    public static function bootHasOrganization(): void
    {
        static::addGlobalScope(new OrganizationScope());
        
        static::creating(function (Model $model): void {
            if ($model->organization_id === null) {
                $model->organization_id = app(TenantContext::class)->organizationIdOrFail();
            }
        });
    }

    public function scopeForOrganization(Builder $query, int $orgId): Builder
    {
        return $query->withoutGlobalScope(OrganizationScope::class)
            ->where($this->qualifyColumn('organization_id'), $orgId);
    }

    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }
}

trait HasUuid
{
    public static function bootHasUuid(): void
    {
        static::creating(function (Model $model): void {
            if ($model->uuid === null) {
                $model->uuid = Str::uuid();
            }
        });
    }

    public function getRouteKeyName(): string
    {
        return 'uuid';
    }
}

trait Auditable
{
    public static function bootAuditable(): void
    {
        static::created(fn (Model $model) => static::logAudit($model, 'created'));
        static::updated(fn (Model $model) => static::logAudit($model, 'updated'));
        static::deleted(fn (Model $model) => static::logAudit($model, 'deleted'));
    }

    protected static function logAudit(Model $model, string $event): void
    {
        AuditLog::create([
            'organization_id' => $model->organization_id ?? null,
            'auditable_type'  => $model::class,
            'auditable_id'    => $model->getKey(),
            'event'           => $event,
            'old_attributes'  => $event === 'updated' ? $model->getOriginal() : null,
            'new_attributes'  => $model->getAttributes(),
        ]);
    }
}

// Usage — compose only what you need
class InventoryItem extends Model
{
    use HasOrganization, HasUuid, Auditable;
}

class User extends Model
{
    use HasOrganization, HasUuid;  // no Auditable — users aren't audited
}

class Order extends Model
{
    use HasOrganization, Auditable;  // no HasUuid — orders use auto-increment IDs
}
```

> **Trap in the `Auditable` trait above — model events do not fire on bulk operations.** `Model::created/updated/deleted` hook the *model* lifecycle, so none of these produce an audit row:
>
> ```php
> InventoryItem::where('organization_id', 5)->update(['active' => false]);  // no events
> InventoryItem::insert([...]);                                            // no events
> DB::table('inventory_items')->delete();                                  // no events
> InventoryItem::where(...)->delete();                                     // no per-model events
> ```
>
> An audit log with silent holes is worse than no audit log, because you'll trust it. If the audit trail is a compliance requirement, the trait is the wrong mechanism — use **database triggers** or logical decoding, which no code path can bypass. If it's best-effort observability, the trait is fine as long as everyone knows the gaps.
>
> Second issue: `AuditLog::create()` inside a model event runs **inside the caller's transaction**. A rollback erases the audit record of the thing that got rolled back (sometimes desirable, sometimes exactly what you needed to keep), and a bulk import of 10,000 rows becomes 20,000 inserts on the hot path. `04-senior.md` §21 covers the outbox alternative.
>
> This is a strong thing to volunteer: "I'd use the trait for convenience auditing, but if it's for compliance I'd push it into the database, because model events don't see `->update()` or `insert()`."

### Composition with real collaborators

```php
// ❌ WRONG — and this is the single most common bug in interview whiteboard code.
// Two concurrent requests both pass the check before either decrements. Both succeed.
// Stock goes negative. This is a check-then-act (TOCTOU) race.
final class InventoryItemService
{
    public function deductStock(int $itemId, int $orgId, int $amount): InventoryItem
    {
        $item = $this->repo->findForOrganization($itemId, $orgId);

        if ($item->quantity < $amount) {                       // ← check
            throw new InsufficientStockException(/* ... */);
        }

        $this->repo->decrementStock($itemId, $orgId, $amount); // ← act; race window above

        return $this->repo->findForOrganization($itemId, $orgId);
    }
}
```

```php
// ✅ CORRECT — the guard lives inside the UPDATE, so the database enforces it atomically.
// Note that EloquentStockRepository::decrementStock() already returns the affected row
// count precisely so the caller can branch on it. Use the return value.
final class InventoryItemService
{
    public function __construct(
        private readonly StockRepository $repo,
        private readonly EventDispatcher $events,
    ) {}

    public function deductStock(int $itemId, int $orgId, int $amount): InventoryItem
    {
        // UPDATE ... SET quantity = quantity - ? WHERE id = ? AND quantity >= ?
        // Zero rows affected means either "not found" or "insufficient" — never a race.
        $affected = $this->repo->decrementStock($itemId, $orgId, $amount);

        if ($affected === 0) {
            $item = $this->repo->findForOrganization($itemId, $orgId);

            throw $item === null
                ? new ModelNotFoundException()
                : new InsufficientStockException($itemId, $amount, $item->quantity);
        }

        $item = $this->repo->findForOrganization($itemId, $orgId);

        $this->events->dispatch(new StockLevelChanged(
            itemId:         $itemId,
            organizationId: $orgId,
            oldQuantity:    $item->quantity + $amount,
            newQuantity:    $item->quantity,
        ));

        return $item;
    }
}
```

> **Why the wrong version is worth studying:** it reads as clean, well-factored OOP — injected collaborators, a guard clause, a domain exception. Good structure and correct concurrency are independent properties, and interviewers deliberately use tidy-looking code to see whether you check for the second one. Composition is still the right design here; the composed pieces just have to be individually correct.

> **Follow-up you should expect:** *what if the business rule is more complex than a single-column comparison?* Then the atomic UPDATE stops being enough and you escalate: `SELECT ... FOR UPDATE` inside a transaction (pessimistic), a `lock_version` column (optimistic), or an append-only reservation ledger. [`04-senior.md`](./04-senior.md) §4 works through all four with the trade-offs.

> **The rule of thumb:** Use traits for framework integration boilerplate (global scopes, route model binding, casts). Use composition (injected collaborators) when the behavior has dependencies, needs testing in isolation, or might change independently.

---

## 5. Dependency Injection & SOLID

### Constructor injection (the default)

```php
// The container resolves all type-hinted constructor params automatically
final class ReportGenerator
{
    public function __construct(
        private readonly Organization $org,
        private readonly ReportRepository $repo,
        private readonly PdfRenderer $pdf,
    ) {}
}

// No binding needed — concrete class with only class-typed params
$report = app(ReportGenerator::class);
```

### Interface injection (requires binding)

```php
// The container can't instantiate an interface — you must bind it
$this->app->bind(PaymentGateway::class, StripeGateway::class);

final class CheckoutService
{
    public function __construct(
        private readonly PaymentGateway $gateway,  // interface
    ) {}
}
```

### Method injection

```php
final class ImportController
{
    public function store(
        StoreImportRequest $request,      // Form Request (validated)
        ImportService $service,           // from container
        TenantContext $tenant,            // from container
    ): JsonResponse {
        $result = $service->import(
            file: $request->file('csv'),
            orgId: $tenant->organizationIdOrFail(),
        );

        return response()->json(['imported' => $result->count]);
    }
}
```

### Property promotion with readonly (PHP 8.1+)

```php
// Clean, immutable, self-documenting
final readonly class DeductStockData
{
    public function __construct(
        public int $itemId,
        public int $organizationId,
        public int $amount,
        public string $reason,
        public string $idempotencyKey,
    ) {}

    public static function fromRequest(Request $request, InventoryItem $item): self
    {
        return new self(
            itemId: $item->id,
            organizationId: $item->organization_id,
            amount: $request->integer('amount'),
            reason: $request->string('reason', 'manual'),
            idempotencyKey: $request->header('Idempotency-Key', Str::uuid()),
        );
    }
}
```

> **Follow-up:** *When NOT to use constructor injection?* When the dependency is optional or varies per call — use method injection. When you need to break circular dependencies — use setter injection or the container's `resolving` hook. When the class is a simple value object with no dependencies — just use `new`.

---

## 6. Interfaces vs Abstract Classes — Deep Comparison

### When to use each

```php
// INTERFACE — when you need polymorphism across unrelated classes
interface Renderable
{
    public function render(): string;
}

class InvoicePdf implements Renderable
{
    public function render(): string { /* PDF generation */ }
}

class InvoiceHtml implements Renderable
{
    public function render(): string { /* HTML generation */ }
}

class InvoiceEmail implements Renderable
{
    public function render(): string { /* email template */ }
}

// ABSTRACT CLASS — when you need shared implementation among close relatives
abstract class BaseRepository
{
    public function __construct(
        protected readonly DatabaseManager $db,
    ) {}

    abstract protected function table(): string;

    protected function scoped(int $orgId): Builder
    {
        return $this->db->table($this->table())->where('organization_id', $orgId);
    }

    public function findForOrganization(int $id, int $orgId): ?Model
    {
        return $this->scoped($orgId)->where('id', $id)->first();
    }

    public function existsForOrganization(int $id, int $orgId): bool
    {
        return $this->scoped($orgId)->where('id', $id)->exists();
    }
}

final class ItemRepository extends BaseRepository
{
    protected function table(): string { return 'inventory_items'; }
    
    public function lowStock(int $orgId, int $threshold = 10): Collection
    {
        return $this->scoped($orgId)
            ->whereColumn('quantity', '<=', 'reorder_point')
            ->get();
    }
}

final class SupplierRepository extends BaseRepository
{
    protected function table(): string { return 'suppliers'; }
    
    public function withItemCounts(int $orgId): Collection
    {
        return $this->scoped($orgId)
            ->withCount('items')
            ->get();
    }
}
```

### Multiple inheritance via interfaces

```php
// PHP allows implementing multiple interfaces
interface Searchable
{
    public function toSearchDocument(): array;
}

interface Exportable
{
    public function toCsvRow(): array;
    public function csvHeaders(): array;
}

interface ProvidesAuditContext
{
    public function auditContext(): array;
}

// This model implements three unrelated behaviors
class InventoryItem extends Model implements Searchable, Exportable, ProvidesAuditContext
{
    use HasOrganization;

    public function toSearchDocument(): array
    {
        return [
            'id' => $this->id,
            'sku' => $this->sku,
            'quantity' => $this->quantity,
        ];
    }

    public function toCsvRow(): array
    {
        return [$this->sku, $this->name, $this->quantity];
    }

    public function csvHeaders(): array
    {
        return ['SKU', 'Name', 'Quantity'];
    }

    public function auditContext(): array
    {
        return ['item_id' => $this->id, 'org_id' => $this->organization_id];
    }
}
```

### Interface as a security boundary

```php
// Restrict what a class can access via type-hinting
interface CanManageInventory
{
    public function viewInventory(): bool;
    public function updateStock(): bool;
    public function deleteItem(): bool;
}

final class InventoryPolicy implements CanManageInventory
{
    public function __construct(
        private readonly User $user,
        private readonly Organization $org,
    ) {}

    public function viewInventory(): bool
    {
        return $this->user->can('inventory.view');
    }

    public function updateStock(): bool
    {
        return $this->user->can('inventory.update');
    }

    public function deleteItem(): bool
    {
        return $this->user->can('inventory.delete');
    }
}

// Service depends only on the permission interface, not the full User model
final class InventoryService
{
    public function deductStock(
        int $itemId,
        int $amount,
        CanManageInventory $permissions,
    ): void {
        if (! $permissions->updateStock()) {
            throw new UnauthorizedException('Cannot update stock');
        }

        // ... business logic
    }
}
```

---

## 7. Traits — Advanced Patterns & Pitfalls

### Boot method pattern

```php
trait HasSlug
{
    public static function bootHasSlug(): void
    {
        static::creating(function (Model $model): void {
            $model->slug ??= $model->generateUniqueSlug();
        });
    }

    protected function generateUniqueSlug(): string
    {
        $base = Str::slug($this->name);

        // Scope the check the same way the unique index is scoped.
        $taken = static::query()
            ->when($this->exists, fn (Builder $q) => $q->whereKeyNot($this->getKey()))
            ->where('organization_id', $this->organization_id)
            ->where('slug', 'like', "{$base}%")
            ->pluck('slug')
            ->all();

        if (! in_array($base, $taken, true)) {
            return $base;
        }

        for ($i = 2; in_array("{$base}-{$i}", $taken, true); $i++) {
        }

        return "{$base}-{$i}";
    }

    public function scopeBySlug(Builder $query, string $slug): Builder
    {
        return $query->where('slug', $slug);
    }
}
```

> **Two bugs this version fixes, and both are interview-grade:**
>
> 1. **`where('id', '!=', $model->id)` during `creating` is dead code.** The model has no key yet, so `$model->id` is `null` and the clause compiles to `id != NULL`, which is never true in SQL — the query returns nothing and the collision branch never runs. Any `!=` against a possibly-null value has this problem; use `whereKeyNot()` guarded by `$this->exists`, or `whereNotNull()` explicitly.
> 2. **`count() + 1` produces collisions.** If `post`, `post-2`, and `post-3` exist and `post-2` is deleted, the count is 2 and you generate `post-3` — which is taken. Probe for the first free suffix instead of arithmetic on a count.

> **The bug this version does NOT fix — and you should say so:** it's still a check-then-act race. Two concurrent inserts both read the same `$taken` list and both generate `post-2`. The only real fix is a **unique index** on `(organization_id, slug)` plus a retry:
>
> ```php
> $table->unique(['organization_id', 'slug']);
> ```
>
> ```php
> retry(3, fn () => Post::create($attributes), function (Throwable $e) {
>     return $e instanceof UniqueConstraintViolationException;
> });
> ```
>
> Application-level uniqueness checks are advisory; the database is the only thing that can actually enforce it. Volunteering this unprompted is a strong senior signal — it's the same reasoning as the reservation-ledger discussion in [`04-senior.md`](./04-senior.md) §4.

### Conflict resolution

```php
trait RecordsAudit
{
    public function getContext(): array
    {
        return ['source' => 'audit', 'id' => $this->getKey()];
    }
}

trait RecordsTenancy
{
    public function getContext(): array
    {
        return ['source' => 'tenancy', 'org_id' => $this->organization_id];
    }
}

class InventoryItem extends Model
{
    use RecordsAudit, RecordsTenancy {
        RecordsAudit::getContext insteadof RecordsTenancy;   // pick a winner
        RecordsTenancy::getContext as getTenancyContext;     // alias the loser
    }
}
```

> **The precedence rules, in order** — this is a standard question and the ordering is counter-intuitive:
>
> 1. The **class's own** method wins over any trait method — silently, with no error.
> 2. A **trait** method wins over an **inherited parent** method.
> 3. Two traits with the same method is a **fatal error** unless resolved with `insteadof`.
>
> So: `class > trait > parent`. Rule 1 is the trap — if your model happens to define `getContext()`, the trait's version is silently ignored and no error tells you. That's how "why isn't my trait's boot logic running?" bugs happen.
>
> Two more details worth having ready:
>
> - `as` **aliases**, it does not remove. `RecordsTenancy::getContext as getTenancyContext` makes the method reachable under a second name; you still need `insteadof` to resolve the original conflict.
> - `as` can also change **visibility**: `RecordsAudit::getContext as protected;`
> - Traits can declare **abstract** methods to demand something from the using class, and (since 8.2) **constants**. Static and non-static properties in traits are per-using-class, not shared — a common misconception.

### When traits become a problem

```php
// BAD — trait with hidden dependencies
trait Searchable
{
    public function syncToSearch(): void
    {
        // This method silently depends on SearchClient — untestable without it
        app(SearchClient::class)->index($this->toSearchDocument());
    }
}

// GOOD — extract the dependency into a service
final class SearchSyncService
{
    public function __construct(
        private readonly SearchClient $client,
    ) {}

    public function sync(Model $model): void
    {
        $this->client->index($model->toSearchDocument());
    }
}
```

> **The decision tree:**
> 1. Does the behavior need external dependencies? → Use a service class with DI
> 2. Is the behavior a simple, self-contained concern (global scope, attribute casting)? → Trait is fine
> 3. Do you need to test the behavior in isolation? → Service class (traits are hard to mock)
> 4. Does the behavior apply to unrelated classes? → Interface + implementation, not trait

---

## 8. Immutability & Value Objects

### Immutable value objects

```php
final readonly class Money
{
    public function __construct(
        public int $amountInCents,
        public string $currency = 'USD',
    ) {
        // Note: negatives are ALLOWED. See the design note below.
        if (strlen($currency) !== 3 || $currency !== strtoupper($currency)) {
            throw new InvalidArgumentException("Invalid ISO 4217 code: {$currency}");
        }
    }

    public static function zero(string $currency = 'USD'): self
    {
        return new self(0, $currency);
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);

        return new self($this->amountInCents + $other->amountInCents, $this->currency);
    }

    public function subtract(self $other): self
    {
        $this->assertSameCurrency($other);

        return new self($this->amountInCents - $other->amountInCents, $this->currency);
    }

    public function multiply(int|float $factor): self
    {
        // Banker's rounding avoids the systematic upward bias of round-half-up
        // when you multiply millions of line items.
        return new self(
            (int) round($this->amountInCents * $factor, 0, PHP_ROUND_HALF_EVEN),
            $this->currency,
        );
    }

    public function isGreaterThan(self $other): bool
    {
        $this->assertSameCurrency($other);

        return $this->amountInCents > $other->amountInCents;
    }

    public function isNegative(): bool
    {
        return $this->amountInCents < 0;
    }

    public function equals(self $other): bool
    {
        return $this->amountInCents === $other->amountInCents
            && $this->currency === $other->currency;
    }

    public function format(): string
    {
        return number_format($this->amountInCents / 100, 2) . ' ' . $this->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($other->currency !== $this->currency) {
            throw new CurrencyMismatchException($this->currency, $other->currency);
        }
    }
}

// Usage — impossible to corrupt
$price    = new Money(1999, 'USD');   // $19.99
$discount = new Money(500, 'USD');    // $5.00
$final    = $price->subtract($discount); // $14.99 — new instance, original unchanged
$final->amountInCents = 0;            // Error: Cannot modify readonly property
```

> **Design note — why negatives are allowed now.** Rejecting negative amounts in the constructor seems safe, but it makes `subtract()` throw `InvalidArgumentException` on an overdraft: a `LogicException` ("the programmer made a mistake") for what is actually a domain condition ("insufficient funds"). It also makes refunds, credit notes, and ledger reversals unrepresentable. The right home for "this particular balance must not go below zero" is the **aggregate that owns the balance**, not the value type. Keep `Money` a faithful model of an amount; put the business rule where the business rule lives.
>
> This is a genuinely good interview answer because the naive version *looks* more defensive. Being able to explain why the stricter type is the worse design shows you reason about where invariants belong.

> **Trap — `readonly` is shallow.** A `readonly` property holding an object prevents *reassignment*, not *mutation* of the referenced object:
>
> ```php
> final readonly class Basket
> {
>     public function __construct(public ArrayObject $items) {}
> }
>
> $b = new Basket(new ArrayObject(['a']));
> $b->items = new ArrayObject();   // ❌ Error: readonly
> $b->items[] = 'b';               // ✅ allowed — the Basket is now "changed"
> ```
>
> Depth requires either genuinely immutable members all the way down, or defensive copying in the constructor. PHP 8.3 added `readonly` class *cloning* support (a `__clone` may reinitialise readonly props), which is what makes wither-methods work:
>
> ```php
> public function withCurrency(string $currency): static
> {
>     $clone = clone $this;
>     $clone->currency = $currency;   // legal inside __clone scope on PHP 8.3+
>     return $clone;
> }
> ```

> **Trap — `__clone` is shallow too.** `clone $order` copies the property table, so both objects point at the *same* `Collection` of lines. Deep copy needs an explicit `__clone`:
>
> ```php
> public function __clone(): void
> {
>     $this->lines = $this->lines->map(fn (OrderLine $l) => clone $l);
> }
> ```
>
> "What's the difference between `clone` and a deep copy, and how does PHP let you control it?" is a standard question and `__clone` is the whole answer.

> **Laravel integration — make the value object a first-class cast:**
>
> ```php
> final class MoneyCast implements CastsAttributes
> {
>     public function get(Model $model, string $key, mixed $value, array $attributes): ?Money
>     {
>         return $value === null
>             ? null
>             : new Money((int) $value, $attributes['currency'] ?? 'USD');
>     }
>
>     public function set(Model $model, string $key, mixed $value, array $attributes): array
>     {
>         if (! $value instanceof Money) {
>             throw new InvalidArgumentException('Expected Money instance');
>         }
>
>         return [$key => $value->amountInCents, 'currency' => $value->currency];
>     }
> }
> ```
>
> ```php
> protected $casts = ['total' => MoneyCast::class];
> ```
>
> Now the value object survives the round trip through the database, and no code outside the cast ever sees a bare integer. Being able to name `CastsAttributes` and show a multi-column cast is a concrete "I've actually done this" signal.

### DTOs with readonly (PHP 8.2+)

```php
final readonly class OrderData
{
    public function __construct(
        public int $organizationId,
        public int $customerId,
        public array $lineItems,
        public Money $total,
        public string $idempotencyKey,
    ) {}

    public static function fromRequest(Request $request): self
    {
        $lines = collect($request->validated('lines'))->map(fn ($line) => new OrderLineData(
            itemId: $line['item_id'],
            quantity: $line['quantity'],
            unitPrice: new Money($line['unit_price_cents']),
        ))->all();

        $total = array_reduce(
            $lines,
            fn (Money $sum, OrderLineData $line) => $sum->add($line->unitPrice->multiply($line->quantity)),
            new Money(0),
        );

        return new self(
            organizationId: $request->user()->organization_id,
            customerId: $request->integer('customer_id'),
            lineItems: $lines,
            total: $total,
            idempotencyKey: $request->header('Idempotency-Key', Str::uuid()),
        );
    }
}
```

> **The benefit:** Once created, an immutable object can be freely shared, cached, and passed between methods without defensive copying. You know it won't change under you. This eliminates an entire class of bugs.

---

## 9. Design by Contract

### Preconditions, postconditions, and invariants

```php
final class BankAccount
{
    private int $balanceCents;
    private string $status;

    public function __construct(int $initialBalanceCents)
    {
        // Precondition: initial balance must be non-negative
        if ($initialBalanceCents < 0) {
            throw new InvalidArgumentException('Initial balance cannot be negative');
        }

        $this->balanceCents = $initialBalanceCents;
        $this->status = 'active';
    }

    public function withdraw(int $amountCents): void
    {
        // Preconditions
        if ($amountCents <= 0) {
            throw new InvalidArgumentException('Withdrawal amount must be positive');
        }
        if ($this->status !== 'active') {
            throw new DomainException('Cannot withdraw from a closed account');
        }
        if ($amountCents > $this->balanceCents) {
            throw new InsufficientFundsException($amountCents, $this->balanceCents);
        }

        $this->balanceCents -= $amountCents;

        // Postcondition: balance must be non-negative
        assert($this->balanceCents >= 0, 'Balance went negative after withdrawal');

        // Invariant: account is still in a valid state
        $this->assertInvariants();
    }

    public function deposit(int $amountCents): void
    {
        if ($amountCents <= 0) {
            throw new InvalidArgumentException('Deposit amount must be positive');
        }
        if ($this->status !== 'active') {
            throw new DomainException('Cannot deposit to a closed account');
        }

        $this->balanceCents += $amountCents;
        $this->assertInvariants();
    }

    public function close(): void
    {
        if ($this->balanceCents !== 0) {
            throw new DomainException('Cannot close account with non-zero balance');
        }
        $this->status = 'closed';
    }

    // Invariant: balance is always non-negative, status is always valid
    private function assertInvariants(): void
    {
        assert($this->balanceCents >= 0, 'Balance must be non-negative');
        assert(in_array($this->status, ['active', 'closed'], true), 'Invalid account status');
    }
}
```

> **Critical caveat — `assert()` is compiled out in production.** This is the single most important thing to know about `assert()` and the reason it's a trap in a Design-by-Contract discussion.
>
> | `zend.assertions` | Behaviour | Typical environment |
> |---|---|---|
> | `1` | Compiled and executed | development |
> | `0` | Compiled but skipped at runtime | — |
> | `-1` | **Not compiled at all — zero cost, never runs** | **production** |
>
> `-1` is the recommended production setting and what most Docker/PHP images ship. So every postcondition and invariant above **silently disappears exactly where a corrupted balance would do real damage**. `zend.assertions` can only be set in `php.ini`, not at runtime, so you cannot flip it on to debug a live incident either.
>
> The practical division:
>
> - **Preconditions on public API** — validating *someone else's* input. Use real `throw`s. These must run in production; they're part of the contract.
> - **Postconditions and invariants** — checking *your own* logic. `assert()` is defensible: they should be unreachable, and paying for them on every production call is waste.
> - **Anything protecting money, stock, or tenant boundaries** — always a real `throw`, never `assert()`. If the check matters enough to write, it matters enough to run.
>
> ```php
> // Production-safe invariant on something that actually matters
> if ($this->balanceCents < 0) {
>     throw new LogicException("Invariant violated: balance {$this->balanceCents} < 0");
> }
> ```
>
> **Follow-up they'll ask:** *"So how do you get contract checking without the production cost?"* Assertions in tests (run the invariant check in a test-only observer), property-based testing to explore the state space, and database `CHECK` constraints for the invariants that can be expressed in SQL — `CHECK (quantity >= 0)` runs in production, costs almost nothing, and cannot be bypassed by any code path including raw queries or a psql session. For your inventory system that last one is the strongest control available.

### Laravel form request as a contract

```php
final class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        $orgId = $this->user()->organization_id;

        return [
            // ⚠️ A bare `exists:customers,id` is a cross-tenant leak: it confirms whether
            // an ID exists in ANY organization. Every existence rule must be tenant-scoped.
            'customer_id' => [
                'required', 'integer',
                Rule::exists('customers', 'id')->where('organization_id', $orgId),
            ],
            'lines'              => ['required', 'array', 'min:1', 'max:500'],
            'lines.*.item_id'    => [
                'required', 'integer',
                Rule::exists('inventory_items', 'id')->where('organization_id', $orgId),
            ],
            'lines.*.quantity'   => ['required', 'integer', 'min:1', 'max:1000'],
            'idempotency_key'    => ['required', 'string', 'uuid'],
        ];
    }

    // The form request IS the contract — if validation passes, the data is guaranteed valid
    public function authorize(): bool
    {
        return $this->user()->can('create', Order::class);
    }
}

// Controller trusts the contract
final class OrderController
{
    public function store(StoreOrderRequest $request, CreateOrderAction $action): JsonResponse
    {
        // By the time we reach here, we KNOW:
        // 1. The user is authenticated and authorized to create orders
        // 2. customer_id exists AND belongs to this user's organization
        // 3. Every item_id exists AND belongs to this user's organization
        // 4. Every quantity is a positive integer ≤ 1000, and there are 1–500 lines
        // 5. A well-formed idempotency_key is present

        $order = $action->execute(OrderData::fromRequest($request));

        return new JsonResponse(new OrderResource($order), 201);
    }
}
```

> **What the contract explicitly does NOT guarantee** — and saying this unprompted is the senior move. Validation is a *point-in-time* check, so it establishes nothing that another request can invalidate:
>
> - **Not that stock is available.** `exists` says the row exists, not that `quantity >= requested`. Between validation and the write, someone else can take the last unit. The stock guard must be atomic and live at the write (§4), never in `rules()`.
> - **Not that the row still exists.** It could be deleted microseconds later. Foreign keys, not validators, enforce referential integrity.
> - **Not that the request is unique.** A duplicate `idempotency_key` needs a unique index and a conflict-handling path, not a `unique:` rule — the rule is another check-then-act race.
>
> The precise framing: **Form Requests establish preconditions about the *shape and provenance* of input. They cannot establish preconditions about *contended state*.** Those belong in the database, atomically, at the moment of the write. Candidates who validate stock levels in `rules()` and think they're done are the ones this question is designed to find.

---

## 10. PHP 8.x OOP Features

### Enums with behavior (8.1+)

```php
enum OrderStatus: string
{
    case Pending   = 'pending';
    case Confirmed = 'confirmed';
    case Paid      = 'paid';
    case Shipped   = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match ($this) {
            self::Pending   => 'Awaiting confirmation',
            self::Confirmed => 'Confirmed',
            self::Paid      => 'Paid',
            self::Shipped   => 'Shipped',
            self::Delivered => 'Delivered',
            self::Cancelled => 'Cancelled',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Pending   => 'yellow',
            self::Confirmed => 'blue',
            self::Paid      => 'teal',
            self::Shipped   => 'purple',
            self::Delivered => 'green',
            self::Cancelled => 'red',
        };
    }

    /** @return list<self> */
    public function allowedTransitions(): array
    {
        return match ($this) {
            self::Pending   => [self::Confirmed, self::Cancelled],
            self::Confirmed => [self::Paid, self::Cancelled],
            self::Paid      => [self::Shipped, self::Cancelled],
            self::Shipped   => [self::Delivered],
            self::Delivered,
            self::Cancelled => [],   // terminal states
        };
    }

    public function canTransitionTo(self $next): bool
    {
        return in_array($next, $this->allowedTransitions(), strict: true);
    }

    public function isTerminal(): bool
    {
        return $this->allowedTransitions() === [];
    }

    /** @return list<self> */
    public static function active(): array
    {
        return array_values(array_filter(
            self::cases(),
            fn (self $status) => ! $status->isTerminal(),
        ));
    }
}

// Usage
$status = OrderStatus::from('pending');    // OrderStatus::Pending
$status = OrderStatus::tryFrom('nope');    // null

// State machine — but note the concurrency caveat below
$order = Order::findOrFail(1);
$newStatus = OrderStatus::Shipped;

if (! $order->status->canTransitionTo($newStatus)) {
    throw new InvalidStatusTransitionException($order->status, $newStatus);
}

$order->status = $newStatus;
$order->save();
```

> **Trap — exhaustive `match` is a feature, not an annoyance.** Because none of these `match` expressions has a `default`, adding a case to the enum makes every one of them throw `UnhandledMatchError` until you handle it. That's the point: the compiler-ish behavior forces you to visit every decision site. Writing `default => 'gray'` throws that safety away and is how a new status silently renders wrong. Say this out loud if asked "why no default?"

> **Trap — `in_array()` with enums needs `strict: true`.** Non-strict `in_array` compares loosely; with backed enums against raw strings that can produce surprising matches. Enum instances are singletons, so `===` and strict `in_array` are both correct and unambiguous.

> **Trap — this state machine is still racy.** Read-check-write on `$order->status` has the same TOCTOU hole as the stock example in §4. Two "ship" requests both read `Paid` and both pass. The atomic form puts the current state in the WHERE clause:
>
> ```php
> $affected = Order::query()
>     ->whereKey($order->id)
>     ->where('status', $order->status)          // ← guard on the state we validated
>     ->update(['status' => $newStatus]);
>
> if ($affected === 0) {
>     throw new ConcurrentStatusChangeException($order->id);
> }
> ```
>
> The enum makes the transition rules *correct*; it does nothing to make them *atomic*. Interviewers love this distinction because it separates people who know the syntax from people who have shipped state machines.

### Fibers (8.1+) — cooperative multitasking

```php
// Fibers are the primitive behind async runtimes (Amp v3, ReactPHP, Octane's Swoole bridge).
// The key mechanic: Fiber::suspend() RETURNS the value later passed to resume().
$fiber = new Fiber(function (): string {
    echo "Start\n";

    $received = Fiber::suspend('first');    // hands 'first' to start(); returns resume()'s arg
    echo "Resumed with: {$received}\n";     // "Resumed with: hello"

    Fiber::suspend('second');               // hands 'second' to the first resume()
    echo "Done\n";

    return 'return value';                  // available via getReturn()
});

$label = $fiber->start();          // 'first'   — runs until the first suspend
echo "Suspended at: {$label}\n";

$label = $fiber->resume('hello');  // 'second'  — 'hello' becomes $received inside
$fiber->resume();                  // runs to completion

echo $fiber->getReturn();          // 'return value' — only valid once terminated
```

The complete API is small: instance methods `start`, `resume`, `throw`, `getReturn`, `isStarted`, `isSuspended`, `isRunning`, `isTerminated`, plus statics `Fiber::suspend()` and `Fiber::getCurrent()`.

> **Trap:** there is no method for reading "the value passed to `resume()`" from outside the suspension point — the value simply *is* the return value of `Fiber::suspend()`. Candidates often invent an accessor here. Data flows both directions through those two return values and nowhere else.

> **Follow-up:** *How do Fibers differ from Generators?* A generator can only yield from its own function body — the whole call stack between you and the `yield` has to be generators too ("what colour is your function"). `Fiber::suspend()` suspends the *entire* call stack from anywhere in it, so ordinary functions deep inside can suspend without every caller knowing. That's precisely what makes transparent async I/O possible in PHP.

### Property hooks (8.4)

```php
class InventoryItem
{
    public int $quantity = 0;
    public int $reorderPoint = 10;

    // VIRTUAL property — neither hook touches $this->isLowStock, so no storage is
    // allocated. Read-only by construction: there is no set hook, so writing errors.
    public bool $isLowStock {
        get => $this->quantity < $this->reorderPoint;
    }

    // BACKED property with a normalising set hook.
    // The arrow expression's VALUE is written to the backing store — you must NOT
    // assign to $this->sku yourself or you re-enter the hook and recurse infinitely.
    public string $sku {
        set => strtoupper(trim($value));
    }

    // Same thing with the parameter spelled out. Omitting it defaults to $value.
    public string $name {
        set (string $value) => ucfirst(trim($value));
    }

    // Multi-statement hooks use a block body, and there you DO assign explicitly.
    public string $barcode {
        get => $this->barcode;
        set (string $value) {
            $normalised = preg_replace('/\D/', '', $value);

            if (strlen($normalised) !== 13) {
                throw new InvalidArgumentException('EAN-13 required');
            }

            $this->barcode = $normalised;   // legal: we are in a block body
        }
    }
}
```

> **The recursion trap, precisely.** The manual's rule is: *"If the `set` hook is only setting a modified version of the passed-in value, it may be simplified to an arrow expression. The value the expression evaluates to will be set on the backing value."* So `set => strtoupper($value)` is correct, and `set => $this->sku = strtoupper($value)` is an infinite loop — the assignment inside re-triggers the hook. In a **block** body the assignment is required and correct. Getting this backwards is the most common property-hook mistake.

> **Two more facts interviewers like:**
> - Hooks are **incompatible with `readonly`**. If you need "public read, restricted write," that's asymmetric visibility (below), not a hook.
> - Hooks change serialization behavior asymmetrically: `var_dump()`, `serialize()`, and array casts use the **raw backing value**, while `json_encode()`, `var_export()`, and `get_object_vars()` go **through the get hook**. That inconsistency bites when a model with hooks is queued as a job payload.

### Asymmetric visibility (8.4)

```php
class Order
{
    // Public read, private write — enforced by the language
    public private(set) OrderStatus $status = OrderStatus::Pending;

    public function markPaid(): void
    {
        $this->status = OrderStatus::Paid;   // allowed: we're inside the class
    }
}

$order = new Order();
echo $order->status;               // allowed: public read
$order->status = OrderStatus::Paid; // Error: private set
```

---

## 11. Anti-Patterns & Code Smells

### God class

```php
// BAD — 800-line model with everything
class InventoryItem extends Model
{
    // 20 attributes, 15 relationships, 10 scopes, 8 accessors, 5 mutators,
    // 3 observers, search sync, email notifications, report generation,
    // import/export logic, pricing calculations, audit logging...
}

// GOOD — focused model + extracted services
class InventoryItem extends Model
{
    use HasOrganization, HasUuid;

    protected $fillable = ['sku', 'name', 'quantity', 'reorder_point'];
    
    public function organization(): BelongsTo { ... }
    public function movements(): HasMany { ... }
    public function supplier(): BelongsTo { ... }

    public function scopeLowStock(Builder $query): void { ... }
}

// Extracted into focused services
final class InventorySearchService { ... }
final class InventoryReportingService { ... }
final class InventoryImportService { ... }
final class InventoryPricingService { ... }
```

> **The framing that works in an interview:** don't measure god classes by line count — count **reasons to change**. If three unrelated stakeholders could each force an edit to the same class (a product manager changing pricing rules, an SRE swapping the search backend, a compliance officer adding audit fields), it has three responsibilities whatever its size. Line count is the symptom; divergent change is the disease.

---

### Anemic domain model

```php
// BAD — all logic in service classes, model is just data
class Order extends Model
{
    protected $fillable = ['total', 'status', 'customer_id'];
    
    // No behavior — just getters and setters
}

class OrderService
{
    public function calculateTotal(Order $order): void
    {
        $order->total = $order->lines->sum(fn ($line) => $line->price * $line->quantity);
    }

    public function markAsPaid(Order $order): void
    {
        if ($order->status !== 'pending') {
            throw new InvalidStatusTransitionException();
        }
        $order->status = 'paid';
        $order->paid_at = now();
        $order->save();
    }

    public function cancel(Order $order): void
    {
        if ($order->status === 'shipped') {
            throw new CannotCancelShippedOrderException();
        }
        $order->status = 'cancelled';
        $order->cancelled_at = now();
        $order->save();
    }
}

// GOOD — domain logic lives in the model
class Order extends Model
{
    protected $casts = ['status' => OrderStatus::class];

    // Laravel dispatches these automatically on the matching lifecycle event.
    protected $dispatchesEvents = [
        'saved' => OrderSaved::class,
    ];

    public function calculateTotal(): Money
    {
        return $this->lines->reduce(
            fn (Money $sum, OrderLine $line) => $sum->add($line->lineTotal()),
            new Money(0),
        );
    }

    public function markAsPaid(): void
    {
        $this->transitionTo(OrderStatus::Paid, ['paid_at' => now()]);

        OrderPaid::dispatch($this);
    }

    public function cancel(): void
    {
        $this->transitionTo(OrderStatus::Cancelled, ['cancelled_at' => now()]);

        OrderCancelled::dispatch($this);
    }

    private function transitionTo(OrderStatus $next, array $attributes = []): void
    {
        if (! $this->status->canTransitionTo($next)) {
            throw new InvalidStatusTransitionException($this->status, $next);
        }

        $affected = static::query()
            ->whereKey($this->getKey())
            ->where('status', $this->status)     // atomic guard, see §10
            ->update([...$attributes, 'status' => $next]);

        if ($affected === 0) {
            throw new ConcurrentStatusChangeException($this->getKey());
        }

        $this->refresh();
    }
}
```

> **Note the fix:** `$this->events()->dispatch(...)` is not an Eloquent method — models have no public event-dispatcher accessor. Your options are the `Dispatchable` trait on the event (`OrderPaid::dispatch($this)`), the global `event()` helper, or `$dispatchesEvents` for lifecycle-driven events. This is a small thing but it's exactly the kind of detail a Laravel-fluent interviewer notices.

> **The test argument:** with the anemic model, testing `OrderService` requires mocking the Order model. With the rich model, you test `Order::markAsPaid()` directly — simpler, faster, more expressive.

> **The honest counter-argument, which you should raise yourself:** "anemic domain model" is a DDD critique, and Eloquent is Active Record, which is *already* a rejection of the DDD separation between entity and persistence. A rich Eloquent model conflates domain behaviour with database access, so `markAsPaid()` can't be unit-tested without a database. Teams that want genuinely pure domain objects put behaviour in plain PHP entities and use Eloquent purely as a data mapper at the boundary — which is a real cost in Laravel and usually only worth it in a complex core domain. The senior answer is: put behaviour on the model by default, and only extract to pure entities when the domain logic is complex enough that database coupling is actually slowing you down.

---

### Primitive obsession

```php
interface RefundService
{
    // BAD — what unit is $amount? what currency? is $email validated? can $orderId be 0?
    // Nothing stops refund($amount, $orderId, ...) — both are int, so it compiles.
    public function refundBad(int $orderId, int $amount, string $currency, string $email): void;

    // GOOD — the types carry the rules, and validation happens once at construction
    public function refund(OrderId $orderId, Money $amount, EmailAddress $email): void;
}
```

Every `string $email` in a signature is a place where an unvalidated email can enter. Every `int $amount` next to a `string $currency` is a place where someone can pass cents as dollars, or mismatch the pair. Value objects (§8) make the illegal states unrepresentable rather than merely unlikely.

> **Where it bites in your multi-tenant SaaS:** `int $organizationId` and `int $userId` are the same type, so nothing stops you transposing them at a call site — and the resulting query returns another tenant's data instead of erroring. A one-line `final readonly class OrganizationId` turns that into a `TypeError` at the boundary. This is a genuinely good answer to "how do you prevent tenant leaks?" because it's a *compile-time* control rather than another runtime check.

---

### Law of Demeter violations (train wrecks)

```php
// BAD — this line knows about four classes and breaks if any relationship changes.
// It also silently triggers up to three lazy-loaded queries.
$city = $order->customer->address->city->name;

// BAD — the classic Laravel version. Four hops, four possible lazy loads,
// and a null anywhere in the chain is a fatal.
if ($user->organization->subscription->plan->features->contains('advanced_reporting')) {
    $this->renderAdvancedReports();
}

// GOOD — ask the object you have, not the object it has
if ($user->canUseAdvancedReporting()) {
    $this->renderAdvancedReports();
}
```

```php
class User extends Model
{
    public function canUseAdvancedReporting(): bool
    {
        return $this->organization->hasFeature('advanced_reporting');
    }
}

class Organization extends Model
{
    public function hasFeature(string $feature): bool
    {
        return $this->subscription?->plan?->hasFeature($feature) ?? false;
    }
}
```

> **Why this one is worth raising in a Laravel interview specifically:** train wrecks aren't just a coupling smell here, they're an **N+1 generator**. Every `->` across a relationship boundary is a potential query, and the chain is invisible to `with()` unless you know to eager-load the whole path (`with('customer.address.city')`). Tying Law of Demeter back to your 88% query-reduction story is a strong move — the coupling problem and the performance problem have the same root cause and the same fix.

> **The nuance:** fluent interfaces (`$query->where()->orderBy()->limit()`) look like train wrecks but aren't — each call returns the *same* object, so you're not reaching through a graph of collaborators. Demeter is about traversing *other people's* structure. Knowing the difference is the follow-up.

---

### Service locator disguised as dependency injection

```php
// BAD — the constructor lies. This class actually needs four things, not one.
final class ReportGenerator
{
    public function __construct(private readonly Container $container) {}

    public function generate(int $orgId): Report
    {
        $repo  = $this->container->make(ReportRepository::class);
        $pdf   = $this->container->make(PdfRenderer::class);
        $cache = $this->container->make(Repository::class);
        // ...
    }
}

// Same problem, more idiomatic disguise — app() and facades inside domain code
final class ReportGenerator
{
    public function generate(int $orgId): Report
    {
        $repo = app(ReportRepository::class);          // hidden dependency
        Cache::put("report:{$orgId}", $report);        // hidden dependency
    }
}

// GOOD — the constructor is an honest declaration of what this class needs
final class ReportGenerator
{
    public function __construct(
        private readonly ReportRepository $repo,
        private readonly PdfRenderer $pdf,
        private readonly CacheRepository $cache,
    ) {}
}
```

> **The argument:** a constructor should be a complete, checkable list of a class's collaborators. Injecting the container replaces that list with "anything, at any time," so you can only discover the real dependencies by reading every method — and you can't construct the object in a test without a booted container. Facades inside domain classes are the same problem wearing Laravel clothing; they're fine in controllers and framework glue, and a liability in the domain layer.

> **Follow-up:** *So are facades bad?* No — they're excellent ergonomics at the framework boundary and they're swappable in tests (`Cache::fake()`), which is more than most static calls offer. The rule is about *layer*: facades in controllers, commands, and service providers are idiomatic; facades in a class you want to unit-test without Laravel are a hidden dependency you'll regret.

---

## 12. Tier 2 Q&A Drill

### Quick Answers

**1. Name the five SOLID principles and give a one-sentence summary of each.**  
SRP: one reason to change. OCP: open for extension, closed for modification. LSP: subtypes substitutable for base types. ISP: small, focused interfaces. DIP: depend on abstractions.

**2. When would you use an interface over an abstract class?**  
When you need multiple inheritance, when implementations are unrelated, or when you're defining a contract without shared implementation.

**3. What is the Liskov Substitution Principle and why does it matter?**  
Subclasses must honor the base class's contract. If a `Square` extends `Rectangle` and changes width behavior to also affect height, code expecting a `Rectangle` breaks.

**4. What's the difference between Strategy and State patterns?**  
Strategy: swap algorithms externally (pricing model). State: transition between states internally (order status machine). Strategy is chosen by the client; State transitions are triggered by the object itself.

**5. Why is the Decorator pattern useful in Laravel?**  
It lets you wrap existing functionality (add caching, metrics, logging) without modifying the original class. `$app->extend()` is Laravel's built-in Decorator mechanism.

**6. When is the Repository pattern justified in Laravel?**  
When the domain must not depend on Eloquent, when you need to swap persistence layers, or when query logic is complex and reusable. NOT justified when the interface just proxies Eloquent methods.

**7. What is anemic domain model and why is it considered an anti-pattern?**  
Models with only data and no behavior, with all logic in service classes. It defeats OOP's purpose of encapsulating behavior with data, making testing harder and code harder to navigate.

**8. Why prefer composition over inheritance?**  
Inheritance creates tight coupling, the fragile base class problem, and makes it hard to change behavior at runtime. Composition lets you swap collaborators, test in isolation, and mix behavior selectively.

**9. What is a value object? Give an example.**  
An immutable object defined by its attributes, not identity. `Money`, `EmailAddress`, `Address`, `DateRange`. They encapsulate validation and behavior (e.g., `Money::add()`).

**10. When would you use the Command pattern in Laravel?**  
When you need to encapsulate a request as an object (for queuing, undo/redo, logging, or audit trails). Laravel's job system is essentially the Command pattern.

**11. What is the difference between an interface and an abstract class in PHP?**  
Interface: no implementation, multiple inheritance allowed, no properties (until 8.4). Abstract class: shared implementation, single inheritance, can have properties and constructors.

**12. How do traits differ from interfaces?**  
Traits provide implementation; interfaces define contracts. Traits can have state (properties); interfaces cannot (until 8.4). Traits are not type-hinted; interfaces are. Traits create hidden coupling; interfaces enable polymorphism.

**13. What is the SOLID violation when a model class has 15+ methods that each do different things?**  
SRP violation. The model has multiple reasons to change (attributes, relationships, business logic, infrastructure, notifications). Extract into focused services.

**14. Why is `readonly` important for value objects?**  
It enforces immutability at the language level. Once created the object cannot be reassigned, eliminating mutation bugs and enabling safe sharing. Caveat: it's **shallow** — a readonly property holding a mutable object still allows that object to be mutated.

**15. What is design by contract?**  
Preconditions (what must be true before), postconditions (what will be true after), and invariants (always true). Laravel Form Requests are contracts — but only for the *shape* of input, not for contended state.

---

### Variance & the type system

**16. What is a covariant return type? Give a PHP example.**  
A child may return a *more specific* type than the parent declares. `RepositoryFactory::make(): Repository` overridden as `make(): StockRepository`. Safe because every caller expected at least a `Repository`.

**17. What is a contravariant parameter type?**  
A child may accept a *wider* type than the parent declares. Parent takes `RuntimeException`, child takes `Throwable`. Safe because every caller passes at most a `RuntimeException`.

**18. Can you narrow a parameter type in an override? Why not?**  
No — fatal error. A caller holding the parent type may pass something the narrowed child rejects, so substitutability breaks. Parameters widen, returns narrow.

**19. Why must property types be invariant?**  
A property is both read and written. Narrowing breaks writers, widening breaks readers. Since it does both, only an exact match is safe.

**20. What does `#[Override]` do and why care?**  
PHP 8.3 attribute asserting at compile time that the method actually overrides a parent's. Without it, a typo in the method name means your override silently never runs and the parent's version executes — an invisible bug.

**21. How does `static` differ from `self` as a return type?**  
`self` is the defining class; `static` is the called class (late static binding). `static` is covariant by definition, which is why fluent builders work correctly in subclasses.

---

### Patterns

**22. Decorator vs Adapter — what's the difference?**  
Decorator implements the *same* interface it wraps and adds behaviour. Adapter *converts* one interface to another. Both wrap; only Decorator is transparent to the caller.

**23. Strategy vs Template Method?**  
Same goal, opposite mechanism. Strategy uses composition and binds at runtime (swap per tenant/request). Template Method uses inheritance and binds at compile time (one subclass per variant). Prefer Strategy unless the shared *sequence* is the valuable part.

**24. Why mark a template method `final`?**  
So a subclass can't override the sequence and skip the transaction, validation, or error handling the base class guarantees. The steps are `abstract protected`; the orchestration is `final public`.

**25. What is the Null Object pattern and where does Laravel use it?**  
An implementation that satisfies the interface and does nothing harmless, removing null checks from call sites. Laravel's `null` cache driver, `array`/`log` mail drivers, and `sync` queue driver are all examples.

**26. When is Null Object the wrong choice?**  
When the absence is meaningful and the caller must react differently. Fine for optional collaborators (a logger); dangerous for missing domain entities, where it hides real bugs.

**27. What problem does the Specification pattern solve?**  
A business rule written twice — once as a query, once as an in-memory `if` — inevitably drifts. Specification forces both readings through one object, so the dashboard count and the job's behaviour can't disagree.

**28. Why does Laravel's `Manager` class matter?**  
It's the framework's own Factory implementation — the base for `Cache`, `Queue`, `Mail`, `Session`, and `Auth` driver resolution, with `extend()` as the registration hook for custom drivers.

**29. Is `Model::factory()` the Factory pattern?**  
No. That's a test data builder (fixture replacement). The GoF Factory pattern is about deferring which concrete class gets instantiated.

**30. What does Abstract Factory give you that separate factories don't?**  
A guarantee that the produced family is *consistent* — you can't pair a domestic packing slip with an international shipping label. If products don't need to match, you don't need an abstract factory.

**31. Why is a mutable builder that returns `$this` sometimes a problem?**  
A shared "base" builder gets mutated by whoever touches it next. Returning a clone makes it safe to store and reuse. Laravel's query builder is mutable, which is why two chains off one `$base` interfere.

---

### Laravel-specific OOP

**32. Eloquent observers vs domain events — when do you use which?**  
Observers fire on *persistence* events and can't distinguish intent (a stock deduction from a name typo). Domain events are dispatched deliberately and carry meaning. Observers for mechanical concerns (slugs, UUIDs, cache busting); domain events for anything a business person would recognise.

**33. Why does a queued listener need `afterCommit`?**  
Dispatched before commit, a worker can pick the job up and read a row the transaction hasn't committed yet, so it fails on a record that "doesn't exist." `$afterCommit = true` defers dispatch until the transaction commits.

**34. Why is a slow synchronous listener dangerous?**  
It runs inside the caller's transaction, holding row locks for its entire duration. An HTTP call in a listener is a classic source of lock contention under load.

**35. Do Eloquent model events fire on `Model::where(...)->update()`?**  
No. Bulk operations (`update`, `insert`, `delete` on a query, and anything via `DB::table()`) bypass the model lifecycle entirely. An audit trait built on model events therefore has silent holes — use database triggers if the audit is a compliance requirement.

**36. Are facades an anti-pattern?**  
Not inherently — they're good ergonomics at the framework boundary and are fakeable in tests. The rule is about *layer*: facades in controllers, commands, and providers are idiomatic; facades inside a domain class you want to unit-test without Laravel are a hidden dependency.

**37. What's wrong with injecting the container into a class?**  
It's service location, not injection. The constructor stops being an honest list of collaborators, you can only discover real dependencies by reading every method, and you can't construct the object in a test without a booted container.

**38. What are the trait method precedence rules?**  
Class method > trait method > inherited parent method. Two traits with the same method is a fatal error unless resolved with `insteadof`. The trap is rule 1: a class method silently shadows the trait's with no error.

**39. Does `as` remove a conflicting trait method?**  
No — it only adds an alias. You still need `insteadof` to resolve the conflict. `as` can also change visibility (`as protected`).

**40. How do you cast a value object onto an Eloquent attribute?**  
A custom cast implementing `CastsAttributes`. `set()` can return multiple columns, which is how `Money` persists as an amount plus a currency.

---

### Correctness & concurrency in OOP code

**41. What's wrong with `if ($item->quantity < $n) throw; $item->decrement($n);`?**  
Check-then-act (TOCTOU) race. Two concurrent requests both pass the check before either decrements. Put the guard in the UPDATE: `where('quantity', '>=', $n)->decrement(...)` and branch on the affected row count.

**42. Why is a `match`-based enum state machine still racy?**  
The enum makes the transition rules *correct*, not *atomic*. Read-check-write on the status column has the same TOCTOU hole. Guard on the current state in the WHERE clause and check the affected count.

**43. Why is `where('id', '!=', $model->id)` dangerous during a `creating` event?**  
The model has no key yet, so it compiles to `id != NULL`, which is never true in SQL. The clause silently matches nothing. Use `whereKeyNot()` guarded by `$this->exists`.

**44. Why can't application-level uniqueness checks work?**  
They're check-then-act. Two concurrent inserts both see the name as free. Only a unique index enforces it; the application's job is to catch the violation and retry.

**45. Where should a "balance cannot go negative" rule live — in the `Money` value object or elsewhere?**  
Elsewhere. `Money` should faithfully model an amount, including negatives (refunds, credits, reversals). "This particular balance must not go below zero" is an aggregate invariant, and ideally also a database `CHECK` constraint.

**46. What's the catch with `assert()` for postconditions?**  
`zend.assertions=-1` is the production default, and at that setting assertions aren't compiled at all — they never run where it matters, and you can't enable them at runtime. Use real `throw`s for anything protecting money, stock, or tenant boundaries.

**47. Is `readonly` deep?**  
No. It prevents reassignment of the property, not mutation of the object it references. `$obj->items[] = 'x'` still works on a readonly `ArrayObject` property.

**48. Is `clone` deep?**  
No. It copies the property table, so both objects share the same referenced objects. Implement `__clone()` to copy them explicitly.

**49. Why is `$order->customer->address->city->name` a problem in Laravel specifically?**  
Beyond the coupling, every `->` across a relationship is a potential lazy-load query — train wrecks are N+1 generators, and they're invisible to `with()` unless you eager-load the whole path.

**50. Why omit `default` from a `match` over an enum?**  
So adding a case makes every unhandled site throw `UnhandledMatchError`, forcing you to visit each decision point. A `default` arm converts that safety into a silent wrong answer.

---

### Scenario Questions

**51. You need to add PDF, HTML, and CSV export for orders. How do you design this?**  
Interface `Exportable` with `render(): string`, three implementations, and a factory or strategy to select based on user preference. Each implementation is independently testable. Follow-up to expect: *what if PDF generation takes 30 seconds?* — then it's a queued job returning a signed download URL, and the interface returns a `PendingExport`, not a string.

**52. You need different notification channels (email, SMS, Slack) per organization. How?**  
Tagged implementations, each answering `supports(Organization $org)`, injected as an iterable into a dispatcher. Adding Teams is one new class plus a tag entry — the dispatcher never changes. A `match` over config in the binding is *not* OCP; it just relocates the modification.

**53. How do you test a class that depends on an external API?**  
Define the interface in *your* domain's language, write a real adapter and a fake, inject the interface. Laravel's `Http::fake()` works for simple cases but tests the vendor's wire format rather than your contract, so it's the weaker option for anything you'd want to swap.

**54. You have a trait that adds search sync to models, but it depends on `SearchClient`. How do you test it?**  
Move the behaviour into a service with constructor injection. A trait reaching for `app(SearchClient::class)` has a hidden dependency and can't be tested without a container. Traits are for self-contained framework glue; anything with collaborators belongs in a class.

**55. How do you prevent a parent class change from breaking child classes?**  
Write the test suite against the *contract* and run it against every implementation (Pest datasets make this one test). Add `#[Override]` so renamed parent methods fail loudly. Favour composition so there are fewer inheritance relationships to break.

**56. A junior submits a PR with a 900-line `OrderService`. What's your review?**  
Count reasons to change, not lines. If pricing, fulfilment, and notification logic all live there, three different stakeholders can force edits to the same file — that's the argument, and it's about merge conflicts and blast radius rather than aesthetics. Suggest extracting along those seams, one at a time, with tests as the safety net.

**57. Your team wants to introduce the Repository pattern across the whole codebase. Do you agree?**  
Not blanket. Ask what it buys: if the interfaces will proxy Eloquent one-to-one, it's indirection with no payoff, and you also lose Eloquent's genuinely useful surface (relations, scopes, eager loading) or leak it through the interface anyway. Justify per-context: worth it where domain logic is complex and must be testable without a database, or where a real second implementation exists.

**58. How would you model an order state machine that can't be corrupted by concurrent requests?**  
A backed enum owning `allowedTransitions()` for the rules, exhaustive `match` with no `default` so new states are caught at every site, and an atomic `UPDATE ... WHERE status = :expected` for the write, throwing on zero affected rows. The enum gives correctness; the WHERE clause gives atomicity. You need both.

**59. You need to add caching to a service used in 40 places, without touching those call sites. How?**  
Decorator plus `$app->extend()` — wrap the existing binding, and nothing that resolves the interface changes. Then say the hard part out loud: invalidation, tenant-scoped keys, and the fact that caching a hydrated Eloquent model is a trap because it serializes relations and its `$exists` flag.

**60. When would you deliberately choose inheritance over composition?**  
When the *sequence* is the shared asset and the variants are a closed, known set — Template Method with a `final` orchestrator. Also when the framework requires it (`extends Model`, `extends Command`). The rule isn't "never inherit," it's "don't inherit to reuse code" — inherit to be substitutable.

---

> **The senior signal is not memorising patterns.** It's knowing when NOT to use them, and noticing when tidy-looking code is still wrong. Half the "good" examples in a typical pattern tutorial contain a race condition, an N+1, or a hidden dependency — structure and correctness are independent properties, and interviewers check both. Use a closure for a one-off callback, a service class for cross-cutting concerns, a trait only for self-contained framework glue. The best OOP code is the code you didn't write.

---

**Next:** [`03-intermediate.md`](./03-intermediate.md) — service container internals, the full relationship catalogue, queues and batching, caching and locks, and a complete testing strategy.

**Back to:** [`README.md`](./README.md) · [`01-basic.md`](./01-basic.md) · [`04-senior.md`](./04-senior.md) · [`05-question-bank.md`](./05-question-bank.md)
