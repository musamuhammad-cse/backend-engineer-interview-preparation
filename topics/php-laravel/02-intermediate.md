# PHP / Laravel — Tier 2: Intermediate (Framework Internals & Applied Patterns)

> This tier is where most Laravel developers plateau. The differentiator is knowing *how* the framework does what it does, and the failure modes of each feature under real load.

---

## Table of Contents

1. [Service Container Internals](#1-service-container-internals)
2. [Middleware: Beyond the Basics](#2-middleware-beyond-the-basics)
3. [Eloquent Relationships — Complete Catalogue](#3-eloquent-relationships--complete-catalogue)
4. [Advanced Querying](#4-advanced-querying)
5. [The N+1 Problem, Exhaustively](#5-the-n1-problem-exhaustively)
6. [Scopes: Local, Global & Removal](#6-scopes-local-global--removal)
7. [Model Events & Observers](#7-model-events--observers)
8. [Serialization & API Resources](#8-serialization--api-resources)
9. [Collections & Lazy Collections](#9-collections--lazy-collections)
10. [Database Transactions](#10-database-transactions)
11. [Queues & Jobs](#11-queues--jobs)
12. [Job Batching & Chaining](#12-job-batching--chaining)
13. [Events, Listeners & Subscribers](#13-events-listeners--subscribers)
14. [Caching & Atomic Locks](#14-caching--atomic-locks)
15. [Redis Beyond Caching](#15-redis-beyond-caching)
16. [Authentication: Guards, Providers, Sanctum vs Passport](#16-authentication-guards-providers-sanctum-vs-passport)
17. [Authorization: Gates, Policies & spatie/permission](#17-authorization-gates-policies--spatiepermission)
18. [API Design in Laravel](#18-api-design-in-laravel)
19. [Testing Strategy with Pest](#19-testing-strategy-with-pest)
20. [Tier 2 Q&A Drill](#20-tier-2-qa-drill)

---

## 1. Service Container Internals

### How autowiring actually works

When you ask for a class the container has never seen, it uses **reflection**:

```php
// Simplified Illuminate\Container\Container::build()
protected function build($concrete)
{
    $reflector = new ReflectionClass($concrete);

    if (! $reflector->isInstantiable()) {
        throw new BindingResolutionException("Target [$concrete] is not instantiable.");
    }

    $constructor = $reflector->getConstructor();

    if (is_null($constructor)) {
        return new $concrete;                       // no deps — just instantiate
    }

    $dependencies = $constructor->getParameters();

    // For each param: if it has a class type-hint, recursively resolve it.
    // If it's a primitive with a default, use the default.
    // Otherwise throw.
    $instances = $this->resolveDependencies($dependencies);

    return $reflector->newInstanceArgs($instances);
}
```

**Consequences:**
- Concrete classes with only class-typed constructor params need **no binding at all**.
- Interfaces **must** be bound — reflection can't instantiate an interface.
- Primitive params (`int $retries`) without defaults **must** be bound contextually or passed via `makeWith`.

### Binding types

```php
// Transient — new instance every resolution
$this->app->bind(StockRepository::class, EloquentStockRepository::class);

// Singleton — one instance for the container's lifetime
$this->app->singleton(TenantContext::class);

// Scoped — one instance per request / per queued job; RESET by Octane between requests
$this->app->scoped(RequestCorrelationId::class);

// Existing instance
$this->app->instance('config.snapshot', $configArray);

// Closure binding — full control
$this->app->singleton(SearchClient::class, function (Application $app) {
    return ClientBuilder::create()
        ->setHosts($app['config']['services.elasticsearch.hosts'])
        ->setRetries(2)
        ->build();
});

// Bind only if not already bound (package-friendly)
$this->app->bindIf(Logger::class, NullLogger::class);

// Interface → closure returning different impls
$this->app->bind(PaymentGateway::class, fn ($app) => match (config('payments.driver')) {
    'stripe' => $app->make(StripeGateway::class),
    'adyen'  => $app->make(AdyenGateway::class),
});
```

| Method | Lifetime | Octane behavior | Use for |
|--------|----------|-----------------|---------|
| `bind` | New each time | New each time | Stateless services |
| `singleton` | App lifetime | **Persists across requests** ⚠️ | Connection pools, config, truly stateless clients |
| `scoped` | Per request/job | Flushed each request ✅ | Request-scoped state (tenant context, correlation ID) |
| `instance` | App lifetime | Persists ⚠️ | Pre-built objects |

> **Trap — the Octane singleton leak:** If you register `TenantContext` as a `singleton` and run under Octane, tenant A's context survives into tenant B's request. **In a multi-tenant app, request-scoped state must be `scoped`, never `singleton`.** This is one of the highest-value things you can say in an interview about your SaaS.

### Contextual binding

```php
// Different implementations of the same interface for different consumers
$this->app->when(ProductImportController::class)
    ->needs(Filesystem::class)
    ->give(fn () => Storage::disk('s3'));

$this->app->when(LocalBackupCommand::class)
    ->needs(Filesystem::class)
    ->give(fn () => Storage::disk('local'));

// Bind a primitive
$this->app->when(SyncSupplierCatalog::class)
    ->needs('$maxRetries')
    ->give(5);

// Bind by tag
$this->app->when(ReportGenerator::class)
    ->needs(ReportSection::class)
    ->giveTagged('report.sections');

// Bind based on config
$this->app->when(StockCalculator::class)
    ->needs('$threshold')
    ->giveConfig('inventory.low_stock_threshold');
```

### Tagging — resolving a group

```php
// Register
$this->app->bind(StockSection::class);
$this->app->bind(MovementSection::class);
$this->app->bind(SupplierSection::class);
$this->app->tag([StockSection::class, MovementSection::class, SupplierSection::class], 'report.sections');

// Resolve all
$sections = $this->app->tagged('report.sections');   // iterable

foreach ($sections as $section) {
    $output[] = $section->render($organizationId);
}
```

This is how you build extensible pipelines (report sections, validation stages, import handlers) without a giant `match`.

### Extending a resolved binding (decorator pattern)

```php
$this->app->extend(StockRepository::class, function (StockRepository $inner, Application $app) {
    return new CachingStockRepository(
        inner: $inner,
        cache: $app->make(Repository::class),
        ttl: 300,
    );
});
```

```php
final class CachingStockRepository implements StockRepository
{
    public function __construct(
        private readonly StockRepository $inner,
        private readonly Repository $cache,
        private readonly int $ttl,
    ) {}

    public function findForOrganization(int $id, int $orgId): ?InventoryItem
    {
        return $this->cache->remember(
            "org:{$orgId}:item:{$id}",
            $this->ttl,
            fn () => $this->inner->findForOrganization($id, $orgId),
        );
    }
}
```

> **Follow-up:** *How would you add caching to a repository without touching it?* Decorate via `$app->extend()`. Zero changes to the original class, satisfies Open/Closed, and the decorator is independently testable. This is a strong SOLID answer with real code behind it.

### Resolution hooks & method injection

```php
// Fire after ANY object is resolved
$this->app->resolving(function (object $object, Application $app) {
    if ($object instanceof NeedsTenant) {
        $object->setTenant($app->make(TenantContext::class));
    }
});

// Fire after a specific class resolves
$this->app->afterResolving(SearchClient::class, fn ($client) => $client->ping());
```

```php
// Method injection — container resolves params of an arbitrary method call
app()->call([$service, 'process'], ['organizationId' => 42]);

// Controller method injection (the everyday case)
public function store(StoreItemRequest $request, InventoryService $service, InventoryItem $item) {}
//                    ^ validated          ^ from container      ^ route model binding
```

### `make` vs `makeWith` vs `resolve` vs `app()`

```php
app(StockRepository::class);                   // helper
resolve(StockRepository::class);               // helper alias
$this->app->make(StockRepository::class);
$this->app->makeWith(Importer::class, ['batchSize' => 500]);   // pass primitives
```

> **Trap:** Calling `app()` deep inside domain classes is the **Service Locator anti-pattern** — it hides dependencies, makes the class untestable without a container, and defeats static analysis. Constructor injection everywhere except the framework boundary (controllers, commands, providers, middleware).

---

## 2. Middleware: Beyond the Basics

### Registering middleware (Laravel 11 style)

```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__ . '/../routes/web.php',
        api: __DIR__ . '/../routes/api.php',
        commands: __DIR__ . '/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Global (every request)
        $middleware->append(AssignRequestId::class);

        // Add to a group
        $middleware->api(prepend: [EnsureFrontendRequestsAreStateful::class]);
        $middleware->api(append: [SetTenantContext::class]);

        // Named aliases
        $middleware->alias([
            'tenant'      => SetTenantContext::class,
            'permission'  => PermissionMiddleware::class,
            'idempotent'  => EnsureIdempotency::class,
        ]);

        // Control execution order explicitly
        $middleware->priority([
            StartSession::class,
            AssignRequestId::class,
            Authenticate::class,
            SetTenantContext::class,     // must run AFTER auth (needs the user)
            SubstituteBindings::class,   // must run AFTER tenant (binding is tenant-scoped)
            PermissionMiddleware::class,
        ]);

        $middleware->throttleApi('api');
    })
    ->create();
```

> **Trap — ordering is a correctness issue, not a style issue.** Your tenant middleware must run after `Authenticate` (it needs `$request->user()`) and **before** `SubstituteBindings` (because tenant-scoped route model binding depends on the tenant context existing). Get this backwards and either the tenant is null or bindings resolve unscoped — a data leak.

### Terminable middleware

```php
final class LogRequestMetrics
{
    public function handle(Request $request, Closure $next): Response
    {
        $request->attributes->set('started_at', microtime(true));

        return $next($request);
    }

    // Runs AFTER the response has been sent to the client
    public function terminate(Request $request, Response $response): void
    {
        $durationMs = (microtime(true) - $request->attributes->get('started_at')) * 1000;

        Log::channel('metrics')->info('request', [
            'method'      => $request->method(),
            'path'        => $request->path(),
            'status'      => $response->getStatusCode(),
            'duration_ms' => round($durationMs, 2),
            'org_id'      => optional($request->user())->organization_id,
            'request_id'  => $request->header('X-Request-Id'),
        ]);
    }
}
```

> **Trap:** `terminate()` runs after `send()` **only with php-fpm/FastCGI** (via `fastcgi_finish_request`). Under Octane, terminate runs in-process before the next request is served, so it still delays throughput. Heavy post-response work belongs in a queue.

### Middleware with parameters

```php
final class EnsurePlanAllows
{
    public function handle(Request $request, Closure $next, string $feature): Response
    {
        $org = $request->user()->organization;

        if (! $org->plan->allows($feature)) {
            return response()->json([
                'error'   => 'plan_upgrade_required',
                'feature' => $feature,
            ], 402);   // Payment Required
        }

        return $next($request);
    }
}

Route::post('/reports/export', ExportController::class)
    ->middleware('plan:advanced_reporting');
```

### Practical middleware for your SaaS: idempotency

```php
final class EnsureIdempotency
{
    public function __construct(private readonly Repository $cache) {}

    public function handle(Request $request, Closure $next): Response
    {
        if (! in_array($request->method(), ['POST', 'PATCH', 'PUT'], true)) {
            return $next($request);
        }

        $key = $request->header('Idempotency-Key');

        if (! $key) {
            return response()->json(['error' => 'idempotency_key_required'], 400);
        }

        $orgId    = $request->user()->organization_id;
        $cacheKey = "idem:org:{$orgId}:" . hash('sha256', $key . '|' . $request->path());

        // Replay: return the stored response
        if ($stored = $this->cache->get($cacheKey)) {
            return response($stored['body'], $stored['status'])
                ->withHeaders(['X-Idempotent-Replay' => 'true']);
        }

        // Claim the key atomically so concurrent duplicates don't both execute
        $lock = $this->cache->lock("{$cacheKey}:lock", 30);

        if (! $lock->get()) {
            return response()->json(['error' => 'request_in_progress'], 409);
        }

        try {
            $response = $next($request);

            if ($response->getStatusCode() < 400) {
                $this->cache->put($cacheKey, [
                    'status' => $response->getStatusCode(),
                    'body'   => $response->getContent(),
                ], now()->addHours(24));
            }

            return $response;
        } finally {
            $lock->release();
        }
    }
}
```

> **Follow-up:** *Why is idempotency essential for your trading platform?* A client that times out and retries an order submission must not create two orders. The `Idempotency-Key` header lets the server recognize the retry and replay the original response. Same pattern applies to stock deductions and payment charges. Stripe's API is the canonical reference here — mentioning it signals you've read real API designs.

---

## 3. Eloquent Relationships — Complete Catalogue

```php
class Organization extends Model
{
    public function users(): HasMany { return $this->hasMany(User::class); }
    public function items(): HasMany { return $this->hasMany(InventoryItem::class); }

    // hasOne with modifiers
    public function primaryWarehouse(): HasOne
    {
        return $this->hasOne(Warehouse::class)->where('is_primary', true);
    }

    // hasOneOfMany (Laravel 8.42+) — latest/oldest related record, single query
    public function latestOrder(): HasOne
    {
        return $this->hasOne(Order::class)->latestOfMany();
    }

    public function largestOrder(): HasOne
    {
        return $this->hasOne(Order::class)->ofMany('total_cents', 'max');
    }

    // hasManyThrough — items' movements via items
    public function movements(): HasManyThrough
    {
        return $this->hasManyThrough(StockMovement::class, InventoryItem::class);
    }
}
```

```php
class InventoryItem extends Model
{
    public function organization(): BelongsTo { return $this->belongsTo(Organization::class); }

    public function supplier(): BelongsTo
    {
        return $this->belongsTo(Supplier::class)->withDefault([
            'name' => 'Unknown supplier',     // null object — avoids null checks in views/resources
        ]);
    }

    public function movements(): HasMany { return $this->hasMany(StockMovement::class); }

    // Many-to-many with pivot data and a custom pivot model
    public function warehouses(): BelongsToMany
    {
        return $this->belongsToMany(Warehouse::class)
            ->using(InventoryItemWarehouse::class)   // custom pivot class
            ->withPivot(['quantity', 'bin_location'])
            ->withTimestamps()
            ->as('stock');                            // access via $item->warehouses[0]->stock->quantity
    }

    // Polymorphic one-to-many
    public function auditLogs(): MorphMany
    {
        return $this->morphMany(AuditLog::class, 'auditable');
    }

    // Polymorphic many-to-many
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}
```

```php
class AuditLog extends Model
{
    public function auditable(): MorphTo { return $this->morphTo(); }
}
```

### Full relationship reference

| Type | Method | Keys involved | Example |
|------|--------|---------------|---------|
| One-to-one | `hasOne` | FK on related table | Organization → primary warehouse |
| Inverse one-to-one/many | `belongsTo` | FK on this table | Item → supplier |
| One-to-many | `hasMany` | FK on related table | Item → movements |
| One-of-many | `hasOne()->latestOfMany()` | FK + aggregate subquery | Org → latest order |
| Many-to-many | `belongsToMany` | pivot table | Item ↔ warehouses |
| Has-one-through | `hasOneThrough` | two FKs | Supplier → account manager via contract |
| Has-many-through | `hasManyThrough` | two FKs | Org → movements via items |
| Polymorphic one-to-many | `morphMany` / `morphTo` | `*_id` + `*_type` | Any model → audit logs |
| Polymorphic one-to-one | `morphOne` | `*_id` + `*_type` | Any model → cover image |
| Polymorphic many-to-many | `morphToMany` / `morphedByMany` | polymorphic pivot | Any model ↔ tags |

### Custom pivot model

```php
class InventoryItemWarehouse extends Pivot
{
    protected $table = 'inventory_item_warehouse';

    public $incrementing = true;

    protected function casts(): array
    {
        return ['quantity' => 'integer'];
    }

    public function isEmpty(): bool
    {
        return $this->quantity === 0;
    }
}
```

### Working with relations

```php
// Query the relationship (returns a Builder — chainable, not loaded)
$item->movements()->where('type', 'outbound')->sum('quantity');

// Access the relation (loads once, then cached on the model)
$item->movements;              // Collection
$item->movements;              // second access: NO new query (cached)
$item->unsetRelation('movements');   // force reload next access

// Attach / detach / sync on many-to-many
$item->warehouses()->attach($warehouseId, ['quantity' => 10]);
$item->warehouses()->detach($warehouseId);
$item->warehouses()->sync([1 => ['quantity' => 5], 2 => ['quantity' => 3]]);  // removes others
$item->warehouses()->syncWithoutDetaching([3]);
$item->warehouses()->updateExistingPivot($warehouseId, ['quantity' => 20]);
$item->warehouses()->toggle([1, 2]);

// Create through relation (auto-sets FK)
$item->movements()->create(['type' => 'outbound', 'quantity' => 5]);
$item->movements()->createMany([[...], [...]]);
$item->supplier()->associate($supplier)->save();
$item->supplier()->dissociate()->save();

// Existence / absence
InventoryItem::has('movements')->get();
InventoryItem::has('movements', '>=', 5)->get();
InventoryItem::doesntHave('movements')->get();
InventoryItem::whereHas('movements', fn ($q) => $q->where('type', 'outbound'))->get();
InventoryItem::whereDoesntHave('supplier')->get();
InventoryItem::whereRelation('supplier', 'country', 'DE')->get();     // shorthand
InventoryItem::whereHasMorph(AuditLog::class, [InventoryItem::class])->get();

// Counts / aggregates without loading
InventoryItem::withCount('movements')->get();                         // movements_count
InventoryItem::withCount(['movements as outbound_count' => fn ($q) => $q->where('type', 'outbound')])->get();
InventoryItem::withSum('movements', 'quantity')->get();               // movements_sum_quantity
InventoryItem::withMax('movements', 'created_at')->get();
InventoryItem::withExists('movements')->get();                        // movements_exists (boolean)
```

> **Trap:** `$item->movements()` (method) returns a query builder — every call hits the DB. `$item->movements` (property) loads once and caches. Calling the method in a loop is a hidden N+1 that `preventLazyLoading()` will **not** catch, because it's an explicit query, not a lazy load.

```php
// This is an N+1 that preventLazyLoading does NOT detect
foreach ($items as $item) {
    $total = $item->movements()->sum('quantity');   // one query PER ITEM
}

// Fix: aggregate in one query
$items = InventoryItem::withSum('movements as total', 'quantity')->get();
foreach ($items as $item) {
    $total = $item->total;
}
```

> **Follow-up:** *Downside of polymorphic relations?* No foreign key constraints (the `*_type` column can't be enforced by the DB), so orphaned rows are possible. Queries can't be indexed as tightly, and joining across types is awkward. Use them for genuinely cross-cutting concerns (audit logs, tags, comments), not to avoid designing proper tables.

---

## 4. Advanced Querying

### Subquery selects — collapse N queries into 1

```php
// Instead of loading the latest movement per item (N+1 or heavy eager load):
$items = InventoryItem::query()
    ->addSelect([
        'last_movement_at' => StockMovement::select('created_at')
            ->whereColumn('inventory_item_id', 'inventory_items.id')
            ->latest()
            ->limit(1),

        'last_movement_type' => StockMovement::select('type')
            ->whereColumn('inventory_item_id', 'inventory_items.id')
            ->latest()
            ->limit(1),
    ])
    ->where('organization_id', $orgId)
    ->get();

// Order by a subquery
$items = InventoryItem::orderBy(
    StockMovement::select('created_at')
        ->whereColumn('inventory_item_id', 'inventory_items.id')
        ->latest()
        ->limit(1),
    'desc'
)->get();
```

This technique is a major part of the "88% query reduction" toolkit — it turns per-row lookups into a single correlated subquery.

### Conditional query building

```php
$items = InventoryItem::query()
    ->when($request->filled('search'), fn ($q) => $q->where(function ($q) use ($request) {
        $search = $request->string('search');
        $q->where('sku', 'ILIKE', "%{$search}%")
          ->orWhere('name', 'ILIKE', "%{$search}%");
    }))
    ->when($request->filled('supplier_id'), fn ($q) => $q->where('supplier_id', $request->integer('supplier_id')))
    ->when($request->boolean('low_stock'), fn ($q) => $q->whereColumn('quantity', '<=', 'reorder_point'))
    ->unless($request->boolean('include_archived'), fn ($q) => $q->whereNull('archived_at'))
    ->when($request->input('sort'), function ($q, $sort) {
        // ALLOWLIST — never pass raw user input to orderBy
        $allowed = ['sku', 'name', 'quantity', 'created_at'];
        $column  = in_array(ltrim($sort, '-'), $allowed, true) ? ltrim($sort, '-') : 'id';
        $q->orderBy($column, str_starts_with($sort, '-') ? 'desc' : 'asc');
    }, fn ($q) => $q->orderBy('id'))
    ->paginate(25)
    ->withQueryString();
```

> **Trap — the `orWhere` grouping bug:** Mixing `where` and `orWhere` without grouping breaks tenant scoping.

```php
// BROKEN — returns other tenants' rows!
// SQL: WHERE organization_id = 5 AND sku ILIKE '%x%' OR name ILIKE '%x%'
InventoryItem::where('organization_id', 5)
    ->where('sku', 'ILIKE', '%x%')
    ->orWhere('name', 'ILIKE', '%x%')
    ->get();

// CORRECT — group the OR
// SQL: WHERE organization_id = 5 AND (sku ILIKE '%x%' OR name ILIKE '%x%')
InventoryItem::where('organization_id', 5)
    ->where(fn ($q) => $q->where('sku', 'ILIKE', '%x%')->orWhere('name', 'ILIKE', '%x%'))
    ->get();
```

This is a **real multi-tenant data leak** and an excellent thing to mention as a bug class you actively review for.

### Joins, unions, CTEs, locks

```php
// Joins
DB::table('inventory_items as i')
    ->join('suppliers as s', 's.id', '=', 'i.supplier_id')
    ->leftJoin('stock_movements as m', function ($join) {
        $join->on('m.inventory_item_id', '=', 'i.id')
             ->where('m.created_at', '>=', now()->subDays(30));
    })
    ->where('i.organization_id', $orgId)
    ->select('i.sku', 's.name as supplier', DB::raw('COUNT(m.id) as movement_count'))
    ->groupBy('i.sku', 's.name')
    ->havingRaw('COUNT(m.id) > ?', [5])
    ->get();

// Union
$low = InventoryItem::where('quantity', '<', 10)->select('id', 'sku');
$stale = InventoryItem::where('updated_at', '<', now()->subYear())->select('id', 'sku');
$low->union($stale)->get();

// Locking
InventoryItem::where('id', $id)->lockForUpdate()->first();    // FOR UPDATE (exclusive)
InventoryItem::where('id', $id)->sharedLock()->first();       // FOR SHARE (read lock)

// Explain
InventoryItem::where('organization_id', $orgId)->explain()->dd();
```

### Iteration strategies (recap with numbers)

```php
// 1M rows, ~1 KB each
InventoryItem::all();                    // ~1 GB — OOM
InventoryItem::chunk(1000, ...);         // 1000 models resident; UNSAFE if mutating filter column
InventoryItem::chunkById(1000, ...);     // 1000 models resident; SAFE for mutation
InventoryItem::cursor();                 // 1 model resident (but driver may buffer result set)
InventoryItem::lazy(1000);               // 1 model resident, chunked queries
InventoryItem::lazyById(1000);           // 1 model resident, keyset-paginated — BEST for backfills
InventoryItem::toBase()->get();          // stdClass rows, no model hydration — ~5-10x less memory
```

> **Follow-up:** *Why is `toBase()` faster?* It skips Eloquent model hydration (no attribute casting, no relation containers, no event plumbing). For read-only reporting over hundreds of thousands of rows, hydration is often the dominant cost, not the query.

### Pagination: offset vs cursor

```php
// Offset pagination — friendly UI (page numbers, total count), degrades at depth
$items = InventoryItem::orderBy('id')->paginate(25);
// SQL: ... LIMIT 25 OFFSET 250000   ← DB must scan and discard 250k rows

// simplePaginate — no COUNT(*) query, so no total; faster
$items = InventoryItem::orderBy('id')->simplePaginate(25);

// Cursor pagination — keyset; O(1) regardless of depth; stable under inserts
$items = InventoryItem::orderBy('id')->cursorPaginate(25);
// SQL: ... WHERE id > :lastId ORDER BY id LIMIT 25
```

| | `paginate` | `simplePaginate` | `cursorPaginate` |
|---|-----------|------------------|------------------|
| Total count | Yes (extra `COUNT(*)`) | No | No |
| Jump to page N | Yes | No | No |
| Deep-page performance | Poor (O(offset)) | Poor | Excellent (O(1)) |
| Stable when rows inserted | No (items shift/duplicate) | No | Yes |
| Requires unique ordered column | No | No | Yes |

> **Follow-up:** *Which would you use for a mobile infinite-scroll feed over 10M rows?* `cursorPaginate` — constant-time regardless of depth, and no duplicate/skipped items when new rows are inserted while the user scrolls. Offset pagination for admin tables where users genuinely need page numbers and the data set is bounded.

---

## 5. The N+1 Problem, Exhaustively

### The five flavors of N+1

```php
// 1. Classic lazy load in a loop
foreach (InventoryItem::all() as $item) {
    echo $item->supplier->name;                 // N queries
}
// Fix:
InventoryItem::with('supplier')->get();

// 2. Nested relation
foreach (Order::with('lines')->get() as $order) {
    foreach ($order->lines as $line) {
        echo $line->item->sku;                  // N×M queries
    }
}
// Fix:
Order::with('lines.item')->get();

// 3. Aggregate per row (method call, NOT caught by preventLazyLoading)
foreach ($items as $item) {
    echo $item->movements()->count();           // N queries
}
// Fix:
InventoryItem::withCount('movements')->get();

// 4. Resource-layer lazy load
'supplier' => SupplierResource::make($this->supplier),   // N queries
// Fix:
'supplier' => SupplierResource::make($this->whenLoaded('supplier')),

// 5. Polymorphic morphTo — needs morphWith
AuditLog::with('auditable')->get();             // queries each type separately
// Fix: constrain what's loaded per type
AuditLog::with(['auditable' => fn (MorphTo $m) => $m->morphWith([
    InventoryItem::class => ['supplier:id,name'],
    Order::class         => ['customer:id,name'],
])])->get();
```

### Eager loading toolkit

```php
InventoryItem::with('supplier')->get();
InventoryItem::with(['supplier', 'movements'])->get();
InventoryItem::with('supplier:id,name')->get();                      // column selection (MUST include FK/PK)
InventoryItem::with(['movements' => fn ($q) => $q->latest()->limit(5)])->get();   // ⚠️ see trap
InventoryItem::withOnly(['supplier'])->get();                        // ignore $with defaults
InventoryItem::without('supplier')->get();                           // drop a default eager load
$items->load('movements');                                            // lazy eager load after the fact
$items->loadMissing('supplier');                                      // only if not already loaded
$items->loadCount('movements');
$items->loadSum('movements', 'quantity');

// Default eager loads on the model (use sparingly — always-on cost)
class InventoryItem extends Model
{
    protected $with = ['supplier'];
}
```

> **Trap — `limit` inside eager load:** `with(['movements' => fn ($q) => $q->limit(5)])` applies the limit to the *whole* combined `WHERE IN` query, not per parent. You get 5 movements total, not 5 per item. Fix: use `hasOne()->latestOfMany()` for one-per-parent, or a window-function query, or Laravel 11's per-relation limit support (`->limit()` on `HasMany` eager loads is supported in newer versions — verify on your target version before claiming it in an interview).

### Detecting N+1

```php
// 1. Hard-fail in dev/test — the single best control
// AppServiceProvider::boot()
Model::preventLazyLoading(! $this->app->isProduction());

Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
    $message = sprintf('Lazy loading [%s] on model [%s].', $relation, get_class($model));

    if (app()->isProduction()) {
        Log::warning($message);          // log, don't break prod
    } else {
        throw new LazyLoadingViolationException($model, $relation);
    }
});
```

```php
// 2. Query counting in tests — assert a performance contract
it('lists inventory in a bounded number of queries', function () {
    $org = Organization::factory()->create();
    InventoryItem::factory()->count(50)->for($org)->has(StockMovement::factory()->count(3))->create();
    $user = User::factory()->for($org)->create();

    DB::enableQueryLog();

    $this->actingAs($user, 'api')->getJson('/api/inventory')->assertOk();

    // Regression guard: fails loudly if someone adds a lazy load later
    expect(count(DB::getQueryLog()))->toBeLessThan(8);
});
```

```php
// 3. Log slow / excessive queries in production
// AppServiceProvider::boot()
DB::listen(function (QueryExecuted $query) {
    if ($query->time > 200) {
        Log::channel('slow_queries')->warning('slow query', [
            'sql'      => $query->sql,
            'bindings' => $query->bindings,
            'time_ms'  => $query->time,
            'org_id'   => app(TenantContext::class)->organizationId(),
        ]);
    }
});

// 4. Per-request query budget alert
app()->terminating(function () {
    $count = count(DB::getQueryLog());
    if ($count > 50) {
        Log::warning('query budget exceeded', ['count' => $count, 'path' => request()->path()]);
    }
});
```

> **This is your 88% story's tooling.** Interviewers love hearing that you didn't just fix N+1s once — you installed guardrails (`preventLazyLoading`, query-count assertions in CI, slow-query logging) so they couldn't come back. Prevention > heroics.

---

## 6. Scopes: Local, Global & Removal

### Local scopes

```php
class InventoryItem extends Model
{
    public function scopeLowStock(Builder $query): void
    {
        $query->whereColumn('quantity', '<=', 'reorder_point');
    }

    public function scopeOfSupplier(Builder $query, int $supplierId): void
    {
        $query->where('supplier_id', $supplierId);
    }

    // Return the builder if you want to allow non-chained use
    public function scopeSearch(Builder $query, ?string $term): Builder
    {
        return $query->when($term, fn ($q) => $q->where(
            fn ($q) => $q->where('sku', 'ILIKE', "%{$term}%")->orWhere('name', 'ILIKE', "%{$term}%")
        ));
    }
}

InventoryItem::lowStock()->ofSupplier(3)->search('widget')->get();
```

### Global scopes (the multi-tenancy engine)

```php
final class OrganizationScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $orgId = app(TenantContext::class)->organizationId();

        // Fail closed: if there is no tenant context in an HTTP request, return nothing
        // rather than everything. Silent full-table access is how leaks happen.
        if ($orgId === null) {
            if (app()->runningInConsole()) {
                return;                        // allow CLI/migrations to see all rows
            }
            $builder->whereRaw('1 = 0');
            return;
        }

        $builder->where($model->qualifyColumn('organization_id'), $orgId);
    }
}
```

```php
trait BelongsToOrganization
{
    public static function bootBelongsToOrganization(): void
    {
        static::addGlobalScope(new OrganizationScope());

        // Auto-stamp on create so callers can't forget (or forge) it
        static::creating(function (Model $model): void {
            if ($model->organization_id === null) {
                $model->organization_id = app(TenantContext::class)->organizationIdOrFail();
            }
        });

        // Defense in depth: refuse to save a row for a different tenant
        static::saving(function (Model $model): void {
            $current = app(TenantContext::class)->organizationId();

            if ($current !== null && $model->organization_id !== $current) {
                throw new TenantMismatchException(
                    "Attempted to write {$model->getTable()} for org {$model->organization_id} while in org {$current}"
                );
            }
        });
    }

    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }
}
```

### Removing scopes

```php
InventoryItem::withoutGlobalScope(OrganizationScope::class)->get();   // for admin/cross-tenant reports
InventoryItem::withoutGlobalScopes()->get();
InventoryItem::withoutGlobalScopes([OrganizationScope::class, SoftDeletingScope::class])->get();
```

> **Trap:** Global scopes make queries lie. A developer writing `InventoryItem::count()` in a console command gets a different answer than in a request. And `DB::table('inventory_items')` bypasses them entirely. Mitigations: **fail closed** (as above), forbid raw `DB::table()` on tenant tables via a static-analysis rule or code review, and write an isolation test per tenant-owned model.

> **Follow-up:** *How do you audit that every tenant table is scoped?* Write an architecture test that reflects over all models, checks each one that has an `organization_id` column uses the trait, and asserts the compiled SQL contains the scope:

```php
it('applies the organization scope to every tenant-owned model', function () {
    $models = collect(File::allFiles(app_path('Models')))
        ->map(fn ($f) => 'App\\Models\\' . $f->getFilenameWithoutExtension())
        ->filter(fn ($class) => is_subclass_of($class, Model::class));

    app(TenantContext::class)->setOrganizationId(42);

    foreach ($models as $class) {
        $model = new $class();

        if (! Schema::hasColumn($model->getTable(), 'organization_id')) {
            continue;
        }

        expect($model->newQuery()->toSql())
            ->toContain('organization_id')
            ->and(class_uses_recursive($class))->toContain(BelongsToOrganization::class);
    }
});
```

---

## 7. Model Events & Observers

### Event order

```
retrieved
saving  → creating → created  → saved
saving  → updating → updated  → saved
deleting → deleted
restoring → restored
replicating
trashed / forceDeleting / forceDeleted
```

```php
final class InventoryItemObserver
{
    public function creating(InventoryItem $item): void
    {
        $item->sku = strtoupper($item->sku);
    }

    public function created(InventoryItem $item): void
    {
        SyncItemToSearchIndex::dispatch($item->id, $item->organization_id);
    }

    public function updated(InventoryItem $item): void
    {
        if ($item->wasChanged('quantity')) {
            StockLevelChanged::dispatch(
                $item->id,
                $item->organization_id,
                (int) $item->getOriginal('quantity'),
                $item->quantity,
            );
        }

        // Tenant-scoped invalidation
        Cache::tags(["org:{$item->organization_id}"])->flush();
    }

    public function deleting(InventoryItem $item): void
    {
        if ($item->movements()->exists()) {
            throw new DomainException('Cannot delete an item with stock movements.');
        }
    }
}

// Register: #[ObservedBy] attribute (L11) or in a provider
#[ObservedBy([InventoryItemObserver::class])]
class InventoryItem extends Model {}
```

### Suppressing events

```php
$item->saveQuietly();
$item->updateQuietly([...]);
$item->deleteQuietly();

InventoryItem::withoutEvents(fn () => InventoryItem::create([...]));

// Factories
InventoryItem::factory()->createQuietly();
```

> **Trap — the transaction/observer race:** An observer that dispatches a job inside a transaction can run the job *before* the transaction commits, so the worker reads a row that doesn't exist yet. Two fixes:

```php
// 1. Per-job / per-listener
class SyncItemToSearchIndex implements ShouldQueue
{
    public $afterCommit = true;
}

// 2. Globally, per queue connection — config/queue.php
'redis' => [
    'driver' => 'redis',
    'after_commit' => true,
],

// 3. Ad-hoc
DB::afterCommit(fn () => SyncItemToSearchIndex::dispatch($item->id));
```

> **Follow-up:** *Why not put all side effects in observers?* Observers fire on *any* save from anywhere, including seeders, imports, and tests — often with surprising cost. They're invisible at the call site, they don't fire on bulk operations, and heavy observers make imports crawl. Rule of thumb: observers for **invariants and cheap derived state**; explicit domain events dispatched from Actions for **business side effects**. That distinction is a senior-level answer.

---

## 8. Serialization & API Resources

```php
class InventoryItem extends Model
{
    protected $hidden = ['cost_price', 'internal_notes'];
    protected $visible = [];                      // whitelist alternative
    protected $appends = ['is_low_stock'];

    // Per-instance overrides
    // $item->makeVisible('cost_price');
    // $item->makeHidden('metadata');
    // $item->setAppends([]);
}
```

```php
final class InventoryItemResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'       => $this->id,
            'sku'      => $this->sku,
            'name'     => $this->name,
            'quantity' => $this->quantity,
            'status'   => $this->status->value,

            // Conditional attributes
            'cost_price' => $this->when(
                $request->user()->can('viewCost', $this->resource),
                fn () => (string) $this->cost_price          // lazy: closure not evaluated unless included
            ),

            // Merge a set of keys conditionally
            $this->mergeWhen($request->user()->hasRole('admin'), [
                'created_by' => $this->created_by,
                'audit_url'  => route('audit.item', $this->id),
            ]),

            // Relations — only if loaded
            'supplier'        => SupplierResource::make($this->whenLoaded('supplier')),
            'movements'       => MovementResource::collection($this->whenLoaded('movements')),
            'movements_count' => $this->whenCounted('movements'),

            // Pivot data
            'stock_at_warehouse' => $this->whenPivotLoaded('inventory_item_warehouse',
                fn () => $this->pivot->quantity
            ),

            'links' => [
                'self' => route('inventory.show', $this->id),
            ],
        ];
    }
}
```

### Custom collection wrapper with meta

```php
final class InventoryItemCollection extends ResourceCollection
{
    public $collects = InventoryItemResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'summary' => [
                'total_units' => $this->collection->sum('quantity'),
                'low_stock'   => $this->collection->where('is_low_stock', true)->count(),
            ],
        ];
    }

    public function withResponse(Request $request, JsonResponse $response): void
    {
        $response->header('X-Resource-Version', 'v1');
    }
}
```

```php
// Control the top-level wrapper
JsonResource::withoutWrapping();                 // no "data" key
class InventoryItemResource extends JsonResource { public static $wrap = 'items'; }
```

> **Trap:** `$this->when($cond, $this->expensiveCall())` evaluates `expensiveCall()` **always**, because arguments evaluate before the method runs. Pass a closure: `$this->when($cond, fn () => $this->expensiveCall())`.

---

## 9. Collections & Lazy Collections

```php
$items = collect([
    ['sku' => 'A', 'qty' => 5,  'supplier' => 'X'],
    ['sku' => 'B', 'qty' => 0,  'supplier' => 'Y'],
    ['sku' => 'C', 'qty' => 12, 'supplier' => 'X'],
]);

$items->groupBy('supplier');                     // ['X' => [...], 'Y' => [...]]
$items->keyBy('sku');                            // ['A' => [...], ...]
$items->partition(fn ($i) => $i['qty'] > 0);     // [inStock, outOfStock]
$items->sum('qty');                              // 17
$items->avg('qty');
$items->sortByDesc('qty')->values();
$items->pluck('qty', 'sku');
$items->filter(fn ($i) => $i['qty'] > 0)->values();   // ← values() to reindex for JSON
$items->reject(fn ($i) => $i['qty'] === 0);
$items->flatMap(fn ($i) => [$i['sku'] => $i['qty']]);
$items->chunk(2);
$items->sliding(2);                              // rolling windows
$items->zip([1, 2, 3]);
$items->unique('supplier');
$items->duplicates('supplier');
$items->countBy('supplier');                     // ['X' => 2, 'Y' => 1]
$items->tap(fn ($c) => Log::debug('count', ['n' => $c->count()]));
$items->pipe(fn ($c) => $c->sum('qty'));
$items->whenNotEmpty(fn ($c) => $c->first());
$items->firstWhere('sku', 'B');
$items->contains(fn ($i) => $i['qty'] > 10);
$items->every(fn ($i) => $i['qty'] >= 0);
$items->sole(fn ($i) => $i['sku'] === 'A');      // exactly one, else throws
$items->ensure('array');                          // type assertion (9.x+)
$items->dd(); $items->dump();
```

### Higher-order messages

```php
$models->each->markAsProcessed();        // calls the method on every item
$models->map->only(['id', 'sku']);
$models->filter->isLowStock();
$models->sum->totalValue();
$models->sortBy->createdAt();
```

### Lazy collections — constant memory over huge sources

```php
// Stream a 5 GB CSV without loading it
LazyCollection::make(function () {
    $handle = fopen(storage_path('imports/catalog.csv'), 'r');
    $header = fgetcsv($handle);

    while (($row = fgetcsv($handle)) !== false) {
        yield array_combine($header, $row);
    }

    fclose($handle);
})
->filter(fn (array $row) => $row['sku'] !== '')
->map(fn (array $row) => [
    'organization_id' => $orgId,
    'sku'             => strtoupper($row['sku']),
    'name'            => $row['name'],
    'quantity'        => (int) $row['quantity'],
])
->chunk(1000)
->each(function (LazyCollection $chunk) {
    InventoryItem::upsert($chunk->all(), ['organization_id', 'sku'], ['name', 'quantity']);
});
```

```php
// Rate-limited streaming (respect an external API's limits)
LazyCollection::make($skus)
    ->throttle(1)                    // one item per second
    ->each(fn ($sku) => $api->refreshPrice($sku));

// takeUntilTimeout — bounded work window, useful in scheduled commands
InventoryItem::query()->lazy()
    ->takeUntilTimeout(now()->addMinutes(4))
    ->each(fn ($item) => $item->recalculate());
```

> **Trap:** `Collection` is eager — every method materializes a new array. `LazyCollection` is a generator pipeline — nothing runs until you iterate, and only one element is in memory at a time. But lazy collections can't be counted or re-iterated without re-running the source.

### Custom collection macros

```php
// AppServiceProvider::boot()
Collection::macro('toCsv', function (): string {
    $out = fopen('php://temp', 'r+');
    fputcsv($out, array_keys($this->first()));
    $this->each(fn ($row) => fputcsv($out, (array) $row));
    rewind($out);
    return stream_get_contents($out);
});

// Model-specific collection
class InventoryItem extends Model
{
    public function newCollection(array $models = []): InventoryItemCollection
    {
        return new InventoryItemCollection($models);
    }
}

class InventoryItemCollection extends Collection
{
    public function totalValueInCents(): int
    {
        return $this->sum(fn (InventoryItem $i) => $i->quantity * $i->cost_price_cents);
    }
}
```

---

## 10. Database Transactions

```php
// Closure form — auto-commits, auto-rolls-back on exception, retries on deadlock
DB::transaction(function () use ($data) {
    $order = Order::create([...]);

    foreach ($data->lines as $line) {
        app(DeductStockAction::class)->execute($line);
        $order->lines()->create([...]);
    }

    return $order;
}, attempts: 3);   // ← retry up to 3 times on deadlock/serialization failure

// Manual form — use only when you truly need control
DB::beginTransaction();
try {
    // ...
    DB::commit();
} catch (Throwable $e) {
    DB::rollBack();
    report($e);
    throw $e;
}
```

### Savepoints (nested transactions)

```php
DB::transaction(function () {
    Order::create([...]);                    // outer: real BEGIN

    DB::transaction(function () {
        AuditLog::create([...]);             // inner: SAVEPOINT trans2
    });                                       // inner commit: RELEASE SAVEPOINT
});                                           // outer commit: COMMIT
```

> **Trap:** Because inner "transactions" are savepoints, catching an exception from an inner transaction does **not** roll back the outer one. And rolling back the outer transaction discards the inner work even though its closure "succeeded." Never assume an inner `DB::transaction` is durable on its own.

```php
// Dangerous pattern — looks safe, isn't
DB::transaction(function () {
    $order = Order::create([...]);

    try {
        DB::transaction(fn () => $this->reserveStock($order));   // rolls back to savepoint
    } catch (InsufficientStockException $e) {
        Log::warning('partial reservation');                      // swallowed!
    }

    // Order is created with no stock reserved — inconsistent state committed.
});
```

### `afterCommit` — the most important transaction gotcha

```php
DB::transaction(function () use ($item) {
    $item->update(['quantity' => 0]);

    // ❌ Job may be picked up by a worker before COMMIT → reads stale/absent data
    NotifyOutOfStock::dispatch($item->id);

    // ✅
    NotifyOutOfStock::dispatch($item->id)->afterCommit();
});

// Or globally in config/queue.php: 'after_commit' => true
// Or per job class: public $afterCommit = true;
// Or per event listener: implements ShouldDispatchAfterCommit
```

### Rules for transaction hygiene

1. **Keep transactions short.** Every millisecond holds locks.
2. **No network I/O inside a transaction** — no HTTP calls, no S3 uploads, no email. If the external call hangs, you hold locks for the timeout duration and can exhaust the connection pool.
3. **No queue dispatch without `afterCommit`.**
4. **Acquire locks in a consistent order** across all code paths (prevents deadlocks).
5. **Set a `lock_timeout`/`statement_timeout`** so a stuck transaction fails fast instead of piling up.

```php
// Postgres: fail fast rather than queueing behind a lock forever
DB::statement("SET LOCAL lock_timeout = '3s'");
DB::statement("SET LOCAL statement_timeout = '10s'");
```

> **Follow-up:** *What happens if the HTTP request dies mid-transaction?* The connection closes and the DB rolls back the open transaction. But with a connection pooler (PgBouncer in transaction mode) or persistent connections, a leaked transaction can hold locks until timeout — which is exactly why `idle_in_transaction_session_timeout` exists.

---

## 11. Queues & Jobs

```php
final class RecalculateStockLevels implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 5;
    public int $timeout = 120;              // seconds; must be < worker --timeout consideration
    public int $maxExceptions = 3;
    public bool $failOnTimeout = true;
    public bool $afterCommit = true;
    public string $queue = 'inventory';
    public int $backoff = 10;               // or: public function backoff(): array

    public function __construct(
        public readonly int $organizationId,   // pass IDs, not models
        public readonly array $itemIds = [],
    ) {}

    // Exponential backoff with jitter
    public function backoff(): array
    {
        return [10, 30, 120, 600];
    }

    // Stop retrying after a wall-clock deadline regardless of $tries
    public function retryUntil(): DateTimeInterface
    {
        return now()->addHours(2);
    }

    public function middleware(): array
    {
        return [
            (new WithoutOverlapping("recalc:{$this->organizationId}"))
                ->releaseAfter(30)
                ->expireAfter(300),
            new RateLimited('search-indexing'),
            new SkipIfBatchCancelled(),
        ];
    }

    public function handle(InventoryService $service): void
    {
        // Re-establish tenant context — global scopes depend on it
        app(TenantContext::class)->setOrganizationId($this->organizationId);

        $service->recalculate($this->itemIds);
    }

    public function failed(?Throwable $e): void
    {
        Log::error('stock recalculation failed', [
            'org_id' => $this->organizationId,
            'error'  => $e?->getMessage(),
        ]);

        // Alert, compensate, or notify an operator
    }

    // Tags for Horizon
    public function tags(): array
    {
        return ['org:' . $this->organizationId, 'inventory'];
    }
}
```

### Dispatching

```php
RecalculateStockLevels::dispatch($orgId);
RecalculateStockLevels::dispatch($orgId)->onQueue('high');
RecalculateStockLevels::dispatch($orgId)->onConnection('sqs');
RecalculateStockLevels::dispatch($orgId)->delay(now()->addMinutes(5));
RecalculateStockLevels::dispatch($orgId)->afterCommit();
RecalculateStockLevels::dispatchIf($shouldRun, $orgId);
RecalculateStockLevels::dispatchUnless($skip, $orgId);
RecalculateStockLevels::dispatchSync($orgId);            // run immediately, in-process
dispatch(fn () => Log::info('queued closure'));           // serialized closure
dispatch(fn () => cleanup())->afterResponse();            // after HTTP response, same process
```

### Running workers

```bash
php artisan queue:work redis \
  --queue=high,inventory,default \      # strict priority: drains 'high' first
  --tries=3 \
  --backoff=10 \
  --timeout=90 \
  --memory=256 \
  --max-jobs=1000 \                     # restart worker after N jobs (bounds leaks)
  --max-time=3600 \
  --sleep=1 \
  --rest=0

php artisan queue:listen               # dev only: reloads code each job, much slower
php artisan queue:restart              # graceful: workers finish current job then exit
php artisan queue:retry all
php artisan queue:failed
php artisan queue:flush
php artisan queue:monitor redis:default --max=1000    # alert when backlog exceeds threshold
```

> **Trap — the deploy gotcha:** `queue:work` boots the app once and keeps it in memory. After a deploy, workers still run **old code** until restarted. You must run `php artisan queue:restart` (or roll the worker pods/containers) as part of every deploy. Forgetting this causes bizarre bugs where the API behaves like the new version but jobs behave like the old one.

> **Trap — timeout hierarchy:** `$job->timeout` must be **less** than `retry_after` in `config/queue.php`, otherwise the queue releases the job back for another worker while the first is still running it → the job runs twice concurrently. Rule: `retry_after > timeout`.

```php
// config/queue.php
'redis' => [
    'driver'      => 'redis',
    'queue'       => 'default',
    'retry_after' => 180,      // must exceed the longest job timeout (120 above)
    'block_for'   => 5,        // long-poll instead of busy-looping
    'after_commit'=> true,
],
```

### Unique jobs

```php
// Only one instance may be QUEUED at a time
class SyncSupplierCatalog implements ShouldQueue, ShouldBeUnique
{
    public int $uniqueFor = 3600;      // lock TTL — safety net if the job dies

    public function uniqueId(): string
    {
        return "supplier:{$this->supplierId}";
    }

    public function uniqueVia(): Repository
    {
        return Cache::store('redis');   // MUST be an atomic, shared store
    }
}

// Unique until the job STARTS processing (allows a new one to queue while this runs)
class RebuildIndex implements ShouldQueue, ShouldBeUniqueUntilProcessing {}
```

| Concern | Tool |
|---------|------|
| Prevent duplicate **queued** jobs | `ShouldBeUnique` |
| Prevent duplicate **concurrent execution** | `WithoutOverlapping` middleware |
| Throttle external API calls | `RateLimited` middleware + `RateLimiter::for()` |
| Skip work if batch was cancelled | `SkipIfBatchCancelled` |

```php
// Rate limiter for jobs (define in a provider)
RateLimiter::for('search-indexing', fn () => Limit::perMinute(300));
RateLimiter::for('supplier-api', fn (SyncSupplierCatalog $job) =>
    Limit::perMinute(60)->by("supplier:{$job->supplierId}")
);
```

### Idempotency in jobs — at-least-once delivery is the default

Every queue driver gives **at-least-once**, not exactly-once. A job can run twice (worker crash after work but before ack, visibility timeout expiry on SQS, `retry_after` misconfiguration). Your handler must tolerate it.

```php
public function handle(): void
{
    // Guard with a durable marker, not just a cache key
    $processed = StockAdjustment::where('idempotency_key', $this->idempotencyKey)->exists();

    if ($processed) {
        Log::info('duplicate job skipped', ['key' => $this->idempotencyKey]);
        return;
    }

    DB::transaction(function () {
        StockAdjustment::create([
            'idempotency_key'  => $this->idempotencyKey,   // UNIQUE index in DB
            'inventory_item_id'=> $this->itemId,
            'delta'            => $this->delta,
        ]);

        InventoryItem::where('id', $this->itemId)->increment('quantity', $this->delta);
    });
}
```

The **unique index on `idempotency_key` is the real guarantee** — a cache check alone races. Catch the unique-violation and treat it as success.

> **Follow-up:** *Redis vs SQS vs database queue — trade-offs?*

| | Database | Redis | SQS |
|---|----------|-------|-----|
| Setup | Zero extra infra | Redis needed | Managed |
| Throughput | Low (row locks, polling) | High | Very high |
| Delayed jobs | Yes | Yes (sorted set) | Yes (max 15 min) |
| Job size limit | DB column | Redis memory | 256 KB |
| Visibility/retry model | `reserved_at` + `retry_after` | same | native visibility timeout |
| Ordering | ~FIFO | ~FIFO | Standard: no order; FIFO queues: yes |
| Horizon support | No | **Yes** | No |
| Durability | Strong (ACID) | Depends on persistence config | Very strong |
| Best for | Small apps, transactional consistency with data | Most Laravel apps, need Horizon metrics | Cross-service, huge scale, AWS-native |

Since you use both Redis and SQS: Redis + Horizon for in-app background work (great observability, `WithoutOverlapping`, batching), SQS when the producer and consumer are **different services** or you need AWS-native fan-out with SNS.

---

## 12. Job Batching & Chaining

### Batching — parallel fan-out with aggregate progress

```php
$orgId = $organization->id;

$batch = Bus::batch(
    InventoryItem::where('organization_id', $orgId)
        ->pluck('id')
        ->chunk(500)
        ->map(fn (Collection $ids) => new ReindexItems($ids->all(), $orgId))
        ->all()
)
    ->name("reindex:org:{$orgId}")
    ->allowFailures()                      // don't cancel the batch on first failure
    ->onQueue('indexing')
    ->before(fn (Batch $b) => Log::info('batch created', ['id' => $b->id]))
    ->progress(fn (Batch $b) => Log::debug('progress', ['pct' => $b->progress()]))
    ->then(fn (Batch $b) => Log::info('all jobs succeeded'))
    ->catch(fn (Batch $b, Throwable $e) => Log::error('first failure', ['e' => $e->getMessage()]))
    ->finally(function (Batch $b) use ($orgId) {
        Cache::tags(["org:{$orgId}"])->flush();
        ReindexCompleted::dispatch($orgId, $b->failedJobs);
    })
    ->dispatch();

return response()->json(['batch_id' => $batch->id]);
```

```php
// Poll progress from the API
$batch = Bus::findBatch($batchId);

return [
    'total'       => $batch->totalJobs,
    'pending'     => $batch->pendingJobs,
    'failed'      => $batch->failedJobs,
    'progress'    => $batch->progress(),      // percentage
    'finished'    => $batch->finished(),
    'cancelled'   => $batch->cancelled(),
];

$batch->cancel();
```

```php
// Inside a batched job
public function handle(): void
{
    if ($this->batch()?->cancelled()) {
        return;
    }

    // Add more jobs to the running batch (dynamic fan-out)
    $this->batch()->add([new FollowUpJob(...)]);
}
```

Requires the `job_batches` table (`php artisan queue:batches-table`).

### Chaining — sequential, stop on first failure

```php
Bus::chain([
    new ValidateImportFile($uploadId),
    new ParseImportFile($uploadId),
    new UpsertInventoryItems($uploadId),
    new ReindexOrganization($orgId),
    new NotifyImportComplete($uploadId),
])
->onQueue('imports')
->catch(fn (Throwable $e) => ImportFailed::dispatch($uploadId, $e->getMessage()))
->dispatch();
```

```php
// Chain from inside a job
public function handle(): void
{
    // ...
    $this->chain([new NextStep()]);         // prepend to remaining chain
}
```

### Batches within chains (the real-world import pipeline)

```php
Bus::chain([
    new ValidateImportFile($uploadId),

    // A whole batch as one chain link — chain waits for all batch jobs
    fn () => Bus::batch(
        collect($chunks)->map(fn ($c) => new ImportChunk($c, $uploadId))->all()
    )->name("import:{$uploadId}")->dispatch(),

    new FinalizeImport($uploadId),
])->dispatch();
```

> **Follow-up:** *When batch vs chain?* Batch = independent work you want to parallelize with aggregate completion tracking (reindex 200k items across 20 workers). Chain = strictly ordered steps where each depends on the previous (validate → parse → import → notify). Combine them for pipelines with a parallel middle stage.

> **Trap:** Chains and batches store state in the DB (`job_batches`) and cache. If a worker is killed with `SIGKILL` mid-job, batch counters can drift. Always design the `finally` callback to be idempotent, and reconcile with actual data rather than trusting counters for correctness.

---

## 13. Events, Listeners & Subscribers

```php
final class StockLevelChanged
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly int $itemId,
        public readonly int $organizationId,
        public readonly int $previousQuantity,
        public readonly int $newQuantity,
        public readonly ?int $userId = null,
    ) {}

    public function crossedReorderPoint(int $reorderPoint): bool
    {
        return $this->previousQuantity > $reorderPoint && $this->newQuantity <= $reorderPoint;
    }
}
```

```php
// Synchronous listener — runs in-request. Keep it fast and non-critical-path.
final class WriteStockAuditLog
{
    public function handle(StockLevelChanged $event): void
    {
        AuditLog::create([
            'organization_id' => $event->organizationId,
            'auditable_type'  => InventoryItem::class,
            'auditable_id'    => $event->itemId,
            'action'          => 'stock_changed',
            'changes'         => ['from' => $event->previousQuantity, 'to' => $event->newQuantity],
            'user_id'         => $event->userId,
        ]);
    }
}

// Queued listener
final class NotifyLowStock implements ShouldQueue
{
    use InteractsWithQueue;

    public int $tries = 3;
    public string $queue = 'notifications';
    public bool $afterCommit = true;

    public function handle(StockLevelChanged $event): void
    {
        app(TenantContext::class)->setOrganizationId($event->organizationId);

        $item = InventoryItem::findOrFail($event->itemId);

        if ($event->crossedReorderPoint($item->reorder_point ?? 0)) {
            Notification::send($item->organization->purchasers, new LowStockAlert($item));
        }
    }

    // Prevent the listener from running for irrelevant events
    public function shouldQueue(StockLevelChanged $event): bool
    {
        return $event->newQuantity < $event->previousQuantity;
    }

    public function failed(StockLevelChanged $event, Throwable $e): void
    {
        Log::error('low stock notification failed', ['item' => $event->itemId]);
    }
}
```

### Registration (Laravel 11 auto-discovers by type-hint)

```php
// Explicit registration when needed
Event::listen(StockLevelChanged::class, WriteStockAuditLog::class);

// Wildcard
Event::listen('stock.*', function (string $eventName, array $data) {});

// Subscriber — group related listeners
class InventorySubscriber
{
    public function subscribe(Dispatcher $events): array
    {
        return [
            StockLevelChanged::class => [WriteStockAuditLog::class, NotifyLowStock::class],
            ItemArchived::class      => 'handleItemArchived',
        ];
    }

    public function handleItemArchived(ItemArchived $event): void {}
}
```

### Stopping propagation & event results

```php
// Returning false from a listener halts remaining listeners
public function handle(OrderPlacing $event): bool
{
    if ($event->order->total_cents > 1_000_000) {
        return false;      // stops the chain
    }
    return true;
}

$results = Event::dispatch(new CalculateDiscount($order));  // array of listener return values
Event::until(new CanCancelOrder($order));                    // first non-null response
```

### Faking in tests

```php
Event::fake();
Event::fake([StockLevelChanged::class]);                     // fake only these; others run normally
Event::fakeExcept([SomeCriticalEvent::class]);

// assertions
Event::assertDispatched(StockLevelChanged::class);
Event::assertDispatched(StockLevelChanged::class, fn ($e) => $e->newQuantity === 0);
Event::assertDispatchedTimes(StockLevelChanged::class, 3);
Event::assertNotDispatched(OrderPlaced::class);
Event::assertListening(StockLevelChanged::class, NotifyLowStock::class);
```

> **Trap:** `Event::fake()` prevents listeners from running — including model observers registered as event listeners, and Eloquent's own internal events in some cases. If your test needs the side effect, fake selectively.

> **Follow-up:** *Event vs Job — when each?* An **event** describes something that happened in the domain (`StockLevelChanged`), and may have zero or many subscribers, none of which the publisher knows about. A **job** is a specific unit of work you want done. Use events for decoupling and extensibility; use jobs when there's exactly one thing to do and you want direct control over retries/queues. Overusing events makes control flow untraceable — a real criticism senior interviewers probe for.

---

## 14. Caching & Atomic Locks

```php
Cache::get('key', 'default');
Cache::put('key', $value, now()->addMinutes(10));
Cache::add('key', $value, 60);                       // only if missing — ATOMIC
Cache::forever('key', $value);
Cache::remember('key', 600, fn () => expensive());
Cache::rememberForever('key', fn () => expensive());
Cache::pull('key');                                   // get + forget
Cache::forget('key');
Cache::increment('counter'); Cache::decrement('counter');
Cache::has('key');
Cache::many(['a', 'b']); Cache::putMany([...], 60);
Cache::flush();                                       // ⚠️ nukes everything on the store
Cache::store('redis')->tags(['org:5'])->flush();
Cache::flexible('key', [300, 900], fn () => expensive());   // L11: SWR-style stale-while-revalidate
```

### Tenant-scoped caching for your SaaS

```php
final class TenantCache
{
    public function __construct(
        private readonly Repository $cache,
        private readonly TenantContext $tenant,
    ) {}

    public function remember(string $key, int $ttl, Closure $callback): mixed
    {
        $orgId = $this->tenant->organizationIdOrFail();

        return $this->cache
            ->tags(["org:{$orgId}"])
            ->remember("org:{$orgId}:{$key}", $ttl, $callback);
    }

    public function flushTenant(): void
    {
        $this->cache->tags(["org:{$this->tenant->organizationIdOrFail()}"])->flush();
    }
}
```

> **Trap — the #1 multi-tenant caching bug:** an unnamespaced key. `Cache::remember('dashboard_stats', ...)` serves Org A's numbers to Org B. Every cache key in a multi-tenant app must include the tenant identifier, and every cache read path must be covered by a test that proves two tenants get different values.

```php
it('does not leak cached dashboard stats between tenants', function () {
    $orgA = Organization::factory()->create();
    $orgB = Organization::factory()->create();
    InventoryItem::factory()->count(5)->for($orgA)->create();
    InventoryItem::factory()->count(2)->for($orgB)->create();

    $userA = User::factory()->for($orgA)->create();
    $userB = User::factory()->for($orgB)->create();

    $a = $this->actingAs($userA, 'api')->getJson('/api/dashboard')->json('data.item_count');
    $b = $this->actingAs($userB, 'api')->getJson('/api/dashboard')->json('data.item_count');

    expect($a)->toBe(5)->and($b)->toBe(2);
});
```

### Cache stampede (thundering herd) and how to fix it

**The problem:** A hot key expires. 500 concurrent requests all miss, all run the expensive query, all hammer the DB. Latency spikes, connections exhaust, possible outage.

```php
// Fix 1: Lock so only one request recomputes; others wait for the result
public function stats(int $orgId): array
{
    $key = "org:{$orgId}:stats";

    if ($cached = Cache::get($key)) {
        return $cached;
    }

    $lock = Cache::lock("{$key}:rebuild", 10);

    if ($lock->get()) {
        try {
            $value = $this->computeStats($orgId);
            Cache::put($key, $value, now()->addMinutes(5));
            return $value;
        } finally {
            $lock->release();
        }
    }

    // Someone else is rebuilding — block briefly, then fall back
    try {
        $lock->block(3);
        $lock->release();
        return Cache::get($key) ?? $this->computeStats($orgId);
    } catch (LockTimeoutException) {
        return Cache::get("{$key}:stale") ?? $this->computeStats($orgId);
    }
}
```

```php
// Fix 2: stale-while-revalidate — serve stale instantly, refresh in the background
// Laravel 11 built-in:
Cache::flexible("org:{$orgId}:stats", [300, 900], fn () => $this->computeStats($orgId));
// fresh for 300s; between 300–900s serve stale AND refresh after the response
```

```php
// Fix 3: probabilistic early expiration — spread out recomputation
$ttl = 300 + random_int(0, 60);      // jitter prevents synchronized expiry across keys
Cache::put($key, $value, $ttl);
```

```php
// Fix 4: cache warming — never let the hot key expire cold
Schedule::call(fn () => Organization::each(
    fn ($org) => Cache::put("org:{$org->id}:stats", computeStats($org->id), 600)
))->everyFiveMinutes();
```

### Atomic locks

```php
$lock = Cache::lock('process-eod-report', 120);

if ($lock->get()) {
    try {
        $this->runEndOfDayReport();
    } finally {
        $lock->release();
    }
}

// Auto-release via callback
Cache::lock('key', 10)->get(function () {
    // lock released automatically when the closure returns
});

// Block and wait up to 5 seconds for the lock
try {
    Cache::lock('key', 10)->block(5, function () {
        // ...
    });
} catch (LockTimeoutException $e) {
    // couldn't acquire
}

// Cross-process lock ownership (acquire in a request, release in a job)
$lock = Cache::lock('long-task', 600);
$lock->get();
$owner = $lock->owner();

// in the job:
Cache::restoreLock('long-task', $owner)->release();
```

> **Trap:** Cache locks require an **atomic, shared** store — Redis, Memcached, DynamoDB, or the database. With `file` or `array` drivers, each server/process has its own store, so the "lock" provides no mutual exclusion at all. Also: locks are **not** a substitute for DB transactions. If the process dies after acquiring the lock but before finishing, the TTL expires and another process proceeds — so the protected operation must still be idempotent or transactional.

### Cache invalidation strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| TTL only | `remember($key, 300, ...)` | Simple; serves stale data up to TTL |
| Write-through | Update cache on every write | Always fresh; more write complexity |
| Tag flush | `Cache::tags(["org:{$id}"])->flush()` | Coarse but easy; Redis/Memcached only |
| Key versioning | `"org:{$id}:v{$version}:stats"`, bump version on write | Atomic, no scan, old keys expire naturally |
| Event-driven | Observer dispatches invalidation | Precise; breaks on bulk ops that skip events |

```php
// Key versioning — avoids tag overhead and works on any store
$version = Cache::get("org:{$orgId}:cache_version", 1);
$value = Cache::remember("org:{$orgId}:v{$version}:stats", 600, $callback);

// Invalidate everything for the tenant with one atomic increment
Cache::increment("org:{$orgId}:cache_version");
```

---

## 15. Redis Beyond Caching

You list Redis as a skill; interviewers will ask what you use it for *besides* `Cache::remember`.

```php
// Sorted set — leaderboard / top movers on your trading platform
Redis::zadd("org:{$orgId}:top-movers", $volume, $symbol);
Redis::zrevrange("org:{$orgId}:top-movers", 0, 9, ['withscores' => true]);

// Sliding-window rate limit
$key = "rate:org:{$orgId}:" . now()->format('YmdHi');
$count = Redis::incr($key);
if ($count === 1) { Redis::expire($key, 120); }
if ($count > 1000) { abort(429); }

// Distributed counter with a single round trip
Redis::pipeline(function ($pipe) use ($orgId) {
    $pipe->hincrby("org:{$orgId}:metrics", 'api_calls', 1);
    $pipe->hincrby("org:{$orgId}:metrics", 'items_read', 25);
});

// Set — dedupe/uniqueness tracking
Redis::sadd("org:{$orgId}:processed-imports", $importId);
Redis::sismember("org:{$orgId}:processed-imports", $importId);

// Hash — session-ish or per-entity field storage
Redis::hset("item:{$itemId}", 'quantity', 42);
Redis::hgetall("item:{$itemId}");

// Pub/Sub — broadcasting
Redis::publish("org:{$orgId}:stock-updates", json_encode($payload));

// Streams — durable event log with consumer groups
Redis::xadd("stock-events", '*', ['item_id' => $itemId, 'delta' => -1]);

// Lua script for true atomicity across multiple keys
$script = <<<'LUA'
local current = tonumber(redis.call('GET', KEYS[1]) or '0')
if current >= tonumber(ARGV[1]) then
    return redis.call('DECRBY', KEYS[1], ARGV[1])
end
return -1
LUA;

$result = Redis::eval($script, 1, "stock:{$itemId}", $amount);
```

> **Trap:** `Redis::get()` then `Redis::set()` is not atomic — classic read-modify-write race. Use `INCR`/`DECRBY`, `SETNX`, `GETSET`, `WATCH`/`MULTI`, or a Lua script (which Redis executes atomically as a single command).

> **Follow-up:** *Is Redis a safe source of truth for inventory counts?* No, not alone. Redis is not durable by default (async AOF/RDB), and a failover can lose recent writes. Use Redis as a fast **reservation/rate-limiting layer** or read cache, and Postgres as the authoritative ledger. For stock, the DB row (or an append-only movements table) is the truth.

> **Follow-up:** *Redis persistence?* RDB = periodic snapshots (fast restart, can lose data since last snapshot). AOF = append-only log (`appendfsync everysec` typical — up to 1s data loss). Redis Cluster/Sentinel for HA; ElastiCache Multi-AZ with automatic failover in AWS.

---

## 16. Authentication: Guards, Providers, Sanctum vs Passport

### The model

```php
// config/auth.php
'guards' => [
    'web'    => ['driver' => 'session', 'provider' => 'users'],
    'api'    => ['driver' => 'passport', 'provider' => 'users'],
    'admin'  => ['driver' => 'session', 'provider' => 'admins'],
],

'providers' => [
    'users'  => ['driver' => 'eloquent', 'model' => User::class],
    'admins' => ['driver' => 'eloquent', 'model' => Admin::class],
],
```

- **Guard** = *how* a request is authenticated (session cookie, bearer token).
- **Provider** = *where* users come from (Eloquent model, database table, LDAP).

```php
Auth::guard('api')->user();
Auth::guard('admin')->check();
auth()->id();
$request->user();
$request->user('api');
Auth::shouldUse('api');
```

### Sanctum vs Passport — know exactly why you chose Passport

| | Sanctum | Passport |
|---|---------|----------|
| Standard | Custom (simple bearer tokens) + SPA cookie mode | Full **OAuth 2.0** server |
| Token format | Opaque, hashed in DB (`personal_access_tokens`) | JWT access tokens signed with RSA keys |
| Grants | N/A | authorization_code (+PKCE), client_credentials, password (deprecated), refresh_token, implicit (deprecated) |
| Third-party apps | Not designed for it | **Yes** — that's the point |
| Refresh tokens | No (issue a new token) | Yes |
| Scopes | Abilities (`$token->can('items:read')`) | Scopes (`Passport::tokensCan([...])`) |
| Revocation | Delete the DB row → immediate | Access token is stateless JWT → revocation needs token introspection or short TTLs |
| Complexity | Very low | High (keys, clients, grants, migrations) |
| Best for | First-party SPA/mobile, simple API tokens | Public API with third-party integrations, SSO, machine-to-machine |

```php
// Passport setup
Passport::tokensCan([
    'inventory:read'  => 'Read inventory',
    'inventory:write' => 'Modify inventory',
    'reports:read'    => 'Read reports',
]);

Passport::setDefaultScope(['inventory:read']);
Passport::tokensExpireIn(now()->addMinutes(15));
Passport::refreshTokensExpireIn(now()->addDays(30));
Passport::personalAccessTokensExpireIn(now()->addMonths(6));
Passport::enablePasswordGrant();          // avoid — deprecated in OAuth 2.1
```

```php
// Scope middleware
Route::middleware(['auth:api', 'scopes:inventory:write'])->post('/inventory', ...);   // ALL scopes
Route::middleware(['auth:api', 'scope:inventory:read,reports:read'])->get(...);       // ANY scope

// In code
if ($request->user()->tokenCan('inventory:write')) {}
```

### The grant flows (be able to draw these)

```
Authorization Code + PKCE (SPA / mobile — the correct modern choice)
  1. Client generates code_verifier (random), code_challenge = SHA256(verifier)
  2. Redirect user → /oauth/authorize?response_type=code&client_id=..&code_challenge=..&code_challenge_method=S256&state=..
  3. User authenticates + consents
  4. Redirect back with ?code=..&state=..     (verify state — CSRF protection)
  5. POST /oauth/token { grant_type=authorization_code, code, code_verifier, client_id }
  6. Server verifies SHA256(code_verifier) == stored code_challenge
  7. → { access_token, refresh_token, expires_in }

Client Credentials (service-to-service — no user involved)
  POST /oauth/token { grant_type=client_credentials, client_id, client_secret, scope }
  → { access_token }

Refresh Token
  POST /oauth/token { grant_type=refresh_token, refresh_token, client_id, client_secret }
  → new access_token (+ rotated refresh_token if rotation is enabled)
```

> **Trap:** *Why is the implicit grant deprecated?* It returns the access token in the URL fragment — it lands in browser history, referrer headers, and server logs, and there's no client authentication. Authorization Code + PKCE replaced it for public clients.

> **Trap:** *Why is the password grant deprecated?* It requires the client to handle the user's raw credentials, defeats MFA and federated login, and encourages credential storage. OAuth 2.1 removes it.

> **Follow-up:** *Where should an SPA store tokens?* Not `localStorage` (any XSS steals it). Best: `httpOnly`, `Secure`, `SameSite=Lax/Strict` cookies (Sanctum's SPA mode does this with CSRF protection). If you must use bearer tokens, keep TTLs very short, store in memory only, and refresh via an httpOnly refresh cookie.

> **Follow-up:** *You use Passport — how do you revoke a JWT access token immediately?* You can't purely statelessly. Options: (a) very short access-token TTL (5–15 min) so revocation window is bounded, (b) check a revocation list / `oauth_access_tokens.revoked` on each request (Passport does this — which makes it not fully stateless), (c) maintain a Redis denylist of `jti` values until natural expiry. Be explicit that this is the fundamental JWT trade-off: statelessness vs instant revocation.

---

## 17. Authorization: Gates, Policies & spatie/permission

### Gates — simple, closure-based

```php
// AppServiceProvider::boot()
Gate::define('access-admin-panel', fn (User $user) => $user->is_super_admin);

Gate::define('export-reports', function (User $user) {
    return $user->organization->plan->allows('exports')
        && $user->can('reports.export');
});

// Runs before all checks — super-admin bypass
Gate::before(fn (User $user) => $user->is_super_admin ? true : null);

// Runs after, only if nothing returned a decision
Gate::after(fn (User $user, string $ability) => $user->isOwner() ? true : null);
```

> **Trap:** `Gate::before` returning `true` bypasses **every** policy including tenant checks. In a multi-tenant SaaS, a careless super-admin bypass means support staff can read any tenant's data — which may be a compliance violation. Gate it behind explicit impersonation with an audit trail.

### Policies

```php
final class InventoryItemPolicy
{
    // Runs before every method in this policy
    public function before(User $user, string $ability): ?bool
    {
        // Hard tenant boundary, enforced once
        return null;   // let individual methods decide
    }

    public function viewAny(User $user): bool
    {
        return $user->can('inventory.view');
    }

    public function view(User $user, InventoryItem $item): bool
    {
        return $user->organization_id === $item->organization_id
            && $user->can('inventory.view');
    }

    public function create(User $user): bool
    {
        return $user->can('inventory.create');
    }

    public function update(User $user, InventoryItem $item): Response
    {
        if ($user->organization_id !== $item->organization_id) {
            return Response::denyAsNotFound();       // 404, not 403 — no existence leak
        }

        if ($item->is_locked) {
            return Response::deny('This item is locked for stocktake.');
        }

        return $user->can('inventory.update')
            ? Response::allow()
            : Response::deny('You lack the inventory.update permission.');
    }

    public function viewCost(User $user, InventoryItem $item): bool
    {
        return $user->organization_id === $item->organization_id
            && $user->hasAnyRole(['owner', 'finance']);
    }
}
```

```php
// Usage
$this->authorize('update', $item);                       // controller — throws 403/404
$request->user()->can('update', $item);
Gate::allows('update', $item);
Gate::forUser($otherUser)->denies('update', $item);
$this->authorize('create', InventoryItem::class);         // no instance yet
Route::put('/inventory/{item}', ...)->can('update', 'item');   // route-level

// In a Form Request
public function authorize(): bool
{
    return $this->user()->can('update', $this->route('item'));
}
```

### spatie/laravel-permission in teams mode

```php
// config/permission.php
'teams' => true,
'team_foreign_key' => 'organization_id',
```

Tables gain `organization_id`: `model_has_roles`, `model_has_permissions`, and `roles` (permissions stay global).

```php
// Set the active team BEFORE any permission check
setPermissionsTeamId($organizationId);
// or
app(PermissionRegistrar::class)->setPermissionsTeamId($organizationId);

// Assign a role scoped to the current team
$user->assignRole('inventory-manager');

// A user can hold different roles per organization
setPermissionsTeamId($orgA->id);
$user->assignRole('owner');

setPermissionsTeamId($orgB->id);
$user->assignRole('viewer');

// Checks are relative to the current team id
setPermissionsTeamId($orgA->id);
$user->hasRole('owner');            // true
$user->can('inventory.delete');     // true

setPermissionsTeamId($orgB->id);
$user->hasRole('owner');            // false
```

> **Trap #1 — the team ID must be set before checks, everywhere.** Middleware handles HTTP requests, but **queued jobs, scheduled commands, and tests have no middleware.** A job that checks permissions without calling `setPermissionsTeamId()` reads whatever the last value was (or none) → wrong authorization decisions.

```php
// Job: set BOTH tenant context and permission team id
public function handle(): void
{
    app(TenantContext::class)->setOrganizationId($this->organizationId);
    setPermissionsTeamId($this->organizationId);

    // now permission checks are correct
}
```

> **Trap #2 — the permission cache.** spatie caches the full permission/role map (default 24h) under a single key. After changing roles you must `forgetCachedPermissions()`, and in tests you must reset it between cases or you'll get bizarre cross-test failures.

```php
app(PermissionRegistrar::class)->forgetCachedPermissions();

// Pest: reset in beforeEach
beforeEach(function () {
    app(PermissionRegistrar::class)->forgetCachedPermissions();
});
```

> **Trap #3 — `hasPermissionTo` triggers a query per check if relations aren't loaded.** For endpoints that check many permissions, eager-load `roles.permissions` and `permissions` on the user, or rely on the cached registrar.

> **Follow-up:** *Roles vs permissions — how do you model this?* Check **permissions** in code (`$user->can('inventory.update')`), never roles. Roles are just bundles that admins can reshape without a deploy. Hardcoding `hasRole('manager')` means every customer request to change who can do what becomes a code change. This is a genuinely senior distinction.

> **Follow-up:** *Policy + spatie together — is that redundant?* No, they're orthogonal. spatie answers "does this user hold this capability?" The policy answers "for **this specific record**, in **this tenant**, in **this state**?" You need both: permission check plus tenant/ownership/state check.

---

## 18. API Design in Laravel

### Versioning

```php
// bootstrap/app.php or a provider
Route::prefix('api/v1')->middleware('api')->group(base_path('routes/api_v1.php'));
Route::prefix('api/v2')->middleware('api')->group(base_path('routes/api_v2.php'));

// Namespaced resources per version
App\Http\Resources\V1\InventoryItemResource
App\Http\Resources\V2\InventoryItemResource
```

| Strategy | Example | Trade-off |
|----------|---------|-----------|
| URL path | `/api/v1/items` | Explicit, cacheable, easy to route. Most common. |
| Header | `Accept: application/vnd.app.v2+json` | "Purer" REST; harder to test/debug, CDN caching needs Vary |
| Query param | `/api/items?version=2` | Easy but pollutes params, awkward caching |

Pick URL path and say why: discoverability, CDN/proxy friendliness, and trivially testable with curl.

### Consistent error envelope

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (Throwable $e, Request $request) {
        if (! $request->is('api/*')) {
            return null;
        }

        [$status, $code] = match (true) {
            $e instanceof ValidationException        => [422, 'validation_failed'],
            $e instanceof AuthenticationException    => [401, 'unauthenticated'],
            $e instanceof AuthorizationException     => [403, 'forbidden'],
            $e instanceof ModelNotFoundException,
            $e instanceof NotFoundHttpException      => [404, 'not_found'],
            $e instanceof MethodNotAllowedHttpException => [405, 'method_not_allowed'],
            $e instanceof ThrottleRequestsException  => [429, 'rate_limited'],
            $e instanceof InsufficientStockException => [409, 'insufficient_stock'],
            default                                  => [500, 'server_error'],
        };

        return response()->json([
            'error' => [
                'code'       => $code,
                'message'    => $status === 500 && app()->isProduction()
                    ? 'An unexpected error occurred.'
                    : $e->getMessage(),
                'details'    => $e instanceof ValidationException ? $e->errors() : null,
                'request_id' => $request->header('X-Request-Id'),
            ],
        ], $status);
    });
})
```

> **Trap:** Leaking `$e->getMessage()` on 500s in production exposes SQL fragments, file paths, and connection strings. Always gate detailed messages behind a non-production check, and log the full exception with the same `request_id` so support can correlate.

### Status code discipline

| Code | Use for | Common mistake |
|------|---------|-----------------|
| 200 | Successful GET/PATCH/PUT | Returning 200 with `{"error": ...}` — never do this |
| 201 | Resource created (include `Location`) | Using 200 |
| 202 | Accepted for async processing | Using 200 and lying about completion |
| 204 | Successful DELETE, empty body | Returning 200 with `{}` |
| 400 | Malformed request (bad JSON) | Using it for validation errors |
| 401 | Not authenticated | Confusing with 403 |
| 403 | Authenticated but not permitted | Using for cross-tenant (should be 404) |
| 404 | Not found / not visible to this tenant | Using 403 and leaking existence |
| 409 | State conflict (insufficient stock, version mismatch) | Using 400 or 422 |
| 422 | Semantically invalid payload | Using 400 |
| 429 | Rate limited (include `Retry-After`) | Omitting `Retry-After` |
| 500 | Unhandled server error | Exposing stack traces |
| 503 | Maintenance / dependency down (include `Retry-After`) | Using 500 |

### Filtering, sorting, sparse fieldsets

```php
// GET /api/v1/inventory?filter[low_stock]=1&filter[supplier_id]=3&sort=-quantity&fields=id,sku,quantity&include=supplier&page[size]=50

final class InventoryQuery
{
    private const SORTABLE = ['sku', 'name', 'quantity', 'created_at'];
    private const INCLUDABLE = ['supplier', 'movements'];

    public function apply(Builder $query, Request $request): Builder
    {
        // Filters (allowlisted)
        collect($request->input('filter', []))->each(function ($value, $key) use ($query) {
            match ($key) {
                'low_stock'   => $query->whereColumn('quantity', '<=', 'reorder_point'),
                'supplier_id' => $query->where('supplier_id', (int) $value),
                'search'      => $query->where(fn ($q) => $q
                    ->where('sku', 'ILIKE', "%{$value}%")
                    ->orWhere('name', 'ILIKE', "%{$value}%")),
                default       => null,     // silently ignore unknown filters
            };
        });

        // Sorting (allowlisted)
        foreach (explode(',', (string) $request->input('sort')) as $sort) {
            $column = ltrim($sort, '-');
            if (in_array($column, self::SORTABLE, true)) {
                $query->orderBy($column, str_starts_with($sort, '-') ? 'desc' : 'asc');
            }
        }
        $query->orderBy('id');   // ALWAYS add a deterministic tiebreaker

        // Includes (allowlisted — prevents include=... N+1 abuse)
        $includes = array_intersect(
            explode(',', (string) $request->input('include')),
            self::INCLUDABLE
        );
        $query->with($includes);

        return $query;
    }
}
```

> **Trap:** Unbounded `page[size]`. A client sending `page[size]=100000` can OOM your app or exhaust the DB. Clamp it: `min($request->integer('page.size', 25), 100)`.

> **Trap:** Non-deterministic ordering. `ORDER BY created_at DESC` with ties means the same row can appear on page 1 and page 2. Always append a unique tiebreaker (`id`).

---

## 19. Testing Strategy with Pest

### The pyramid, applied to a Laravel API

```
        /\        E2E / smoke (few)         — real HTTP against a deployed env
       /  \
      /----\      Feature tests (many)      — HTTP → DB, auth, validation, tenant isolation
     /      \
    /--------\    Unit tests (most)         — Actions, DTOs, value objects, pure logic
```

### Configuration

```php
// tests/Pest.php
uses(Tests\TestCase::class, Illuminate\Foundation\Testing\RefreshDatabase::class)
    ->in('Feature');

uses(Tests\TestCase::class)->in('Unit');

// Shared helper
function actingAsOrgUser(Organization $org, array $permissions = []): User
{
    $user = User::factory()->for($org)->create();

    setPermissionsTeamId($org->id);
    foreach ($permissions as $permission) {
        Permission::findOrCreate($permission);
        $user->givePermissionTo($permission);
    }

    app(PermissionRegistrar::class)->forgetCachedPermissions();
    app(TenantContext::class)->setOrganizationId($org->id);

    test()->actingAs($user, 'api');

    return $user;
}

// Custom expectations
expect()->extend('toBeScopedTo', function (int $orgId) {
    expect($this->value->organization_id)->toBe($orgId);
    return $this;
});
```

### Unit test — pure domain logic

```php
it('rejects a deduction larger than available stock', function () {
    $item = new InventoryItem(['quantity' => 3]);

    expect(fn () => $item->deduct(5))
        ->toThrow(InsufficientStockException::class);
});

it('calculates low stock status', function (int $qty, int $reorder, bool $expected) {
    $item = new InventoryItem(['quantity' => $qty, 'reorder_point' => $reorder]);

    expect($item->isLowStock())->toBe($expected);
})->with([
    'at threshold'    => [10, 10, true],
    'below threshold' => [5, 10, true],
    'above threshold' => [50, 10, false],
    'zero stock'      => [0, 10, true],
    'no reorder set'  => [0, 0, true],
]);
```

### Feature test — the tenant isolation suite (your most important tests)

```php
describe('tenant isolation', function () {
    beforeEach(function () {
        $this->orgA = Organization::factory()->create();
        $this->orgB = Organization::factory()->create();
        $this->itemA = InventoryItem::factory()->for($this->orgA)->create();
        $this->itemB = InventoryItem::factory()->for($this->orgB)->create();
    });

    it('lists only the current organizations items', function () {
        actingAsOrgUser($this->orgA, ['inventory.view']);

        $this->getJson('/api/v1/inventory')
            ->assertOk()
            ->assertJsonCount(1, 'data')
            ->assertJsonPath('data.0.id', $this->itemA->id);
    });

    it('returns 404 for another organizations item', function () {
        actingAsOrgUser($this->orgA, ['inventory.view']);

        $this->getJson("/api/v1/inventory/{$this->itemB->id}")
            ->assertNotFound();          // NOT 403
    });

    it('cannot update another organizations item', function () {
        actingAsOrgUser($this->orgA, ['inventory.update']);

        $this->patchJson("/api/v1/inventory/{$this->itemB->id}", ['name' => 'hacked'])
            ->assertNotFound();

        expect($this->itemB->fresh()->name)->not->toBe('hacked');
    });

    it('cannot reference another organizations supplier', function () {
        actingAsOrgUser($this->orgA, ['inventory.create']);
        $foreignSupplier = Supplier::factory()->for($this->orgB)->create();

        $this->postJson('/api/v1/inventory', [
            'sku'         => 'NEW-1',
            'name'        => 'New item',
            'quantity'    => 5,
            'supplier_id' => $foreignSupplier->id,
        ])->assertStatus(422)
          ->assertJsonValidationErrors('supplier_id');
    });

    it('ignores a client-supplied organization_id', function () {
        actingAsOrgUser($this->orgA, ['inventory.create']);

        $this->postJson('/api/v1/inventory', [
            'sku'             => 'NEW-2',
            'name'            => 'New item',
            'quantity'        => 1,
            'organization_id' => $this->orgB->id,     // attempted forgery
        ])->assertCreated();

        expect(InventoryItem::withoutGlobalScopes()->where('sku', 'NEW-2')->first())
            ->toBeScopedTo($this->orgA->id);
    });

    it('does not leak cached data between tenants', function () {
        actingAsOrgUser($this->orgA, ['inventory.view']);
        $a = $this->getJson('/api/v1/dashboard')->json('data.item_count');

        actingAsOrgUser($this->orgB, ['inventory.view']);
        $b = $this->getJson('/api/v1/dashboard')->json('data.item_count');

        expect($a)->toBe(1)->and($b)->toBe(1);   // both have exactly 1 item, from their own org
    });
});
```

### Faking external systems

```php
// Queue
Queue::fake();
Queue::fake([SyncItemToSearchIndex::class]);            // fake only this job
Queue::assertPushed(SyncItemToSearchIndex::class, 1);
Queue::assertPushedOn('indexing', SyncItemToSearchIndex::class);
Queue::assertPushed(fn (SyncItemToSearchIndex $j) => $j->itemId === $item->id);
Queue::assertNothingPushed();

// Bus (chains and batches)
Bus::fake();
Bus::assertChained([ValidateImportFile::class, ParseImportFile::class]);
Bus::assertBatched(fn (PendingBatch $b) => $b->jobs->count() === 20);
Bus::assertDispatchedAfterResponse(CleanupJob::class);

// Events, Mail, Notifications
Event::fake([StockLevelChanged::class]);
Mail::fake(); Mail::assertQueued(LowStockDigest::class, fn ($m) => $m->hasTo($user->email));
Notification::fake();
Notification::assertSentTo($user, LowStockAlert::class, fn ($n, $channels) => in_array('mail', $channels));

// HTTP client
Http::preventStrayRequests();          // fail the test if any un-faked request escapes
Http::fake([
    'api.supplier.com/v1/prices*' => Http::response(['price' => 1299], 200),
    'api.supplier.com/v1/stock*'  => Http::sequence()
        ->push([], 500)
        ->push(['qty' => 5], 200),      // test retry behavior
    '*'                            => Http::response(null, 404),
]);
Http::assertSent(fn (Request $r) => $r->hasHeader('Authorization') && $r['sku'] === 'ABC');
Http::assertSentCount(2);

// Storage
Storage::fake('s3');
Storage::disk('s3')->assertExists("imports/{$uploadId}.csv");

// Time
$this->travelTo(now()->addDays(31));
$this->travel(5)->hours();
$this->freezeTime(function () { /* ... */ });
$this->travelBack();
```

### Mocking with care

```php
// Container mock — good for external boundaries
$this->mock(SupplierApiClient::class, function (MockInterface $mock) {
    $mock->shouldReceive('fetchPrice')
         ->once()
         ->with('SKU-1')
         ->andReturn(1299);
});

// Partial mock — keep real behavior except one method
$this->partialMock(InventoryService::class, fn ($m) => $m->shouldReceive('notifyWarehouse'));

// Spy — assert after the fact
$spy = $this->spy(AuditLogger::class);
// ... exercise code ...
$spy->shouldHaveReceived('log')->once();
```

> **Trap:** Over-mocking. If you mock the repository, the service, and the cache, your test proves only that your mocks were configured correctly — it can pass while the app is broken. **Mock at process boundaries** (HTTP APIs, payment gateways, email, S3). Use the real database (SQLite in-memory or a real Postgres in CI) for anything involving queries, because query bugs and scope bugs are exactly what you need to catch.

> **Follow-up:** *SQLite in-memory vs real Postgres in CI?* SQLite is fast but differs meaningfully: no `jsonb` operators, different `ILIKE`/case-sensitivity semantics, weaker constraint enforcement, no `FOR UPDATE SKIP LOCKED`, different transaction behavior. If you use Postgres in production — and you do — **run tests against Postgres in CI** (a Docker service container). Otherwise you'll ship migrations and queries that pass CI and fail in prod. Use SQLite only for fast local unit runs.

### `RefreshDatabase` vs `DatabaseTransactions` vs `DatabaseMigrations`

| Trait | Mechanism | Speed | Notes |
|-------|-----------|-------|-------|
| `RefreshDatabase` | Migrate once, then wrap each test in a transaction | Fast | Default choice |
| `DatabaseTransactions` | Wrap each test in a transaction (no migration) | Fastest | Assumes schema already exists |
| `DatabaseMigrations` | Full migrate + rollback per test | Very slow | Only when you must test migrations themselves |

> **Trap:** Transaction-wrapped tests break any code that relies on real commits — `DB::afterCommit()` callbacks, `$afterCommit` jobs, and `LOCK` behavior across connections. If you're testing `afterCommit` semantics, you need a non-transactional test (truncate-based cleanup) or `Bus::fake()` plus asserting the intent.

### Architecture tests (excellent senior signal)

```php
arch('domain layer does not depend on the framework')
    ->expect('App\Domain')
    ->not->toUse(['Illuminate\Http', 'Illuminate\Support\Facades']);

arch('controllers are final and thin')
    ->expect('App\Http\Controllers')
    ->toBeFinal()
    ->toHaveSuffix('Controller');

arch('actions expose a single entry point')
    ->expect('App\Domain\Actions')
    ->toHaveMethod('execute');

arch('no debug statements ship')
    ->expect(['dd', 'dump', 'ray', 'var_dump', 'die'])
    ->not->toBeUsed();

arch('DTOs are immutable')
    ->expect('App\Domain\Data')
    ->toBeReadonly();

arch()->preset()->security();     // Pest 3: bans md5, sha1, eval, extract, etc.
```

---

## 20. Tier 2 Q&A Drill

### Container & providers

1. **How does the container resolve a class it has no binding for?**  
   Reflection on the constructor: recursively resolve class-typed params, use defaults for primitives, throw `BindingResolutionException` if it can't. Interfaces must be bound explicitly.

2. **`bind` vs `singleton` vs `scoped`?**  
   `bind` = new instance per resolution. `singleton` = one per container lifetime. `scoped` = one per request/job, flushed by Octane between requests. Request-scoped state in a multi-tenant app must be `scoped`.

3. **Why is `singleton` dangerous under Octane?**  
   The instance survives across requests, so tenant A's state can leak into tenant B's request. Use `scoped` for anything request-specific.

4. **How do you give two classes different implementations of the same interface?**  
   Contextual binding: `$app->when(A::class)->needs(Interface::class)->give(...)`.

5. **How do you add caching to a repository without modifying it?**  
   `$app->extend()` with a decorator that implements the same interface and wraps the inner instance.

6. **What is the service locator anti-pattern and where does it appear in Laravel?**  
   Calling `app()`/`resolve()` inside domain classes to fetch dependencies. It hides the dependency graph, breaks static analysis, and requires a container to test. Restrict it to the framework boundary.

7. **What is a deferred provider and when should you avoid it?**  
   A provider loaded only when one of its `provides()` bindings is resolved, reducing bootstrap cost. Avoid if `boot()` must always run (routes, listeners, gates).

### Middleware & routing

8. **Why does middleware order matter for multi-tenancy?**  
   Tenant middleware needs `$request->user()` so it must run after `Authenticate`, and route model binding must be tenant-aware so it must run before `SubstituteBindings`. Wrong order = null tenant or unscoped bindings.

9. **What runs in `terminate()` and what's the caveat?**  
   Post-response work, via `fastcgi_finish_request` under php-fpm. Under Octane it still occupies the worker, so heavy work belongs in a queue.

10. **How would you implement an idempotent POST endpoint?**  
    Require an `Idempotency-Key` header, hash it with the route and tenant, take an atomic lock to prevent concurrent duplicates, store the response body/status on success, and replay the stored response on retry. Plus a unique DB index on the key as the durable guarantee.

### Eloquent

11. **`$item->movements` vs `$item->movements()` — what's the difference and why does it matter?**  
    Property returns a cached Collection (loaded once). Method returns a Builder — every call hits the DB. Aggregates via the method in a loop are an N+1 that `preventLazyLoading` won't catch.

12. **Name five distinct kinds of N+1 in a Laravel API.**  
    Lazy load in loop; nested relation; per-row aggregate via relation method; lazy load inside an API Resource; unconstrained `morphTo` eager load.

13. **How do you eliminate N+1 permanently rather than case by case?**  
    `Model::preventLazyLoading()` outside production, `whenLoaded()`/`whenCounted()` in resources, `withCount`/`withSum`/subquery selects instead of per-row queries, query-count assertions in feature tests as regression guards, and slow-query/query-budget logging in production.

14. **Why does `with(['movements' => fn($q) => $q->limit(5)])` not do what you expect?**  
    The limit applies to the single combined `WHERE IN` query, giving 5 rows total rather than 5 per parent. Use `latestOfMany()`, a window function, or per-relation limit support if your Laravel version provides it.

15. **What is a global scope and how does it fail?**  
    A query constraint applied automatically to every query on the model. It fails via raw `DB::table()` queries, console commands without tenant context, and forgotten scope removal. Make it **fail closed** (return nothing when there's no tenant in an HTTP context) rather than fail open.

16. **How do you prove every tenant-owned model is scoped?**  
    An architecture test that reflects over all models, skips those without an `organization_id` column, and asserts both that the trait is applied and that compiled SQL contains the scope.

17. **Which operations skip model events and why does it matter?**  
    `insert`, `upsert`, query-builder `update`/`delete`, `DB::table`, `saveQuietly`, `withoutEvents`. Anything relying on observers — audit logs, cache invalidation, search indexing — silently breaks.

18. **What's the transaction/observer race and how do you fix it?**  
    An observer dispatches a job inside a transaction; a worker picks it up before COMMIT and can't see the row. Fix with `$afterCommit = true` on the job, `after_commit` on the queue connection, or `DB::afterCommit()`.

19. **`chunk` vs `chunkById` vs `cursor` vs `lazyById` — pick one for a 15M-row backfill and justify it.**  
    `lazyById`: keyset pagination so mutating the filtered column can't skip rows, and only one model resident at a time. Add throttling and a resumable checkpoint.

20. **Offset vs cursor pagination?**  
    Offset supports page numbers and totals but is O(offset) and unstable under concurrent inserts. Cursor is O(1) at any depth and stable, but no page jumping and requires a unique ordered column. Cursor for infinite scroll over large tables; offset for bounded admin tables.

21. **Why is `toBase()` faster than `get()`?**  
    It skips Eloquent hydration — no casting, no relation containers, no event plumbing. For large read-only reports, hydration often dominates.

22. **Show the `orWhere` bug that leaks tenant data.**  
    `where('organization_id', 5)->where('sku','like',...)->orWhere('name','like',...)` compiles to `... AND ... OR ...` which matches other tenants' rows. Wrap the OR in a closure.

### Queues

23. **Why must `retry_after` exceed the job `timeout`?**  
    Otherwise the queue releases the job to another worker while the first is still executing it, so it runs twice concurrently.

24. **What's the deploy trap with `queue:work`?**  
    Workers hold the old code in memory. You must `queue:restart` (or roll worker containers) on every deploy or jobs run stale code.

25. **Why should you never serialize a full model into a job?**  
    `SerializesModels` stores the key and re-fetches on execution; if the row is deleted you get `ModelNotFoundException`. Passing IDs explicitly makes the contract clear and keeps payloads small. Also, re-fetching gets fresh data instead of a stale snapshot.

26. **Queue delivery guarantee?**  
    At-least-once. Jobs can run twice (crash after work before ack, visibility timeout, misconfigured `retry_after`). Handlers must be idempotent, ideally enforced by a unique index on an idempotency key.

27. **`ShouldBeUnique` vs `WithoutOverlapping`?**  
    `ShouldBeUnique` prevents duplicate jobs from being **queued**. `WithoutOverlapping` prevents duplicate **concurrent execution** (the job stays queued and gets released). Both need an atomic shared store.

28. **Batch vs chain?**  
    Batch = parallel independent jobs with aggregate progress and completion callbacks. Chain = strictly ordered, stop-on-failure. Combine for import pipelines with a parallel middle stage.

29. **Redis vs SQS for queues?**  
    Redis + Horizon for in-app work (metrics, unique jobs, batching, sub-ms latency). SQS when producer and consumer are different services, you need AWS-native fan-out with SNS, or you want managed durability at very high scale. SQS has no Horizon and a 256 KB payload limit.

30. **A job needs tenant context — what must it do?**  
    Carry `organization_id` in its payload and set both `TenantContext` and `setPermissionsTeamId()` at the start of `handle()`. There's no middleware to do it for you.

### Caching

31. **What is a cache stampede and how do you prevent it?**  
    A hot key expires and every concurrent request recomputes it, hammering the DB. Fixes: lock so one request rebuilds while others wait or serve stale; stale-while-revalidate (`Cache::flexible`); TTL jitter; scheduled cache warming.

32. **What's the most common multi-tenant caching bug?**  
    An unnamespaced key serving one tenant's data to another. Every key must include the tenant ID, and every cached endpoint needs a two-tenant isolation test.

33. **Why won't `Cache::lock()` work with the `file` driver across servers?**  
    Locks need an atomic shared store. File/array stores are per-process/per-server, so mutual exclusion doesn't exist.

34. **Alternatives to tag-based invalidation?**  
    Key versioning: embed a version number in the key and `increment` it to invalidate atomically. Works on any store (tags require Redis/Memcached) and avoids scanning.

35. **Is Redis safe as the source of truth for stock levels?**  
    No — not durable by default, and failover can lose recent writes. Use it as a reservation/rate-limit/read-cache layer; keep the authoritative ledger in Postgres.

### Auth & authorization

36. **Sanctum vs Passport — when each?**  
    Sanctum for first-party SPA/mobile and simple API tokens: opaque hashed tokens, instant revocation, trivial setup. Passport when you need a real OAuth 2.0 server for third-party clients, refresh tokens, and standard grants.

37. **Why are the implicit and password grants deprecated?**  
    Implicit exposes tokens in URLs (history, referrer, logs) with no client auth. Password grant requires handling raw user credentials, defeats MFA/federation, and encourages credential storage. Use authorization code + PKCE.

38. **How do you revoke a JWT immediately?**  
    You can't purely statelessly. Short TTLs to bound the window, a revocation check against stored token state (what Passport does), or a Redis denylist of `jti` until expiry. Name the statelessness-vs-revocation trade-off explicitly.

39. **In spatie teams mode, what breaks in queued jobs?**  
    No middleware sets the team ID, so permission checks read a stale or missing team context and return wrong answers. Call `setPermissionsTeamId()` in `handle()`.

40. **Why check permissions rather than roles in code?**  
    Roles are admin-editable bundles; permissions are the actual capabilities. Hardcoding `hasRole('manager')` turns every "who can do X" change into a deploy.

41. **Policy vs spatie permission — redundant?**  
    No. spatie answers "does the user hold this capability?"; the policy answers "for this record, in this tenant, in this state?" You need both.

42. **Why does `Gate::before` deserve scrutiny in a multi-tenant SaaS?**  
    Returning `true` bypasses every policy including tenant boundaries, so a super-admin flag becomes cross-tenant read access. Require explicit, audited impersonation instead.

### API & testing

43. **What status code for cross-tenant access, and why?**  
    404 — a 403 confirms the resource exists and enables enumeration.

44. **What status for insufficient stock?**  
    409 Conflict — the request is well-formed and authorized but conflicts with current state. Not 422 (that's payload validity) and not 400 (that's malformed).

45. **Two things every paginated list endpoint must do?**  
    Clamp page size to a maximum, and include a deterministic tiebreaker in `ORDER BY` so rows don't repeat or vanish across pages.

46. **Where should you mock, and where shouldn't you?**  
    Mock at process boundaries: HTTP APIs, payment gateways, mail, S3. Don't mock the database or your own repositories in feature tests — query and scope bugs are precisely what you need to catch.

47. **SQLite in-memory vs Postgres in CI?**  
    Run Postgres in CI if you run Postgres in prod. SQLite differs on `jsonb`, `ILIKE`, constraints, `FOR UPDATE SKIP LOCKED`, and transaction semantics, so it hides real bugs.

48. **What breaks under `RefreshDatabase`?**  
    Anything depending on a real commit: `DB::afterCommit`, `$afterCommit` jobs, cross-connection lock visibility. Test those without transaction wrapping or assert intent with `Bus::fake()`.

49. **How do you write a test that prevents N+1 regressions?**  
    Enable `Model::preventLazyLoading()` in the test environment, and add a query-count assertion around the endpoint so the test fails when someone introduces extra queries.

50. **Give three architecture tests worth having.**  
    Domain layer must not import `Illuminate\Http` or facades; DTOs must be readonly; `dd`/`dump`/`var_dump` must never be used. Add one asserting every tenant-owned model uses the tenant trait.

---

**Next:** [`03-senior.md`](./03-senior.md) — multi-tenancy architecture, concurrency and isolation levels, performance engineering, zero-downtime migrations, DDD/CQRS, scaling, observability, and security.
