# PHP / Laravel — Tier 1: Basic (Deep Foundations)

> Do not skip this tier. Interviewers plant "easy" questions here specifically to test whether your 8 years were deep or repetitive. Type juggling rules, `self` vs `static`, copy-on-write, and the real request lifecycle are where shallow candidates get exposed.

---

## Table of Contents

**Part A — PHP Language & Engine**
1. [How PHP Actually Executes](#1-how-php-actually-executes)
2. [The Type System](#2-the-type-system)
3. [Comparison, Coercion & Type Juggling](#3-comparison-coercion--type-juggling)
4. [Variables, References & Copy-on-Write](#4-variables-references--copy-on-write)
5. [Arrays: The Ordered Hash Map](#5-arrays-the-ordered-hash-map)
6. [Strings](#6-strings)
7. [Functions, Closures & Callables](#7-functions-closures--callables)
8. [OOP Deep Dive](#8-oop-deep-dive)
9. [Enums](#9-enums)
10. [Magic Methods & Late Static Binding](#10-magic-methods--late-static-binding)
11. [Exceptions & Error Handling](#11-exceptions--error-handling)
12. [Generators & Iterators](#12-generators--iterators)
13. [Namespaces, Composer & PSR-4 Autoloading](#13-namespaces-composer--psr-4-autoloading)
14. [PHP 8.0 → 8.4 Feature Timeline](#14-php-80--84-feature-timeline)

**Part B — Laravel Fundamentals**
15. [The Request Lifecycle, Step by Step](#15-the-request-lifecycle-step-by-step)
16. [Service Providers](#16-service-providers)
17. [Facades](#17-facades)
18. [Configuration & Environment](#18-configuration--environment)
19. [Routing](#19-routing)
20. [Controllers](#20-controllers)
21. [Requests, Responses & API Resources](#21-requests-responses--api-resources)
22. [Validation](#22-validation)
23. [Eloquent Basics](#23-eloquent-basics)
24. [Migrations, Seeders & Factories](#24-migrations-seeders--factories)
25. [Artisan & Console Commands](#25-artisan--console-commands)
26. [Baseline Security](#26-baseline-security)
27. [Tier 1 Q&A Drill](#27-tier-1-qa-drill)

---

# Part A — PHP Language & Engine

## 1. How PHP Actually Executes

Most candidates say "PHP is interpreted" and stop. That answer loses points. Here is the real pipeline.

```
HTTP Request
   ↓
Nginx  ──(FastCGI)──►  php-fpm master
                            ↓ (dispatches to idle worker)
                       php-fpm worker process
                            ↓
                       1. Lexing      (source → tokens)
                       2. Parsing     (tokens → AST)
                       3. Compilation  (AST → opcodes)   ← OPcache caches THIS
                       4. Execution    (Zend VM runs opcodes)
                            ↓
                       Response → Nginx → Client
                            ↓
                       Worker resets memory, waits for next request
```

### Key facts a senior should state

**Shared-nothing architecture.** Each request gets a fresh process state. Globals, statics, and in-memory objects are destroyed at request end. This is why PHP scales horizontally so easily — and why Laravel Octane (which keeps state alive) breaks naive code.

**OPcache** stores compiled opcodes in shared memory so steps 1–3 are skipped on subsequent requests. Without OPcache, PHP re-parses every included file on every request. In production this is typically a 2–3× throughput difference.

```ini
; production php.ini
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0   ; never stat files; requires restart/reload on deploy
opcache.jit=tracing             ; PHP 8+; marginal for typical web I/O-bound work
opcache.jit_buffer_size=64M
```

**JIT** (8.0+) compiles hot opcodes to machine code. It helps CPU-bound numeric work; it barely helps typical web apps that are I/O bound on DB and Redis. Saying "JIT made our API 3× faster" is a red flag; saying "JIT is mostly irrelevant for I/O-bound CRUD, we got more from query and cache work" is a green flag.

**SAPI** (Server API) is the interface between PHP and its host: `fpm-fcgi`, `cli`, `cli-server`, `apache2handler`, `swoole` (via Octane). Different SAPIs mean different lifetimes — `cli` has no request timeout by default, `fpm` does.

### php-fpm process management

```ini
pm = dynamic
pm.max_children = 40         ; hard ceiling on concurrent PHP requests
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 15
pm.max_requests = 500        ; recycle worker to bound memory leaks
```

**How to size `pm.max_children`:** `available_RAM_for_PHP / avg_worker_memory`. If a worker averages 80 MB and you allocate 4 GB, that's ~50. Over-provisioning causes swapping; under-provisioning causes 502s under load.

> **Trap:** "How would you handle a traffic spike on your Laravel API?" A junior answer is "add more servers." A senior answer covers: check whether workers are saturated (`fpm` status page), whether they're blocked on DB (then more workers makes it worse), connection limits on Postgres, and whether the work belongs in a queue at all.

> **Follow-up:** *Why do 502 Bad Gateway errors appear under load?* All php-fpm workers busy → backlog full → Nginx gets no response within `fastcgi_read_timeout`. Root cause is usually slow downstream (DB lock, external API) not PHP itself.

---

## 2. The Type System

### Scalar and compound types

```php
<?php

declare(strict_types=1);

// Scalars
int $qty = 100;
float $price = 19.99;
string $sku = 'WIDGET-01';
bool $active = true;

// Compound
array $tags = ['a', 'b'];
object $o = new stdClass();
callable $fn = fn (int $x): int => $x * 2;
iterable $it = [1, 2, 3];      // array | Traversable

// Special
null $nothing = null;
mixed $anything = 'any type at all';
```

### Type declarations across PHP versions

```php
// Nullable (7.1+)
function find(?int $id): ?InventoryItem {}

// Union types (8.0+)
function parseQty(int|string $raw): int {}

// Intersection types (8.1+)
function process(Countable&Traversable $collection): void {}

// DNF — Disjunctive Normal Form (8.2+)
function handle((Countable&Traversable)|array $input): void {}

// never — function never returns (8.1+)
function fail(string $msg): never
{
    throw new RuntimeException($msg);
}

// void — returns nothing
function log(string $msg): void {}

// static — returns instance of called class (8.0+), enables fluent chains
class Query
{
    public function where(string $col): static
    {
        return $this;
    }
}
```

### `declare(strict_types=1)` — what it really does

It affects **only the file it appears in**, and only for *calls made from that file*. It changes how scalar arguments and return values are checked at the boundary.

```php
// file: strict.php
declare(strict_types=1);

function double(int $n): int { return $n * 2; }

double('5');   // TypeError: must be of type int, string given
double(5.0);   // TypeError (even though 5.0 is integral)
double(true);  // TypeError
```

```php
// file: coercive.php  (no declare)
function double(int $n): int { return $n * 2; }

double('5');    // 10   — numeric string coerced
double(5.9);    // 10   — float truncated, emits Deprecated in 8.1+ for lossy
double(true);   // 2    — bool coerced
double('abc');  // TypeError — non-numeric string still fails
```

> **Trap:** "Does `strict_types` apply to the whole application?" No — it is per-file, and it applies to the *caller's* file, not the callee's. If your file is strict and you call a non-strict vendor function, strict rules still apply to your call.

> **Follow-up:** *Why would you enable it?* Fails fast on real bugs, documents intent, prevents silent precision loss. On a trading platform, silent `'19.99'` → `19` truncation is a financial incident. Also: static analysis (PHPStan/Psalm) is far more effective with strict types.

> **Follow-up:** *What about float money?* Don't use floats for money. Use integer minor units (cents) or a decimal library (`brick/math`), and `DECIMAL`/`NUMERIC` in Postgres. Floats give `0.1 + 0.2 !== 0.3`.

```php
// Why floats fail for money
var_dump(0.1 + 0.2 === 0.3);        // false
var_dump(0.1 + 0.2);                // float(0.30000000000000004)

// Correct: integer cents
final readonly class Money
{
    public function __construct(
        public int $amountInCents,
        public string $currency = 'USD',
    ) {}

    public function add(self $other): self
    {
        if ($other->currency !== $this->currency) {
            throw new InvalidArgumentException('Currency mismatch');
        }
        return new self($this->amountInCents + $other->amountInCents, $this->currency);
    }
}
```

---

## 3. Comparison, Coercion & Type Juggling

This is the single most common "gotcha" question in PHP interviews.

### `==` (loose) vs `===` (identical)

`===` requires same type AND same value. For objects, `===` means *same instance*; `==` means same class with `==`-equal properties.

```php
$a = new stdClass(); $a->x = 1;
$b = new stdClass(); $b->x = 1;

var_dump($a == $b);   // true  — same class, equal properties
var_dump($a === $b);  // false — different instances
```

### PHP 8 changed string ↔ number comparison (important!)

Before PHP 8, `0 == 'foo'` was `true` (string cast to 0). PHP 8 made it saner: if the string is **not numeric**, the number is cast to string instead.

| Expression | PHP 7 | PHP 8 |
|-----------|-------|-------|
| `0 == 'foo'` | `true` ⚠️ | `false` ✅ |
| `0 == ''` | `true` ⚠️ | `false` ✅ |
| `0 == '0'` | `true` | `true` |
| `'1' == '01'` | `true` | `true` (both numeric) |
| `'10' == '1e1'` | `true` | `true` |
| `100 == '1e2'` | `true` | `true` |
| `'abc' == 0` | `true` ⚠️ | `false` ✅ |

### The loose-truthiness table (memorize)

| Value | `(bool)` | `== null` | `empty()` | `isset()` |
|-------|----------|-----------|-----------|-----------|
| `null` | `false` | `true` | `true` | `false` |
| `false` | `false` | `true` | `true` | `true` |
| `0` | `false` | `true` | `true` | `true` |
| `0.0` | `false` | `true` | `true` | `true` |
| `''` | `false` | `true` | `true` | `true` |
| `'0'` | `false` | `false` | `true` ⚠️ | `true` |
| `'0.0'` | `true` ⚠️ | `false` | `false` | `true` |
| `'false'` | `true` | `false` | `false` | `true` |
| `[]` | `false` | `true` | `true` | `true` |
| `[0]` | `true` | `false` | `false` | `true` |
| `-1` | `true` ⚠️ | `false` | `false` | `true` |
| `new stdClass` | `true` | `false` | `false` | `true` |

> **Trap:** `'0'` is falsy but `'0.0'` is truthy. And `-1` is truthy. This causes real bugs: `if (!$quantity)` treats `0` and `null` and `''` identically — usually wrong for inventory where `0` is a legitimate stock level.

```php
// BAD — 0 stock is a valid value but gets treated as "missing"
if (!$item->quantity) {
    throw new Exception('Quantity missing');
}

// GOOD
if ($item->quantity === null) {
    throw new Exception('Quantity missing');
}
```

### Null-handling operators

```php
$name = $data['name'] ?? 'unknown';         // null coalescing (isset-based, no notice)
$config['retries'] ??= 3;                   // null coalescing assignment (7.4+)
$city = $user?->address?->city;              // nullsafe chain (8.0+) — short-circuits to null
$label = $flag ? 'on' : 'off';
$value = $input ?: 'fallback';               // Elvis — uses truthiness, NOT isset (emits notice if unset)
```

> **Trap:** `?:` vs `??`. `?:` checks truthiness (so `'0'` and `0` fall through to the default — often a bug). `??` checks only null/unset.

```php
$qty = $request->input('quantity') ?: 10;   // BUG: sending 0 gives 10
$qty = $request->input('quantity') ?? 10;   // Correct: only missing/null gives 10
```

### `match` vs `switch`

```php
// switch: loose comparison (==), fallthrough, no return value
switch ($status) {
    case 1:
    case '1':      // both match; loose
        $label = 'active';
        break;
    default:
        $label = 'unknown';
}

// match (8.0+): strict comparison (===), no fallthrough, is an expression, throws if unhandled
$label = match ($status) {
    1, 2 => 'active',
    3 => 'archived',
    default => 'unknown',
};

// match(true) for conditional chains — very idiomatic
$tier = match (true) {
    $qty === 0        => 'out_of_stock',
    $qty < 10         => 'low',
    $qty < 100        => 'normal',
    default           => 'high',
};
```

> **Follow-up:** *What happens with `match` and no `default`?* `UnhandledMatchError` is thrown. This is a feature — it makes non-exhaustive handling loud instead of silent.

---

## 4. Variables, References & Copy-on-Write

### Value semantics with copy-on-write (CoW)

PHP does not copy an array on assignment. It increments a refcount and only copies when one side is modified.

```php
$a = range(1, 1_000_000);   // ~32 MB
$b = $a;                     // no copy yet — refcount = 2, memory unchanged
$b[] = 'x';                  // NOW it copies — memory doubles
```

```php
function inspect(array $rows): int    // by value; no copy while read-only
{
    return count($rows);
}

function mutate(array $rows): array   // copy happens on first write inside
{
    $rows[0] = 'changed';
    return $rows;
}
```

> **Trap / real relevance:** In your 15M-record migration, passing a large array by value into a function that writes to it silently doubles memory. This is exactly why you chunk and why generators matter (§12).

### Objects are handles, not values

```php
class Item { public int $qty = 5; }

function bump(Item $i): void { $i->qty++; }   // mutates the caller's object

$item = new Item();
bump($item);
echo $item->qty;   // 6  — objects are passed by handle
```

```php
// To get a true copy you must clone — and shallow clone is not enough for nested objects
$copy = clone $item;
```

### Explicit references

```php
$a = 1;
$b = &$a;    // $b and $a are the same variable slot
$b = 2;
echo $a;     // 2

// Reference in foreach — classic bug source
$prices = [10, 20, 30];
foreach ($prices as &$p) { $p *= 2; }
unset($p);   // ALWAYS unset — otherwise $p still references the last element
```

```php
// The infamous bug if you forget unset()
$arr = [1, 2, 3];
foreach ($arr as &$v) {}
foreach ($arr as $v) {}      // $v is still a reference to $arr[2]
print_r($arr);               // [1, 2, 2]  ← surprising
```

### Garbage collection

PHP uses **reference counting** plus a **cycle collector**. Refcount frees immediately when it hits zero. Circular references need the cycle GC (`gc_collect_cycles()`).

```php
class Node { public ?Node $next = null; }

$a = new Node();
$b = new Node();
$a->next = $b;
$b->next = $a;     // cycle — refcount never reaches 0
unset($a, $b);     // leaked until gc runs
gc_collect_cycles();
```

> **Follow-up:** *When does this matter?* In long-running processes: queue workers, Octane, and long Artisan commands. A worker leaking cycles will OOM. Mitigations: `pm.max_requests`, `queue:work --max-jobs`, `--max-time`, and Horizon's memory limit.

---

## 5. Arrays: The Ordered Hash Map

A PHP array is a single data structure that behaves as list, map, stack, and queue. Internally (PHP 7+) it is a `zend_array` — a hash table with an ordered bucket list.

```php
// "Packed" array: sequential int keys from 0 — optimized, less memory
$packed = [1, 2, 3];

// "Hashed" array: string or non-sequential keys — full hash table
$hashed = ['sku' => 'A1', 'qty' => 5];

// Unsetting breaks packing
$a = [1, 2, 3];
unset($a[1]);
// keys are now 0, 2 — no longer packed; json_encode gives an OBJECT not an array
echo json_encode($a);            // {"0":1,"2":3}   ← common API bug
echo json_encode(array_values($a)); // [1,3]        ← fix
```

> **Trap (very common in API work):** After `filter`, keys are preserved, so `json_encode` emits an object instead of an array. In Laravel: `$collection->filter()->values()` before returning.

```php
// Laravel equivalent of the same trap
return Item::all()
    ->filter(fn ($i) => $i->quantity > 0)
    ->values()          // ← required, otherwise JSON object with gappy keys
    ->toArray();
```

### Essential array functions (know these cold)

```php
$items = [
    ['sku' => 'A', 'qty' => 5],
    ['sku' => 'B', 'qty' => 0],
    ['sku' => 'C', 'qty' => 12],
];

array_map(fn ($i) => $i['sku'], $items);                  // ['A','B','C']
array_filter($items, fn ($i) => $i['qty'] > 0);           // keys preserved!
array_reduce($items, fn ($c, $i) => $c + $i['qty'], 0);   // 17
array_column($items, 'qty', 'sku');                        // ['A'=>5,'B'=>0,'C'=>12]
array_combine(['a','b'], [1,2]);                           // ['a'=>1,'b'=>2]
array_key_exists('sku', $items[0]);                        // true even if value is null
isset($items[0]['sku']);                                   // FALSE if value is null ⚠️
usort($items, fn ($a, $b) => $a['qty'] <=> $b['qty']);    // spaceship operator
array_unique([1,'1',2]);                                   // loose comparison by default ⚠️
array_merge(['a'=>1], ['a'=>2]);                           // ['a'=>2] string keys overwrite
array_merge([1,2], [3]);                                   // [1,2,3] int keys renumbered
[...['a'=>1], ...['b'=>2]];                                // spread with string keys (8.1+)
```

> **Trap:** `isset()` returns `false` for keys whose value is `null`; `array_key_exists()` returns `true`. This matters when `null` is meaningful (e.g. "field explicitly cleared" vs "field not sent" in a PATCH endpoint).

```php
// PATCH semantics done right
if ($request->has('description')) {          // key present, even if null
    $item->description = $request->input('description');
}
// vs $request->filled('description') which requires a non-empty value
```

---

## 6. Strings

```php
$sku = 'WIDGET';

echo 'No $interpolation and \n literal';       // single quotes: fastest, no parsing
echo "Interpolated: $sku and {$item->name}\n"; // double quotes: parses escapes + vars

// Heredoc — interpolates
$sql = <<<SQL
    SELECT * FROM inventory_items
    WHERE organization_id = {$orgId}
SQL;

// Nowdoc — does NOT interpolate
$template = <<<'TXT'
    Literal $notAVariable
TXT;
```

### Multibyte safety

```php
$name = 'Café';

strlen($name);        // 5  — BYTES
mb_strlen($name);     // 4  — CHARACTERS
strtoupper($name);    // 'CAFé' — broken
mb_strtoupper($name); // 'CAFÉ' — correct
substr($name, 0, 3);  // may cut mid-byte → mojibake
mb_substr($name, 0, 3);
```

> **Trap:** Truncating user input with `substr` for a `VARCHAR(255)` column can produce invalid UTF-8 and a DB error. Use `mb_substr`, or better, validate length with `max:255` (Laravel counts characters for strings).

### Safe comparison for secrets

```php
// VULNERABLE to timing attacks — returns early on first differing byte
if ($providedToken === $storedToken) {}

// CORRECT — constant time
if (hash_equals($storedToken, $providedToken)) {}

// Passwords: never compare hashes manually
password_verify($plain, $hash);       // native
Hash::check($plain, $user->password); // Laravel
```

---

## 7. Functions, Closures & Callables

```php
// Default + variadic + named args
function makeQuery(string $table, string ...$columns): string
{
    return sprintf('SELECT %s FROM %s', implode(', ', $columns ?: ['*']), $table);
}

makeQuery('items', 'id', 'sku');

// Named arguments (8.0+) — skip optional params, self-documenting
function paginate(int $page = 1, int $perPage = 25, bool $withTrashed = false) {}
paginate(perPage: 100, withTrashed: true);
```

> **Trap:** Named arguments make parameter names part of your public API. Renaming a parameter becomes a breaking change for library code.

### Closures: `use` by value vs by reference

```php
$multiplier = 2;

$byValue = function (int $n) use ($multiplier): int {
    return $n * $multiplier;   // captured at DEFINITION time
};

$byRef = function (int $n) use (&$multiplier): int {
    return $n * $multiplier;   // reads CURRENT value at CALL time
};

$multiplier = 10;

echo $byValue(5);   // 10  (used 2)
echo $byRef(5);     // 50  (used 10)
```

### Arrow functions (7.4+) capture automatically by value

```php
$multiplier = 3;
$fn = fn (int $n): int => $n * $multiplier;   // implicit by-value capture, single expression only
```

### `$this` binding and `static` closures

```php
class InventoryService
{
    private int $threshold = 10;

    public function lowStockFilter(): Closure
    {
        // $this is bound automatically inside a method's closure
        return fn (Item $i) => $i->quantity < $this->threshold;
    }

    public function stateless(): Closure
    {
        // static closure: $this NOT bound — prevents accidental object retention
        return static fn (Item $i) => $i->quantity < 10;
    }
}
```

> **Trap / real relevance:** A non-static closure captures `$this`. If that closure is stored in a long-lived container binding or a queued job, it keeps the whole object graph alive → memory leak in workers/Octane. Use `static fn` when you don't need `$this`.

### First-class callable syntax (8.1+)

```php
$strlen = strlen(...);                     // Closure from function
$method = $service->process(...);          // Closure from instance method
$static = InventoryService::make(...);     // Closure from static method

// Very useful in collections
$skus = $items->map(fn ($i) => $i->sku);
$names = $items->map(Item::displayName(...));
```

### `Closure::bind` — how Laravel does some of its magic

```php
class Locked { private string $secret = 'hidden'; }

$reader = function () { return $this->secret; };
$bound = Closure::bind($reader, new Locked(), Locked::class);
echo $bound();   // 'hidden'
```

Laravel uses this for macros (`Macroable`), route model binding internals, and testing helpers.

---

## 8. OOP Deep Dive

### Constructor property promotion + readonly

```php
final class InventoryItem
{
    public function __construct(
        public readonly int $id,
        public readonly int $organizationId,
        public readonly string $sku,
        private int $quantity = 0,
    ) {}

    public function quantity(): int
    {
        return $this->quantity;
    }
}
```

`readonly` (8.1+) can be assigned exactly once, from inside the declaring class scope. Attempting reassignment throws `Error`.

```php
$item = new InventoryItem(1, 42, 'ABC');
$item->id = 2;   // Error: Cannot modify readonly property
```

`readonly class` (8.2+) makes all properties readonly:

```php
final readonly class DeductStockData
{
    public function __construct(
        public int $itemId,
        public int $amount,
        public int $organizationId,
    ) {}
}
```

> **Trap:** `readonly` is shallow. A readonly property holding an array cannot be reassigned, but a readonly property holding an *object* still allows mutating that object's internals.

```php
final readonly class Order
{
    public function __construct(public Collection $lines) {}
}

$o = new Order(collect([1]));
$o->lines->push(2);   // ALLOWED — object internals mutable
$o->lines = collect(); // Error — reassignment blocked
```

### Visibility

| Modifier | Accessible from |
|----------|-----------------|
| `public` | anywhere |
| `protected` | declaring class + subclasses |
| `private` | declaring class only (not subclasses) |

```php
class Base
{
    private function secret(): string { return 'base'; }
    public function call(): string { return $this->secret(); }
}

class Child extends Base
{
    private function secret(): string { return 'child'; }
}

echo (new Child)->call();   // 'base' — private methods are NOT polymorphic
```

> **Trap:** Private methods are resolved at compile time in the declaring class scope; they are not overridable. Change `private` → `protected` and the output becomes `'child'`. This appears in interviews as a "what does this print" puzzle.

### Interfaces vs abstract classes

```php
interface StockRepository
{
    public function findForOrganization(int $id, int $orgId): ?InventoryItem;
    public function decrement(InventoryItem $item, int $by): InventoryItem;
}

abstract class BaseRepository
{
    public function __construct(protected readonly DatabaseManager $db) {}

    abstract protected function table(): string;

    // shared implementation
    protected function scoped(int $orgId): Builder
    {
        return $this->db->table($this->table())->where('organization_id', $orgId);
    }
}

final class EloquentStockRepository extends BaseRepository implements StockRepository
{
    protected function table(): string { return 'inventory_items'; }
    // ...
}
```

| | Interface | Abstract class |
|---|-----------|----------------|
| Multiple inheritance | Yes (implement many) | No (extend one) |
| Constants | Yes | Yes |
| Properties | No (8.x: no properties; 8.4 adds prop hooks in interfaces) | Yes |
| Constructor | No | Yes |
| Method bodies | No (only signatures) | Yes |
| Use when | Defining a contract for swappable impls | Sharing implementation among close relatives |

### Traits — horizontal reuse and its dangers

```php
trait HasOrganization
{
    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }

    // Traits can have abstract members to demand contract from host
    abstract public function getTable(): string;

    // Traits can have static and private members, and boot hooks (Laravel convention)
    protected static function bootHasOrganization(): void
    {
        static::addGlobalScope(new OrganizationScope());
    }
}
```

#### Conflict resolution

```php
trait A { public function hello(): string { return 'A'; } }
trait B { public function hello(): string { return 'B'; } }

class C
{
    use A, B {
        A::hello insteadof B;    // resolve collision
        B::hello as helloFromB;  // alias the other
    }
}

// Change visibility on import
class D
{
    use A { hello as protected; }
}
```

**Precedence:** own class methods > trait methods > inherited parent methods.

```php
trait T { public function who(): string { return 'trait'; } }
class Parent_ { public function who(): string { return 'parent'; } }
class Child extends Parent_ { use T; }

echo (new Child)->who();   // 'trait' — trait beats inherited parent method
```

> **Trap:** "Traits are just copy-paste at compile time." Mostly true, and that's the criticism: they create hidden coupling, cannot be type-hinted, and are hard to mock. Prefer composition (inject a collaborator) when the behavior has dependencies or needs testing in isolation. Traits are fine for framework-integration boilerplate (Laravel's `SoftDeletes`, `HasFactory`).

> **Follow-up:** *How do you test a trait?* Either test it through a real host class (preferred), or create an anonymous test class:

```php
it('scopes queries to organization', function () {
    $model = new class extends Model {
        use HasOrganization;
        protected $table = 'inventory_items';
    };

    expect($model->newQuery()->toSql())->toContain('organization_id');
});
```

### `static` vs `self` vs `parent` — late static binding

```php
class Model
{
    public static function create(): static   // 'static' = the CALLED class
    {
        return new static();
    }

    public static function createSelf(): self  // 'self' = the DEFINING class
    {
        return new self();
    }
}

class InventoryItem extends Model {}

get_class(InventoryItem::create());      // 'InventoryItem'  ✅
get_class(InventoryItem::createSelf());  // 'Model'          ⚠️
```

> **Trap:** This is *the* classic PHP OOP interview question. `self` resolves at compile time to the class where the code is written. `static` resolves at runtime to the class that was actually called ("late static binding"). Every fluent framework (Eloquent, query builders) depends on `static`.

---

## 9. Enums

```php
// Pure enum — no scalar value
enum StockStatus
{
    case InStock;
    case LowStock;
    case OutOfStock;
}

// Backed enum — has a scalar value (int or string), needed for DB storage
enum MovementType: string
{
    case Inbound    = 'inbound';
    case Outbound   = 'outbound';
    case Adjustment = 'adjustment';
    case Transfer   = 'transfer';

    // Enums can have methods, constants, implement interfaces, use traits
    public function affectsStock(): bool
    {
        return $this !== self::Transfer;
    }

    public function sign(): int
    {
        return match ($this) {
            self::Inbound, self::Adjustment => 1,
            self::Outbound                  => -1,
            self::Transfer                  => 0,
        };
    }

    public function label(): string
    {
        return ucfirst($this->value);
    }

    public static function forApi(): array
    {
        return array_map(
            fn (self $c) => ['value' => $c->value, 'label' => $c->label()],
            self::cases()
        );
    }
}
```

```php
MovementType::cases();                    // all cases
MovementType::from('inbound');            // MovementType::Inbound; throws ValueError if invalid
MovementType::tryFrom('nope');            // null instead of throwing
MovementType::Outbound->value;            // 'outbound'
MovementType::Outbound->name;             // 'Outbound'
```

### Enums in Laravel

```php
class StockMovement extends Model
{
    protected function casts(): array
    {
        return ['type' => MovementType::class];   // auto-cast to/from enum
    }
}

// Validation
'type' => ['required', Rule::enum(MovementType::class)],

// Route binding — Laravel resolves enum params automatically
Route::get('/movements/{type}', fn (MovementType $type) => $type->label());
```

> **Trap:** Enums are singletons — `MovementType::Inbound === MovementType::Inbound` is always `true`, and you cannot instantiate them or add state. They also cannot have (non-constant) properties.

> **Follow-up:** *When would you use an enum vs a DB lookup table?* Enum for a small, code-controlled, rarely-changing set where behavior lives with the value (movement types, order states). Lookup table when non-developers must add values at runtime, or you need per-tenant configurability — which matters for your multi-tenant SaaS where different organizations may want custom categories.

---

## 10. Magic Methods & Late Static Binding

```php
final class Config implements ArrayAccess, Countable, JsonSerializable, Stringable
{
    private array $items = [];

    // Property access
    public function __get(string $key): mixed { return $this->items[$key] ?? null; }
    public function __set(string $key, mixed $value): void { $this->items[$key] = $value; }
    public function __isset(string $key): bool { return isset($this->items[$key]); }
    public function __unset(string $key): void { unset($this->items[$key]); }

    // Method calls
    public function __call(string $method, array $args): mixed
    {
        throw new BadMethodCallException("No method {$method}");
    }
    public static function __callStatic(string $method, array $args): mixed
    {
        return (new static())->$method(...$args);
    }

    // Object as function
    public function __invoke(string $key): mixed { return $this->__get($key); }

    // Casting / serialization
    public function __toString(): string { return json_encode($this->items); }
    public function jsonSerialize(): array { return $this->items; }

    // Cloning
    public function __clone(): void { /* deep-copy nested objects here */ }

    // Lifecycle
    public function __construct() {}
    public function __destruct() {}

    // Serialization hooks (8.1+ replaces __sleep/__wakeup)
    public function __serialize(): array { return $this->items; }
    public function __unserialize(array $data): void { $this->items = $data; }

    // ArrayAccess
    public function offsetExists(mixed $o): bool { return isset($this->items[$o]); }
    public function offsetGet(mixed $o): mixed { return $this->items[$o] ?? null; }
    public function offsetSet(mixed $o, mixed $v): void { $this->items[$o] = $v; }
    public function offsetUnset(mixed $o): void { unset($this->items[$o]); }

    public function count(): int { return count($this->items); }
}
```

### Where Laravel uses which magic method

| Magic method | Laravel usage |
|-------------|---------------|
| `__get` / `__set` | Eloquent attributes (`$item->sku` reads `$attributes['sku']`) |
| `__call` | Eloquent query forwarding (`Item::where()` → Builder), `Macroable` |
| `__callStatic` | **Facades** (`Cache::get()` → resolve from container → call method) |
| `__invoke` | Single-action controllers, invokable rules, pipeline stages |
| `__toString` | `Stringable`, `HtmlString`, route URLs |
| `ArrayAccess` | `$request['key']`, `Collection`, `Fluent` |

### Shallow vs deep clone

```php
class Order
{
    public function __construct(
        public Address $shipping,
        public array $lines = [],
    ) {}

    public function __clone(): void
    {
        // Without this, $clone->shipping is the SAME Address object
        $this->shipping = clone $this->shipping;
        $this->lines = array_map(fn ($l) => clone $l, $this->lines);
    }
}
```

> **Trap:** "What does `clone` do?" A *shallow* copy: scalars and arrays are copied (arrays via CoW), but object properties still point to the same instances. Deep copy requires `__clone`.

> **Follow-up:** *Downside of `__get`/`__call`?* No IDE autocomplete, no static analysis, slower than real properties, and harder to debug. Eloquent pays this cost for developer ergonomics; mitigate with `@property` docblocks or generated model stubs (`barryvdh/laravel-ide-helper`).

---

## 11. Exceptions & Error Handling

### The hierarchy

```
Throwable (interface)
├── Error                          ← engine-level problems (PHP 7+)
│   ├── TypeError                  ← wrong argument/return type
│   ├── ValueError                 ← right type, invalid value (8.0+)
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   ├── UnhandledMatchError        ← match with no default
│   └── ArgumentCountError
└── Exception                      ← application-level problems
    ├── ErrorException
    ├── RuntimeException
    │   ├── OutOfBoundsException
    │   ├── OverflowException / UnderflowException
    │   ├── RangeException
    │   └── UnexpectedValueException
    ├── LogicException
    │   ├── BadFunctionCallException
    │   │   └── BadMethodCallException
    │   ├── DomainException
    │   ├── InvalidArgumentException
    │   ├── LengthException
    │   └── OutOfRangeException
    ├── JsonException
    └── PDOException
```

> **Trap:** `catch (Exception $e)` does **not** catch `TypeError` or any `Error`. To catch everything, use `catch (Throwable $e)`. This bites people whose "global error handling" silently misses type errors.

```php
try {
    $result = 10 / 0;
} catch (Exception $e) {
    // NOT reached — DivisionByZeroError extends Error, not Exception
} catch (DivisionByZeroError $e) {
    // reached
}
```

### `LogicException` vs `RuntimeException` — semantics matter

- **`LogicException`** = a programming bug that should be fixed in code (wrong argument, invalid state, calling a method out of order). Should generally *not* be caught and recovered from.
- **`RuntimeException`** = a condition only detectable at runtime (network failure, insufficient stock, external API error). Recoverable.

```php
final class InsufficientStockException extends RuntimeException
{
    public function __construct(
        public readonly int $itemId,
        public readonly int $requested,
        public readonly int $available,
    ) {
        parent::__construct(
            "Item {$itemId}: requested {$requested}, available {$available}"
        );
    }

    // Laravel: control the HTTP response for this exception
    public function render(): JsonResponse
    {
        return response()->json([
            'error'     => 'insufficient_stock',
            'message'   => $this->getMessage(),
            'item_id'   => $this->itemId,
            'available' => $this->available,
        ], 409);
    }

    // Laravel: control logging context
    public function context(): array
    {
        return ['item_id' => $this->itemId, 'requested' => $this->requested];
    }
}
```

### Exception chaining — never lose the original cause

```php
try {
    $this->externalApi->fetchPricing($sku);
} catch (ConnectionException $e) {
    throw new PricingUnavailableException(
        message: "Pricing lookup failed for {$sku}",
        code: 0,
        previous: $e,          // ← preserves original stack trace
    );
}

// Later
$root = $e;
while ($root->getPrevious()) { $root = $root->getPrevious(); }
```

### `finally` semantics (and the trap)

```php
function f(): string
{
    try {
        return 'try';
    } finally {
        // runs even though we returned
        Log::info('cleanup');
    }
}

function g(): string
{
    try {
        return 'try';
    } finally {
        return 'finally';   // ⚠️ OVERRIDES the try return AND swallows exceptions
    }
}
```

> **Trap:** `return` inside `finally` discards the try block's return value and *swallows in-flight exceptions*. Never return from `finally`.

### Laravel exception handling (11.x)

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->dontReport([InsufficientStockException::class]);

    $exceptions->report(function (PaymentGatewayException $e) {
        Sentry::captureException($e);
    });

    $exceptions->render(function (TenantMismatchException $e, Request $request) {
        // Return 404, not 403 — do not confirm the resource exists
        return response()->json(['message' => 'Not found'], 404);
    });

    $exceptions->shouldRenderJsonWhen(
        fn (Request $request) => $request->is('api/*') || $request->expectsJson()
    );
})
```

> **Follow-up:** *For a multi-tenant API, should cross-tenant access return 403 or 404?* **404.** A 403 confirms the resource exists, which leaks information (an attacker can enumerate other tenants' IDs). Laravel's scoped queries naturally produce 404 via `findOrFail`.

---

## 12. Generators & Iterators

Generators let you iterate huge sequences with constant memory. This is directly relevant to your 15M-row migration.

```php
// Loads 15M rows into memory → OOM
function allIdsBad(): array
{
    return DB::table('inventory_items')->pluck('id')->all();
}

// Constant memory — yields one row at a time
function allIds(): Generator
{
    foreach (DB::table('inventory_items')->orderBy('id')->cursor() as $row) {
        yield $row->id;
    }
}

foreach (allIds() as $id) {
    // memory stays flat
}
```

### Generator mechanics

```php
function counter(int $limit): Generator
{
    for ($i = 1; $i <= $limit; $i++) {
        $received = yield $i;          // yield can RECEIVE a value via send()
        if ($received === 'stop') {
            return 'stopped early';    // return value available via getReturn()
        }
    }
    return 'completed';
}

$gen = counter(5);
echo $gen->current();          // 1
$gen->next();
echo $gen->current();          // 2
$gen->send('stop');            // resumes yield with 'stop'
echo $gen->getReturn();        // 'stopped early'
```

```php
// yield from — delegate to another iterable, flattening
function chunks(): Generator
{
    yield from [1, 2];
    yield from anotherGenerator();
    yield 3;
}
```

> **Trap:** Generators are **one-shot forward-only**. You cannot `count()` them, `foreach` them twice, or access them by index. `Cannot rewind a generator that was already run` is a common error.

### Laravel's generator-backed helpers

```php
// cursor(): one query, hydrates one model at a time (uses PDO unbuffered-ish behavior)
foreach (InventoryItem::where('organization_id', $orgId)->cursor() as $item) {}

// lazy(): chunks under the hood (default 1000), returns LazyCollection
InventoryItem::lazy(1000)->each(fn ($i) => process($i));

// lazyById(): chunks by primary key — SAFE when you MUTATE rows while iterating
InventoryItem::whereNull('status')->lazyById(1000)->each(function ($item) {
    $item->update(['status' => 'active']);
});
```

### `chunk` vs `chunkById` vs `cursor` vs `lazy`

| Method | Queries | Memory | Safe while mutating the filter column? |
|--------|---------|--------|----------------------------------------|
| `chunk(1000)` | N (OFFSET-based) | 1000 models | ❌ **No** — rows shift, records get skipped |
| `chunkById(1000)` | N (`WHERE id > last`) | 1000 models | ✅ Yes |
| `cursor()` | 1 | 1 model (but full result set buffered by driver) | ❌ No |
| `lazy(1000)` | N | 1 model at a time | ❌ No |
| `lazyById(1000)` | N | 1 model at a time | ✅ Yes |

> **Trap — this is a great senior question:** *Why does `chunk()` skip records when you modify the rows you're iterating?* `chunk` uses `LIMIT/OFFSET`. If your loop updates rows so they no longer match the `WHERE`, the result set shrinks and OFFSET now points past records you never saw. `chunkById` uses keyset pagination (`WHERE id > :last ORDER BY id`) and is immune. In your 15M backfill, using `chunk` would have silently missed rows — a data-integrity bug you'd only find via reconciliation counts.

> **Follow-up:** *`cursor()` says "one model at a time" — why can it still OOM?* Because the *database driver* may buffer the entire result set client-side. With MySQL/PDO you need `PDO::MYSQL_ATTR_USE_BUFFERED_QUERY = false`; with Postgres, PDO buffers by default too. For truly huge sets, prefer `lazyById`.

---

## 13. Namespaces, Composer & PSR-4 Autoloading

```php
<?php

namespace App\Domain\Inventory\Actions;

use App\Domain\Inventory\Data\DeductStockData;
use App\Models\InventoryItem;
use Illuminate\Support\Facades\DB;
use function array_map;          // function import
use const PHP_INT_MAX;           // constant import

final class DeductStockAction {}
```

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Modules\\": "modules/"
        },
        "files": ["app/helpers.php"]
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

### How autoloading resolves

1. Code references `App\Models\InventoryItem`.
2. PHP doesn't know it → calls registered autoloaders (`spl_autoload_register`).
3. Composer's autoloader strips the `App\` prefix, maps to `app/`, converts `\` → `/`, appends `.php` → `app/Models/InventoryItem.php`.
4. `require` that file. If the class still isn't defined → fatal error.

```bash
composer dump-autoload -o          # optimized: build a static classmap (production)
composer dump-autoload --classmap-authoritative  # classmap only; skip filesystem checks
composer install --no-dev --optimize-autoloader   # production install
```

> **Trap:** Class name and file name must match exactly, **case-sensitively**, on Linux. Works on macOS (case-insensitive FS), breaks in production. A real deploy-only failure.

> **Follow-up:** *Difference between `composer install` and `composer update`?* `install` reads `composer.lock` (reproducible, use in CI/prod). `update` re-resolves versions and rewrites the lock. Never run `update` in a deploy pipeline.

> **Follow-up:** *What are the PSRs you use?* PSR-1/PSR-12 (coding style), PSR-4 (autoloading), PSR-3 (logging — Laravel's `LoggerInterface`), PSR-7/PSR-17 (HTTP messages), PSR-11 (container — Laravel's container implements it), PSR-15 (middleware), PSR-18 (HTTP client).

---

## 14. PHP 8.0 → 8.4 Feature Timeline

Interviewers use this to check whether you've kept current or coasted on PHP 7 habits.

### PHP 8.0
- Union types, `mixed`, `static` return type
- Constructor property promotion
- Named arguments
- `match` expression
- Nullsafe operator `?->`
- Attributes (`#[Route]`) — native annotations
- Saner string↔number comparison
- Throw as an expression: `$x = $y ?? throw new Exception()`
- Non-capturing catch: `catch (Exception)`
- JIT
- `str_contains`, `str_starts_with`, `str_ends_with`

### PHP 8.1
- **Enums**
- `readonly` properties
- Fibers (basis for async runtimes)
- First-class callable syntax `foo(...)`
- Pure intersection types
- `never` return type
- `new` in initializers: `public function __construct(private Logger $l = new NullLogger())`
- Final class constants
- Array unpacking with string keys
- Deprecated: implicit float→int lossy conversion, `null` to non-nullable internal params

### PHP 8.2
- `readonly` classes
- DNF types `(A&B)|null`
- `true`, `false`, `null` as standalone types
- Deprecated dynamic properties (`#[AllowDynamicProperties]` to opt out)
- Constants in traits
- `SensitiveParameter` attribute — redacts values in stack traces

```php
function login(string $email, #[SensitiveParameter] string $password): void {}
// password no longer appears in stack traces / logs — useful for OWASP compliance
```

### PHP 8.3
- Typed class constants
- `#[Override]` attribute — compile-time check that you're really overriding
- `json_validate()` — validate JSON without allocating the decoded structure
- Readonly property reinitialization inside `__clone`
- Dynamic class constant fetch `Foo::{$name}`

### PHP 8.4
- **Property hooks** — computed properties without full getters/setters
- **Asymmetric visibility** — `public private(set) string $sku;`
- `new` without parentheses for member access: `new Foo()->bar()`
- Lazy objects (proxy/ghost) — relevant for ORMs
- `array_find`, `array_any`, `array_all`
- Deprecated implicit nullable params (`function f(string $s = null)` → must write `?string`)

```php
// 8.4 property hooks
class InventoryItem
{
    public int $quantity = 0;

    public bool $isLowStock {
        get => $this->quantity < $this->reorderPoint;
    }

    public string $sku {
        set (string $value) => strtoupper(trim($value));
    }
}

// 8.4 asymmetric visibility — public read, private write
class Order
{
    public private(set) OrderStatus $status = OrderStatus::Pending;

    public function markPaid(): void
    {
        $this->status = OrderStatus::Paid;   // only from inside
    }
}
```

> **Follow-up:** *What PHP version are you on and why?* Have a real answer: supported versions, security-support windows, performance gains per version, and what blocked you from upgrading (extension compatibility, deprecations in dependencies). Mentioning that you led a PHP 7.4 → 8.x upgrade with a deprecation-sweeping strategy (`E_DEPRECATED` logging in staging, PHPStan level bump, Rector for automated fixes) is strong senior signal.

---

# Part B — Laravel Fundamentals

## 15. The Request Lifecycle, Step by Step

Memorize this with real class names. "Walk me through the Laravel request lifecycle" is asked in nearly every Laravel interview, and naming actual classes separates users from experts.

```
1.  Nginx → php-fpm → public/index.php
2.  require vendor/autoload.php                       (Composer autoloader)
3.  $app = require bootstrap/app.php                  (creates Illuminate\Foundation\Application)
      · L11+: Application::configure() fluently registers routing, middleware, exceptions
      · binds paths, registers base providers (Event, Log, Routing)
4.  $kernel = $app->make(Illuminate\Contracts\Http\Kernel::class)
5.  $response = $kernel->handle($request)             (Request::capture() from globals)
      5a. bootstrap():  run bootstrappers in order
            · LoadEnvironmentVariables   (.env via Dotenv — skipped if config cached)
            · LoadConfiguration          (config/*.php or bootstrap/cache/config.php)
            · HandleExceptions           (set_error_handler / set_exception_handler)
            · RegisterFacades            (Facade::setFacadeApplication)
            · RegisterProviders          (calls register() on ALL providers)
            · BootProviders              (calls boot() on ALL providers)
      5b. sendRequestThroughRouter():
            · global middleware stack (TrustProxies, HandleCors, ValidatePostSize,
              TrimStrings, ConvertEmptyStringsToNull, PreventRequestsDuringMaintenance)
            · Router::dispatch() → match route → collect route/group middleware
            · route middleware pipeline (auth, throttle, your tenant middleware)
            · resolve controller from container (constructor injection)
            · resolve method params (route model binding, Form Request validation, DI)
            · run controller action → return value
            · convert return value to Illuminate\Http\Response (Responsable, JsonSerializable, etc.)
            · unwind middleware "after" side (session save, cookie queue, CORS headers)
6.  $response->send()                                  (headers + body to client)
7.  $kernel->terminate($request, $response)            (terminable middleware, e.g. session
                                                        write-close, log flush, Telescope)
8.  Process/state destroyed (unless Octane)
```

### The pipeline is an onion, not a queue

Middleware is implemented with `array_reduce` over a reversed stack, building nested closures:

```php
// Simplified Illuminate\Pipeline\Pipeline
public function then(Closure $destination)
{
    $pipeline = array_reduce(
        array_reverse($this->pipes),
        $this->carry(),
        $this->prepareDestination($destination)
    );

    return $pipeline($this->passable);
}

protected function carry(): Closure
{
    return fn ($stack, $pipe) => fn ($passable) => $pipe->handle($passable, $stack);
}
```

This is why every middleware can act **before** (`before $next($request)`) and **after** (`after $next($request)`) the request:

```php
class MeasureLatency
{
    public function handle(Request $request, Closure $next): Response
    {
        $start = microtime(true);        // BEFORE

        $response = $next($request);     // inner onion layers run here

        $ms = (microtime(true) - $start) * 1000;   // AFTER
        $response->headers->set('X-Response-Time', (string) round($ms, 2));

        return $response;
    }
}
```

> **Trap:** "Where do you put logic that must run *after* the response is sent to the client?" Not in `handle()` after `$next()` — that still delays the response. Use **terminable middleware** (`terminate()`), a queued job, or `dispatch(fn () => ...)->afterResponse()`.

> **Follow-up:** *`register()` vs `boot()` in providers?* `register()` only binds things into the container — you must not resolve or use other services there, because they may not be registered yet. `boot()` runs after ALL providers have registered, so you can safely resolve dependencies, register event listeners, define gates, publish routes.

> **Follow-up:** *Why doesn't `.env` work after `config:cache`?* Because `LoadEnvironmentVariables` is skipped when a cached config file exists. `env()` outside of `config/*.php` returns `null` in production. **Rule: call `env()` only in config files; call `config()` everywhere else.** This is one of the most common Laravel production bugs.

```php
// BAD — breaks with config:cache
class PaymentService
{
    public function __construct()
    {
        $this->key = env('STRIPE_KEY');   // null in production
    }
}

// GOOD
// config/services.php: 'stripe' => ['key' => env('STRIPE_KEY')]
$this->key = config('services.stripe.key');
```

---

## 16. Service Providers

```php
namespace App\Providers;

class InventoryServiceProvider extends ServiceProvider
{
    // Only container bindings here. Do NOT resolve services.
    public function register(): void
    {
        $this->app->bind(StockRepository::class, EloquentStockRepository::class);

        $this->app->singleton(TenantContext::class);

        $this->mergeConfigFrom(__DIR__ . '/../../config/inventory.php', 'inventory');
    }

    // Everything else. All providers are registered by now.
    public function boot(): void
    {
        Gate::policy(InventoryItem::class, InventoryItemPolicy::class);

        Event::listen(StockLevelChanged::class, NotifyLowStock::class);

        Route::middleware('api')
            ->prefix('api/inventory')
            ->group(base_path('routes/inventory.php'));

        // Fail loudly in dev if someone introduces an N+1
        Model::preventLazyLoading(! $this->app->isProduction());

        // Load migrations/views/translations for a package
        $this->loadMigrationsFrom(__DIR__ . '/../../database/migrations');

        // Custom validation rule
        Validator::extend('tenant_owned', function ($attr, $value, $params) {
            return InventoryItem::where('id', $value)
                ->where('organization_id', app(TenantContext::class)->organizationId())
                ->exists();
        });
    }
}
```

### Deferred providers

If a provider only registers bindings and declares `provides()`, Laravel defers loading it until one of those bindings is actually resolved — reducing per-request bootstrap cost.

```php
class ReportingServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(ReportGenerator::class, fn ($app) => new ReportGenerator(
            $app->make(StockRepository::class)
        ));
    }

    public function provides(): array
    {
        return [ReportGenerator::class];
    }
}
```

> **Trap:** A deferrable provider must not do anything in `boot()` that the app relies on unconditionally (like registering routes or event listeners), because `boot()` may never run.

---

## 17. Facades

A Facade is a static proxy to a container binding. It is *not* a static class.

```php
namespace Illuminate\Support\Facades;

abstract class Facade
{
    public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();     // resolve from container

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
}

class Cache extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'cache';    // container key
    }
}
```

So `Cache::get('key')` is really `app('cache')->get('key')`.

### Custom facade

```php
// 1. The real class
namespace App\Services;

class StockCalculator
{
    public function available(int $itemId): int { /* ... */ }
}

// 2. Bind it
$this->app->singleton(StockCalculator::class);

// 3. The facade
namespace App\Facades;

use Illuminate\Support\Facades\Facade;

/**
 * @method static int available(int $itemId)
 */
class Stock extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return \App\Services\StockCalculator::class;
    }
}

// 4. Use it
Stock::available(42);
```

### Real-time facades — facade any class without writing one

```php
use Facades\App\Services\StockCalculator;   // note the `Facades\` prefix

StockCalculator::available(42);              // works, and is mockable
```

### Testing facades

```php
Cache::shouldReceive('get')->with('key')->once()->andReturn('value');
Queue::fake();
Event::fake([StockLevelChanged::class]);
Http::fake(['api.supplier.com/*' => Http::response(['ok' => true], 200)]);
```

> **Trap:** "Facades make code untestable." False — Laravel facades are swappable mocks. The real criticism is that facades **hide dependencies**: a class using 6 facades has 6 invisible collaborators, and its constructor lies about what it needs. For domain/service classes, prefer constructor injection; facades are fine in controllers, commands, and glue code.

> **Follow-up:** *Facade vs helper function vs DI?* Helpers (`cache()`, `config()`) are thin wrappers over the container too. DI is the most explicit and testable. My rule: **inject in domain/service layer, facade in framework layer.**

---

## 18. Configuration & Environment

```php
config('inventory.low_stock_threshold', 10);       // read (dot notation)
config(['inventory.threshold' => 5]);              // set at runtime (not persisted)
config()->has('services.stripe.key');
app()->environment('production');
app()->environment(['local', 'staging']);
```

```bash
php artisan config:cache      # merge all config into bootstrap/cache/config.php
php artisan config:clear
php artisan route:cache       # requires no closure-based routes
php artisan event:cache
php artisan view:cache
php artisan optimize          # config + route + view + events
php artisan optimize:clear
```

> **Trap (containers):** In Docker, if you run `config:cache` at image build time, the cached config bakes in build-time env values. Runtime env vars from ECS task definitions / Kubernetes secrets are then ignored. Either run `config:cache` in the container entrypoint (after env injection), or don't cache config and accept the small cost.

> **Follow-up:** *How do you manage secrets across environments?* AWS Secrets Manager / SSM Parameter Store injected as env vars at task startup, never in the image or repo. Rotate via Secrets Manager rotation. For `APP_KEY` rotation, you must re-encrypt anything encrypted with `Crypt` — plan a dual-key window.

---

## 19. Routing

```php
// routes/api.php
Route::middleware(['auth:api', 'tenant'])->group(function () {

    Route::get('/inventory', [InventoryController::class, 'index'])->name('inventory.index');
    Route::post('/inventory', [InventoryController::class, 'store']);

    // Constraints
    Route::get('/inventory/{item}', [InventoryController::class, 'show'])
        ->whereNumber('item');

    Route::get('/orgs/{org:slug}/items/{item:sku}', [ItemController::class, 'show'])
        ->scopeBindings();       // ← forces item to be resolved as a CHILD of org

    // Optional param with default
    Route::get('/reports/{period?}', ReportController::class);

    // Resource routes
    Route::apiResource('suppliers', SupplierController::class);
    Route::apiResource('suppliers.items', SupplierItemController::class)->shallow();
});

Route::fallback(fn () => response()->json(['message' => 'Not Found'], 404));
```

### Route model binding

```php
// Implicit — {item} resolves InventoryItem by primary key
public function show(InventoryItem $item) {}

// Custom key on the model
class InventoryItem extends Model
{
    public function getRouteKeyName(): string { return 'uuid'; }
}

// Custom resolution logic — CRITICAL for multi-tenancy
class InventoryItem extends Model
{
    public function resolveRouteBinding($value, $field = null): ?Model
    {
        return $this->where($field ?? $this->getRouteKeyName(), $value)
            ->where('organization_id', app(TenantContext::class)->organizationId())
            ->firstOrFail();
    }
}

// Explicit binding in a provider
Route::bind('item', function (string $value) {
    return InventoryItem::where('sku', $value)
        ->where('organization_id', auth()->user()->organization_id)
        ->firstOrFail();
});
```

> **Trap — IDOR:** Default implicit binding does `InventoryItem::findOrFail($id)` with **no tenant check**. If your global scope isn't applied (or is bypassed), user from Org A can read Org B's item by guessing an ID. Always either (a) rely on a verified global scope, or (b) override `resolveRouteBinding`, or (c) use `scopeBindings()` with nested routes. And **test it**.

```php
it('returns 404 when accessing another organizations item', function () {
    $orgA = Organization::factory()->create();
    $orgB = Organization::factory()->create();
    $foreignItem = InventoryItem::factory()->for($orgB)->create();
    $user = User::factory()->for($orgA)->create();

    $this->actingAs($user, 'api')
        ->getJson("/api/inventory/{$foreignItem->id}")
        ->assertNotFound();          // NOT 403 — don't confirm existence
});
```

### Signed URLs & rate limiting

```php
URL::temporarySignedRoute('invoice.download', now()->addMinutes(15), ['invoice' => $id]);

// Verify via middleware
Route::get('/invoice/{invoice}', ...)->middleware('signed');

// Named rate limiter (define in a provider)
RateLimiter::for('api', fn (Request $r) => $r->user()
    ? Limit::perMinute(1000)->by("org:{$r->user()->organization_id}")   // per TENANT
    : Limit::perMinute(60)->by($r->ip()));

Route::middleware('throttle:api')->group(...);
```

> **Follow-up:** *How do you rate limit fairly in a multi-tenant API?* Key the limiter by `organization_id`, not by IP or user — otherwise one tenant's traffic burst starves others (noisy neighbor), and a single tenant behind one NAT IP gets throttled unfairly. Consider tiered limits per plan.

> **Follow-up:** *Why does `route:cache` fail?* Closure-based routes cannot be serialized. Move them to controllers.

---

## 20. Controllers

```php
final class InventoryController extends Controller
{
    public function __construct(
        private readonly InventoryService $inventory,
    ) {}

    public function index(IndexInventoryRequest $request): AnonymousResourceCollection
    {
        return InventoryItemResource::collection(
            $this->inventory->paginate($request->validated())
        );
    }

    public function store(StoreInventoryItemRequest $request): JsonResponse
    {
        $item = $this->inventory->create(
            CreateItemData::fromRequest($request)
        );

        return InventoryItemResource::make($item)
            ->response()
            ->setStatusCode(201);
    }

    public function update(UpdateInventoryItemRequest $request, InventoryItem $item): InventoryItemResource
    {
        return InventoryItemResource::make(
            $this->inventory->update($item, $request->validated())
        );
    }

    public function destroy(InventoryItem $item): Response
    {
        $this->authorize('delete', $item);
        $this->inventory->delete($item);

        return response()->noContent();     // 204
    }
}
```

### Single-action (invokable) controller

```php
final class DeductStockController
{
    public function __construct(private readonly DeductStockAction $action) {}

    public function __invoke(DeductStockRequest $request, InventoryItem $item): JsonResponse
    {
        $updated = $this->action->execute(
            DeductStockData::fromRequest($request, $item)
        );

        return response()->json(['quantity' => $updated->quantity]);
    }
}

Route::post('/inventory/{item}/deduct', DeductStockController::class);
```

> **Trap:** Fat controllers. A senior interviewer will ask "where does business logic go?" Answer: controllers translate HTTP ↔ domain; business rules live in Actions/Services/domain models; persistence in Eloquent/repositories. Controllers should be readable in under 10 lines per method.

---

## 21. Requests, Responses & API Resources

### Reading input safely

```php
$request->input('sku');                 // any input source, dot notation supported
$request->query('page', 1);             // query string only
$request->post('sku');                  // body only
$request->boolean('active');            // "1","true","on","yes" → true
$request->integer('page');
$request->date('from');                 // Carbon instance
$request->enum('type', MovementType::class);
$request->collect('ids');               // Collection
$request->only(['sku', 'name']);
$request->except(['_token']);
$request->has('sku');                   // key present (even if null/empty)
$request->filled('sku');                // present AND non-empty
$request->missing('sku');
$request->whenFilled('sku', fn ($v) => ...);
$request->bearerToken();
$request->header('X-Request-Id');
$request->ip();
$request->file('import')->store('imports');
```

> **Trap:** `$request->all()` includes anything the client sent. Passing it into `create()`/`update()` is a mass-assignment vulnerability — a client could send `organization_id` and, if it's in `$fillable`, cross-tenant-write. Always use `$request->validated()` and never put `organization_id` in `$fillable` (set it server-side).

### API Resources

```php
final class InventoryItemResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'sku'        => $this->sku,
            'name'       => $this->name,
            'quantity'   => $this->quantity,
            'status'     => $this->status->value,
            'is_low'     => $this->quantity < $this->reorder_point,

            // Only include if relation is already loaded — prevents N+1 from the resource layer
            'supplier'   => SupplierResource::make($this->whenLoaded('supplier')),
            'movements_count' => $this->whenCounted('movements'),

            // Conditional on permission
            'cost_price' => $this->when(
                $request->user()->can('viewCost', $this->resource),
                fn () => $this->cost_price
            ),

            'created_at' => $this->created_at->toIso8601String(),
        ];
    }

    public function with(Request $request): array
    {
        return ['meta' => ['version' => 'v1']];
    }
}
```

> **Trap — the resource-layer N+1:** Writing `'supplier' => SupplierResource::make($this->supplier)` lazily loads the relation for every item in a collection → N+1. `whenLoaded()` returns a `MissingValue` that gets stripped from output when the relation isn't eager-loaded, so it's safe.

```php
// Pair the resource with an eager load in the controller
InventoryItemResource::collection(
    InventoryItem::with(['supplier:id,name'])->withCount('movements')->paginate(25)
);
```

### Response types

```php
return response()->json($data, 200, ['X-Total' => $count]);
return response()->noContent();                        // 204
return response()->streamDownload(fn () => echo $csv, 'export.csv');
return response()->json($data)->withHeaders([...]);
return new JsonResponse($data, 422);
return $item;                                          // model → auto JSON
return InventoryItemResource::make($item);             // Responsable
abort(404);
abort_if($item->organization_id !== $orgId, 404);
abort_unless($user->can('view', $item), 403);
throw new HttpResponseException(response()->json([...], 409));
```

---

## 22. Validation

### Form Requests

```php
final class StoreInventoryItemRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', InventoryItem::class);
    }

    public function rules(): array
    {
        $orgId = $this->user()->organization_id;

        return [
            'sku' => [
                'required', 'string', 'max:64', 'regex:/^[A-Z0-9\-]+$/',
                // Tenant-scoped uniqueness — a global unique rule would leak/conflict across tenants
                Rule::unique('inventory_items', 'sku')->where('organization_id', $orgId),
            ],
            'name'           => ['required', 'string', 'max:255'],
            'quantity'       => ['required', 'integer', 'min:0', 'max:1000000'],
            'reorder_point'  => ['nullable', 'integer', 'min:0', 'lte:quantity'],
            'type'           => ['required', Rule::enum(MovementType::class)],
            'supplier_id'    => [
                'nullable',
                // MUST scope the exists check to the tenant, else cross-tenant reference
                Rule::exists('suppliers', 'id')->where('organization_id', $orgId),
            ],
            'tags'           => ['array', 'max:10'],
            'tags.*'         => ['string', 'max:32', 'distinct'],
            'metadata'       => ['nullable', 'array'],
            'expires_at'     => ['nullable', 'date', 'after:today'],
        ];
    }

    // Normalize BEFORE validation
    protected function prepareForValidation(): void
    {
        $this->merge([
            'sku' => strtoupper(trim((string) $this->input('sku'))),
        ]);
    }

    // Cross-field / DB checks AFTER individual rules pass
    public function after(): array
    {
        return [
            function (Validator $validator) {
                if ($this->filled('reorder_point') && $this->integer('reorder_point') > $this->integer('quantity')) {
                    $validator->errors()->add('reorder_point', 'Reorder point cannot exceed quantity.');
                }
            },
        ];
    }

    public function messages(): array
    {
        return ['sku.regex' => 'SKU may only contain uppercase letters, numbers, and hyphens.'];
    }

    public function attributes(): array
    {
        return ['reorder_point' => 'reorder threshold'];
    }
}
```

### Conditional & advanced rules

```php
'card_number' => ['required_if:payment_method,card'],
'discount'    => ['exclude_unless:type,promo', 'numeric'],
'password'    => ['required', Password::min(12)->mixedCase()->numbers()->symbols()->uncompromised()],
'email'       => ['required', 'email:rfc,dns'],
'role'        => ['required', Rule::in(['admin', 'manager', 'viewer'])],
'avatar'      => ['image', 'mimes:jpg,png', 'max:2048', 'dimensions:max_width=2000'],

// Conditional rule sets
$validator->sometimes('warehouse_id', 'required', fn ($input) => $input->type === 'transfer');
```

### Custom rule object

```php
final class SufficientStock implements ValidationRule
{
    public function __construct(private readonly InventoryItem $item) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if ((int) $value > $this->item->quantity) {
            $fail("Only {$this->item->quantity} units available.");
        }
    }
}

// Usage
'amount' => ['required', 'integer', 'min:1', new SufficientStock($this->route('item'))],
```

> **Trap:** Validating stock availability in a validation rule is a **TOCTOU** (time-of-check-to-time-of-use) race: between validation and the actual decrement, another request can take the stock. Validation gives a nice error message; the *authoritative* check must be inside the transaction with a lock or an atomic conditional update (see Tier 3, §Concurrency).

> **Follow-up:** *What HTTP status does a failed Form Request return?* `422 Unprocessable Entity` with `{"message": ..., "errors": {"field": ["..."]}}` for JSON requests; a redirect back with flashed errors for web requests. Laravel decides based on `expectsJson()`.

> **Follow-up:** *Where does `authorize()` failing land?* `403` via `AuthorizationException` — thrown before `rules()` runs.

---

## 23. Eloquent Basics

```php
final class InventoryItem extends Model
{
    use HasFactory, SoftDeletes, BelongsToOrganization;

    protected $table = 'inventory_items';        // inferred: snake_case plural
    protected $primaryKey = 'id';
    public $incrementing = true;
    protected $keyType = 'int';
    public $timestamps = true;
    protected $dateFormat = 'Y-m-d H:i:s';
    protected $connection = 'pgsql';

    // Mass assignment: whitelist. organization_id deliberately EXCLUDED (set server-side).
    protected $fillable = ['sku', 'name', 'quantity', 'reorder_point', 'supplier_id', 'metadata'];

    protected $hidden = ['cost_price'];          // excluded from array/JSON
    protected $appends = ['is_low_stock'];       // computed attrs added to JSON

    protected $attributes = ['quantity' => 0];   // default attribute values

    protected function casts(): array            // Laravel 11 method form
    {
        return [
            'quantity'      => 'integer',
            'cost_price'    => 'decimal:2',
            'metadata'      => 'array',
            'status'        => StockStatus::class,
            'expires_at'    => 'datetime',
            'is_active'     => 'boolean',
            'secret_note'   => 'encrypted',
            'settings'      => AsArrayObject::class,
        ];
    }

    // Accessor + mutator (Laravel 9+ single-method style)
    protected function sku(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => strtoupper($value),
            set: fn (string $value) => strtoupper(trim($value)),
        );
    }

    // Computed attribute (no DB column)
    protected function isLowStock(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->quantity <= ($this->reorder_point ?? 0),
        )->shouldCache();
    }

    public function supplier(): BelongsTo
    {
        return $this->belongsTo(Supplier::class);
    }

    public function movements(): HasMany
    {
        return $this->hasMany(StockMovement::class);
    }

    // Local scope
    public function scopeLowStock(Builder $query): void
    {
        $query->whereColumn('quantity', '<=', 'reorder_point');
    }
}
```

### CRUD and the differences that matter

```php
// Create
$item = InventoryItem::create([...]);          // mass assignment + events
$item = new InventoryItem(); $item->sku = 'A'; $item->save();
InventoryItem::insert([[...], [...]]);         // ⚠️ bulk: NO events, NO timestamps, NO casts
InventoryItem::firstOrCreate(['sku' => 'A'], ['name' => 'X']);   // find or insert
InventoryItem::updateOrCreate(['sku' => 'A'], ['quantity' => 5]);
InventoryItem::upsert($rows, ['organization_id', 'sku'], ['quantity']);  // ⚠️ no events

// Read
InventoryItem::find(1);                        // null if missing
InventoryItem::findOrFail(1);                  // ModelNotFoundException → 404
InventoryItem::findMany([1, 2, 3]);
InventoryItem::first();
InventoryItem::firstWhere('sku', 'A');
InventoryItem::where('quantity', '<', 10)->get();
InventoryItem::lowStock()->paginate(25);
InventoryItem::value('sku');                   // single column, single row
InventoryItem::pluck('name', 'id');            // ['1' => 'Widget']
InventoryItem::count(); ::sum('quantity'); ::avg('quantity'); ::max('quantity');
InventoryItem::exists(); ::doesntExist();

// Update
$item->update(['quantity' => 5]);              // events fire
$item->fill([...])->save();
$item->increment('quantity', 5);               // single atomic SQL, saves other dirty attrs too
InventoryItem::where(...)->update(['quantity' => 0]);   // ⚠️ mass update: NO model events

// Delete
$item->delete();                               // soft delete if trait present
$item->forceDelete();
$item->restore();
InventoryItem::withTrashed()->find(1);
InventoryItem::onlyTrashed()->get();
InventoryItem::destroy([1, 2, 3]);
InventoryItem::where(...)->delete();           // ⚠️ no model events
```

> **Trap (asked constantly):** *Which Eloquent operations skip model events?* `insert()`, `upsert()`, mass `update()`/`delete()` on a query builder, `DB::table()` anything, and `saveQuietly()`/`withoutEvents()`. This matters enormously if you rely on observers for audit logs, cache invalidation, or search indexing. In your multi-tenant SaaS, a bulk `update()` that bypasses an observer would leave Elasticsearch stale.

```php
// If you need events on a bulk operation, you must iterate
InventoryItem::where('quantity', 0)->lazyById(500)->each(
    fn ($i) => $i->update(['status' => StockStatus::OutOfStock])
);
// ...or dispatch invalidation explicitly after the bulk update
InventoryItem::where('quantity', 0)->update(['status' => 'out_of_stock']);
Cache::tags(["org:{$orgId}"])->flush();
```

### Dirty / changed state

```php
$item->sku = 'NEW';
$item->isDirty();               // true — changed but not saved
$item->isDirty('sku');          // true
$item->getOriginal('sku');      // old value
$item->getDirty();              // ['sku' => 'NEW']

$item->save();
$item->wasChanged('sku');       // true — changed by the last save
$item->isClean();               // true
```

Useful in observers:

```php
public function updated(InventoryItem $item): void
{
    if ($item->wasChanged('quantity')) {
        StockLevelChanged::dispatch(
            $item,
            $item->getOriginal('quantity'),
            $item->quantity,
        );
    }
}
```

### Timestamps, UUIDs, primary keys

```php
class StockMovement extends Model
{
    use HasUuids;                  // UUID v4 primary key
    // or use HasUlids;            // sortable, better for index locality

    public $timestamps = true;
    const UPDATED_AT = null;       // append-only ledger: created_at only
}
```

> **Follow-up:** *UUID vs auto-increment PK for a multi-tenant SaaS?* UUID/ULID avoids leaking row counts and makes cross-tenant ID guessing useless, and lets clients generate IDs offline. Cost: 16 bytes vs 8, random UUIDv4 destroys B-tree insert locality (page splits, worse cache hit rate). **ULID or UUIDv7** are time-ordered and get most of the benefit without the index churn. Many teams use a `bigint` PK internally plus a public `uuid` column exposed via `getRouteKeyName()`.

---

## 24. Migrations, Seeders & Factories

```php
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('inventory_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
            $table->foreignId('supplier_id')->nullable()->constrained()->nullOnDelete();
            $table->string('sku', 64);
            $table->string('name');
            $table->integer('quantity')->default(0);
            $table->integer('reorder_point')->nullable();
            $table->decimal('cost_price', 12, 2)->nullable();
            $table->jsonb('metadata')->nullable();          // Postgres: jsonb, not json
            $table->string('status')->default('in_stock');
            $table->timestamps();
            $table->softDeletes();

            // Tenant-scoped uniqueness — NOT a global unique on sku
            $table->unique(['organization_id', 'sku']);

            // Composite index: tenant first (highest selectivity in every query)
            $table->index(['organization_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('inventory_items');
    }
};
```

### Column types worth knowing

```php
$table->id();                          // bigIncrements
$table->uuid('public_id')->unique();
$table->ulid('trace_id');
$table->string('name', 255);
$table->text('notes'); $table->longText('payload');
$table->integer(); ->bigInteger(); ->unsignedBigInteger(); ->smallInteger();
$table->decimal('amount', 12, 2);      // exact — use for money
$table->float(); $table->double();     // ⚠️ never for money
$table->boolean('active');
$table->json('meta'); $table->jsonb('meta');
$table->enum('status', ['a', 'b']);    // ⚠️ painful to alter; prefer string + app enum
$table->timestamp('paid_at')->nullable();
$table->timestampTz('occurred_at');
$table->date(); $table->time(); $table->year();
$table->binary('blob');
$table->ipAddress(); $table->macAddress();
$table->morphs('subject');             // subject_id + subject_type
$table->uuidMorphs('subject');
$table->foreignIdFor(Organization::class);
$table->rememberToken();
$table->comment('...');
```

### Modifiers and index operations

```php
->nullable() ->default(0) ->unsigned() ->after('col') ->useCurrent()
->useCurrentOnUpdate() ->storedAs('...') ->virtualAs('...') ->change()

$table->index('col');
$table->index(['a', 'b'], 'custom_name');
$table->unique(['a', 'b']);
$table->fullText('name');                 // MySQL / Postgres tsvector
$table->dropIndex(['a', 'b']);
$table->dropForeign(['supplier_id']);
$table->renameColumn('old', 'new');       // ⚠️ locks; see Tier 3
$table->dropColumn(['a', 'b']);
```

### Raw DDL when the schema builder isn't enough

```php
public function up(): void
{
    // Partial index — only rows that matter for the low-stock dashboard
    DB::statement('
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_items_low_stock
        ON inventory_items (organization_id, quantity)
        WHERE deleted_at IS NULL AND quantity < 10
    ');
}

// CREATE INDEX CONCURRENTLY cannot run in a transaction
public $withinTransaction = false;
```

> **Trap:** Laravel wraps migrations in a transaction on databases that support transactional DDL (Postgres). `CREATE INDEX CONCURRENTLY` errors inside a transaction. Set `public $withinTransaction = false;`.

### Factories

```php
class InventoryItemFactory extends Factory
{
    protected $model = InventoryItem::class;

    public function definition(): array
    {
        return [
            'organization_id' => Organization::factory(),
            'sku'             => strtoupper($this->faker->unique()->bothify('???-####')),
            'name'            => $this->faker->words(3, true),
            'quantity'        => $this->faker->numberBetween(0, 500),
            'reorder_point'   => 10,
            'status'          => StockStatus::InStock,
        ];
    }

    public function outOfStock(): static
    {
        return $this->state(fn () => [
            'quantity' => 0,
            'status'   => StockStatus::OutOfStock,
        ]);
    }

    public function forOrganization(Organization $org): static
    {
        return $this->state(fn () => ['organization_id' => $org->id]);
    }
}
```

```php
InventoryItem::factory()->count(3)->outOfStock()->create();
InventoryItem::factory()->for($org)->has(StockMovement::factory()->count(5))->create();
InventoryItem::factory()->make();                 // not persisted
InventoryItem::factory()->createQuietly();        // skip observers
InventoryItem::factory()->count(1000)->create();  // ⚠️ slow; consider ->make() + insert()
```

---

## 25. Artisan & Console Commands

```php
final class BackfillItemStatus extends Command
{
    protected $signature = 'inventory:backfill-status
                            {--org= : Limit to a single organization}
                            {--chunk=1000 : Rows per batch}
                            {--sleep=100 : Milliseconds to sleep between batches}
                            {--dry-run : Report only, write nothing}';

    protected $description = 'Backfill the status column for existing inventory items';

    public function handle(): int
    {
        $chunk  = (int) $this->option('chunk');
        $sleep  = (int) $this->option('sleep');
        $dryRun = (bool) $this->option('dry-run');

        $query = InventoryItem::withoutGlobalScopes()->whereNull('status');

        if ($org = $this->option('org')) {
            $query->where('organization_id', $org);
        }

        $total = $query->clone()->count();
        $bar   = $this->output->createProgressBar($total);
        $done  = 0;

        // lazyById is essential: we're mutating the column we filter on
        $query->lazyById($chunk)->each(function (InventoryItem $item) use (&$done, $bar, $dryRun, $sleep, $chunk) {
            if (! $dryRun) {
                $item->updateQuietly([
                    'status' => $item->quantity > 0
                        ? StockStatus::InStock
                        : StockStatus::OutOfStock,
                ]);
            }

            $bar->advance();

            // Throttle to protect replication lag and lock contention
            if (++$done % $chunk === 0) {
                usleep($sleep * 1000);
            }
        });

        $bar->finish();
        $this->newLine();
        $this->info(($dryRun ? '[DRY RUN] ' : '') . "Processed {$done} of {$total} rows.");

        return self::SUCCESS;
    }
}
```

```php
// Scheduling (Laravel 11: routes/console.php or bootstrap/app.php)
Schedule::command('inventory:backfill-status')
    ->hourly()
    ->withoutOverlapping(10)      // mutex with 10-minute expiry
    ->onOneServer()               // requires a shared cache lock store (Redis)
    ->runInBackground()
    ->appendOutputTo(storage_path('logs/backfill.log'))
    ->onFailure(fn () => Log::error('Backfill failed'));
```

> **Trap:** `withoutOverlapping()` uses a cache lock. With the `file` or `array` cache driver on multiple servers, each server has its own lock and the job runs concurrently anyway. `onOneServer()` requires Redis/Memcached/DB — a **shared** atomic lock store.

> **Follow-up:** *Command exit codes?* Return `self::SUCCESS` (0) / `self::FAILURE` (1). Non-zero matters for CI, cron alerting, and Kubernetes Jobs.

---

## 26. Baseline Security

| Threat | Laravel control | What you must still do |
|--------|-----------------|------------------------|
| SQL injection | Eloquent/Query Builder use PDO prepared statements | Never interpolate into `DB::raw()`, `whereRaw()`, `orderByRaw()`; bind params |
| XSS | Blade `{{ }}` escapes via `htmlspecialchars` | `{!! !!}` only for sanitized HTML; escape in JSON→JS contexts; set CSP |
| CSRF | `VerifyCsrfToken` on web routes | API routes with token auth don't need it; never blanket-disable |
| Mass assignment | `$fillable` / `$guarded` | Exclude `organization_id`, `role`, `is_admin`; use `validated()` not `all()` |
| Broken auth | Hashing via bcrypt/argon2 | Rate limit login, rotate tokens, short TTLs, revoke on password change |
| IDOR | Route model binding + policies | Scope every lookup by tenant; return 404 not 403 |
| Sensitive data exposure | `$hidden`, `encrypted` casts | Redact in logs, `#[SensitiveParameter]`, no PII in URLs |
| Timing attacks | — | `hash_equals`, `Hash::check` |

```php
// SQL injection — the ways you can still shoot yourself
DB::select("SELECT * FROM items WHERE sku = '{$sku}'");            // ❌ injectable
DB::select('SELECT * FROM items WHERE sku = ?', [$sku]);            // ✅ bound
InventoryItem::whereRaw("sku = '{$sku}'")->get();                   // ❌
InventoryItem::whereRaw('sku = ?', [$sku])->get();                  // ✅
InventoryItem::orderBy($request->input('sort'))->get();             // ❌ column injection
InventoryItem::orderBy(
    in_array($request->input('sort'), ['sku', 'quantity'], true)
        ? $request->input('sort') : 'id'
)->get();                                                            // ✅ allowlist
```

> **Trap:** Bindings protect *values*, not *identifiers*. Column names, table names, and sort directions cannot be parameterized — you must allowlist them. Dynamic-sort endpoints are the most common injection hole in otherwise-safe Laravel apps.

```php
// Hashing
$hash = Hash::make($password);                       // bcrypt by default
Hash::check($plain, $hash);
Hash::needsRehash($hash);                            // rehash on login after cost/algo change

// config/hashing.php
'driver' => 'argon2id',
'argon' => ['memory' => 65536, 'threads' => 1, 'time' => 4],
```

> **Follow-up:** *Why not MD5/SHA-256 for passwords?* They're fast — which is exactly wrong. Password hashing must be *slow and memory-hard* (bcrypt/argon2id) to resist GPU brute force, and salted per user (bcrypt embeds the salt).

---

## 27. Tier 1 Q&A Drill

Answer out loud, then check. Vague answers = the gap to close.

### PHP language

1. **What does `declare(strict_types=1)` do, and what is its scope?**  
   Enables strict scalar type checking for calls made *from that file only*. Without it, PHP coerces scalars (numeric strings to int, etc.). It's per-file and applies to the caller's file.

2. **`0 == 'foo'` — true or false?**  
   `false` in PHP 8 (non-numeric string → number is cast to string). `true` in PHP 7. Know this version difference.

3. **Explain copy-on-write.**  
   Assigning an array increments a refcount instead of copying. The copy only happens when either side is written to. Objects are exempt — they're handles, so assignment shares the instance.

4. **`self` vs `static`?**  
   `self` binds at compile time to the defining class; `static` binds at runtime to the called class (late static binding). Fluent APIs and Eloquent depend on `static`.

5. **Can a private method be overridden?**  
   No. Private methods are resolved in the declaring class's scope and aren't polymorphic. `protected` is required for overriding.

6. **`isset()` vs `array_key_exists()`?**  
   `isset()` is `false` for keys whose value is `null`. `array_key_exists()` is `true`. Matters for PATCH semantics ("explicitly set to null" vs "not sent").

7. **`??` vs `?:`?**  
   `??` triggers on null/unset only. `?:` triggers on any falsy value, so `0` and `'0'` fall through to the default — usually a bug for quantities and IDs.

8. **Does `catch (Exception $e)` catch a `TypeError`?**  
   No. `TypeError` extends `Error`, not `Exception`. Both implement `Throwable`. Catch `Throwable` for everything.

9. **What's wrong with `return` inside `finally`?**  
   It overrides the `try` block's return value and swallows in-flight exceptions.

10. **What is a generator and when would you use one?**  
    A function with `yield` that produces values lazily with constant memory. Used for streaming large datasets — e.g. iterating 15M rows during a backfill instead of loading them into an array.

11. **Why can't you `foreach` a generator twice?**  
    It's a forward-only, one-shot iterator with no rewind. You get "Cannot rewind a generator that was already run."

12. **Traits vs interfaces vs abstract classes — when each?**  
    Interface = contract, multiple allowed, enables DI/mocking. Abstract class = shared implementation + contract, single inheritance. Trait = horizontal code reuse with no type identity; convenient but creates hidden coupling and resists mocking.

13. **Method resolution order with traits?**  
    Own class > trait > inherited parent.

14. **Is `readonly` deep?**  
    No. You can't reassign the property, but if it holds an object you can still mutate that object's state.

15. **`clone` — shallow or deep?**  
    Shallow. Object-typed properties still reference the same instances. Implement `__clone()` for deep copies.

16. **Difference between `match` and `switch`?**  
    `match` is an expression, uses `===`, has no fallthrough, and throws `UnhandledMatchError` if nothing matches and there's no default. `switch` uses `==`, falls through, is a statement.

17. **How does Composer autoloading find a class?**  
    PSR-4 prefix→directory mapping; namespace separators become path separators; case-sensitive on Linux. `composer dump-autoload -o` builds a static classmap for production.

18. **`composer install` vs `composer update` in a deploy?**  
    Always `install` (respects `composer.lock`, reproducible). `update` re-resolves dependencies and must never run in a pipeline.

19. **Why is `hash_equals` needed?**  
    `===` on strings short-circuits at the first differing byte, leaking length/prefix info through timing. `hash_equals` is constant-time.

20. **Why can't you use floats for money?**  
    Binary floating point can't represent decimal fractions exactly; `0.1 + 0.2 !== 0.3`. Use integer minor units or `DECIMAL`/`NUMERIC` plus a decimal library.

### Laravel

21. **Walk through the request lifecycle.**  
    `public/index.php` → autoload → `bootstrap/app.php` builds the Application → resolve HTTP Kernel → `handle()` runs bootstrappers (env, config, exceptions, facades, register providers, boot providers) → global middleware → router match → route middleware → resolve controller from container → route model binding + Form Request validation → action → response conversion → middleware unwind → `send()` → `terminate()`.

22. **`register()` vs `boot()`?**  
    `register()` only binds into the container; other providers may not be registered yet so you must not resolve services. `boot()` runs after all providers are registered — safe place for gates, listeners, routes, macros.

23. **How does a Facade work?**  
    `__callStatic` resolves the container binding named by `getFacadeAccessor()` and forwards the call to that instance. It's a static-looking proxy, not a static class, which is why it's mockable.

24. **Why is `env()` `null` in production?**  
    Because `config:cache` skips `LoadEnvironmentVariables`. Only call `env()` inside `config/*.php`; use `config()` everywhere else.

25. **What is middleware and how is the pipeline built?**  
    A layered filter around the request. `Pipeline` uses `array_reduce` over the reversed pipe list to build nested closures, so each middleware runs code before and after `$next($request)`.

26. **How do you run work after the response is sent?**  
    Terminable middleware (`terminate()`), a queued job, or `dispatch(...)->afterResponse()`.

27. **What is route model binding, and how does it become a security bug?**  
    Laravel resolves `{item}` to a model by key. Default binding has no tenant filter, so a user can fetch another tenant's record by ID (IDOR). Fix with a verified global scope, `resolveRouteBinding()` override, or `scopeBindings()` — and a test asserting 404.

28. **Why 404 instead of 403 for cross-tenant access?**  
    403 confirms the record exists, enabling enumeration of other tenants' data. 404 reveals nothing.

29. **`$fillable` vs `$guarded`?**  
    Whitelist vs blacklist for mass assignment. Prefer `$fillable`. Never include `organization_id`, `role`, or `is_admin`.

30. **Which Eloquent operations skip model events?**  
    `insert()`, `upsert()`, query-builder `update()`/`delete()`, `DB::table()`, `saveQuietly()`, `withoutEvents()`. Anything relying on observers (audit log, cache invalidation, search indexing) silently breaks.

31. **`chunk()` vs `chunkById()` — why does it matter?**  
    `chunk()` uses LIMIT/OFFSET; if your loop changes rows so they leave the result set, OFFSET skips records. `chunkById()` uses keyset pagination (`WHERE id > last`) and is safe for mutating iterations. Essential for backfills.

32. **What does `Rule::unique('items','sku')` do wrong in a multi-tenant app?**  
    It enforces global uniqueness across all tenants, so Org B can't use a SKU Org A already has. Scope it: `->where('organization_id', $orgId)`, and back it with a composite DB unique index.

33. **How do you prevent an N+1 from your API Resource layer?**  
    Use `whenLoaded()` for relations and `whenCounted()` for counts, and eager-load in the controller. Enable `Model::preventLazyLoading()` outside production so lazy loads throw during development and tests.

34. **What status code does Form Request validation failure return?**  
    422 with an `errors` object for JSON; redirect-back with flashed errors for web. `authorize()` returning false gives 403 before rules run.

35. **`whereRaw` with user input — is it safe?**  
    Only with bindings, and only for values. Column names, table names, and sort direction can't be bound — allowlist them. Dynamic `orderBy($request->input('sort'))` is an injection vector.

36. **Why does `route:cache` fail?**  
    Closure routes can't be serialized. Move them to controller classes.

37. **Why might `withoutOverlapping()` still let a scheduled command run twice?**  
    It relies on a cache lock. With the `file` or `array` driver each server has its own lock store. Use Redis/DB and `onOneServer()`.

38. **`chunk` a 15M-row backfill — what else do you need besides `chunkById`?**  
    Throttling (`usleep`) to bound replication lag and lock contention, a resumable checkpoint, `withoutGlobalScopes()` if it must cross tenants, `updateQuietly()` if observers would be too expensive, dry-run mode, and a reconciliation count to prove completeness.

39. **How do you make a partial/concurrent index in a Laravel migration?**  
    `DB::statement('CREATE INDEX CONCURRENTLY ... WHERE ...')` plus `public $withinTransaction = false;` because concurrent index builds can't run in a transaction.

40. **What's the difference between `increment()` and `$item->quantity++; $item->save();`?**  
    `increment()` issues a single `UPDATE ... SET quantity = quantity + 1` — atomic at the DB level. The read-modify-write version has a lost-update race between concurrent requests. (Full treatment in Tier 3.)

---

**Next:** [`02-intermediate.md`](./02-intermediate.md) — service container internals, the full relationship catalogue, queues and batching, caching and locks, and a complete testing strategy.
