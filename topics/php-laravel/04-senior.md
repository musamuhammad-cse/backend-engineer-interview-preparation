# PHP / Laravel — Tier 3: Senior (Architecture, Concurrency, Scale & Operations)

> This tier is about **trade-offs and failure modes**, not features. Senior interviewers are testing whether you've operated systems under pressure, not whether you can name a design pattern. Every section here ties back to your real work: multi-tenant SaaS, 88% query reduction, 15M-row zero-downtime migration, 20K DAU trading platform.

---

## Table of Contents

1. [Multi-Tenancy Architecture](#1-multi-tenancy-architecture)
2. [spatie/laravel-permission Teams — Internals & Pitfalls](#2-spatielaravel-permission-teams--internals--pitfalls)
3. [Passport & OAuth 2.0 Deep Dive](#3-passport--oauth-20-deep-dive)
4. [Concurrency & Race Conditions](#4-concurrency--race-conditions)
5. [Transaction Isolation Levels](#5-transaction-isolation-levels)
6. [Deadlocks](#6-deadlocks)
7. [Performance Engineering — The 88% Story](#7-performance-engineering--the-88-story)
8. [Indexing Strategy](#8-indexing-strategy)
9. [Zero-Downtime Migrations — The 15M Story](#9-zero-downtime-migrations--the-15m-story)
10. [Read Replicas & Connection Pooling](#10-read-replicas--connection-pooling)
11. [Clean Architecture & DDD in Laravel](#11-clean-architecture--ddd-in-laravel)
12. [CQRS, Event Sourcing & the Outbox Pattern](#12-cqrs-event-sourcing--the-outbox-pattern)
13. [SOLID at Architectural Scale](#13-solid-at-architectural-scale)
14. [Design Patterns in the Laravel Source](#14-design-patterns-in-the-laravel-source)
15. [Scaling & Deployment](#15-scaling--deployment)
16. [Laravel Octane & Long-Lived Processes](#16-laravel-octane--long-lived-processes)
17. [Observability](#17-observability)
18. [Security — OWASP Mapped to Laravel](#18-security--owasp-mapped-to-laravel)
19. [PHP Internals for Seniors](#19-php-internals-for-seniors)
20. [Tier 3 Q&A Drill](#20-tier-3-qa-drill)

---

## 1. Multi-Tenancy Architecture

### The three models

| | Row-level (`organization_id`) | Schema-per-tenant | Database-per-tenant |
|---|---|---|---|
| Isolation | Logical (app-enforced) | Strong (DB-enforced) | Strongest |
| Cost per tenant | Near zero | Low-medium | High |
| Migrations | One run, all tenants | N runs (one per schema) | N runs |
| Noisy neighbor | Yes — shared tables/indexes | Partial | No |
| Per-tenant backup/restore | Hard (row filtering) | Moderate | Trivial |
| Cross-tenant analytics | Trivial | Needs UNION across schemas | Needs ETL |
| Connection overhead | One pool | One pool (`SET search_path`) | N pools ⚠️ |
| Max practical tenants | Millions | Thousands (Postgres degrades past ~10k schemas) | Hundreds |
| Blast radius of a bug | **All tenants** ⚠️ | One tenant | One tenant |
| Your SaaS | ✅ | | |

**How to present your choice in an interview:** "We chose row-level because we needed thousands of small tenants with near-zero marginal cost, single-run migrations, and cross-tenant analytics. The trade-off is that isolation is application-enforced, so a single missing `WHERE` clause is a data breach across all customers. We compensated with defense in depth: global scopes that fail closed, tenant-stamped writes, a `saving` guard, composite unique indexes, tenant-namespaced cache keys, and an isolation test suite per model. For enterprise customers with regulatory requirements we'd offer database-per-tenant as a premium tier."

That answer shows you understand it's a **business trade-off**, not just a technical one.

### Complete implementation

#### 1. Tenant context — request-scoped, never singleton

```php
namespace App\Support\Tenancy;

final class TenantContext
{
    private ?int $organizationId = null;
    private bool $bypassed = false;

    public function setOrganizationId(?int $id): void
    {
        $this->organizationId = $id;
    }

    public function organizationId(): ?int
    {
        return $this->bypassed ? null : $this->organizationId;
    }

    public function organizationIdOrFail(): int
    {
        return $this->organizationId
            ?? throw new MissingTenantContextException(
                'Tenant context is required but was not set.'
            );
    }

    public function hasTenant(): bool
    {
        return $this->organizationId !== null;
    }

    /**
     * Explicit, auditable escape hatch for cross-tenant admin work.
     * Deliberately verbose so it shows up in code review and grep.
     */
    public function runWithoutTenancy(Closure $callback): mixed
    {
        $previous = $this->bypassed;
        $this->bypassed = true;

        Log::withContext(['tenancy_bypassed' => true]);

        try {
            return $callback();
        } finally {
            $this->bypassed = $previous;
        }
    }

    public function runAs(int $organizationId, Closure $callback): mixed
    {
        $previous = $this->organizationId;
        $this->organizationId = $organizationId;

        try {
            return $callback();
        } finally {
            $this->organizationId = $previous;
        }
    }
}
```

```php
// Provider — scoped(), NOT singleton(). Critical for Octane.
$this->app->scoped(TenantContext::class);
```

#### 2. Middleware

```php
final class SetTenantContext
{
    public function __construct(private readonly TenantContext $tenant) {}

    public function handle(Request $request, Closure $next): Response
    {
        $user = $request->user();

        if (! $user) {
            throw new AuthenticationException();
        }

        $orgId = $this->resolveOrganizationId($request, $user);

        $this->tenant->setOrganizationId($orgId);
        setPermissionsTeamId($orgId);

        // Every log line and error report is now tenant-attributed
        Log::withContext(['organization_id' => $orgId, 'user_id' => $user->id]);
        Sentry::configureScope(fn (Scope $s) => $s->setTag('organization_id', (string) $orgId));

        return $next($request);
    }

    private function resolveOrganizationId(Request $request, Authenticatable $user): int
    {
        // Support users belonging to multiple orgs via an explicit header
        if ($header = $request->header('X-Organization-Id')) {
            $orgId = (int) $header;

            abort_unless(
                $user->organizations()->whereKey($orgId)->exists(),
                404          // not 403 — don't confirm the org exists
            );

            return $orgId;
        }

        return $user->organization_id
            ?? throw new MissingTenantContextException('User has no organization.');
    }
}
```

#### 3. Global scope that fails closed

```php
final class OrganizationScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $tenant = app(TenantContext::class);

        if (! $tenant->hasTenant()) {
            // Console: allow (migrations, backfills, cross-tenant reports)
            if (app()->runningInConsole()) {
                return;
            }

            // HTTP with no tenant is a bug. Return NOTHING rather than everything.
            Log::error('Tenant-scoped query executed without tenant context', [
                'model' => $model::class,
            ]);

            $builder->whereRaw('1 = 0');
            return;
        }

        $builder->where(
            $model->qualifyColumn('organization_id'),
            $tenant->organizationIdOrFail()
        );
    }
}
```

> **The "fail closed vs fail open" decision is a strong senior talking point.** Failing open (no constraint when context is missing) turns a forgotten middleware into a full cross-tenant data dump. Failing closed turns the same bug into an obvious empty-list bug that gets caught in QA.

#### 4. The trait with write-side guards

```php
trait BelongsToOrganization
{
    public static function bootBelongsToOrganization(): void
    {
        static::addGlobalScope(new OrganizationScope());

        static::creating(function (Model $model): void {
            $tenant = app(TenantContext::class);

            if ($model->organization_id === null && $tenant->hasTenant()) {
                $model->organization_id = $tenant->organizationIdOrFail();
            }

            if ($model->organization_id === null) {
                throw new MissingTenantContextException(
                    'Cannot create ' . $model::class . ' without an organization.'
                );
            }
        });

        static::saving(function (Model $model): void {
            $tenant = app(TenantContext::class);

            if ($tenant->hasTenant()
                && (int) $model->organization_id !== $tenant->organizationIdOrFail()) {
                throw new TenantMismatchException(sprintf(
                    'Refusing to write %s for org %d while acting as org %d.',
                    $model::class,
                    $model->organization_id,
                    $tenant->organizationIdOrFail(),
                ));
            }
        });
    }

    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }

    public function scopeForOrganization(Builder $q, int $orgId): Builder
    {
        return $q->withoutGlobalScope(OrganizationScope::class)
                 ->where('organization_id', $orgId);
    }
}
```

#### 5. Schema rules

```php
Schema::create('inventory_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
    $table->foreignId('supplier_id')->nullable()->constrained()->nullOnDelete();
    $table->string('sku', 64);
    // ...

    // 1. Uniqueness is ALWAYS tenant-scoped
    $table->unique(['organization_id', 'sku']);

    // 2. Every query-supporting index leads with organization_id
    $table->index(['organization_id', 'status']);
    $table->index(['organization_id', 'created_at']);
});
```

> **Trap — the cross-tenant foreign key.** A plain `foreign key (supplier_id) references suppliers(id)` allows an item in Org A to reference a supplier in Org B. The DB can't see the tenant relationship. Two defenses:

```sql
-- Defense A: composite FK — makes cross-tenant references structurally impossible
ALTER TABLE suppliers ADD CONSTRAINT suppliers_org_id_unique UNIQUE (organization_id, id);

ALTER TABLE inventory_items
  ADD CONSTRAINT items_supplier_same_org
  FOREIGN KEY (organization_id, supplier_id)
  REFERENCES suppliers (organization_id, id);
```

```php
// Defense B: tenant-scoped exists validation (app level, always do this too)
'supplier_id' => [
    'nullable',
    Rule::exists('suppliers', 'id')->where('organization_id', $orgId),
],
```

### The complete leak-vector checklist

Memorize this list. "Name the ways row-level multi-tenancy leaks" is a top-tier senior question and most candidates name two or three.

| # | Vector | Example | Defense |
|---|--------|---------|---------|
| 1 | Raw queries | `DB::table('items')->get()` | Ban raw queries on tenant tables; static analysis rule; code review |
| 2 | Global scope removed | `withoutGlobalScopes()` left in | Grep in CI; require an explicit, logged bypass API |
| 3 | Route model binding | `findOrFail($id)` unscoped | Verified scope + `resolveRouteBinding` override + 404 test per resource |
| 4 | `orWhere` ungrouped | `where(org)->where(a)->orWhere(b)` | Always wrap OR groups in a closure; lint for it |
| 5 | Unnamespaced cache keys | `Cache::remember('stats')` | Tenant prefix on every key; two-tenant cache test |
| 6 | Queued jobs | Job runs with no tenant context | Carry `organization_id` in the payload; set context in `handle()` |
| 7 | Scheduled commands | Cron has no HTTP context | Loop tenants explicitly with `runAs()` |
| 8 | Cross-tenant FK | Item references another org's supplier | Composite FK + scoped `exists` validation |
| 9 | Validation `unique`/`exists` | Global uniqueness across tenants | `Rule::unique(...)->where('organization_id', $orgId)` |
| 10 | Search index | Elasticsearch doc without tenant filter | Tenant field in every doc + mandatory filter in every query, or index-per-tenant |
| 11 | File storage | `storage/exports/report.csv` collision | Path includes tenant: `exports/{orgId}/{uuid}.csv` |
| 12 | Broadcasting | Public channel per resource | Private channels authorized against tenant |
| 13 | Aggregate/report endpoints | `SUM()` across all rows | Reports go through the same scoped repository |
| 14 | Super-admin bypass | `Gate::before` returns true | Explicit impersonation with audit trail, time-boxed |
| 15 | Error messages | "SKU already exists" reveals another tenant's data | Tenant-scoped uniqueness makes this impossible |
| 16 | Octane singleton | Tenant context persists across requests | `scoped()` bindings; Octane request-flush test |

### Tenant-aware background work

```php
// Scheduled command that must run per tenant
final class SendDailyDigests extends Command
{
    protected $signature = 'digests:send';

    public function handle(TenantContext $tenant): int
    {
        Organization::active()->cursor()->each(function (Organization $org) use ($tenant) {
            $tenant->runAs($org->id, function () use ($org) {
                setPermissionsTeamId($org->id);

                // Isolate failures: one tenant's bad data must not stop the rest
                try {
                    SendOrganizationDigest::dispatch($org->id);
                } catch (Throwable $e) {
                    report($e);
                    $this->error("Digest failed for org {$org->id}");
                }
            });
        });

        return self::SUCCESS;
    }
}
```

```php
// Base job that enforces tenant context — no job author can forget
abstract class TenantAwareJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public readonly int $organizationId) {}

    final public function handle(): void
    {
        app(TenantContext::class)->setOrganizationId($this->organizationId);
        setPermissionsTeamId($this->organizationId);

        Log::withContext(['organization_id' => $this->organizationId]);

        $this->handleForTenant();
    }

    abstract protected function handleForTenant(): void;

    public function tags(): array
    {
        return ['org:' . $this->organizationId, static::class];
    }
}
```

> **Follow-up:** *How do you handle noisy neighbors?* Per-tenant rate limits keyed on `organization_id`; dedicated queues (or Horizon supervisors) for large tenants; per-tenant query budgets with slow-query alerting tagged by org; connection pool limits; and for the largest tenants, an isolated shard or dedicated database. Also: partition huge tables by `organization_id` (Postgres declarative partitioning) so one tenant's data volume doesn't degrade everyone's index scans.

> **Follow-up:** *How do you delete a tenant (GDPR right to erasure)?* Cascade deletes via FK, then a verification query per table proving zero remaining rows, plus purging derived stores: search index documents, cache keys under the tenant namespace, S3 prefix, queued jobs referencing the org, log retention, and backups (document the backup retention window in your DPA since you can't surgically edit backups). Emit a deletion certificate/audit record. Being able to enumerate the *derived* stores is what separates a real answer from a textbook one.

---

## 2. spatie/laravel-permission Teams — Internals & Pitfalls

### Schema in teams mode

```
permissions            (id, name, guard_name)                       ← GLOBAL, not per team
roles                  (id, name, guard_name, organization_id)      ← per team
role_has_permissions   (permission_id, role_id)
model_has_roles        (role_id, model_type, model_id, organization_id)
model_has_permissions  (permission_id, model_type, model_id, organization_id)
```

Key insight: **permissions are global; roles and assignments are team-scoped.** So `inventory.update` exists once, but "Alice is a manager in Org 7" is a team-scoped row.

### The registrar and the cache

```php
$registrar = app(PermissionRegistrar::class);

$registrar->setPermissionsTeamId($orgId);      // MUST happen before any check
$registrar->getPermissionsTeamId();
$registrar->forgetCachedPermissions();          // after any role/permission mutation
```

spatie caches the entire permissions+roles graph under one cache key (default 24h TTL). The cache is **not** team-partitioned — team filtering happens in PHP after the cached collection is loaded. That's why forgetting `setPermissionsTeamId()` gives wrong answers rather than errors.

### The five pitfalls

**1. Queued jobs and commands have no middleware.**

```php
// WRONG — team id is whatever the worker last processed, or null
public function handle(): void
{
    if ($this->user->can('inventory.delete')) { /* ... */ }
}

// RIGHT
public function handle(): void
{
    setPermissionsTeamId($this->organizationId);
    app(PermissionRegistrar::class)->forgetCachedPermissions();  // if roles may have changed

    if ($this->user->can('inventory.delete')) { /* ... */ }
}
```

**2. Test pollution via the permission cache.**

```php
// tests/Pest.php
beforeEach(function () {
    app(PermissionRegistrar::class)->forgetCachedPermissions();
});
```

Without this you get tests that pass alone and fail in a suite — the worst kind of flake.

**3. Cache invalidation across servers.** With the `file` cache driver, invalidating permissions on server A leaves server B serving stale authorization for up to 24h. **Use Redis** for the permission cache in any multi-server deployment.

**4. N+1 on permission checks.**

```php
// Each ->can() may hydrate roles/permissions if not cached
$users->each(fn ($u) => $u->can('inventory.view'));

// Eager load
$users = User::with('roles.permissions', 'permissions')->get();
```

**5. Role name collisions across teams.** Two orgs can both have a role named `admin` — they're different rows with different `organization_id`. `Role::findByName('admin')` without a team context is ambiguous. Always pass the team: `Role::findByName('admin', 'api', $orgId)` or set the team id first.

### Modeling advice

```php
// Check permissions, not roles
$user->can('inventory.update');                 // ✅ admin-configurable
$user->hasRole('manager');                       // ❌ hardcodes policy in code

// Permission naming convention: resource.action
'inventory.view', 'inventory.create', 'inventory.update', 'inventory.delete',
'inventory.view_cost', 'reports.export', 'users.invite', 'billing.manage',
```

> **Follow-up:** *A customer wants a custom role "Warehouse Auditor" that can view inventory and cost but not edit. How long does that take?* With permission-based checks: zero deploys — the org admin creates a role in the UI and assigns permissions. With role-based checks: a code change, review, and deploy. That's the business value of the distinction, and it's exactly the framing a senior interviewer wants.

---

## 3. Passport & OAuth 2.0 Deep Dive

### The roles

| Role | Who | Example |
|------|-----|---------|
| Resource Owner | The user | Your customer |
| Client | The app requesting access | A partner's integration |
| Authorization Server | Issues tokens | Your Laravel + Passport app |
| Resource Server | Serves protected data | Your Laravel API |

### Authorization Code + PKCE, in detail

```
┌────────┐                                     ┌──────────────┐
│ Client │                                     │ Auth Server  │
└───┬────┘                                     └──────┬───────┘
    │ 1. verifier = random(43-128 chars)              │
    │    challenge = BASE64URL(SHA256(verifier))      │
    │                                                  │
    │ 2. GET /oauth/authorize                          │
    │    ?response_type=code                           │
    │    &client_id=X                                  │
    │    &redirect_uri=https://app/cb                  │
    │    &scope=inventory:read                         │
    │    &state=RANDOM            ← CSRF protection    │
    │    &code_challenge=CHALLENGE                     │
    │    &code_challenge_method=S256                   │
    ├─────────────────────────────────────────────────►│
    │                                        3. user logs in + consents
    │ 4. 302 → https://app/cb?code=AUTH_CODE&state=RANDOM
    │◄─────────────────────────────────────────────────┤
    │ 5. verify state matches what we sent             │
    │                                                  │
    │ 6. POST /oauth/token                             │
    │    grant_type=authorization_code                 │
    │    code=AUTH_CODE                                │
    │    redirect_uri=https://app/cb                   │
    │    client_id=X                                   │
    │    code_verifier=VERIFIER   ← proves same client │
    ├─────────────────────────────────────────────────►│
    │                              7. SHA256(verifier) == stored challenge?
    │ 8. { access_token, refresh_token, expires_in }   │
    │◄─────────────────────────────────────────────────┤
```

**What PKCE defends against:** an attacker who intercepts the authorization code (via a malicious app registering the same custom URL scheme on mobile, or a redirect leak) still can't exchange it, because they don't have the `code_verifier`.

**What `state` defends against:** CSRF — an attacker tricking a victim's browser into completing an authorization flow the attacker initiated.

### Configuration for your SaaS

```php
// AppServiceProvider::boot()
Passport::tokensCan([
    'inventory:read'  => 'View inventory',
    'inventory:write' => 'Modify inventory',
    'orders:read'     => 'View orders',
    'orders:write'    => 'Create and modify orders',
    'reports:read'    => 'Access reports',
]);

Passport::setDefaultScope(['inventory:read']);

Passport::tokensExpireIn(now()->addMinutes(15));          // short — bounds revocation window
Passport::refreshTokensExpireIn(now()->addDays(30));
Passport::personalAccessTokensExpireIn(now()->addMonths(3));

Passport::hashClientSecrets();                             // store bcrypt, not plaintext
Passport::enablePasswordGrant();                           // ← don't, unless legacy forces it
```

```bash
php artisan passport:keys                    # generates oauth-private.key / oauth-public.key
php artisan passport:client --client         # client credentials (machine-to-machine)
php artisan passport:client --public         # public client for PKCE (no secret)
php artisan passport:purge                   # remove revoked/expired tokens
```

### Scopes vs permissions — they answer different questions

```php
Route::middleware(['auth:api', 'scopes:inventory:write'])
    ->post('/inventory/{item}/deduct', DeductStockController::class);
```

- **Scope** = what the *client application* was authorized to do with this token. ("This partner's integration may write inventory.")
- **Permission** (spatie) = what the *user* is allowed to do. ("Alice may update inventory in Org 7.")

**You need both, and the effective permission is the intersection.** A token with `inventory:write` used by a user who lacks `inventory.update` must be denied. Conversely an admin using a read-only token must be denied writes.

```php
public function authorize(): bool
{
    return $this->user()->tokenCan('inventory:write')          // client scope
        && $this->user()->can('update', $this->route('item')); // user permission + tenant
}
```

> **This intersection point is a high-value senior answer.** Most candidates conflate scopes and permissions.

### Revocation and the JWT trade-off

Passport access tokens are RSA-signed JWTs, but Passport *does* check `oauth_access_tokens.revoked` on each request — so it's not stateless in practice. That's the deliberate trade-off: a DB lookup per request in exchange for instant revocation.

```php
// Revoke on logout
$request->user()->token()->revoke();

// Revoke everything for a user (e.g. after password change or suspected compromise)
$user->tokens()->update(['revoked' => true]);
DB::table('oauth_refresh_tokens')
    ->whereIn('access_token_id', $user->tokens()->pluck('id'))
    ->update(['revoked' => true]);
```

| Need | Approach |
|------|----------|
| Instant revocation | DB/Redis check per request (Passport default) |
| Truly stateless | Short TTL (5–15 min) + accept the revocation window |
| Middle ground | Redis denylist of `jti` values, TTL = token remaining lifetime |

### Key rotation

```php
// Rotating oauth keys invalidates every existing access token.
// Plan: 1) generate new keypair, 2) support verifying with BOTH keys during a
// transition window, 3) issue new tokens with the new key, 4) retire the old key
// after max token TTL has elapsed.
```

> **Follow-up:** *Why did you pick Passport over Sanctum for your SaaS?* Because the product exposes a public API to third-party integrators (ERP connectors, marketplace sync). That needs standard OAuth grants, per-client scopes, refresh tokens, and a consent screen — which is exactly Passport's job. For your own first-party dashboard SPA, Sanctum's cookie mode would be simpler and safer. **A strong answer says you'd use both: Sanctum for first-party, Passport for third-party.**

---

## 4. Concurrency & Race Conditions

This is the highest-signal section for a senior backend interview. Inventory and trading are the canonical domains, and you've worked in both.

### The lost update

```
Time  Request A                      Request B                     DB quantity
────────────────────────────────────────────────────────────────────────────
t0    SELECT quantity → 1                                          1
t1                                   SELECT quantity → 1           1
t2    check 1 >= 1 ✓                                               1
t3                                   check 1 >= 1 ✓                1
t4    UPDATE SET quantity = 0                                      0
t5                                   UPDATE SET quantity = 0       0
────────────────────────────────────────────────────────────────────────────
Result: TWO orders fulfilled from ONE unit. Oversold.
```

```php
// The vulnerable code — looks completely reasonable
$item = InventoryItem::findOrFail($id);

if ($item->quantity < $amount) {
    throw new InsufficientStockException();
}

$item->quantity -= $amount;
$item->save();
```

### Solution 1 — Atomic conditional UPDATE (best for simple counters)

```php
$affected = InventoryItem::query()
    ->where('id', $itemId)
    ->where('organization_id', $orgId)
    ->where('quantity', '>=', $amount)          // the guard is IN the UPDATE
    ->decrement('quantity', $amount);

if ($affected === 0) {
    throw new InsufficientStockException();      // either missing or insufficient
}
```

Compiles to one statement:

```sql
UPDATE inventory_items
SET quantity = quantity - 5, updated_at = NOW()
WHERE id = 42 AND organization_id = 7 AND quantity >= 5;
```

The database applies a row lock for the duration of the UPDATE, so the check and the write are a single atomic operation. **No race window exists.**

- ✅ Fastest, no explicit transaction needed, highest concurrency
- ❌ Can't do complex multi-step logic; can't easily distinguish "not found" from "insufficient" without an extra query; skips model events

### Solution 2 — Pessimistic locking (`SELECT ... FOR UPDATE`)

```php
return DB::transaction(function () use ($itemId, $amount, $orgId) {
    $item = InventoryItem::query()
        ->where('id', $itemId)
        ->where('organization_id', $orgId)
        ->lockForUpdate()                        // SELECT ... FOR UPDATE
        ->firstOrFail();

    if ($item->quantity < $amount) {
        throw new InsufficientStockException($itemId, $amount, $item->quantity);
    }

    $item->decrement('quantity', $amount);

    StockMovement::create([
        'organization_id'   => $orgId,
        'inventory_item_id' => $itemId,
        'type'              => MovementType::Outbound,
        'quantity'          => $amount,
    ]);

    return $item->refresh();
}, attempts: 3);
```

The `FOR UPDATE` lock is held until COMMIT. Concurrent transactions touching the same row **block** until then.

- ✅ Handles arbitrary multi-step logic safely; clear error messages; model events fire
- ❌ Serializes access to the row (throughput ceiling); lock held for the whole transaction; deadlock risk if lock order is inconsistent

> **Critical rule:** Never do network I/O inside a `lockForUpdate` transaction. An HTTP call to a payment gateway that hangs for 30s holds the row lock for 30s, and every other buyer blocks.

### Solution 3 — Optimistic locking (version column)

```php
// Migration: $table->unsignedBigInteger('version')->default(0);

$attempts = 0;

do {
    $item = InventoryItem::findOrFail($itemId);

    if ($item->quantity < $amount) {
        throw new InsufficientStockException();
    }

    $affected = InventoryItem::query()
        ->where('id', $itemId)
        ->where('version', $item->version)        // nobody changed it since we read
        ->update([
            'quantity' => $item->quantity - $amount,
            'version'  => $item->version + 1,
        ]);

    if ($affected === 1) {
        return;                                    // success
    }

    usleep(random_int(10_000, 50_000));            // jittered backoff
} while (++$attempts < 3);

throw new ConcurrentModificationException('Item was modified concurrently; please retry.');
```

- ✅ No locks held → excellent throughput under **low** contention; works across requests (a user editing a form for 5 minutes)
- ❌ Wasted work and retries under **high** contention; needs retry logic everywhere

> **Follow-up:** *When optimistic vs pessimistic?* Optimistic when conflicts are rare and the work is cheap to redo (editing a product description). Pessimistic when conflicts are common and correctness is critical with expensive work (deducting the last unit of hot stock during a flash sale). **Contention rate is the deciding factor, not preference.**

> **Optimistic locking also solves the "lost edit" UX problem:** two admins open the same item, both save — without a version column, the second silently overwrites the first. With it, the second gets a 409 and can be shown a diff.

### Solution 4 — Reservation ledger (append-only, no hot row)

The most scalable approach: never update a shared counter. Insert immutable rows and derive the balance.

```php
DB::transaction(function () use ($itemId, $amount, $orgId, $orderId) {
    // Serialize per item with an advisory lock (no row lock held on the item table)
    DB::select('SELECT pg_advisory_xact_lock(?, ?)', [
        crc32('inventory'),
        $itemId,
    ]);

    $available = DB::table('stock_movements')
        ->where('inventory_item_id', $itemId)
        ->sum(DB::raw('quantity * direction'));

    if ($available < $amount) {
        throw new InsufficientStockException();
    }

    DB::table('stock_movements')->insert([
        'organization_id'   => $orgId,
        'inventory_item_id' => $itemId,
        'quantity'          => $amount,
        'direction'         => -1,
        'reference_type'    => 'order',
        'reference_id'      => $orderId,
        'idempotency_key'   => $idempotencyKey,     // UNIQUE index
        'created_at'        => now(),
    ]);
});
```

Maintain a materialized `quantity` on the item via a trigger or periodic rollup for fast reads, but treat the ledger as truth.

- ✅ Full audit trail (essential for inventory and finance), no lost updates, easy reconciliation, idempotency key fits naturally
- ❌ Reads need aggregation or a maintained rollup; more moving parts

**This is what real inventory and trading systems do.** Mentioning double-entry/ledger design is strong senior signal.

### Solution 5 — Postgres advisory locks

```php
// Transaction-scoped: released automatically at COMMIT/ROLLBACK. Prefer this.
DB::select('SELECT pg_advisory_xact_lock(?)', [$lockKey]);

// Non-blocking attempt
$acquired = DB::selectOne('SELECT pg_try_advisory_xact_lock(?) AS locked', [$lockKey])->locked;

if (! $acquired) {
    throw new ResourceBusyException();
}
```

Advisory locks are application-defined locks living in the DB. They serialize logical operations without locking table rows — useful when the thing you're protecting isn't a single row (e.g. "only one import per organization at a time").

> **Trap:** `pg_advisory_lock()` (session-scoped) leaks if you forget to unlock or the connection is reused from a pool. Always prefer `pg_advisory_xact_lock()`.

### Solution 6 — Redis distributed lock (and why to be careful)

```php
$lock = Cache::lock("stock:{$itemId}", 10);

if (! $lock->get()) {
    throw new ResourceBusyException();
}

try {
    // ...
} finally {
    $lock->release();
}
```

> **Trap — the big one:** A Redis lock is **not** safe for correctness across a failover, and the TTL can expire mid-operation, letting a second process in while the first is still working. Martin Kleppmann's critique of Redlock is the canonical reference. **If the invariant is "never oversell," enforce it in the database** (unique constraint, conditional UPDATE, or `FOR UPDATE`). Use Redis locks for *efficiency* (avoiding duplicate expensive work), not for *correctness*.

### Choosing — the decision table

| Situation | Use |
|-----------|-----|
| Simple counter increment/decrement | Atomic conditional UPDATE |
| Multi-step logic on one row, high contention | `lockForUpdate` in a short transaction |
| Multi-step logic, low contention, long user think-time | Optimistic version column |
| Need audit trail, reconciliation, high scale | Append-only ledger + advisory lock |
| Coordinating a non-row resource (one import per tenant) | Advisory lock or Redis lock |
| Avoiding duplicate expensive work (not correctness) | Redis lock |
| Preventing duplicate client submissions | Idempotency key + unique index |

### Idempotency — the client-side race

Locking solves concurrent *server* operations. It does nothing about a client that times out and retries.

```php
final class SubmitOrderAction
{
    public function execute(SubmitOrderData $data): Order
    {
        return DB::transaction(function () use ($data) {
            try {
                $order = Order::create([
                    'organization_id' => $data->organizationId,
                    'idempotency_key' => $data->idempotencyKey,   // UNIQUE(org_id, idempotency_key)
                    'total_cents'     => $data->totalCents,
                ]);
            } catch (UniqueConstraintViolationException) {
                // The retry lost the race — return the original result
                return Order::where('organization_id', $data->organizationId)
                    ->where('idempotency_key', $data->idempotencyKey)
                    ->firstOrFail();
            }

            foreach ($data->lines as $line) {
                $this->deductStock->execute($line);
                $order->lines()->create($line->toArray());
            }

            return $order;
        }, attempts: 3);
    }
}
```

**The unique index is the guarantee.** A `SELECT ... if not exists ... INSERT` check races; letting the database reject the duplicate does not.

> **Follow-up:** *Your trading platform at 20K DAU — how do you make order submission safe?* Layered: (1) `Idempotency-Key` header enforced by middleware and a unique index so retries can't double-submit; (2) atomic conditional UPDATE or ledger insert so concurrent orders can't oversell; (3) short transactions with no external I/O; (4) `DB::transaction(..., attempts: 3)` for serialization failures; (5) queue-based serialization per instrument for non-latency-critical paths; (6) reconciliation job that recomputes balances from the ledger and alerts on drift. Point out that you monitor for drift rather than assuming correctness — that's operational maturity.

---

## 5. Transaction Isolation Levels

### The anomalies

| Anomaly | What happens |
|---------|--------------|
| **Dirty read** | You read another transaction's uncommitted data |
| **Non-repeatable read** | You read a row twice in one transaction and get different values |
| **Phantom read** | You run the same range query twice and get different *rows* |
| **Lost update** | Two transactions read-modify-write; one overwrites the other |
| **Write skew** | Two transactions each read a set, each writes based on it, and together they violate an invariant neither violated alone |

### Levels and what they prevent

| Level | Dirty read | Non-repeatable | Phantom | Lost update | Write skew |
|-------|-----------|----------------|---------|-------------|------------|
| READ UNCOMMITTED | ❌ possible | ❌ | ❌ | ❌ | ❌ |
| READ COMMITTED (**Postgres default**) | ✅ prevented | ❌ | ❌ | ❌ | ❌ |
| REPEATABLE READ (**MySQL InnoDB default**) | ✅ | ✅ | ✅ in InnoDB (gap locks); ✅ in PG (snapshot) | ❌ (PG aborts with 40001) | ❌ |
| SERIALIZABLE | ✅ | ✅ | ✅ | ✅ | ✅ |

> **Know both defaults and that they differ.** Postgres = READ COMMITTED. MySQL InnoDB = REPEATABLE READ. Same application code behaves differently on each — a genuinely important thing for someone who has used both.

### Write skew — the anomaly people forget

```
Invariant: at least one warehouse must remain active per organization.
Currently: Warehouse A active, Warehouse B active.

Tx1: SELECT COUNT(*) WHERE active = true  → 2. "Fine, I can deactivate A."
Tx2: SELECT COUNT(*) WHERE active = true  → 2. "Fine, I can deactivate B."
Tx1: UPDATE A SET active = false. COMMIT.
Tx2: UPDATE B SET active = false. COMMIT.

Result: zero active warehouses. Neither transaction violated the invariant
individually, but together they did. REPEATABLE READ does NOT prevent this.
```

**Fixes:** SERIALIZABLE isolation, materializing the conflict (`SELECT ... FOR UPDATE` on all rows in the set), an advisory lock on the invariant, or a DB constraint/trigger that enforces it.

### Using isolation levels in Laravel

```php
// Postgres
DB::transaction(function () {
    DB::statement('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
    // ...
}, attempts: 5);   // SERIALIZABLE WILL abort with 40001; you MUST retry
```

```php
// Explicit retry on serialization failure
$attempts = 0;

while (true) {
    try {
        return DB::transaction(function () {
            DB::statement('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
            return $this->performCriticalOperation();
        });
    } catch (QueryException $e) {
        // 40001 = serialization_failure, 40P01 = deadlock_detected
        if (! in_array($e->getCode(), ['40001', '40P01'], true) || ++$attempts >= 5) {
            throw $e;
        }
        usleep((2 ** $attempts) * 10_000 + random_int(0, 10_000));   // exp backoff + jitter
    }
}
```

> **Trap:** People say "just use SERIALIZABLE." Postgres implements it with Serializable Snapshot Isolation, which doesn't block — it **aborts** transactions that would violate serializability. If your code doesn't retry on SQLSTATE `40001`, raising the isolation level makes your app *less* reliable, not more. Also, throughput drops meaningfully under contention.

> **Follow-up:** *So how do you decide?* Default to READ COMMITTED and handle specific invariants explicitly with atomic UPDATEs, `FOR UPDATE`, unique constraints, or advisory locks — targeted, cheap, and predictable. Reserve SERIALIZABLE for genuinely complex multi-row invariants where explicit locking would be error-prone, and always pair it with retry logic.

---

## 6. Deadlocks

```
Tx1: UPDATE items WHERE id = 1    (holds lock on row 1)
Tx2: UPDATE items WHERE id = 2    (holds lock on row 2)
Tx1: UPDATE items WHERE id = 2    (waits for Tx2)
Tx2: UPDATE items WHERE id = 1    (waits for Tx1)  → DEADLOCK
```

The DB detects the cycle and kills one transaction (Postgres: SQLSTATE `40P01`; MySQL: error 1213).

### Prevention

**1. Consistent lock ordering — the most important rule.**

```php
// BAD: locks acquired in whatever order the input array happens to be
foreach ($data->lines as $line) {
    InventoryItem::where('id', $line->itemId)->lockForUpdate()->first();
}

// GOOD: always lock in ascending primary key order
$itemIds = collect($data->lines)->pluck('itemId')->sort()->values()->all();

$items = InventoryItem::whereIn('id', $itemIds)
    ->orderBy('id')
    ->lockForUpdate()
    ->get()
    ->keyBy('id');
```

Two concurrent multi-item orders now request locks in the same order, so one simply waits instead of deadlocking.

**2. Keep transactions short.** Less time holding locks = smaller collision window.

**3. Lock the parent first** in parent-child updates, consistently.

**4. Retry — deadlocks are normal, not exceptional.**

```php
DB::transaction(fn () => $this->processOrder($order), attempts: 3);
```

Laravel's `attempts` parameter automatically retries on deadlock detection.

**5. Set `lock_timeout` so waiting fails fast** instead of piling up connections.

```php
DB::statement("SET LOCAL lock_timeout = '3s'");
```

### Diagnosing in production

```sql
-- Postgres: who is blocking whom, right now
SELECT
    blocked.pid          AS blocked_pid,
    blocked.query        AS blocked_query,
    blocking.pid         AS blocking_pid,
    blocking.query       AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';

-- Long-running idle transactions (the silent killer)
SELECT pid, state, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '30 seconds';
```

```sql
-- Prevent them at the server level
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
```

> **Follow-up:** *You see deadlocks spike after a release. How do you debug it?* Enable `log_lock_waits` and `deadlock_timeout` logging in Postgres to capture the two conflicting statements, identify the lock ordering difference introduced by the new code path, normalize the order, add a retry, and add a regression test that runs two concurrent transactions against a real database to prove the fix.

---

## 7. Performance Engineering — The 88% Story

### The framework to tell it

Interviewers want a **method**, not a war story. Use this five-step structure every time.

```
1. MEASURE   — establish a baseline with real numbers
2. PROFILE   — find where the time actually goes (don't guess)
3. HYPOTHESIZE — form a specific, falsifiable theory
4. FIX       — smallest change that tests the hypothesis
5. VERIFY    — same measurement, prove the delta, then prevent regression
```

### The narrative (adapt the specifics to your real numbers)

**Situation.** The inventory dashboard endpoint had a p95 latency of ~2.4s and was the top source of database load. Under peak concurrency, Postgres connections saturated and unrelated endpoints degraded.

**Measure.** Enabled query logging in staging with production-shaped data and counted queries per request: ~520 queries for a 50-row dashboard. Added `DB::listen` slow-query logging in production, tagged by organization, and confirmed the same pattern with real tenants.

**Profile.** Four distinct causes, in order of impact:

| Cause | Queries | Fix |
|-------|---------|-----|
| Lazy-loaded `supplier` per item in the API Resource | 50 | `with('supplier:id,name')` + `whenLoaded()` |
| `$item->movements()->count()` per row | 50 | `withCount('movements')` |
| `$item->movements()->latest()->first()` per row | 50 | subquery select via `addSelect` |
| Per-item permission check hydrating roles | 50 | eager-load `roles.permissions`, rely on registrar cache |
| Per-row category lookup with no eager load | 50 | `with('category')` |
| Repeated `Organization::find()` inside a loop | 50 | hoist out of the loop |
| Aggregate stats recomputed per request | ~200 | Redis cache keyed by tenant, 5-min TTL, tag invalidation on write |

**Fix.** Eager loading with column selection, `withCount`/`withSum` aggregates, correlated subquery selects for "latest per row," hoisting invariants out of loops, and a tenant-namespaced Redis cache for the aggregate block with observer-driven invalidation.

**Verify.** ~520 → ~62 queries (88% reduction). p95 latency 2.4s → ~310ms. Database CPU on the primary dropped noticeably at peak.

**Prevent regression** — this is the part that impresses:
- `Model::preventLazyLoading()` enabled outside production, so a lazy load throws in dev and CI
- A query-count assertion in the feature test for that endpoint (`expect(count(DB::getQueryLog()))->toBeLessThan(8)`)
- Slow-query logging with a 200ms threshold, tagged by `organization_id`
- A per-request query-budget warning in `app()->terminating()`

> **The last section is what separates senior from mid.** Anyone can fix an N+1 once. Installing guardrails so it can't come back is engineering.

### Diagnostic toolkit

```php
// Ad-hoc query inspection
DB::enableQueryLog();
$this->runSuspiciousCode();
dd(DB::getQueryLog());

// See the compiled SQL with bindings substituted
InventoryItem::where('organization_id', 7)->toRawSql();      // Laravel 10.15+
InventoryItem::where('organization_id', 7)->dumpRawSql();

// Get the DB's execution plan
InventoryItem::where('organization_id', 7)->explain()->dd();

// Time a block
$start = hrtime(true);
$result = $this->expensive();
$ms = (hrtime(true) - $start) / 1e6;
```

```sql
-- The real tool: EXPLAIN (ANALYZE, BUFFERS) in Postgres
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM inventory_items
WHERE organization_id = 7 AND quantity < 10
ORDER BY created_at DESC LIMIT 25;
```

**How to read it:**
- `Seq Scan` on a large table with a selective filter → missing index
- `Rows Removed by Filter: 900000` → the index isn't selective enough, or the filter isn't indexable
- `actual rows` wildly different from estimated `rows` → stale statistics, run `ANALYZE`
- `Sort Method: external merge Disk: 12MB` → `work_mem` too low, or add an index that provides the ordering
- `Nested Loop` with high loop count → consider a hash join; check statistics
- High `shared read` vs `shared hit` → poor cache locality, data not in buffer cache

```sql
-- Find the actual worst offenders in production
SELECT
    calls,
    round(total_exec_time::numeric, 2)      AS total_ms,
    round(mean_exec_time::numeric, 2)       AS mean_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

> **Key insight to state out loud:** optimize by **total time** (`calls × mean`), not by slowest single query. A 5ms query called 100,000 times costs far more than a 2s query called twice.

```sql
-- Unused indexes (they cost write throughput and disk for nothing)
SELECT schemaname, relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Table bloat / vacuum health
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup, 0), 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### The optimization hierarchy (apply in this order)

1. **Don't do the work** — cache, precompute, or remove the feature
2. **Do less work** — pagination, column selection, partial indexes
3. **Do the work fewer times** — eager loading, batching, deduplication
4. **Do the work faster** — indexes, query rewrite, denormalization
5. **Do the work elsewhere** — queue it, move to a read replica
6. **Do the work on bigger hardware** — last resort, buys time not architecture

---

## 8. Indexing Strategy

### Composite index column order — the rule

**Equality columns first, then range/sort columns.** An index on `(a, b, c)` can serve queries filtering on `a`, `(a,b)`, or `(a,b,c)` — a left-prefix. It cannot efficiently serve a query filtering only on `b`.

```sql
-- Query pattern
WHERE organization_id = ? AND status = ? ORDER BY created_at DESC

-- Correct index
CREATE INDEX idx_items_org_status_created
  ON inventory_items (organization_id, status, created_at DESC);
```

For a multi-tenant app, **`organization_id` leads every index** because every query filters on it.

### Index types (Postgres)

```sql
-- B-tree (default): equality, ranges, sorting, prefix LIKE 'abc%'
CREATE INDEX idx_items_sku ON inventory_items (sku);

-- Partial: index only the rows you query, much smaller and faster
CREATE INDEX idx_items_low_stock ON inventory_items (organization_id, quantity)
  WHERE deleted_at IS NULL AND quantity < 10;

-- Covering (INCLUDE): index-only scan, avoids the heap fetch
CREATE INDEX idx_items_org_sku_covering ON inventory_items (organization_id, sku)
  INCLUDE (name, quantity);

-- Expression: index the computed value you actually filter on
CREATE INDEX idx_items_lower_name ON inventory_items (LOWER(name));
-- serves: WHERE LOWER(name) = 'widget'

-- GIN: jsonb containment, arrays, full-text search
CREATE INDEX idx_items_metadata ON inventory_items USING GIN (metadata jsonb_path_ops);
-- serves: WHERE metadata @> '{"color":"red"}'

CREATE INDEX idx_items_fts ON inventory_items
  USING GIN (to_tsvector('english', name || ' ' || coalesce(description, '')));

-- Trigram: fast ILIKE '%substring%' (needs pg_trgm extension)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_items_name_trgm ON inventory_items USING GIN (name gin_trgm_ops);

-- BRIN: huge append-only tables with natural ordering; tiny index
CREATE INDEX idx_movements_created_brin ON stock_movements USING BRIN (created_at);

-- Unique partial: enforce "one active per group"
CREATE UNIQUE INDEX idx_one_primary_warehouse
  ON warehouses (organization_id) WHERE is_primary = true;
```

### When an index is NOT used

| Cause | Example | Fix |
|-------|---------|-----|
| Function on the column | `WHERE LOWER(name) = 'x'` | Expression index, or store normalized |
| Leading wildcard | `WHERE name LIKE '%widget%'` | Trigram GIN index, or full-text search |
| Type mismatch | `WHERE varchar_col = 123` | Match types; cast the literal not the column |
| Low selectivity | `WHERE is_active = true` (95% true) | Partial index on the rare value |
| Not a left prefix | index `(a,b)`, query `WHERE b = ?` | Reorder or add an index |
| Stale statistics | planner underestimates | `ANALYZE table_name` |
| `OR` across columns | `WHERE a = 1 OR b = 2` | Rewrite as `UNION`, or add a multi-column index the planner can bitmap-OR |
| Small table | 500 rows | Seq scan is genuinely faster; not a problem |

### The cost of indexes

Every index must be updated on every INSERT/UPDATE/DELETE that touches its columns. Ten indexes on a hot write table can halve write throughput. In Postgres, a non-HOT update rewrites all index entries for the row.

**Rule:** index for the queries you actually run (verified via `pg_stat_statements`), then drop indexes with `idx_scan = 0`.

> **Follow-up:** *How do you add an index to a 15M-row table in production?* `CREATE INDEX CONCURRENTLY` — it doesn't take an `ACCESS EXCLUSIVE` lock, so writes continue. It's slower, takes two table scans, can't run inside a transaction, and if it fails it leaves an `INVALID` index you must drop and retry. In Laravel: `DB::statement(...)` with `public $withinTransaction = false;`.

---

## 9. Zero-Downtime Migrations — The 15M Story

### The core principle: Expand → Migrate → Contract

Never change schema and code in a single lockstep step. Split into deploys where **old code and new code can both run against the intermediate schema**.

```
Deploy 1 (EXPAND):   Schema accepts old AND new shape. Code writes old, tolerates new.
Backfill (MIGRATE):  Batched, throttled, resumable. No downtime, no locks held.
Deploy 2 (SWITCH):   Code reads/writes new shape. Old shape still present.
Deploy 3 (CONTRACT): Remove old shape after a soak period and verification.
```

This works because at every moment, **the currently-deployed code and the currently-deployed schema are compatible** — which is required during a rolling deploy where old and new pods serve traffic simultaneously.

### Case A — Adding a NOT NULL column with a default

```php
// ── Deploy 1: add nullable column ────────────────────────────────────
Schema::table('inventory_items', function (Blueprint $table) {
    $table->string('status')->nullable()->index();
});
```

> **Postgres nuance worth stating:** since PG 11, `ADD COLUMN ... DEFAULT x NOT NULL` is a metadata-only operation — it does **not** rewrite the table. On PG 10 and MySQL 5.7 it rewrites the whole table under an exclusive lock, which on 15M rows means minutes of downtime. **Know your engine and version.** On MySQL 8 with `ALGORITHM=INSTANT`, adding a column is also instant.

```php
// ── Deploy 1 (same deploy): code dual-writes ─────────────────────────
final class InventoryItemObserver
{
    public function saving(InventoryItem $item): void
    {
        // New writes populate the new column; old readers ignore it
        $item->status ??= $item->quantity > 0 ? 'in_stock' : 'out_of_stock';
    }
}
```

```php
// ── Backfill: batched, throttled, resumable ──────────────────────────
final class BackfillItemStatus extends Command
{
    protected $signature = 'inventory:backfill-status
                            {--chunk=2000}
                            {--sleep=100 : ms between batches}
                            {--max-lag=5 : seconds of replica lag to tolerate}
                            {--dry-run}';

    public function handle(): int
    {
        $chunk  = (int) $this->option('chunk');
        $sleep  = (int) $this->option('sleep');
        $maxLag = (int) $this->option('max-lag');
        $dryRun = (bool) $this->option('dry-run');

        // Resume from a checkpoint so a crash doesn't restart from zero
        $lastId = (int) Cache::get('backfill:item_status:last_id', 0);
        $total  = 0;

        $this->info("Resuming from id > {$lastId}");

        while (true) {
            $ids = DB::table('inventory_items')
                ->whereNull('status')
                ->where('id', '>', $lastId)
                ->orderBy('id')
                ->limit($chunk)
                ->pluck('id');

            if ($ids->isEmpty()) {
                break;
            }

            if (! $dryRun) {
                // Set-based update: one statement per batch, not per row
                DB::table('inventory_items')
                    ->whereIn('id', $ids)
                    ->update([
                        'status' => DB::raw("CASE WHEN quantity > 0 THEN 'in_stock' ELSE 'out_of_stock' END"),
                    ]);
            }

            $lastId = $ids->last();
            $total += $ids->count();

            Cache::put('backfill:item_status:last_id', $lastId, now()->addDays(7));

            $this->line("Processed {$total} (last id {$lastId})");

            // Backpressure: pause if read replicas fall behind
            while ($this->replicaLagSeconds() > $maxLag) {
                $this->warn('Replica lag high — pausing 5s');
                sleep(5);
            }

            usleep($sleep * 1000);
        }

        $this->info("Backfill complete: {$total} rows.");

        // Reconciliation — prove completeness, don't assume it
        $remaining = DB::table('inventory_items')->whereNull('status')->count();
        if ($remaining > 0) {
            $this->error("{$remaining} rows still NULL — investigate before proceeding.");
            return self::FAILURE;
        }

        return self::SUCCESS;
    }

    private function replicaLagSeconds(): float
    {
        return (float) (DB::connection('read')->selectOne(
            'SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag'
        )->lag ?? 0);
    }
}
```

```php
// ── Deploy 2: reads switch to the new column ─────────────────────────
// (code now uses $item->status everywhere)

// ── Deploy 3: enforce the constraint ─────────────────────────────────
// Adding NOT NULL normally requires a full table scan under ACCESS EXCLUSIVE.
// Postgres trick: add a NOT VALID check constraint (instant), validate it
// concurrently (SHARE UPDATE EXCLUSIVE — allows reads and writes), then
// PG 12+ recognizes it and SET NOT NULL becomes cheap.
DB::statement('ALTER TABLE inventory_items ADD CONSTRAINT status_not_null CHECK (status IS NOT NULL) NOT VALID');
DB::statement('ALTER TABLE inventory_items VALIDATE CONSTRAINT status_not_null');
DB::statement('ALTER TABLE inventory_items ALTER COLUMN status SET NOT NULL');
DB::statement('ALTER TABLE inventory_items DROP CONSTRAINT status_not_null');
```

> **The `NOT VALID` → `VALIDATE` trick is a genuinely strong senior detail.** It converts a long exclusive lock into a brief one plus a concurrent scan.

### Case B — Renaming a column with zero downtime

Never `renameColumn` on a large hot table: old code breaks the instant the rename lands.

```
Deploy 1:  ADD new column (nullable)
Deploy 2:  Code writes BOTH old and new; reads old
Backfill:  Copy old → new in batches
Deploy 3:  Code reads new; still writes both
Verify:    Soak period; assert old and new agree
Deploy 4:  Code writes only new
Deploy 5:  DROP old column
```

```php
// Deploy 2 — dual write via a mutator
protected function reorderPoint(): Attribute
{
    return Attribute::make(
        get: fn () => $this->attributes['reorder_point'] ?? $this->attributes['low_stock_threshold'],
        set: fn (?int $value) => [
            'low_stock_threshold' => $value,   // old
            'reorder_point'       => $value,   // new
        ],
    );
}
```

```sql
-- Verification before dropping
SELECT COUNT(*) FROM inventory_items
WHERE low_stock_threshold IS DISTINCT FROM reorder_point;
-- must be 0
```

### Case C — Changing a column type (int → bigint on a 15M PK)

```
1. ADD COLUMN id_new bigint                                     (instant)
2. Trigger or dual-write keeps id_new in sync with id
3. Backfill id_new in batches
4. CREATE UNIQUE INDEX CONCURRENTLY on id_new
5. Recreate all FKs referencing id, pointing to id_new (NOT VALID, then VALIDATE)
6. In one short transaction: swap the PK constraint and rename columns
7. DROP the old column after a soak period
```

For MySQL, `pt-online-schema-change` or `gh-ost` automate this by building a shadow table and replaying the binlog.

### The DDL lock reference (Postgres)

| Operation | Lock | Blocks reads? | Blocks writes? | Safe on 15M rows? |
|-----------|------|---------------|----------------|-------------------|
| `ADD COLUMN` (nullable, no default) | ACCESS EXCLUSIVE, brief | Momentarily | Momentarily | ✅ |
| `ADD COLUMN ... DEFAULT` (PG 11+) | ACCESS EXCLUSIVE, brief | Momentarily | Momentarily | ✅ |
| `ADD COLUMN ... DEFAULT` (PG ≤10) | ACCESS EXCLUSIVE, table rewrite | **Yes, minutes** | **Yes** | ❌ |
| `DROP COLUMN` | ACCESS EXCLUSIVE, brief (marks invisible) | Momentarily | Momentarily | ✅ |
| `ALTER COLUMN TYPE` | ACCESS EXCLUSIVE, rewrite | **Yes** | **Yes** | ❌ use shadow column |
| `SET NOT NULL` (naive) | ACCESS EXCLUSIVE, full scan | **Yes** | **Yes** | ❌ use NOT VALID trick |
| `CREATE INDEX` | SHARE | No | **Yes** | ❌ |
| `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | No | No | ✅ |
| `ADD FOREIGN KEY` | SHARE ROW EXCLUSIVE + scan | No | **Yes** | ❌ use NOT VALID |
| `ADD FOREIGN KEY ... NOT VALID` then `VALIDATE` | Brief, then SHARE UPDATE EXCLUSIVE | No | No | ✅ |
| `RENAME COLUMN` | ACCESS EXCLUSIVE, brief | Momentarily | Momentarily | ⚠️ **breaks running code** |

> **Critical operational detail:** even a "brief" ACCESS EXCLUSIVE lock is dangerous, because it queues **behind** any long-running query and then **everything else queues behind it**. A migration waiting on a 5-minute analytics query blocks all traffic to that table for 5 minutes. Always set `lock_timeout` and retry:

```php
public function up(): void
{
    DB::statement("SET lock_timeout = '3s'");

    retry(10, function () {
        Schema::table('inventory_items', fn (Blueprint $t) => $t->string('status')->nullable());
    }, 5000);   // 10 attempts, 5s apart
}
```

### Deploy-order rules

| Change | Migrate before or after deploy? |
|--------|--------------------------------|
| Add nullable column | Before (new code needs it; old code ignores it) |
| Add index | Before or anytime (concurrently) |
| Drop column | **After** the deploy that stops referencing it |
| Rename | Never in one step — use the dual-write dance |
| Add NOT NULL constraint | After backfill + after all writers populate it |

### Telling the story

**Situation.** Needed to add a `status` column and a composite index to a 15M-row `inventory_items` table in a multi-tenant SaaS with 24/7 usage across time zones. Naive migration would have locked the table for minutes.

**Approach.** Expand → migrate → contract across four deploys. Added the column nullable (metadata-only on PG 11+). Dual-wrote from the application so new rows were correct immediately. Backfilled with a resumable, checkpointed command using keyset pagination, 2000-row batches, 100ms inter-batch sleep, and replica-lag backpressure that paused when lag exceeded 5 seconds. Built the index with `CREATE INDEX CONCURRENTLY`. Enforced NOT NULL via a `NOT VALID` check constraint validated concurrently. Set `lock_timeout` to 3s with retries so a DDL statement could never queue behind a long analytics query.

**Verification.** Reconciliation query proving zero remaining NULLs before advancing. Row-count comparison against a pre-migration snapshot. A soak period reading from the new column with the old still present, so rollback was a code revert rather than a data restore.

**Result.** Zero downtime, zero customer-visible errors, p99 latency unchanged during the backfill window.

> **The rollback plan is what interviewers probe for.** Answer: at every stage before the final `DROP`, rollback is a code deploy, not a data restore — because the old column is still present and populated. The backfill is idempotent and resumable, so a partial run is safe to re-run. That's why expand/contract exists.

---

## 10. Read Replicas & Connection Pooling

```php
// config/database.php
'pgsql' => [
    'read' => [
        'host' => [env('DB_READ_HOST_1'), env('DB_READ_HOST_2')],
    ],
    'write' => [
        'host' => [env('DB_WRITE_HOST')],
    ],
    'sticky' => true,        // ← critical
    'driver' => 'pgsql',
    // ...
],
```

**`sticky = true`** means: once a write happens in this request/job, subsequent reads in that same request go to the **primary**. Without it, you write a row and immediately read it back from a replica that hasn't caught up yet, and it isn't there.

```php
// Force a connection explicitly
DB::connection('pgsql::read')->select(...);
InventoryItem::on('pgsql::read')->get();

DB::transaction(function () {
    // All queries inside a transaction always use the write connection
});
```

### Replication lag — the class of bug it creates

```php
// Bug: create then immediately read
$item = InventoryItem::create([...]);              // primary
$fresh = InventoryItem::find($item->id);            // replica → may be NULL

// Bug: job dispatched during a request reads a replica before it catches up
ProcessNewItem::dispatch($item->id);                // worker reads replica → not found
```

**Fixes:** `sticky = true` for the request path; in jobs, either read from the primary for freshly-created entities, or add a retry with backoff, or pass the needed data in the job payload rather than re-fetching.

```php
// Monitor lag
SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

### Connection pooling

PHP's shared-nothing model means **every php-fpm worker opens its own DB connection**. 40 workers × 6 app servers = 240 connections. Postgres `max_connections` defaults to 100, and each backend costs ~10 MB. This is the classic scaling wall for PHP applications.

```
Without pooler:  [200 PHP workers] ──200 connections──► [Postgres: dies]
With PgBouncer:  [200 PHP workers] ──200 connections──► [PgBouncer] ──20──► [Postgres: fine]
```

**PgBouncer pool modes:**

| Mode | Connection reused after | Works with Laravel? |
|------|------------------------|---------------------|
| `session` | Client disconnects | Yes, but little benefit for php-fpm |
| **`transaction`** | Each transaction ends | **Yes — the right choice**, with caveats |
| `statement` | Each statement | No — breaks transactions |

**Transaction-mode caveats you must know:**
- Session-level state doesn't persist: `SET` statements, session advisory locks (`pg_advisory_lock`), `LISTEN/NOTIFY`, temp tables, and **prepared statements**.
- Laravel/PDO uses prepared statements — set `PDO::ATTR_EMULATE_PREPARES => true` (or use PgBouncer 1.21+ which supports protocol-level prepared statement tracking).
- Use `pg_advisory_xact_lock` (transaction-scoped), never `pg_advisory_lock` (session-scoped).

```php
'pgsql' => [
    'options' => extension_loaded('pdo_pgsql') ? [
        PDO::ATTR_EMULATE_PREPARES => true,
    ] : [],
],
```

> **Follow-up:** *When do you add a read replica?* When read load dominates and you've already exhausted caching and query optimization. Replicas add eventual-consistency complexity to every read path, so they are not a free win. Route reporting, analytics, exports, and search backfills to replicas; keep anything read-after-write on the primary.

---

## 11. Clean Architecture & DDD in Laravel

### Layering

```
┌──────────────────────────────────────────────────────┐
│  Presentation   Controllers, Requests, Resources,    │  Framework
│                 Console Commands, Middleware          │  (Laravel)
├──────────────────────────────────────────────────────┤
│  Application    Actions / Use Cases, DTOs,           │  Orchestration
│                 Application Services                  │
├──────────────────────────────────────────────────────┤
│  Domain         Entities, Value Objects, Aggregates, │  Pure PHP
│                 Domain Events, Repository INTERFACES │  Zero framework
├──────────────────────────────────────────────────────┤
│  Infrastructure Eloquent repos, HTTP clients,        │  Framework
│                 Queue adapters, Cache adapters        │  + external
└──────────────────────────────────────────────────────┘

Dependency rule: dependencies point INWARD. Domain knows nothing about Laravel.
```

```
app/
├── Domain/
│   └── Inventory/
│       ├── Models/          InventoryItem.php (Eloquent, or a pure entity)
│       ├── ValueObjects/    Sku.php, Quantity.php, Money.php
│       ├── Events/          StockDeducted.php
│       ├── Exceptions/      InsufficientStockException.php
│       ├── Repositories/    StockRepositoryInterface.php   ← interface only
│       └── Services/        StockCalculator.php
├── Application/
│   └── Inventory/
│       ├── Actions/         DeductStockAction.php
│       └── Data/            DeductStockData.php
├── Infrastructure/
│   └── Inventory/
│       ├── Repositories/    EloquentStockRepository.php    ← implementation
│       └── Search/          ElasticsearchItemIndexer.php
└── Http/
    └── Controllers/         DeductStockController.php
```

### Value objects — make invalid states unrepresentable

```php
namespace App\Domain\Inventory\ValueObjects;

final readonly class Quantity
{
    private function __construct(public int $value) {}

    public static function fromInt(int $value): self
    {
        if ($value < 0) {
            throw new InvalidArgumentException("Quantity cannot be negative: {$value}");
        }

        return new self($value);
    }

    public static function zero(): self
    {
        return new self(0);
    }

    public function add(self $other): self
    {
        return new self($this->value + $other->value);
    }

    public function subtract(self $other): self
    {
        if ($other->value > $this->value) {
            throw new InsufficientQuantityException($this->value, $other->value);
        }

        return new self($this->value - $other->value);
    }

    public function isZero(): bool
    {
        return $this->value === 0;
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }
}
```

```php
final readonly class Sku implements Stringable
{
    private function __construct(public string $value) {}

    public static function fromString(string $raw): self
    {
        $normalized = strtoupper(trim($raw));

        if (! preg_match('/^[A-Z0-9\-]{1,64}$/', $normalized)) {
            throw new InvalidSkuException($raw);
        }

        return new self($normalized);
    }

    public function __toString(): string
    {
        return $this->value;
    }
}
```

**Why this matters:** once you have `Quantity`, a negative quantity is impossible anywhere in the system — not by convention, by construction. That's the DDD payoff, and it's a much better answer than reciting the definition of a value object.

```php
// Bridge value objects into Eloquent with a custom cast
final class QuantityCast implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes): Quantity
    {
        return Quantity::fromInt((int) $value);
    }

    public function set($model, string $key, $value, array $attributes): array
    {
        return [$key => $value instanceof Quantity ? $value->value : (int) $value];
    }
}

// Model
protected function casts(): array
{
    return ['quantity' => QuantityCast::class, 'sku' => SkuCast::class];
}
```

### Action / Use Case

```php
namespace App\Application\Inventory\Actions;

final readonly class DeductStockAction
{
    public function __construct(
        private StockRepositoryInterface $stock,
        private DatabaseManager $db,
        private Dispatcher $events,
    ) {}

    public function execute(DeductStockData $data): InventoryItem
    {
        return $this->db->transaction(function () use ($data) {
            $item = $this->stock->lockForUpdate($data->itemId, $data->organizationId);

            $newQuantity = $item->quantity->subtract($data->amount);   // VO enforces the rule

            $this->stock->updateQuantity($item, $newQuantity);

            $this->stock->recordMovement(new StockMovementData(
                itemId:         $item->id,
                organizationId: $data->organizationId,
                type:           MovementType::Outbound,
                quantity:       $data->amount,
                reference:      $data->reference,
                idempotencyKey: $data->idempotencyKey,
            ));

            $this->events->dispatch(new StockDeducted(
                itemId:         $item->id,
                organizationId: $data->organizationId,
                amount:         $data->amount->value,
                remaining:      $newQuantity->value,
            ));

            return $item->refresh();
        }, attempts: 3);
    }
}
```

### When NOT to do this — the answer that shows judgment

Full Clean Architecture in Laravel means: hand-mapping between Eloquent models and domain entities, giving up Eloquent's relationship/eager-loading ergonomics inside the domain, and a lot of interfaces with one implementation.

**Say this in an interview:**

> "I apply the layering pragmatically. For a CRUD-heavy admin module, a thin controller plus Eloquent is the right amount of structure — adding repositories and entities there is pure ceremony. I introduce Actions, DTOs, value objects, and repository interfaces where the business logic is genuinely complex and long-lived: stock deduction, pricing, order state machines. In our inventory SaaS, that was the stock and movements domain — everything else stayed conventional Laravel. Uniform architecture across a codebase usually means it was chosen dogmatically rather than for the problem."

That answer signals seniority far more than a perfect recitation of the dependency rule.

### The repository debate — have a position

**Against:** Eloquent already implements Active Record + a query builder. Wrapping it in a repository often produces `findById`, `save`, `delete` passthroughs plus a leaky `Builder` return type. You lose eager loading, scopes, and pagination ergonomics.

**For:** It gives the domain a persistence-agnostic interface, lets you swap or decorate implementations (caching, read replicas, search), and makes unit testing trivial.

**Your position:** "I use repository interfaces when the domain layer must not know about Eloquent, or when I need to decorate persistence — for instance wrapping the stock repository with a caching decorator via `$app->extend()`, or routing heavy reads to a search index. I don't create a repository per model as a reflex. If the interface method set mirrors Eloquent one-to-one, the abstraction is not earning its keep."

---

## 12. CQRS, Event Sourcing & the Outbox Pattern

### CQRS — separate the read and write models

```
                      ┌────────────────┐
  Command ──────────► │  Write Model   │ ──► Postgres (normalized, constrained)
  (DeductStock)       │  Actions,      │         │
                      │  Aggregates    │         │ domain events
                      └────────────────┘         ▼
                                          ┌──────────────┐
  Query ────────────────────────────────► │  Read Model  │ ◄── projector
  (InventoryDashboard)                    │  ES / Redis / │
                                          │  mat. view    │
                                          └──────────────┘
```

```php
// Write side — enforces invariants, normalized, transactional
final class DeductStockAction { /* as above */ }

// Read side — denormalized, optimized purely for query shape
final class InventoryDashboardQuery
{
    public function __construct(private readonly Connection $db) {}

    public function forOrganization(int $orgId): array
    {
        // Hits a materialized view; no Eloquent hydration, no joins at query time
        return $this->db->table('mv_inventory_dashboard')
            ->where('organization_id', $orgId)
            ->get()
            ->toArray();
    }
}
```

```sql
CREATE MATERIALIZED VIEW mv_inventory_dashboard AS
SELECT
    i.organization_id,
    COUNT(*)                                            AS total_items,
    COUNT(*) FILTER (WHERE i.quantity = 0)              AS out_of_stock,
    COUNT(*) FILTER (WHERE i.quantity <= i.reorder_point AND i.quantity > 0) AS low_stock,
    SUM(i.quantity * i.cost_price)                      AS inventory_value
FROM inventory_items i
WHERE i.deleted_at IS NULL
GROUP BY i.organization_id;

CREATE UNIQUE INDEX ON mv_inventory_dashboard (organization_id);

-- Refresh without blocking readers (requires the unique index above)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_inventory_dashboard;
```

> **Follow-up:** *When is CQRS worth it?* When read and write workloads have genuinely different shapes and scaling characteristics — e.g. writes are single-row transactional updates while reads are cross-entity aggregations over millions of rows. It buys independent optimization and scaling at the cost of eventual consistency and two models to keep in sync. It is **not** a default architecture, and applying it to CRUD is a well-known way to make a simple system complicated.

### Transactional Outbox — the dual-write problem

**The problem:** you must update the database *and* publish an event to Kafka/SQS. These are two different systems; there is no distributed transaction.

```php
// BROKEN — four ways to be inconsistent
DB::transaction(function () use ($item) {
    $item->decrement('quantity');
    Kafka::publish('stock.deducted', [...]);   // published, then DB rolls back → phantom event
});                                             // or DB commits, then publish fails → lost event
```

**The fix:** write the event to an outbox table **in the same transaction** as the business data. A separate relay reads the outbox and publishes.

```php
Schema::create('outbox_events', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->foreignId('organization_id');
    $table->string('aggregate_type');
    $table->string('aggregate_id');
    $table->string('event_type');
    $table->jsonb('payload');
    $table->timestamp('created_at');
    $table->timestamp('published_at')->nullable();
    $table->unsignedTinyInteger('attempts')->default(0);

    $table->index(['published_at', 'created_at']);
});
```

```php
DB::transaction(function () use ($item, $amount) {
    $item->decrement('quantity', $amount);

    OutboxEvent::create([
        'id'             => Str::uuid(),
        'organization_id'=> $item->organization_id,
        'aggregate_type' => 'inventory_item',
        'aggregate_id'   => (string) $item->id,
        'event_type'     => 'stock.deducted',
        'payload'        => ['item_id' => $item->id, 'amount' => $amount],
        'created_at'     => now(),
    ]);
});
// Atomic: either both the decrement and the event row exist, or neither does.
```

```php
// Relay — claims rows with SKIP LOCKED so multiple relays can run safely
final class PublishOutboxEvents extends Command
{
    public function handle(): int
    {
        while (true) {
            $published = DB::transaction(function () {
                $events = DB::table('outbox_events')
                    ->whereNull('published_at')
                    ->orderBy('created_at')
                    ->limit(100)
                    ->lockForUpdate()
                    ->skipLocked()          // ← other relay instances take different rows
                    ->get();

                foreach ($events as $event) {
                    Kafka::publish($event->event_type, json_decode($event->payload, true));

                    DB::table('outbox_events')
                        ->where('id', $event->id)
                        ->update(['published_at' => now()]);
                }

                return $events->count();
            });

            if ($published === 0) {
                sleep(1);
            }
        }
    }
}
```

**Guarantee:** at-least-once delivery. If the relay crashes after publishing but before marking `published_at`, the event republishes. Consumers must be idempotent — which is the standard contract in event-driven systems anyway.

> **`SKIP LOCKED` is worth calling out explicitly.** It's how you build a safe work-claiming queue in SQL, and it connects directly to your Chronos scheduler project — the same primitive prevents two schedulers from claiming the same job.

### Event sourcing (know the shape, know when to refuse)

Store the events as the source of truth; derive current state by replay.

```
stock_events: [ItemCreated, StockAdded(100), StockDeducted(5), StockDeducted(3)]
current quantity = 0 + 100 - 5 - 3 = 92
```

- ✅ Perfect audit trail, time travel, replay into new projections, natural fit for finance/inventory
- ❌ Very high complexity, snapshotting required for performance, schema evolution of old events is painful, queries need projections for everything

> **Say this:** "I've used the *ledger* pattern — append-only stock movements with a maintained rollup — which gives most of the audit and reconciliation benefits of event sourcing without committing the whole system to it. Full event sourcing I'd reserve for a bounded context where auditability is a hard regulatory requirement, not as a system-wide architecture."

---

## 13. SOLID at Architectural Scale

> **Full treatment with worked code lives in [`02-oop-php.md`](./02-oop-php.md) §1** — the five principles, covariance/contravariance as the mechanics behind LSP, and a "when this principle is the wrong call" note for each. Don't re-read it here; this section covers only what changes when the unit of design is a *service boundary* rather than a class.

At senior level the acronym itself is table stakes. What differentiates the answer is applying the same reasoning one level up, and knowing when the ceremony isn't worth it.

| Principle | At class scale | At architectural scale |
|-----------|----------------|------------------------|
| **SRP** | One reason to change | One team owns it; one deploy cadence; one on-call rotation |
| **OCP** | Add a class, don't edit one | Add a consumer to the event stream, don't edit the producer |
| **LSP** | Subtype honours the contract | A new service version honours the old API contract (see contract testing) |
| **ISP** | Narrow interfaces | Narrow, versioned API surfaces — don't force every client through one fat endpoint |
| **DIP** | Depend on abstractions | The domain owns the contract; adapters for Stripe, S3, and Elasticsearch depend on it |

### The one example worth keeping here — LSP and read replicas

This ties directly to §10 (read replicas and connection pooling), which is why it lives in this file rather than in the OOP tier.

```php
// ❌ Subtype strengthens a precondition — breaks substitutability
class ReadOnlyStockRepository extends EloquentStockRepository
{
    public function updateQuantity(InventoryItem $i, Quantity $q): void
    {
        throw new BadMethodCallException('Read-only');   // caller can't substitute safely
    }
}

// ✅ Segregate the contract instead — the type system now prevents writing to a replica
interface StockReader { public function find(int $id, int $orgId): ?InventoryItem; }
interface StockWriter { public function updateQuantity(InventoryItem $i, Quantity $q): void; }

final class EloquentStockRepository implements StockReader, StockWriter {}
final class ReplicaStockRepository implements StockReader {}
```

A reporting service typed against `StockReader` **cannot** be handed something that writes, and a replica-backed implementation **cannot** be passed where writes are expected. This is ISP and LSP doing real work: the replica/primary split stops being a convention people have to remember and becomes a compile-time guarantee.

> **Interview framing — do not recite the acronym.** Say: *"SOLID is mostly about isolating what changes. In our inventory domain the alerting channels changed constantly while the deduction logic didn't, so I made channels open for extension via container tagging and kept the deduction action closed. Everywhere else I skipped the ceremony — most of the codebase had one implementation and no prospect of a second, and an interface there is just a file you have to open twice."*
>
> The second half of that answer is what marks it as senior. Anyone can argue *for* abstraction; knowing where it doesn't pay for itself is the harder judgement, and it's what interviewers are actually probing.

---

## 14. Design Patterns in the Laravel Source

> **Applying** these patterns in your own code is covered in [`02-oop-php.md`](./02-oop-php.md) §3, including Adapter, Template Method, Null Object, Builder, and Specification with full implementations. This section is the complementary skill: pointing at where the **framework itself** uses each one.

Naming a pattern is cheap. Naming the class in the Laravel source that implements it is not, and it's a fast way to signal you've read the framework rather than just used it.

| Pattern | Where in Laravel | Mechanism |
|---------|-----------------|-----------|
| **Service Locator / IoC Container** | `Illuminate\Container\Container` | Reflection-based autowiring |
| **Facade (static proxy)** | `Illuminate\Support\Facades\*` | `__callStatic` → container resolution |
| **Builder** | `Eloquent\Builder`, `Query\Builder` | Fluent chained method calls |
| **Active Record** | Eloquent `Model` | Model contains its own persistence |
| **Chain of Responsibility / Pipeline** | `Illuminate\Pipeline\Pipeline` (middleware) | `array_reduce` building nested closures |
| **Observer** | Model events, `Illuminate\Events\Dispatcher` | Listener registry |
| **Strategy** | Cache/queue/mail/filesystem drivers | Interface + `Manager` selecting an implementation |
| **Factory** | `Illuminate\Support\Manager` subclasses, model factories | `createRedisDriver()`, `createSqsDriver()` |
| **Decorator** | `$app->extend()`, cache repository wrapping a store | Wrap an instance preserving the interface |
| **Adapter** | Flysystem in `Filesystem`, PSR-3 logging | Translate one interface to another |
| **Template Method** | `Illuminate\Support\Manager`, `TestCase::setUp` | Base defines skeleton, subclass fills steps |
| **Repository** | `Cache\Repository` | Abstracts the underlying store |
| **Singleton** | `$app->singleton()` bindings | Container-managed, not the GoF static kind |
| **Macro / Open class** | `Macroable` trait | `__call`/`__callStatic` + `Closure::bind` |
| **Specification** | Query scopes | Composable, named query constraints |
| **Null Object** | `belongsTo()->withDefault()` | Return a benign default instead of null |

```php
// The Manager pattern — how Laravel picks a driver
abstract class Manager
{
    protected array $drivers = [];

    public function driver(?string $driver = null)
    {
        $driver ??= $this->getDefaultDriver();

        return $this->drivers[$driver] ??= $this->createDriver($driver);
    }

    protected function createDriver(string $driver)
    {
        $method = 'create' . Str::studly($driver) . 'Driver';

        if (method_exists($this, $method)) {
            return $this->$method();
        }

        throw new InvalidArgumentException("Driver [{$driver}] not supported.");
    }
}

// Extend without touching the framework
Queue::extend('my-broker', fn ($app, $config) => new MyBrokerConnector($config));
Cache::extend('dynamodb-custom', fn ($app, $config) => Cache::repository(new MyStore($config)));
Auth::extend('jwt-custom', fn ($app, $name, $config) => new JwtGuard($app['request'], $config));
Validator::extend(
    'tenant_owned',
    fn (string $attribute, mixed $value, array $params, Validator $validator): bool => DB::table($params[0])
        ->where('id', $value)
        ->where('organization_id', app(TenantContext::class)->organizationIdOrFail())
        ->exists()
);
```

---

## 15. Scaling & Deployment

### The stateless requirement

Horizontal scaling requires that any request can hit any server. That means **nothing may live on local disk or in local process memory**:

| State | Wrong | Right |
|-------|-------|-------|
| Sessions | `file` driver | Redis / DynamoDB / cookie |
| Cache | `file` / `array` | Redis / Memcached |
| Uploads | `local` disk | S3 |
| Queue | `sync` / `database` on local | Redis / SQS |
| Locks | `file` cache lock | Redis / DB |
| Logs | local file only | stdout → CloudWatch/ELK |
| Scheduler | cron on every box | `onOneServer()` + shared cache, or a single scheduler container |

### Reference architecture on AWS (matches your stack)

```
Route 53
   │
CloudFront (static assets, CDN)
   │
   ALB (TLS termination, health checks on /up)
   │
   ├── ECS/EKS: web service (nginx + php-fpm, N tasks, autoscale on CPU + ALB RequestCount)
   │
   ├── ECS/EKS: queue workers (separate service, autoscale on queue DEPTH not CPU)
   │
   └── ECS/EKS: scheduler (exactly 1 task running `schedule:work`)

   RDS PostgreSQL Multi-AZ  ──► read replicas
        └── PgBouncer (sidecar or dedicated) for connection pooling
   ElastiCache Redis (cluster mode, Multi-AZ) — cache, sessions, queues, locks
   S3 — uploads, exports, backups
   SQS/SNS — cross-service events
   Secrets Manager — credentials, injected as env at task start
   CloudWatch / Prometheus + Grafana — metrics, alarms
```

**Autoscale workers on queue depth, not CPU** — a worker blocked on I/O has low CPU while the backlog grows. This is a detail interviewers notice.

```php
// Expose queue depth as a CloudWatch custom metric
Schedule::call(function () {
    foreach (['default', 'inventory', 'notifications'] as $queue) {
        CloudWatch::putMetricData([
            'Namespace'  => 'App/Queues',
            'MetricData' => [[
                'MetricName' => 'QueueDepth',
                'Dimensions' => [['Name' => 'QueueName', 'Value' => $queue]],
                'Value'      => Queue::size($queue),
            ]],
        ]);
    }
})->everyMinute();
```

### Graceful worker shutdown

```
ECS/K8s sends SIGTERM
   → Laravel worker sets shouldQuit = true
   → finishes the CURRENT job
   → exits cleanly
   → (if not exited within the grace period) SIGKILL — job dies mid-execution
```

```yaml
# Kubernetes: grace period must exceed your longest job timeout
terminationGracePeriodSeconds: 120
```

If a worker is SIGKILLed mid-job, the job is retried (at-least-once) — which is exactly why handlers must be idempotent.

### Zero-downtime deploy sequence

```bash
# 1. Build the artifact
composer install --no-dev --optimize-autoloader --no-interaction
npm ci && npm run build

# 2. Run backward-compatible migrations BEFORE shifting traffic
php artisan migrate --force --isolated     # --isolated: only one instance runs them

# 3. Warm caches in the new image/release
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# 4. Shift traffic (rolling / blue-green), health check on /up

# 5. Restart queue workers so they pick up the new code
php artisan queue:restart

# 6. Verify: error rate, latency, queue depth, then decommission old tasks
```

```php
// Maintenance mode with a bypass secret and a warm-up allowance
php artisan down --secret="deploy-2026-07-26" --render="errors::503" --retry=60
php artisan up
```

> **Trap:** `php artisan down` on multiple servers only affects the server it ran on unless you use a shared cache-based maintenance driver (`'driver' => 'cache'` in `config/app.php`, Laravel 11+). Otherwise half your fleet stays up.

> **Trap:** `config:cache` at Docker **build** time bakes in build-time env values. Run it in the entrypoint after runtime secrets are injected.

### Horizon configuration for a multi-tenant queue

```php
// config/horizon.php
'environments' => [
    'production' => [
        'supervisor-default' => [
            'connection'   => 'redis',
            'queue'        => ['high', 'default'],
            'balance'      => 'auto',           // auto-shift workers to busy queues
            'minProcesses' => 3,
            'maxProcesses' => 30,
            'balanceMaxShift'  => 3,
            'balanceCooldown'  => 3,
            'tries'        => 3,
            'timeout'      => 90,
            'memory'       => 256,
        ],
        'supervisor-heavy' => [
            'connection'   => 'redis',
            'queue'        => ['imports', 'exports'],
            'balance'      => 'simple',
            'maxProcesses' => 5,
            'timeout'      => 900,              // long-running imports
            'memory'       => 512,
        ],
    ],
],
```

```php
// Alert on queue backlog
Horizon::routeSlackNotificationsTo(config('services.slack.webhook'));
'waits' => ['redis:default' => 60, 'redis:imports' => 600],   // seconds before alerting
```

---

## 16. Laravel Octane & Long-Lived Processes

Octane boots the application **once** and serves many requests from memory (Swoole, RoadRunner, or FrankenPHP). This removes bootstrap cost (~20–50ms per request) but breaks PHP's shared-nothing guarantee.

### What breaks

```php
// 1. Singleton holding request state — LEAKS ACROSS TENANTS
$this->app->singleton(TenantContext::class);     // ❌ use scoped()

// 2. Static properties persist forever
class Counter { public static int $count = 0; }  // grows across requests

// 3. Container instances captured in constructors go stale
class Service
{
    public function __construct(private Request $request) {}   // ❌ frozen first request
}
// ✅ resolve per call, or inject a Closure/callable that resolves fresh

// 4. Closures capturing $this keep whole object graphs alive
Event::listen('x', function () use ($hugeObject) {});   // retained forever

// 5. Config mutated at runtime persists into other requests
config(['app.timezone' => $tenant->timezone]);   // ❌ leaks

// 6. Auth/session state must be flushed between requests
```

```php
// config/octane.php — flush stateful services between requests
'flush' => [
    TenantContext::class,
    RequestCorrelationId::class,
],

'warm' => [
    // Services safe and expensive to pre-boot
    \Illuminate\Cache\CacheManager::class,
    \Illuminate\Database\DatabaseManager::class,
],

'listeners' => [
    RequestReceived::class => [...],
    RequestTerminated::class => [FlushTemporaryContainerInstances::class],
],

'max_execution_time' => 30,
```

```php
// Octane-specific goodies
Octane::concurrently([
    fn () => InventoryItem::count(),
    fn () => Order::sum('total_cents'),
    fn () => Http::get('https://api.supplier.com/status')->json(),
]);   // parallel execution via Swoole task workers

Octane::tick('refresh-stats', fn () => Cache::put('stats', compute()))->seconds(30);
Octane::table('metrics')->set('requests', ['count' => $n]);   // shared memory across workers
```

> **Follow-up:** *Would you run your multi-tenant SaaS on Octane?* Only after an audit. The upside is real (removing 30–50ms of bootstrap on every request matters at scale), but a single `singleton` holding tenant state becomes a **cross-tenant data leak** rather than a slow endpoint. I'd require: all request-scoped bindings converted to `scoped`, an explicit `flush` list, a test that asserts `TenantContext` is empty at the start of each request, memory-leak monitoring per worker, and a staged rollout behind a load balancer weight. And I'd first confirm the bottleneck is actually bootstrap — for a DB-bound API, query optimization usually returns more.

> **Follow-up:** *How do you find a memory leak in an Octane/queue worker?* Log `memory_get_usage(true)` per request/job tagged with the route or job class, look for monotonic growth, then use `--max-requests`/`--max-jobs` as a mitigation while you find the retaining reference — usually a static array, an unbounded in-memory cache, a growing event listener registry, or closures capturing large graphs.

---

## 17. Observability

### Structured logging with tenant context

```php
// Middleware — every log line in the request inherits this context
Log::withContext([
    'request_id'      => $request->header('X-Request-Id') ?? (string) Str::uuid(),
    'organization_id' => $orgId,
    'user_id'         => $user->id,
    'route'           => $request->route()?->getName(),
]);
```

```php
// config/logging.php — JSON to stdout for container log shipping
'channels' => [
    'stack' => ['driver' => 'stack', 'channels' => ['stdout', 'sentry'], 'ignore_exceptions' => false],

    'stdout' => [
        'driver'    => 'monolog',
        'handler'   => StreamHandler::class,
        'with'      => ['stream' => 'php://stdout'],
        'formatter' => JsonFormatter::class,
        'level'     => env('LOG_LEVEL', 'info'),
    ],

    'slow_queries' => ['driver' => 'daily', 'path' => storage_path('logs/slow.log'), 'days' => 7],
],
```

> **Trap:** Logging PII or secrets. Redact tokens, passwords, card numbers, and full request bodies. Use `#[SensitiveParameter]` so credentials don't appear in stack traces, and add a Monolog processor that scrubs known-sensitive keys.

```php
// Monolog processor to scrub sensitive keys
$logger->pushProcessor(function (LogRecord $record) {
    $scrub = ['password', 'token', 'secret', 'authorization', 'card_number'];

    $record->context = collect($record->context)
        ->map(fn ($v, $k) => in_array(strtolower($k), $scrub, true) ? '[REDACTED]' : $v)
        ->all();

    return $record;
});
```

### Correlation across services

```php
final class AssignRequestId
{
    public function handle(Request $request, Closure $next): Response
    {
        $id = $request->header('X-Request-Id') ?: (string) Str::uuid();

        $request->headers->set('X-Request-Id', $id);
        Log::withContext(['request_id' => $id]);

        // Propagate to outbound HTTP calls automatically
        Http::globalRequestMiddleware(fn ($req) => $req->withHeader('X-Request-Id', $id));

        $response = $next($request);
        $response->headers->set('X-Request-Id', $id);

        return $response;
    }
}
```

```php
// Propagate into queued jobs so async work stays correlated
abstract class TenantAwareJob implements ShouldQueue
{
    public string $requestId;

    public function __construct(public readonly int $organizationId)
    {
        $this->requestId = Log::sharedContext()['request_id'] ?? (string) Str::uuid();
    }

    final public function handle(): void
    {
        Log::withContext([
            'request_id'      => $this->requestId,
            'organization_id' => $this->organizationId,
            'job'             => static::class,
        ]);

        // ...
    }
}
```

### OpenTelemetry

```php
// Manual span around a critical operation
$tracer = Globals::tracerProvider()->getTracer('inventory');

$span = $tracer->spanBuilder('stock.deduct')
    ->setSpanKind(SpanKind::KIND_INTERNAL)
    ->startSpan();

$scope = $span->activate();

try {
    $span->setAttribute('organization.id', $orgId);
    $span->setAttribute('item.id', $itemId);
    $span->setAttribute('deduct.amount', $amount);

    $result = $this->action->execute($data);

    $span->setStatus(StatusCode::STATUS_OK);
    return $result;
} catch (Throwable $e) {
    $span->recordException($e);
    $span->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
    throw $e;
} finally {
    $scope->detach();
    $span->end();
}
```

**What to instrument:** HTTP handler (auto), DB queries, Redis calls, outbound HTTP, queue publish and consume (propagate trace context in the job payload so the async work joins the same trace). Tag every span with `organization_id` so you can slice latency per tenant and find the noisy neighbor.

### The four golden signals + queue metrics

| Signal | Metric | Alert threshold example |
|--------|--------|------------------------|
| Latency | p50 / p95 / p99 per route | p95 > 500ms for 5 min |
| Traffic | requests/sec, per tenant | sudden 3× spike from one org |
| Errors | 5xx rate, exception count | > 1% of requests |
| Saturation | php-fpm busy workers, DB connections, Redis memory | > 80% workers busy |
| Queue | depth, oldest job age, failure rate | oldest job > 60s |
| DB | slow queries, replica lag, deadlocks/min, cache hit ratio | lag > 10s |

```php
// Prometheus-style counters
Prometheus::counter('stock_deductions_total')
    ->labels(['organization_id' => $orgId, 'result' => 'success'])
    ->inc();

Prometheus::histogram('stock_deduction_duration_seconds')
    ->labels(['organization_id' => $orgId])
    ->observe($durationSeconds);

Prometheus::gauge('queue_depth')->labels(['queue' => 'inventory'])->set(Queue::size('inventory'));
```

> **Follow-up:** *An endpoint is slow for one customer only. How do you debug it?* Filter traces and slow-query logs by `organization_id` — which works only because you tagged them. Typical causes: that tenant has vastly more rows so a query that index-scans for everyone else seq-scans for them; a missing composite index leading with `organization_id`; an unbounded page size; or a cache key with poor hit rate for their access pattern. Then check whether they're hitting a shared resource (a hot table partition, a queue everyone shares).

---

## 18. Security — OWASP Mapped to Laravel

| OWASP 2021 | Risk in your SaaS | Control |
|-----------|-------------------|---------|
| **A01 Broken Access Control** | IDOR across tenants; missing policy check | Verified global scopes, `resolveRouteBinding`, policies, 404 not 403, isolation test per resource |
| **A02 Cryptographic Failures** | Secrets in env files, unencrypted PII | Secrets Manager, `encrypted` casts, TLS everywhere, `APP_KEY` rotation plan |
| **A03 Injection** | `orderBy($userInput)`, `whereRaw` interpolation | Bindings for values, allowlist for identifiers, no string-built SQL |
| **A04 Insecure Design** | No rate limiting, no idempotency, trusting client IDs | Threat model per feature, tenant-keyed rate limits, idempotency keys |
| **A05 Security Misconfiguration** | `APP_DEBUG=true` in prod, Telescope exposed | Config audit in CI, `APP_DEBUG=false`, gate Telescope/Horizon behind auth |
| **A06 Vulnerable Components** | Outdated packages | `composer audit` in CI, Dependabot, pinned lockfile |
| **A07 Auth Failures** | Credential stuffing, no MFA, long token TTL | Login throttling, MFA, short access tokens, revoke on password change |
| **A08 Data Integrity Failures** | `unserialize()` on user input, unsigned webhooks | Never unserialize untrusted data; verify webhook HMAC signatures |
| **A09 Logging Failures** | No audit trail; PII in logs | Structured logs, audit table, scrubbing processor, retention policy |
| **A10 SSRF** | User-supplied webhook URL fetched by the server | Allowlist hosts, block private IP ranges, no redirects, separate egress |

### The controls that matter most for your SaaS

```php
// 1. SSRF protection — this is easy to get wrong
final class SafeUrlRule implements ValidationRule
{
    private const BLOCKED_CIDRS = [
        '127.0.0.0/8', '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',
        '169.254.0.0/16',    // AWS metadata endpoint — the classic SSRF target
        '::1/128', 'fc00::/7',
    ];

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $parsed = parse_url((string) $value);

        if (! in_array($parsed['scheme'] ?? '', ['https'], true)) {
            $fail('Only HTTPS URLs are allowed.');
            return;
        }

        $ip = gethostbyname($parsed['host'] ?? '');

        foreach (self::BLOCKED_CIDRS as $cidr) {
            if (IpUtils::checkIp($ip, $cidr)) {
                $fail('URL resolves to a disallowed address.');
                return;
            }
        }
    }
}

// Also: disable redirects, set a timeout, and re-check the IP after DNS resolution
Http::withOptions(['allow_redirects' => false])->timeout(5)->get($url);
```

> **Note the DNS rebinding caveat:** validating the IP then fetching separately is a TOCTOU race — an attacker can make DNS return a public IP for validation and a private IP for the fetch. Real defense: resolve once and connect to the resolved IP, or route webhook egress through a proxy that enforces the allowlist.

```php
// 2. Webhook signature verification (constant-time)
final class VerifyWebhookSignature
{
    public function handle(Request $request, Closure $next): Response
    {
        $signature = $request->header('X-Signature');
        $expected  = hash_hmac('sha256', $request->getContent(), config('services.webhook.secret'));

        abort_unless($signature && hash_equals($expected, $signature), 401);

        // Replay protection
        $timestamp = (int) $request->header('X-Timestamp');
        abort_if(abs(time() - $timestamp) > 300, 401, 'Stale webhook');

        return $next($request);
    }
}

// 3. Login throttling
RateLimiter::for('login', fn (Request $r) => [
    Limit::perMinute(5)->by($r->input('email') . '|' . $r->ip()),
    Limit::perMinute(20)->by($r->ip()),
]);

// 4. Encrypted attributes + key rotation awareness
protected function casts(): array
{
    return ['tax_id' => 'encrypted', 'api_credentials' => 'encrypted:array'];
}
// Rotating APP_KEY requires re-encrypting every encrypted column.
// Plan: config('app.previous_keys') supports decrypt-with-old, encrypt-with-new.

// 5. Security headers
$response->headers->add([
    'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains; preload',
    'X-Content-Type-Options'    => 'nosniff',
    'X-Frame-Options'           => 'DENY',
    'Referrer-Policy'           => 'strict-origin-when-cross-origin',
    'Content-Security-Policy'   => "default-src 'self'; frame-ancestors 'none'",
    'Permissions-Policy'        => 'geolocation=(), microphone=(), camera=()',
]);
```

### Audit logging for compliance

```php
final class AuditLogger
{
    public function record(string $action, Model $subject, array $changes = []): void
    {
        AuditLog::create([
            'organization_id' => app(TenantContext::class)->organizationId(),
            'user_id'         => auth()->id(),
            'action'          => $action,
            'auditable_type'  => $subject::class,
            'auditable_id'    => $subject->getKey(),
            'changes'         => $changes,
            'ip_address'      => request()->ip(),
            'user_agent'      => substr((string) request()->userAgent(), 0, 255),
            'request_id'      => request()->header('X-Request-Id'),
            'created_at'      => now(),
        ]);
    }
}
```

Make the audit table append-only (revoke UPDATE/DELETE at the DB role level) so it's trustworthy, and partition it by month so retention pruning is a partition drop rather than a mass delete.

---

## 19. PHP Internals for Seniors

### Memory and the GC

```php
memory_get_usage();          // PHP-tracked
memory_get_usage(true);      // real allocated from OS
memory_get_peak_usage(true);
gc_collect_cycles();
gc_status();
```

PHP frees by refcount immediately; cyclic references need the cycle collector, which runs when the root buffer fills (10,000 possible roots) or on explicit call. In **long-lived processes** (queue workers, Octane) cycles accumulate — which is why `--max-jobs` and `pm.max_requests` exist as pragmatic mitigations.

### OPcache and preloading

```ini
opcache.enable=1
opcache.memory_consumption=512
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=30000
opcache.validate_timestamps=0
opcache.preload=/var/www/preload.php        ; PHP 7.4+
opcache.preload_user=www-data
opcache.jit=tracing
opcache.jit_buffer_size=128M
```

**Preloading** loads and links classes into shared memory at server start, eliminating per-request include and linking cost entirely. It's very effective for frameworks — but preloaded files can't change without a full php-fpm restart, and Laravel's dynamic container makes a complete preload list awkward.

### php-fpm sizing (be able to do this arithmetic on a whiteboard)

```
avg worker RSS         = 80 MB
container memory limit = 4096 MB
reserved for nginx/OS  = 512 MB
────────────────────────────────
pm.max_children = (4096 - 512) / 80 ≈ 44

Also bound by the database:
  Postgres max_connections = 200
  6 app tasks × 44 workers = 264 connections   ← exceeds the limit
  → either lower max_children, or add PgBouncer (the right answer)
```

**Key insight:** more workers do not help when the bottleneck is downstream. If every request waits 200ms on Postgres, doubling workers just doubles the queue at the database. Fix the query, add caching, or add capacity where the contention actually is.

### `fastcgi_finish_request`

```php
// Send the response, then keep working (php-fpm only)
fastcgi_finish_request();
$this->doExpensiveCleanup();
```

This is what powers Laravel's terminable middleware and `dispatch(...)->afterResponse()`. It does not exist under CLI or Octane.

### Fibers (8.1+)

The primitive under async runtimes (ReactPHP, Amp, Swoole coroutines). Fibers allow suspending and resuming a call stack, enabling non-blocking I/O without callback pyramids. Laravel itself is synchronous, but Octane/Swoole uses coroutines under the hood.

> **Follow-up:** *Is PHP suitable for high-concurrency workloads?* Traditional php-fpm handles concurrency with processes — simple and robust, but memory-bound (one process per in-flight request). For I/O-bound fan-out (calling 50 microservices), that's inefficient compared to Go's goroutines. Options: Octane/Swoole coroutines, `Http::pool()` for concurrent outbound requests, or — the answer I'd give given your background — **use the right tool**: keep the CRUD API in Laravel and build the concurrency-heavy component in Go, which is exactly the reasoning behind your Chronos scheduler.

```php
// Concurrent outbound HTTP without leaving Laravel
$responses = Http::pool(fn (Pool $pool) => [
    $pool->as('stock')   ->get('https://api.supplier.com/stock'),
    $pool->as('pricing') ->get('https://api.supplier.com/pricing'),
    $pool->as('lead')    ->get('https://api.supplier.com/lead-times'),
]);

$stock = $responses['stock']->json();
```

---

## 20. Tier 3 Q&A Drill

### Multi-tenancy

1. **Compare row-level, schema-per-tenant, and database-per-tenant.**  
   Row-level: cheapest, single migration run, cross-tenant analytics easy, but isolation is app-enforced and one bug affects everyone. Schema-per-tenant: DB-enforced isolation, N migration runs, Postgres degrades past a few thousand schemas. Database-per-tenant: strongest isolation and per-tenant backup/restore, but connection overhead and operational cost scale linearly. Frame it as a business trade-off tied to tenant count, isolation requirements, and cost per tenant.

2. **Name eight ways row-level tenancy leaks.**  
   Raw `DB::table` queries; removed global scopes; unscoped route model binding (IDOR); ungrouped `orWhere`; unnamespaced cache keys; queued jobs without tenant context; scheduled commands; cross-tenant foreign keys; global `unique`/`exists` validation; unfiltered search index; shared file paths; public broadcast channels; aggregate/report endpoints; super-admin gate bypass; Octane singleton state.

3. **Fail open or fail closed when tenant context is missing?**  
   Fail closed — `whereRaw('1=0')` in HTTP contexts. Failing open turns a forgotten middleware into a full cross-tenant dump; failing closed turns it into an obvious empty-list bug caught in QA. Allow console contexts through for migrations and backfills.

4. **How does the DB prevent an item from referencing another tenant's supplier?**  
   A composite foreign key: add `UNIQUE (organization_id, id)` on suppliers, then `FOREIGN KEY (organization_id, supplier_id) REFERENCES suppliers (organization_id, id)`. Structurally impossible, not merely validated. Pair with a scoped `exists` rule for a good error message.

5. **How do you prove tenant isolation rather than assume it?**  
   A per-resource isolation suite: list returns only own rows; fetching another tenant's ID returns 404; update/delete of a foreign record returns 404 and doesn't mutate; a forged `organization_id` in the payload is ignored; cached responses differ per tenant. Plus an architecture test asserting every model with an `organization_id` column uses the tenant trait.

6. **Handle noisy neighbors.**  
   Rate limits keyed on `organization_id`; dedicated queues/supervisors for large tenants; per-tenant query budgets and slow-query alerts tagged by org; table partitioning by tenant for the largest tables; and a dedicated shard or database as a premium tier.

7. **GDPR deletion for one tenant.**  
   FK cascade for relational data, then purge derived stores: search index, tenant-namespaced cache keys, S3 prefix, queued jobs, logs; document the backup retention window since backups can't be surgically edited; emit an auditable deletion certificate; verify with per-table zero-row queries.

### Concurrency

8. **Two users buy the last unit. Give four solutions with trade-offs.**  
   Atomic conditional UPDATE (fastest, simple counters only). `SELECT FOR UPDATE` in a short transaction (handles complex logic, serializes the row). Optimistic version column (great at low contention, retries at high). Append-only ledger with advisory lock (audit trail, scales, more moving parts). Choose based on contention rate and whether you need an audit trail.

9. **Why isn't a Redis lock enough for "never oversell"?**  
   It isn't safe across failover, and the TTL can expire mid-operation letting a second process in. Redis locks are for efficiency (avoiding duplicate work), not correctness. Enforce hard invariants in the database with constraints, conditional UPDATEs, or row locks.

10. **Explain write skew and why REPEATABLE READ doesn't stop it.**  
    Two transactions each read a set, each writes based on that read, and together they break an invariant neither broke alone — e.g. both deactivate a warehouse after each observes two active. REPEATABLE READ guarantees a stable snapshot but doesn't detect the write-write conflict on different rows. Fix with SERIALIZABLE, `FOR UPDATE` on the whole set, an advisory lock, or a DB constraint.

11. **Postgres vs MySQL default isolation, and why it matters.**  
    Postgres READ COMMITTED, MySQL InnoDB REPEATABLE READ. The same code exhibits different anomalies on each — for example non-repeatable reads inside a transaction occur on Postgres by default but not MySQL. You must know your engine before reasoning about correctness.

12. **What must you do if you raise isolation to SERIALIZABLE in Postgres?**  
    Retry on SQLSTATE `40001`. Postgres uses Serializable Snapshot Isolation, which aborts rather than blocks. Without retry logic, raising the isolation level makes the app less reliable.

13. **How do you prevent deadlocks in a multi-item order?**  
    Acquire row locks in a deterministic order (sort item IDs ascending), keep transactions short, set `lock_timeout`, and retry with `DB::transaction(..., attempts: 3)`. Deadlocks are expected under concurrency, not exceptional.

14. **Design an idempotent order endpoint.**  
    Require an `Idempotency-Key` header; middleware hashes it with tenant and route, takes an atomic lock to serialize concurrent duplicates, and replays a stored response on retry. The durable guarantee is a `UNIQUE (organization_id, idempotency_key)` index — catch the unique-violation and return the original record rather than doing a check-then-insert, which races.

15. **What delivery guarantee do queues give, and what follows from it?**  
    At-least-once. Handlers must be idempotent, ideally enforced by a unique index on a natural or supplied key rather than a cache check.

### Performance

16. **Tell me about a significant performance win.**  
    Use the five-step frame: measure baseline (520 queries, p95 2.4s), profile the causes (resource-layer lazy loads, per-row counts, per-row "latest" lookups, per-row permission hydration, uncached aggregates), fix each with the appropriate tool (eager loading with column selection, `withCount`, correlated subquery selects, eager-loaded permissions, tenant-namespaced Redis cache), verify (62 queries, p95 310ms, 88% reduction), then prevent regression (`preventLazyLoading`, query-count assertions in CI, slow-query logging, per-request query budget).

17. **How do you decide which query to optimize?**  
    Total time (`calls × mean_exec_time`) from `pg_stat_statements`, not slowest single execution. A 5ms query run 100k times costs far more than a 2s query run twice.

18. **How do you read `EXPLAIN ANALYZE`?**  
    Look for seq scans on large tables with selective filters, large "Rows Removed by Filter", estimate-vs-actual divergence (stale stats → `ANALYZE`), external merge sorts (raise `work_mem` or add an ordering index), and high shared-read vs shared-hit ratios (poor cache locality).

19. **Composite index column order rule?**  
    Equality columns first, then range/sort columns. An index on `(a,b,c)` serves left prefixes only. In a multi-tenant app, `organization_id` leads every index.

20. **Give five reasons an index isn't used.**  
    Function applied to the column, leading wildcard `LIKE`, type mismatch, not a left prefix, low selectivity, stale statistics, or the table is small enough that a seq scan wins.

21. **How do you add an index to a 15M-row live table?**  
    `CREATE INDEX CONCURRENTLY`, which takes only SHARE UPDATE EXCLUSIVE so reads and writes continue. It can't run inside a transaction (`public $withinTransaction = false` in the migration), takes two scans, and leaves an INVALID index if it fails — so check and retry.

22. **When do you add a read replica?**  
    After caching and query optimization are exhausted and reads dominate. Set `sticky = true` so reads after a write in the same request hit the primary; route reporting, exports, and analytics to replicas; monitor replication lag and add backpressure to backfills.

23. **Why does a PHP app exhaust database connections, and what fixes it?**  
    Shared-nothing means one connection per php-fpm worker. 6 servers × 40 workers = 240 connections against a default `max_connections` of 100. Fix with PgBouncer in **transaction** pool mode — with the caveats that session state, session advisory locks, `LISTEN/NOTIFY`, and prepared statements don't survive, so use `pg_advisory_xact_lock` and emulated prepares.

### Migrations

24. **Give the zero-downtime playbook for adding a NOT NULL column to 15M rows.**  
    Expand: add the column nullable (metadata-only on PG 11+). Dual-write from the app so new rows are correct. Backfill in keyset-paginated batches with a checkpoint, inter-batch sleep, and replica-lag backpressure. Switch reads. Enforce NOT NULL via `ADD CONSTRAINT ... CHECK ... NOT VALID`, then `VALIDATE CONSTRAINT` (concurrent-friendly), then `SET NOT NULL`. Verify with a reconciliation count. Contract after a soak period.

25. **Why never `renameColumn` on a big hot table?**  
    Old code breaks the instant the rename lands, and during a rolling deploy both versions run simultaneously. Use add → dual-write → backfill → switch reads → stop writing old → drop.

26. **Why is a "brief" ACCESS EXCLUSIVE lock still dangerous?**  
    It queues behind any long-running query, and every subsequent query queues behind it. A migration waiting on a 5-minute analytics query blocks all traffic to that table for 5 minutes. Set `lock_timeout` and retry.

27. **What's your rollback plan for a large migration?**  
    At every stage before the final drop, rollback is a code deploy, not a data restore, because the old column still exists and is populated. The backfill is idempotent and resumable via checkpoint, so a partial run is safe to re-run. That property is the entire point of expand/contract.

28. **How do you prove the backfill was complete?**  
    A reconciliation query asserting zero remaining unmigrated rows before advancing a stage, plus a comparison against a pre-migration row-count snapshot, plus a soak period during which old and new columns are compared for divergence.

29. **Deploy order: migrate before or after shifting traffic?**  
    Additive, backward-compatible changes before. Destructive changes (dropping columns, tightening constraints) only after the deploy that stops using the old shape.

### Architecture

30. **When would you NOT use Clean Architecture / repositories?**  
    For CRUD-heavy modules where the domain logic is trivial — the abstractions are pure ceremony. Apply the layering where business rules are complex and long-lived (stock deduction, pricing, order state machines). Uniform architecture across an entire codebase usually indicates dogma rather than judgment.

31. **What's the dual-write problem and how does the outbox pattern solve it?**  
    Writing to the DB and publishing to a broker are separate systems with no shared transaction, so you can get phantom events (published then rolled back) or lost events (committed then publish failed). The outbox writes the event to a table in the same transaction; a relay reads unpublished rows with `FOR UPDATE SKIP LOCKED` and publishes them, giving at-least-once delivery with a consistent database.

32. **Why `SKIP LOCKED`?**  
    It lets multiple relay/worker instances claim different rows concurrently without blocking each other — the SQL primitive for a safe work queue. Same mechanism prevents two schedulers claiming the same job in a distributed scheduler.

33. **When is CQRS worth it?**  
    When read and write workloads differ fundamentally in shape and scale — transactional single-row writes versus large cross-entity aggregations. It costs eventual consistency and two models to maintain. Not a default; applying it to CRUD is a classic over-engineering failure.

34. **Event sourcing — would you use it?**  
    Rarely as a system-wide architecture. The ledger pattern (append-only movements plus a maintained rollup) gives most of the audit and reconciliation benefit at a fraction of the complexity. Full event sourcing only where auditability is a hard regulatory requirement in a bounded context.

35. **Give a concrete Open/Closed example from your work.**  
    Alert channels for low-stock notifications: an `AlertChannel` interface with implementations tagged in the container, so adding a webhook channel is a new class plus a tag — no modification to the dispatcher. The deduction logic, which never changed, stayed closed.

### Operations & security

36. **Why autoscale queue workers on depth rather than CPU?**  
    An I/O-blocked worker has low CPU while the backlog grows, so CPU-based scaling never triggers. Publish queue depth and oldest-job age as metrics and scale on those.

37. **What breaks under Octane and why does it matter more for multi-tenancy?**  
    Singletons, statics, captured request objects, and mutated config persist across requests. In a multi-tenant app, persisted tenant state is a cross-tenant data leak rather than a mere bug. Convert request-scoped bindings to `scoped`, configure the `flush` list, and add a test asserting tenant context is empty at request start.

38. **How do you correlate a request across API and queue workers?**  
    Assign or accept an `X-Request-Id`, put it in the log context, propagate it on outbound HTTP calls, and carry it in the job payload so the worker re-establishes the same context. With OpenTelemetry, propagate trace context in the job payload so async spans join the same trace.

39. **What do you tag every log line and span with in a multi-tenant app?**  
    `request_id`, `organization_id`, `user_id`, and route/job name. Without `organization_id` you cannot answer "why is it slow for this one customer."

40. **Walk through preventing SSRF on a user-supplied webhook URL.**  
    Require HTTPS, resolve the host and reject private/link-local ranges including 169.254.169.254, disable redirects, set a short timeout — and note the DNS-rebinding TOCTOU: validating then fetching separately is racy, so connect to the resolved IP or route egress through an enforcing proxy.

41. **How do you verify an incoming webhook?**  
    HMAC-SHA256 over the raw body compared with `hash_equals`, plus a timestamp header with a tolerance window to prevent replay, plus idempotency on the event ID.

42. **What's the risk of `Gate::before` in a multi-tenant SaaS?**  
    A super-admin bypass returns true before every policy, including tenant boundaries — support staff can then read any customer's data, which may be a compliance violation. Require explicit, time-boxed, audited impersonation instead.

43. **How would you size php-fpm workers?**  
    `(container_memory - reserved) / avg_worker_RSS`, then cross-check against the database connection ceiling (`app_tasks × workers ≤ max_connections`, or add PgBouncer). And note that more workers don't help when the bottleneck is downstream — they just deepen the queue at the database.

44. **Is PHP suitable for high-concurrency I/O fan-out?**  
    Process-per-request is memory-bound for that workload. Mitigations within Laravel: `Http::pool()` for concurrent outbound calls, Octane/Swoole coroutines. But the honest senior answer is to use the right tool — keep the CRUD API in Laravel and build the concurrency-heavy component in Go, which is the reasoning behind Chronos.

45. **What's your deployment checklist for zero downtime?**  
    Backward-compatible migrations before traffic shift (`migrate --force --isolated`), cache warming after env injection (not at image build), rolling or blue-green with health checks on `/up`, `queue:restart` so workers pick up new code, graceful shutdown period longer than the longest job timeout, and post-deploy verification on error rate, latency, and queue depth before decommissioning old tasks.

---

**Next:** [`05-question-bank.md`](./05-question-bank.md) — 200+ questions with answers, system design prompts, live-coding exercises, and STAR stories built from your real experience.
