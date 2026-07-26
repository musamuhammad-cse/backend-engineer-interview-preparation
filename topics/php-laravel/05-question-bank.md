# PHP / Laravel — Question Bank & Drill Material

> Use this daily. Cover the answer, respond out loud, then compare. If your spoken answer was vaguer than the written one, that's the gap.

---

## Contents

1. [Rapid-Fire: PHP Language (60)](#1-rapid-fire-php-language)
2. [Rapid-Fire: Laravel Framework (60)](#2-rapid-fire-laravel-framework)
3. [Rapid-Fire: Database & Performance (40)](#3-rapid-fire-database--performance)
4. [Rapid-Fire: Architecture & Systems (40)](#4-rapid-fire-architecture--systems)
5. [Code Puzzles — "What Does This Print?"](#5-code-puzzles--what-does-this-print)
6. [Live-Coding Exercises](#6-live-coding-exercises)
7. [Debugging Scenarios](#7-debugging-scenarios)
8. [System Design Prompts](#8-system-design-prompts)
9. [STAR Stories from Your Experience](#9-star-stories-from-your-experience)
10. [Questions to Ask the Interviewer](#10-questions-to-ask-the-interviewer)
11. [Red Flags to Avoid](#11-red-flags-to-avoid)

---

## 1. Rapid-Fire: PHP Language

**1. What's the output of `var_dump(0 == 'foo')` in PHP 8?**  
`false`. PHP 8 casts the number to string when the string isn't numeric. In PHP 7 it was `true`.

**2. `'1' == '01'`?**  
`true` — both are numeric strings, compared numerically.

**3. `'abc' == 0` in PHP 8?**  
`false`.

**4. `null == false`?**  
`true`. `null === false` is `false`.

**5. `'0'` truthy or falsy?**  
Falsy. But `'0.0'` and `'false'` are truthy.

**6. `[] == false`?**  
`true`. Empty array is falsy.

**7. Difference between `isset()` and `!empty()`?**  
`isset` = variable exists and isn't null. `!empty` = exists and is truthy. `isset($x)` is true for `$x = 0`; `!empty($x)` is false.

**8. `array_key_exists` vs `isset` on an array?**  
`isset` returns false when the value is null; `array_key_exists` returns true. Matters for distinguishing "sent as null" from "not sent."

**9. What does `??=` do?**  
Null coalescing assignment: assign only if the left side is null or unset.

**10. `?:` vs `??`?**  
`?:` falls through on any falsy value (`0`, `''`, `'0'`). `??` only on null/unset.

**11. What's `?->` and what does it return?**  
Nullsafe operator; short-circuits the entire chain to `null` if any link is null. It doesn't suppress errors on non-null values.

**12. `self` vs `static`?**  
`self` binds at compile time to the defining class; `static` binds at runtime to the called class (late static binding).

**13. `static::` inside a parent method returns what class?**  
The class the method was called on, not the class it was written in.

**14. Can you override a private method?**  
No — private methods aren't polymorphic. The parent's copy always runs from parent code.

**15. Trait vs interface vs abstract class?**  
Interface = contract, multiple, no implementation. Abstract = single inheritance with shared implementation. Trait = horizontal code copy, no type identity.

**16. Method precedence with traits?**  
Own class > trait > inherited parent.

**17. How do you resolve two traits with the same method?**  
`use A, B { A::hello insteadof B; B::hello as helloB; }`

**18. Is `readonly` deep?**  
No. You can't reassign the property, but an object it holds is still mutable internally.

**19. What does `clone` copy?**  
Shallow: scalars and arrays are copied; object properties still reference the same instances. Implement `__clone()` for deep copies.

**20. Difference between `Error` and `Exception`?**  
Both implement `Throwable`. `Error` is engine-level (TypeError, DivisionByZeroError); `Exception` is application-level. `catch (Exception)` does not catch `Error`.

**21. What catches everything?**  
`catch (Throwable $e)`.

**22. `LogicException` vs `RuntimeException`?**  
Logic = programming bug that should be fixed (don't catch). Runtime = condition only detectable at runtime (recoverable).

**23. What's wrong with `return` inside `finally`?**  
It overrides the try's return value and swallows in-flight exceptions.

**24. What is a generator?**  
A function containing `yield` that produces values lazily with constant memory, returning a `Generator` (one-shot forward-only iterator).

**25. Can you iterate a generator twice?**  
No — "Cannot rewind a generator that was already run."

**26. What does `yield from` do?**  
Delegates to another iterable, flattening its values into the current generator.

**27. Explain copy-on-write.**  
Assigning an array bumps a refcount instead of copying; the copy occurs on first write to either variable.

**28. Are objects passed by reference?**  
No — by handle. The handle is copied, but both handles point to the same instance, so mutations are visible to the caller. Reassigning the parameter doesn't affect the caller.

**29. Why `unset($v)` after `foreach ($arr as &$v)`?**  
`$v` remains a reference to the last element; a subsequent `foreach` overwrites that element on each iteration.

**30. How does PHP free memory?**  
Refcounting frees immediately at zero; a cycle collector handles circular references when the root buffer fills or on `gc_collect_cycles()`.

**31. Why do queue workers leak memory?**  
Long-lived process + reference cycles + static accumulation. Mitigate with `--max-jobs`, `--max-time`, and a memory limit.

**32. `match` vs `switch`?**  
`match` is an expression, uses `===`, no fallthrough, throws `UnhandledMatchError` without a default. `switch` uses `==`, falls through, is a statement.

**33. What does `match(true)` do?**  
Turns a match into a conditional chain, evaluating each arm's expression for truthiness against `true`.

**34. Union vs intersection type?**  
Union `A|B` = either. Intersection `A&B` = must satisfy both.

**35. What is `never`?**  
Return type meaning the function never returns normally — it throws or exits. Helps static analysis prove unreachable code.

**36. What is `static` as a return type?**  
Returns an instance of the called class, enabling fluent chains in subclasses.

**37. Purpose of `declare(strict_types=1)`?**  
Disables scalar coercion for calls made from that file. Per-file, applies to the caller.

**38. Why not use floats for money?**  
Binary floats can't represent decimal fractions exactly; `0.1 + 0.2 !== 0.3`. Use integer minor units or `DECIMAL`.

**39. What's `hash_equals` for?**  
Constant-time string comparison, preventing timing attacks on tokens and signatures.

**40. Why bcrypt/argon2 instead of SHA-256 for passwords?**  
Password hashes must be slow and memory-hard to resist GPU brute force; SHA-256 is fast by design. bcrypt/argon2 also salt per user.

**41. What are PHP attributes?**  
Native metadata (`#[Route(...)]`) readable via reflection — replacing docblock annotations.

**42. What's a first-class callable?**  
`strlen(...)` / `$obj->method(...)` creates a `Closure` from a callable (8.1+).

**43. Arrow function capture semantics?**  
Automatic by-value capture of the enclosing scope; single expression only.

**44. `use ($x)` vs `use (&$x)`?**  
By value captures at definition time; by reference reads the current value at call time.

**45. Why use `static fn`?**  
It doesn't bind `$this`, preventing accidental retention of the enclosing object — important in long-lived processes.

**46. What is `Closure::bind` used for?**  
Rebinding a closure's `$this` and scope, allowing access to private members. Laravel uses it for macros.

**47. Pure vs backed enum?**  
Pure has cases only; backed has a scalar value and supports `from`/`tryFrom` — required for DB persistence.

**48. `from` vs `tryFrom`?**  
`from` throws `ValueError` on invalid input; `tryFrom` returns null.

**49. Can enums have properties?**  
No mutable properties. They can have methods, constants, implement interfaces, and use traits.

**50. What are the SPL interfaces you use?**  
`ArrayAccess`, `Countable`, `IteratorAggregate`, `JsonSerializable`, `Stringable`.

**51. What is PSR-4?**  
Autoloading standard mapping namespace prefixes to directories.

**52. PSR-11?**  
Container interface — Laravel's container implements it.

**53. `composer install` vs `update`?**  
`install` honors `composer.lock` (reproducible, use in CI/prod); `update` re-resolves and rewrites the lock.

**54. What does `-o` do on `dump-autoload`?**  
Builds a static classmap so the autoloader doesn't hit the filesystem per class — a production optimization.

**55. Why does a class fail to load in prod but work on macOS?**  
Linux filesystems are case-sensitive; the class name and filename must match exactly.

**56. What is OPcache?**  
Shared-memory cache of compiled opcodes, skipping lex/parse/compile per request.

**57. What is JIT good for?**  
CPU-bound numeric work. Negligible for typical I/O-bound web apps.

**58. What is a SAPI?**  
The interface between PHP and its host — `fpm-fcgi`, `cli`, `swoole`, etc. Different lifetimes and defaults.

**59. What does `fastcgi_finish_request()` do?**  
Flushes the response to the client while PHP continues executing. Powers terminable middleware. Not available under CLI/Octane.

**60. What is a Fiber?**  
A suspendable/resumable call stack (8.1+), the primitive under async runtimes like Amp and Swoole coroutines.

---

## 2. Rapid-Fire: Laravel Framework

**61. Walk the request lifecycle in one breath.**  
`index.php` → autoload → `bootstrap/app.php` builds the Application → HTTP Kernel → bootstrappers (env, config, exceptions, facades, register providers, boot providers) → global middleware → route match → route middleware → controller resolved from container → bindings + Form Request → action → response conversion → middleware unwind → send → terminate.

**62. `register()` vs `boot()`?**  
`register` binds only (other providers may not exist yet); `boot` runs after all registration, safe for gates, listeners, routes, macros.

**63. What is a deferred provider?**  
Loaded lazily when one of its `provides()` bindings resolves, reducing bootstrap cost. Its `boot()` may never run.

**64. How does a Facade work?**  
`__callStatic` resolves the container binding from `getFacadeAccessor()` and forwards the call. Not a static class.

**65. What is a real-time facade?**  
Prefix any class with `Facades\` to use it statically without writing a facade class.

**66. Criticism of facades?**  
They hide dependencies — the constructor lies about what the class needs. Fine at the framework boundary, avoid in domain code.

**67. Why is `env()` null in production?**  
`config:cache` skips loading `.env`. Only call `env()` in config files.

**68. Why does `route:cache` fail?**  
Closure routes can't be serialized.

**69. How is the middleware pipeline built?**  
`array_reduce` over reversed pipes producing nested closures, so each middleware wraps the next — an onion, allowing before and after logic.

**70. Where does post-response work go?**  
Terminable middleware, a queued job, or `dispatch(...)->afterResponse()`.

**71. Why does middleware order matter for tenancy?**  
Tenant middleware needs the authenticated user, and route model binding must be tenant-aware — so tenant must run after `Authenticate` and before `SubstituteBindings`.

**72. `bind` vs `singleton` vs `scoped`?**  
New each resolution / one per container lifetime / one per request-or-job and flushed by Octane.

**73. Which do you use for tenant context and why?**  
`scoped` — a singleton persists across requests under Octane and leaks tenant state.

**74. What is contextual binding?**  
`$app->when(X)->needs(Y)->give(Z)` — different implementations of the same interface per consumer.

**75. What is container tagging for?**  
Resolving a group of related bindings (`$app->tagged('report.sections')`) to build extensible pipelines without a giant conditional.

**76. How do you decorate a binding?**  
`$app->extend(Interface::class, fn ($inner) => new Decorator($inner))`.

**77. What's the service locator anti-pattern?**  
Calling `app()`/`resolve()` inside domain classes; hides dependencies and breaks testability/static analysis.

**78. Implicit vs explicit route model binding?**  
Implicit resolves by type-hint and route key; explicit uses `Route::bind()` with custom logic.

**79. How do you make route binding tenant-safe?**  
Override `resolveRouteBinding()` to add the tenant filter, or rely on a verified global scope, or use `scopeBindings()` on nested routes — and test for 404.

**80. Why 404 rather than 403 cross-tenant?**  
403 confirms the resource exists, enabling enumeration.

**81. `$fillable` vs `$guarded`?**  
Whitelist vs blacklist for mass assignment. Prefer `$fillable`; never include `organization_id` or role flags.

**82. What does `$request->validated()` protect against that `all()` doesn't?**  
Mass assignment of unexpected fields the client injected.

**83. Which Eloquent operations skip events?**  
`insert`, `upsert`, query-builder `update`/`delete`, `DB::table`, `saveQuietly`, `withoutEvents`.

**84. `$item->relation` vs `$item->relation()`?**  
Property returns a cached collection; method returns a fresh Builder that queries every call.

**85. What is `withCount` and when do you use it?**  
Adds a `{relation}_count` via subquery, avoiding a per-row count query.

**86. `whenLoaded` in an API Resource — why?**  
Returns `MissingValue` (stripped from output) when the relation isn't eager-loaded, preventing a resource-layer N+1.

**87. `chunk` vs `chunkById`?**  
`chunk` uses OFFSET and skips rows if you mutate the filtered column; `chunkById` uses keyset pagination and is safe.

**88. `cursor` vs `lazy` vs `lazyById`?**  
`cursor` = one query, one model at a time (driver may still buffer). `lazy` = chunked queries, one model at a time. `lazyById` = same but keyset-paginated, safe for mutation.

**89. What does `toBase()` do?**  
Returns raw `stdClass` rows without Eloquent hydration — much lower memory and CPU for read-only reporting.

**90. Offset vs cursor pagination?**  
Offset supports totals and page jumps but degrades with depth and is unstable under inserts. Cursor is O(1) and stable but forward/backward only.

**91. What is a global scope and how does it fail?**  
An automatic query constraint. Bypassed by raw queries, removed explicitly, or absent in console contexts. Make it fail closed.

**92. Local scope syntax?**  
`public function scopeLowStock(Builder $q): void` used as `Model::lowStock()`.

**93. Model event order for an update?**  
`saving` → `updating` → `updated` → `saved`.

**94. `isDirty` vs `wasChanged`?**  
`isDirty` = modified but not yet saved. `wasChanged` = changed by the most recent save.

**95. What's the observer + transaction race?**  
An observer dispatches a job inside a transaction; the worker runs before COMMIT and can't see the row. Fix with `$afterCommit`.

**96. Why pass IDs to jobs instead of models?**  
Smaller payloads, fresh data on execution, and an explicit contract. `SerializesModels` re-fetches and throws if the row is gone.

**97. Why must `retry_after` exceed job `timeout`?**  
Otherwise the job is released to another worker while still running — concurrent duplicate execution.

**98. What's the queue deploy trap?**  
Workers hold old code in memory; run `queue:restart` (or roll worker containers) every deploy.

**99. `ShouldBeUnique` vs `WithoutOverlapping`?**  
Prevents duplicate **queued** jobs vs prevents duplicate **concurrent execution**.

**100. Batch vs chain?**  
Parallel with aggregate progress vs strictly ordered stop-on-failure.

**101. How do you track batch progress from an API?**  
`Bus::findBatch($id)` exposing `totalJobs`, `pendingJobs`, `failedJobs`, `progress()`, `finished()`.

**102. What delivery guarantee do queues give?**  
At-least-once. Handlers must be idempotent.

**103. Redis vs SQS queue?**  
Redis + Horizon for in-app work with rich metrics and unique/overlap middleware; SQS for cross-service, AWS-native fan-out, huge scale, 256 KB payload cap, no Horizon.

**104. What must a tenant-aware job do first?**  
Set `TenantContext` and `setPermissionsTeamId()` from the payload — no middleware runs for jobs.

**105. What is a cache stampede?**  
A hot key expires and all concurrent requests recompute simultaneously, overwhelming the DB.

**106. Four stampede fixes?**  
Lock-and-rebuild with others waiting or serving stale; stale-while-revalidate (`Cache::flexible`); TTL jitter; scheduled warming.

**107. Why do cache locks need Redis?**  
Locks require an atomic shared store; file/array stores are per-process.

**108. Alternative to cache tags?**  
Key versioning — embed a version in the key and `increment` it to invalidate atomically; works on any store.

**109. What's the #1 multi-tenant caching bug?**  
An unnamespaced key serving one tenant's data to another.

**110. Guard vs provider?**  
Guard = how a request is authenticated (session, token). Provider = where users come from (Eloquent, database).

**111. Sanctum vs Passport?**  
Sanctum for first-party SPA/mobile with opaque revocable tokens; Passport for full OAuth 2.0 with third-party clients, scopes, refresh tokens.

**112. Why is the implicit grant deprecated?**  
Token in the URL fragment (history, referrer, logs) with no client authentication. Replaced by authorization code + PKCE.

**113. What does PKCE protect against?**  
Interception of the authorization code — the attacker lacks the `code_verifier`.

**114. What does the `state` parameter protect against?**  
CSRF on the authorization flow.

**115. Scope vs permission?**  
Scope = what the client app may do with this token. Permission = what the user is allowed to do. The effective right is the intersection.

**116. How do you revoke a JWT instantly?**  
Not purely statelessly — short TTLs, a stored revocation check (Passport's approach), or a Redis `jti` denylist.

**117. Gate vs Policy?**  
Gate = simple closure ability. Policy = class of abilities for a model, resolved by type.

**118. Why check permissions not roles?**  
Roles are admin-editable bundles; hardcoding roles turns policy changes into deploys.

**119. What breaks with spatie teams in queued jobs?**  
No middleware sets the team id, so permission checks use stale/missing context. Call `setPermissionsTeamId()` in `handle()`.

**120. Why must you reset the permission cache in tests?**  
spatie caches the whole permission graph; leftover cache causes tests that pass alone and fail in a suite.

---

## 3. Rapid-Fire: Database & Performance

**121. What does `EXPLAIN ANALYZE` add over `EXPLAIN`?**  
It actually executes the query and reports real timings and row counts, letting you compare estimates against reality.

**122. Composite index column order rule?**  
Equality columns first, then range/sort. Left-prefix rule governs usability.

**123. Which column leads every index in a multi-tenant app?**  
`organization_id`.

**124. What is a covering index?**  
An index containing all columns the query needs (via `INCLUDE` in Postgres), enabling an index-only scan without a heap fetch.

**125. What is a partial index?**  
An index with a `WHERE` clause covering only the rows you query — smaller, faster, cheaper to maintain.

**126. When would you use GIN?**  
`jsonb` containment, array membership, full-text search, and trigram `ILIKE '%x%'`.

**127. How do you make `LIKE '%term%'` fast?**  
`pg_trgm` extension with a GIN trigram index, or move to full-text search / Elasticsearch.

**128. Why might the planner ignore your index?**  
Function on the column, leading wildcard, type mismatch, not a left prefix, low selectivity, stale stats, or a small table.

**129. What does `ANALYZE` do?**  
Refreshes planner statistics so estimates match reality.

**130. Cost of an index?**  
Write amplification on every insert/update/delete touching its columns, plus disk. Drop indexes with `idx_scan = 0`.

**131. How do you find the queries actually costing you?**  
`pg_stat_statements` ordered by `total_exec_time`, not by slowest single execution.

**132. How do you add an index to a huge live table?**  
`CREATE INDEX CONCURRENTLY`, outside a transaction, with retry handling for INVALID index cleanup.

**133. Postgres default isolation?**  
READ COMMITTED.

**134. MySQL InnoDB default isolation?**  
REPEATABLE READ.

**135. What is a phantom read?**  
The same range query returns different *rows* within one transaction.

**136. What is write skew?**  
Two transactions read overlapping sets, each writes based on its read, and together they violate an invariant neither broke alone.

**137. What does SERIALIZABLE cost in Postgres?**  
It aborts conflicting transactions (SQLSTATE 40001) rather than blocking, so you must implement retries; throughput drops under contention.

**138. What causes a deadlock?**  
Two transactions acquiring the same locks in opposite order.

**139. Primary deadlock prevention?**  
Deterministic lock ordering (e.g. sort IDs ascending), short transactions, `lock_timeout`, and retries.

**140. How does Laravel retry deadlocks?**  
`DB::transaction($closure, attempts: 3)`.

**141. What is `SELECT ... FOR UPDATE`?**  
An exclusive row lock held until COMMIT; concurrent transactions touching those rows block.

**142. What is `SKIP LOCKED` for?**  
Claiming work rows concurrently without blocking — the SQL primitive behind safe job queues and outbox relays.

**143. `pg_advisory_lock` vs `pg_advisory_xact_lock`?**  
Session-scoped (leaks if not released, dangerous with poolers) vs transaction-scoped (auto-released at commit/rollback). Prefer the latter.

**144. Optimistic vs pessimistic locking — deciding factor?**  
Contention rate. Low contention → optimistic. High contention with expensive work → pessimistic.

**145. Why can't a `SELECT`-then-`INSERT` uniqueness check be trusted?**  
Two requests can both pass the check before either inserts. The unique index is the actual guarantee.

**146. Why is `increment()` safe but `$m->qty++; $m->save()` isn't?**  
`increment` issues `SET qty = qty + 1` as a single atomic statement; the other is a read-modify-write with a race window.

**147. What does `sticky = true` do?**  
After a write in a request, subsequent reads use the primary — avoiding read-after-write failures against a lagging replica.

**148. Why do PHP apps exhaust DB connections?**  
One connection per php-fpm worker; workers × servers quickly exceeds `max_connections`.

**149. Which PgBouncer mode works with Laravel?**  
Transaction mode — with `PDO::ATTR_EMULATE_PREPARES` and no session-scoped state (no `pg_advisory_lock`, no `LISTEN`, no temp tables).

**150. What is `idle_in_transaction_session_timeout` for?**  
Killing transactions left open by a crashed or hung client that would otherwise hold locks indefinitely.

**151. When is `REFRESH MATERIALIZED VIEW CONCURRENTLY` possible?**  
When the view has a unique index; it avoids blocking readers during refresh.

**152. Why is `ADD COLUMN ... DEFAULT` safe in PG 11+ but not PG 10?**  
PG 11+ stores the default as metadata rather than rewriting every row.

**153. How do you add NOT NULL without a long lock?**  
`ADD CONSTRAINT ... CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` (concurrent-friendly), then `SET NOT NULL`, then drop the check.

**154. Why is even a brief ACCESS EXCLUSIVE lock risky?**  
It queues behind long-running queries and everything else queues behind it, converting a 5-minute analytics query into a 5-minute outage for that table.

**155. Why never `renameColumn` on a hot table?**  
Rolling deploys run old and new code simultaneously; the rename breaks whichever version doesn't know about it.

**156. What is expand → migrate → contract?**  
Make the schema accept both shapes, backfill, switch reads, then remove the old shape — so code and schema are compatible at every instant.

**157. What makes a backfill resumable?**  
A persisted checkpoint (last processed ID) plus keyset pagination and idempotent updates.

**158. What is replica-lag backpressure?**  
Pausing a backfill when replication lag exceeds a threshold, so bulk writes don't degrade read replicas.

**159. How do you prove a backfill completed?**  
A reconciliation query asserting zero unmigrated rows, plus comparison against a pre-migration count snapshot.

**160. Why use `DECIMAL` not `FLOAT` for money in Postgres?**  
Exact decimal arithmetic; floats introduce representation error that compounds across aggregations.

---

## 4. Rapid-Fire: Architecture & Systems

**161. Three multi-tenancy models and the deciding factors?**  
Row-level, schema-per-tenant, database-per-tenant. Decide on tenant count, isolation/compliance requirements, cost per tenant, and operational complexity.

**162. Biggest risk of row-level tenancy?**  
Isolation is application-enforced — one missing `WHERE` affects every customer.

**163. Fail open or fail closed?**  
Fail closed in HTTP contexts; a missing tenant should return nothing, not everything.

**164. How do you stop cross-tenant foreign keys?**  
Composite FK on `(organization_id, id)` plus tenant-scoped `exists` validation.

**165. What's the outbox pattern for?**  
Atomically committing a state change and its event, avoiding the dual-write problem between database and message broker.

**166. What guarantee does an outbox relay give?**  
At-least-once — so consumers must be idempotent.

**167. When is CQRS justified?**  
When read and write workloads differ fundamentally in shape and scale, and you accept eventual consistency plus two models.

**168. Would you use event sourcing?**  
Rarely system-wide; a ledger (append-only movements plus rollup) gives most of the audit benefit at far lower complexity. Reserve full ES for regulated bounded contexts.

**169. When NOT to use the repository pattern?**  
When the interface mirrors Eloquent one-to-one. Use it when the domain must not know about Eloquent or you need to decorate/swap persistence.

**170. When NOT to use Clean Architecture?**  
CRUD modules with trivial logic — the abstractions become ceremony. Apply it where business rules are complex and long-lived.

**171. Give a real Open/Closed example.**  
Alert channels tagged in the container: adding a channel is a new class plus a tag, with no change to the dispatcher.

**172. What is the Manager pattern in Laravel?**  
A driver factory (`createRedisDriver`, `createSqsDriver`) selecting an implementation by config — Strategy plus Factory. It's how you extend cache/queue/auth drivers.

**173. Where does Laravel use Chain of Responsibility?**  
The middleware pipeline, built with `array_reduce` over reversed pipes.

**174. Where does it use Decorator?**  
`$app->extend()`, and `Cache\Repository` wrapping a store.

**175. Where does it use Null Object?**  
`belongsTo()->withDefault()`.

**176. Why autoscale workers on queue depth not CPU?**  
I/O-blocked workers show low CPU while the backlog grows.

**177. What must be externalized for horizontal scaling?**  
Sessions, cache, uploads, queues, locks, logs — nothing on local disk or in process memory.

**178. What breaks under Octane?**  
Singletons holding request state, statics, injected `Request` objects captured in constructors, runtime config mutation, closures retaining large graphs.

**179. Why is Octane riskier in multi-tenant apps?**  
Persisted state becomes a cross-tenant data leak, not just a bug.

**180. What's the graceful shutdown requirement for workers?**  
The termination grace period must exceed the longest job timeout, or SIGKILL kills jobs mid-execution — relying on at-least-once retry.

**181. Why must `php artisan down` use the cache driver in a fleet?**  
Otherwise only the server that ran it enters maintenance mode.

**182. Why not `config:cache` at Docker build time?**  
It bakes in build-time env values and ignores runtime secrets; run it in the entrypoint.

**183. What's `migrate --isolated` for?**  
Ensures only one instance runs migrations during a parallel rolling deploy, using a cache lock.

**184. What do you tag every log line with?**  
`request_id`, `organization_id`, `user_id`, route/job name.

**185. How do you correlate an HTTP request with its queued work?**  
Carry the request ID in the job payload and re-establish log context in `handle()`; with OTel, propagate trace context so spans join the same trace.

**186. Four golden signals?**  
Latency, traffic, errors, saturation — plus queue depth and oldest-job age for async systems.

**187. How do you debug "slow for one customer only"?**  
Filter traces and slow-query logs by `organization_id`; usually data-volume skew causing a plan flip, a missing composite index, or an unbounded page size.

**188. Main SSRF defenses?**  
HTTPS-only, resolve and reject private/link-local ranges (especially 169.254.169.254), disable redirects, short timeouts, and connect to the resolved IP or use an enforcing egress proxy to avoid DNS rebinding.

**189. How do you verify a webhook?**  
HMAC-SHA256 over the raw body compared with `hash_equals`, a timestamp tolerance window against replay, and idempotency on the event ID.

**190. Where can SQL injection still occur in Laravel?**  
Identifiers — `orderBy($userInput)`, dynamic column/table names, `whereRaw` with interpolation. Bindings protect values only; allowlist identifiers.

**191. What's the risk of `Gate::before`?**  
It bypasses every policy including tenant boundaries; require audited, time-boxed impersonation instead.

**192. Why is a 403 sometimes worse than a 404?**  
It confirms the resource exists, enabling enumeration of other tenants' IDs.

**193. Where do you mock, and where don't you?**  
Mock process boundaries (HTTP, payments, mail, S3). Don't mock the database — query and scope bugs are what you need to catch.

**194. Why run Postgres in CI rather than SQLite?**  
SQLite differs on jsonb, `ILIKE`, constraints, `FOR UPDATE SKIP LOCKED`, and transaction semantics, so it hides real production bugs.

**195. What breaks under `RefreshDatabase`?**  
Anything depending on a real commit — `DB::afterCommit`, `$afterCommit` jobs, cross-connection lock visibility.

**196. How do you prevent N+1 regressions permanently?**  
`preventLazyLoading` outside production, query-count assertions in feature tests, slow-query logging, and a per-request query budget alert.

**197. Give three worthwhile architecture tests.**  
Domain layer must not import framework HTTP/facades; DTOs must be readonly; `dd`/`dump`/`var_dump` never used. Add: every model with `organization_id` uses the tenant trait.

**198. How do you size php-fpm workers?**  
`(memory_limit − reserved) / avg_worker_RSS`, cross-checked against the DB connection ceiling; add PgBouncer if the DB is the binding constraint.

**199. When does adding workers make things worse?**  
When the bottleneck is downstream — more workers just deepen the queue at the database and can exhaust connections.

**200. Is PHP right for high-concurrency I/O fan-out?**  
Process-per-request is memory-bound for that pattern. Use `Http::pool()` or Octane coroutines within Laravel, or build that component in Go — the reasoning behind Chronos.

---

## 5. Code Puzzles — "What Does This Print?"

### Puzzle 1

```php
class A {
    public static function create(): self { return new self(); }
    public static function make(): static { return new static(); }
}
class B extends A {}

echo get_class(B::create()), ' ', get_class(B::make());
```

<details><summary>Answer</summary>

`A B`

`self` binds at compile time to the defining class (`A`). `static` uses late static binding and resolves to the called class (`B`).
</details>

### Puzzle 2

```php
$arr = [1, 2, 3];
foreach ($arr as &$v) {}
foreach ($arr as $v) {}
print_r($arr);
```

<details><summary>Answer</summary>

`[1, 2, 2]`

After the first loop, `$v` is still a reference to `$arr[2]`. The second loop assigns each element into `$v`, i.e. into `$arr[2]`: it becomes 1, then 2, then finally the value of `$arr[2]` itself (2). Fix with `unset($v)`.
</details>

### Puzzle 3

```php
function f(): string {
    try {
        return 'try';
    } finally {
        return 'finally';
    }
}
echo f();
```

<details><summary>Answer</summary>

`finally` — a return in `finally` overrides the try's return and would also swallow an exception. Never do this.
</details>

### Puzzle 4

```php
class Base {
    private function name(): string { return 'base'; }
    public function get(): string { return $this->name(); }
}
class Child extends Base {
    private function name(): string { return 'child'; }
}
echo (new Child)->get();
```

<details><summary>Answer</summary>

`base` — private methods aren't polymorphic; `get()` resolves `name()` in `Base`'s scope. Change to `protected` and it prints `child`.
</details>

### Puzzle 5

```php
$m = 2;
$a = fn ($n) => $n * $m;
$b = function ($n) use (&$m) { return $n * $m; };
$m = 10;
echo $a(5), ' ', $b(5);
```

<details><summary>Answer</summary>

`10 50` — arrow functions capture by value at definition; `use (&$m)` reads the current value at call time.
</details>

### Puzzle 6

```php
$a = [1, 2, 3];
unset($a[1]);
echo json_encode($a);
```

<details><summary>Answer</summary>

`{"0":1,"2":3}` — the keys are no longer sequential, so `json_encode` emits an object. This is the classic API bug after `filter()`. Fix with `array_values()` / `->values()`.
</details>

### Puzzle 7

```php
var_dump(0 == 'a');
var_dump('1' == '01');
var_dump('10' == '1e1');
var_dump(100 == '1e2');
var_dump(null == 0);
var_dump('' == null);
```

<details><summary>Answer</summary>

PHP 8: `false, true, true, true, true, true`.  
The first is `true` in PHP 7 — know the version difference. Numeric strings compare numerically; non-numeric strings force a string comparison in PHP 8.
</details>

### Puzzle 8

```php
$items = InventoryItem::with(['movements' => fn ($q) => $q->limit(3)])->get();
echo $items->sum(fn ($i) => $i->movements->count());
```

<details><summary>Answer</summary>

`3` (total, not 3 per item). The limit applies to the single combined `WHERE IN` eager-load query, so only three movement rows are fetched overall and distributed to whichever parents they belong to. Use `hasOne()->latestOfMany()` or a window function.
</details>

### Puzzle 9

```php
DB::transaction(function () {
    $order = Order::create(['total' => 100]);

    try {
        DB::transaction(function () {
            throw new RuntimeException('inner fail');
        });
    } catch (RuntimeException) {
        // swallowed
    }
});

echo Order::count();
```

<details><summary>Answer</summary>

`1` — the inner transaction is a savepoint. Rolling back to the savepoint doesn't affect the outer transaction, which commits the order. This is exactly how partial/inconsistent state gets committed when people assume nested transactions are independent.
</details>

### Puzzle 10

```php
class Item { public array $tags = []; }

final readonly class Box {
    public function __construct(public Item $item) {}
}

$box = new Box(new Item());
$box->item->tags[] = 'new';
echo count($box->item->tags);
```

<details><summary>Answer</summary>

`1` — `readonly` prevents reassigning `$box->item` but does not make the referenced object immutable. Readonly is shallow.
</details>

### Puzzle 11

```php
$item = InventoryItem::find(1);   // quantity = 10
InventoryItem::where('id', 1)->update(['quantity' => 5]);
echo $item->quantity;
echo $item->fresh()->quantity;
```

<details><summary>Answer</summary>

`10` then `5`. The in-memory model is a snapshot; a query-builder update doesn't refresh it (and fires no model events). `fresh()` re-queries; `refresh()` reloads in place.
</details>

### Puzzle 12

```php
Cache::put('k', 'v', 60);
Cache::tags(['t'])->put('k', 'v2', 60);
echo Cache::get('k');
echo Cache::tags(['t'])->get('k');
```

<details><summary>Answer</summary>

`v` then `v2` — tagged cache entries live in a separate namespace. A tagged `put` is not visible to an untagged `get` and vice versa. This trips people writing with tags and reading without them.
</details>

---

## 6. Live-Coding Exercises

### Exercise 1 — Safe stock deduction

> Implement an endpoint that deducts stock. It must be safe under concurrency, tenant-scoped, idempotent, and record an audit trail.

```php
// routes/api.php
Route::post('/inventory/{item}/deduct', DeductStockController::class)
    ->middleware(['auth:api', 'tenant', 'idempotent', 'scopes:inventory:write']);

// Controller
final class DeductStockController
{
    public function __construct(private readonly DeductStockAction $action) {}

    public function __invoke(DeductStockRequest $request, InventoryItem $item): JsonResponse
    {
        $result = $this->action->execute(DeductStockData::fromRequest($request, $item));

        return response()->json([
            'item_id'   => $result->id,
            'remaining' => $result->quantity,
        ]);
    }
}

// Action
final readonly class DeductStockAction
{
    public function __construct(private DatabaseManager $db, private Dispatcher $events) {}

    public function execute(DeductStockData $data): InventoryItem
    {
        return $this->db->transaction(function () use ($data) {
            // Atomic conditional update: check and write in one statement
            $affected = InventoryItem::query()
                ->whereKey($data->itemId)
                ->where('organization_id', $data->organizationId)
                ->where('quantity', '>=', $data->amount)
                ->decrement('quantity', $data->amount);

            if ($affected === 0) {
                $item = InventoryItem::query()
                    ->whereKey($data->itemId)
                    ->where('organization_id', $data->organizationId)
                    ->first();

                throw $item === null
                    ? new ModelNotFoundException()
                    : new InsufficientStockException($data->itemId, $data->amount, $item->quantity);
            }

            // Ledger row; unique index on (organization_id, idempotency_key) is the real guard
            StockMovement::create([
                'organization_id'   => $data->organizationId,
                'inventory_item_id' => $data->itemId,
                'type'              => MovementType::Outbound,
                'quantity'          => $data->amount,
                'reference_type'    => $data->referenceType,
                'reference_id'      => $data->referenceId,
                'idempotency_key'   => $data->idempotencyKey,
                'user_id'           => $data->userId,
            ]);

            $item = InventoryItem::whereKey($data->itemId)->firstOrFail();

            $this->events->dispatch(new StockDeducted(
                itemId:         $item->id,
                organizationId: $data->organizationId,
                amount:         $data->amount,
                remaining:      $item->quantity,
            ));

            return $item;
        }, attempts: 3);
    }
}
```

**Talk through while coding:** why atomic UPDATE over `lockForUpdate` (higher concurrency, no lock held across the whole transaction); why 404 vs 409 distinction requires a second query; why the ledger row carries the idempotency key; why `attempts: 3` (deadlock/serialization retry); why no HTTP calls inside the transaction.

### Exercise 2 — Fix this N+1

> Given this endpoint, reduce query count without changing the response shape.

```php
// BEFORE — ~200 queries for 50 items
public function index(): JsonResponse
{
    $items = InventoryItem::all();

    return response()->json($items->map(fn ($item) => [
        'sku'            => $item->sku,
        'supplier'       => $item->supplier->name,
        'movement_count' => $item->movements()->count(),
        'last_movement'  => $item->movements()->latest()->first()?->created_at,
        'can_edit'       => auth()->user()->can('update', $item),
    ]));
}
```

```php
// AFTER — 3 queries, paginated, tenant-scoped
public function index(Request $request): AnonymousResourceCollection
{
    $items = InventoryItem::query()
        ->with('supplier:id,name')
        ->withCount('movements')
        ->addSelect(['last_movement_at' => StockMovement::select('created_at')
            ->whereColumn('inventory_item_id', 'inventory_items.id')
            ->latest()
            ->limit(1),
        ])
        ->paginate(min($request->integer('per_page', 25), 100));

    // Load the permission graph once instead of per row
    $request->user()->loadMissing('roles.permissions');

    return InventoryItemResource::collection($items);
}
```

**Points to make:** `all()` → paginate with a clamped max; eager load with column selection; `withCount` replaces per-row counts; correlated subquery replaces per-row "latest"; permission relations loaded once; then add a query-count assertion so it can't regress.

### Exercise 3 — Tenant-scoped rate limiter

```php
// AppServiceProvider::boot()
RateLimiter::for('api', function (Request $request) {
    $user = $request->user();

    if (! $user) {
        return Limit::perMinute(30)->by($request->ip());
    }

    $plan = $user->organization->plan;

    return [
        // Fair-share across tenants: one org can't starve others
        Limit::perMinute($plan->requestsPerMinute)
            ->by("org:{$user->organization_id}")
            ->response(fn () => response()->json([
                'error' => 'rate_limited',
                'scope' => 'organization',
            ], 429)),

        // Per-user cap inside the org
        Limit::perMinute(120)->by("user:{$user->id}"),
    ];
});
```

### Exercise 4 — Idempotent import command

```php
final class ImportCatalog extends Command
{
    protected $signature = 'catalog:import {file} {--org=} {--chunk=1000}';

    public function handle(): int
    {
        $orgId = (int) $this->option('org');
        $chunk = (int) $this->option('chunk');

        app(TenantContext::class)->setOrganizationId($orgId);

        $rows = LazyCollection::make(function () {
            $handle = fopen($this->argument('file'), 'r');
            $header = fgetcsv($handle);

            while (($row = fgetcsv($handle)) !== false) {
                yield array_combine($header, $row);
            }

            fclose($handle);
        });

        $imported = 0;

        $rows->filter(fn ($r) => filled($r['sku'] ?? null))
            ->map(fn ($r) => [
                'organization_id' => $orgId,
                'sku'             => strtoupper(trim($r['sku'])),
                'name'            => $r['name'],
                'quantity'        => (int) $r['quantity'],
                'updated_at'      => now(),
                'created_at'      => now(),
            ])
            ->chunk($chunk)
            ->each(function (LazyCollection $batch) use (&$imported) {
                // Idempotent: re-running the same file converges to the same state
                InventoryItem::upsert(
                    $batch->all(),
                    uniqueBy: ['organization_id', 'sku'],
                    update: ['name', 'quantity', 'updated_at'],
                );

                $imported += $batch->count();
                $this->line("Imported {$imported}");
            });

        // upsert skips model events — invalidate and reindex explicitly
        Cache::tags(["org:{$orgId}"])->flush();
        ReindexOrganization::dispatch($orgId);

        return self::SUCCESS;
    }
}
```

**Points to make:** constant memory via `LazyCollection`; `upsert` with the tenant-scoped unique key makes re-runs idempotent; explicitly acknowledge that `upsert` skips events so cache invalidation and reindexing are manual.

---

## 7. Debugging Scenarios

> These are "you're on call" questions. The interviewer wants your diagnostic *process*, not a lucky guess.

### Scenario 1 — API latency spiked 10× after a deploy, no code changes to that endpoint

**Process:**
1. Confirm scope — all endpoints or one? All tenants or one? Check p50 vs p99 (broad slowdown vs tail).
2. Check saturation first: php-fpm busy workers, DB connections, Redis memory, CPU. Saturation means the cause is elsewhere but manifests everywhere.
3. Check `pg_stat_statements` deltas — did a query's mean time change, or its call count?
4. Check whether a new index was added (write amplification) or dropped, or whether stats went stale after a bulk load (plan flip from index scan to seq scan → `ANALYZE`).
5. Check queue depth — if workers are backed up, they may be saturating the DB connection pool and starving the web tier.
6. Check whether the deploy changed eager loading, added an observer, or enabled a feature flag.

**Most likely causes:** a plan flip from stale statistics after a data load; an observer added to a hot model; a new N+1 introduced in a shared resource/serializer; a cache key change causing a cold-cache stampede.

### Scenario 2 — One tenant reports the dashboard times out; everyone else is fine

**Process:**
1. Filter traces and slow-query logs by that `organization_id` — this only works because you tagged them.
2. Compare that tenant's data volume against the median. Usually they have 100× the rows, so a query that index-scans for everyone else now seq-scans or does an expensive sort.
3. `EXPLAIN ANALYZE` the query with their actual `organization_id` — parameter-dependent plans differ.
4. Check whether their request bypasses cache (different filters, an unbounded page size, or a cache key with low hit rate for their access pattern).

**Fixes:** composite index leading with `organization_id` covering their filter/sort; clamp page size; pre-aggregate their dashboard into a materialized view or cached rollup; consider table partitioning by tenant if they're an outlier.

### Scenario 3 — Customers report duplicate orders

**Process:**
1. Query for duplicates and inspect timestamps — milliseconds apart means client retry or double-click; seconds apart means a job retry.
2. Check whether the endpoint enforces idempotency and whether there's a unique index (a check-then-insert without the index races).
3. Check job retry configuration — is `timeout` ≥ `retry_after`? That causes concurrent duplicate execution.
4. Check for a load balancer or client-side retry on timeout: the server succeeded but the response was lost.

**Fixes:** `Idempotency-Key` header with a unique index on `(organization_id, idempotency_key)`, replay the stored response on retry, fix the `retry_after`/`timeout` relationship, and make the job handler idempotent.

### Scenario 4 — Queue backlog growing, workers appear idle

**Process:**
1. Check whether workers are actually consuming: Horizon throughput, `queue:monitor`, worker logs.
2. Check queue *names* — jobs dispatched to `inventory` while workers listen only to `default` is the most common cause.
3. Check `WithoutOverlapping` / `ShouldBeUnique` locks: a stale lock from a killed worker blocks the queue until TTL expiry.
4. Check whether workers are alive but blocked on a slow external API or a DB lock.
5. Check Redis memory/eviction — if the queue store is evicting keys, jobs vanish.

**Fixes:** align queue names, set `expireAfter` on overlap locks, add per-job timeouts, move slow external calls behind a circuit breaker, autoscale on queue depth.

### Scenario 5 — Memory usage climbing steadily in queue workers until OOM

**Process:**
1. Log `memory_get_usage(true)` per job with the job class; find which class correlates with growth.
2. Look for accumulating static arrays, an in-memory cache without bounds, event listeners registered per job, or closures capturing large graphs.
3. Check for `Model::all()` or unchunked iteration inside a job.
4. Check for reference cycles (parent↔child model references) that the cycle collector isn't reaching.

**Immediate mitigation:** `--max-jobs`, `--max-time`, `--memory` so workers recycle. **Real fix:** find the retaining reference.

### Scenario 6 — A user in Org A can see Org B's data

**This is a Sev-1. Process:**
1. Contain: identify the endpoint, disable it or hotfix the scope, and preserve logs.
2. Determine scope of exposure: which records, which tenants, over what window — from access logs tagged with `organization_id`.
3. Root cause: run through the leak-vector checklist — raw query, removed scope, unscoped route binding, ungrouped `orWhere`, unnamespaced cache key, job without tenant context, Octane singleton.
4. Fix and add a regression test that would have caught it.
5. Audit for the same pattern elsewhere: grep for `DB::table` on tenant tables, `withoutGlobalScope`, `orWhere` chains, and cache keys lacking a tenant prefix.
6. Notify per your incident and data-protection obligations.

**Saying "and I'd audit for the same class of bug everywhere else, then add a test" is the answer that lands.**

---

## 8. System Design Prompts

For each: clarify requirements, estimate scale, sketch the data model, name the bottleneck, and state trade-offs. Spend the first 3–5 minutes on requirements — jumping to a diagram is the most common failure.

### Design 1 — Multi-tenant inventory management SaaS

*(Your actual system — you should be able to do this cold and in depth.)*

**Clarify:** number of tenants and size distribution; items per tenant; read/write ratio; real-time requirements; integrations; compliance/isolation requirements; multi-warehouse; offline support.

**Scale estimate:** 5,000 tenants × 10,000 items avg = 50M items; 100 stock movements/tenant/day = 500K writes/day ≈ 6/s average, ~50/s peak; reads 10× writes.

**Core model:**
```
organizations (id, name, plan_id, settings)
users (id, organization_id, email)
inventory_items (id, organization_id, sku, name, quantity, reorder_point, ...)
    UNIQUE (organization_id, sku)
    INDEX (organization_id, status), (organization_id, updated_at)
stock_movements (id, organization_id, inventory_item_id, type, quantity,
                 reference_type, reference_id, idempotency_key, created_at)
    UNIQUE (organization_id, idempotency_key)
    INDEX (organization_id, inventory_item_id, created_at)
warehouses, inventory_item_warehouse (pivot with quantity)
audit_logs (append-only, partitioned monthly)
```

**Key decisions to articulate:**
- Row-level tenancy with global scopes failing closed; composite FKs preventing cross-tenant references
- Stock as an append-only ledger with a maintained `quantity` rollup — audit trail plus fast reads
- Atomic conditional UPDATE for deduction; idempotency key for client retries
- Redis cache namespaced per tenant, key-versioned invalidation
- Search via Elasticsearch with a mandatory tenant filter on every query
- Queue per workload class (`imports`, `notifications`, `indexing`) so a bulk import can't starve alerts
- Rate limits keyed by `organization_id` for fair-share
- Read replicas for reporting with `sticky = true`

**Bottleneck and evolution:** the shared `inventory_items` table under the largest tenants. First: composite indexes and per-tenant caching. Then: partition by `organization_id`. Then: dedicated database for outlier tenants as a premium tier.

### Design 2 — Rate limiter for a public API

**Clarify:** per-user, per-org, or per-IP? Fixed or sliding window? Distributed across N servers? What happens on Redis failure?

**Algorithms:**

| Algorithm | Pros | Cons |
|-----------|------|------|
| Fixed window counter | Trivial, cheap | Burst at window boundaries (2× limit) |
| Sliding window log | Exact | Memory grows with request count |
| Sliding window counter | Good approximation, cheap | Slight inaccuracy |
| Token bucket | Allows controlled bursts | Two values to track |
| Leaky bucket | Smooths output rate | Doesn't allow bursts |

**Implementation (token bucket in Redis Lua for atomicity):**

```lua
local key = KEYS[1]
local rate = tonumber(ARGV[1])       -- tokens per second
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(bucket[1]) or capacity
local ts = tonumber(bucket[2]) or now

tokens = math.min(capacity, tokens + (now - ts) * rate)

if tokens < requested then
  redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
  redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
  return 0
end

redis.call('HMSET', key, 'tokens', tokens - requested, 'ts', now)
redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
return 1
```

**Discuss:** fail-open vs fail-closed when Redis is down (fail open for availability, with an alarm, is usually right for a rate limiter); returning `X-RateLimit-*` and `Retry-After` headers; tiered limits per plan; separate limits for expensive endpoints.

### Design 3 — Distributed job scheduler

*(Your Chronos project — tie it in.)*

**Requirements:** cron-like schedules, exactly-once-ish execution across N nodes, survives node failure, observable, supports retries and backoff.

**Core problems:** leader election (or lock-based claiming), preventing duplicate execution, handling clock skew, recovering orphaned jobs.

**Two approaches:**
- **Raft-based leader election** (Chronos): one leader dispatches; followers stand by. Strong consistency, more complex, needs a quorum.
- **Database claiming with `SELECT ... FOR UPDATE SKIP LOCKED`**: any node claims due jobs atomically; no leader needed. Much simpler, scales to moderate throughput.

```sql
-- Lease-based claiming: any worker, no leader, safe under concurrency
UPDATE scheduled_jobs
SET locked_by = :worker_id, locked_until = now() + interval '5 minutes', attempts = attempts + 1
WHERE id IN (
    SELECT id FROM scheduled_jobs
    WHERE next_run_at <= now()
      AND (locked_until IS NULL OR locked_until < now())
    ORDER BY next_run_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

**Discuss:** leases with heartbeat renewal so a dead worker's jobs are reclaimed; idempotent handlers because exactly-once is impossible across a network; monotonic clocks and NTP for skew; jitter to avoid thundering herd when many jobs share a schedule; separating scheduling from execution (scheduler enqueues, workers execute). Mention that you built the Raft version to learn consensus but would ship the `SKIP LOCKED` version for most production needs — that's a mature trade-off statement.

### Design 4 — Real-time stock updates to 20K concurrent clients

**Options:** polling (simple, wasteful), long polling, SSE (one-way, HTTP/2-friendly, auto-reconnect), WebSockets (bidirectional, more infra).

**Given one-way server→client updates, SSE or WebSockets via Laravel Reverb/Pusher/Soketi.**

```php
// Private, tenant-authorized channel
Broadcast::channel('org.{orgId}.inventory', fn (User $user, int $orgId) =>
    (int) $user->organization_id === $orgId
);

class StockLevelChanged implements ShouldBroadcast
{
    public function broadcastOn(): PrivateChannel
    {
        return new PrivateChannel("org.{$this->organizationId}.inventory");
    }

    public function broadcastWith(): array
    {
        return ['item_id' => $this->itemId, 'quantity' => $this->newQuantity];
    }
}
```

**Discuss:** channel authorization is a tenant boundary (a public channel here is a data leak); batching/debouncing high-frequency updates so one bulk import doesn't emit 10,000 events; the fan-out cost; horizontal scaling of the WebSocket layer with a Redis adapter; graceful degradation to polling; and that broadcast payloads must not include fields the client isn't authorized to see.

### Design 5 — Bulk import of 1M product rows

**Flow:**
```
Upload → S3 (presigned URL, bypass the app server)
      → job: validate structure and headers, count rows
      → Bus::batch of chunk jobs (each handles 5,000 rows via upsert)
      → finally: reindex search, invalidate tenant cache, notify user
```

**Discuss:** streaming the CSV with `LazyCollection` (never `file_get_contents`); `upsert` with the tenant-scoped unique key so re-runs are idempotent; per-row validation errors collected into a downloadable report rather than failing the whole import; progress tracking via `Bus::findBatch`; throttling so the import doesn't saturate the DB or replicas; `upsert` skipping model events so cache invalidation and indexing must be explicit; resumability if the batch partially fails.

### Design 6 — Audit log for a compliance requirement

**Requirements:** immutable, queryable per tenant and per record, retained N years, must not slow the write path.

**Design:** append-only table partitioned by month; DB role lacking UPDATE/DELETE; write asynchronously via a queued listener (or the outbox) so it doesn't add latency to the transaction — but note that async means a crash can lose an entry, so for strict compliance write synchronously in the same transaction and accept the cost. Retention via partition drop. Archive old partitions to S3/Glacier. Index on `(organization_id, auditable_type, auditable_id, created_at)`.

**The trade-off to name explicitly:** synchronous audit = guaranteed completeness, added write latency. Asynchronous = fast, but a crash between commit and enqueue loses the record. Compliance usually forces synchronous within the same transaction.

---

## 9. STAR Stories from Your Experience

Prepare these as spoken narratives, 2–3 minutes each. Practice out loud. **Always end with a measurable result and what you'd do differently.**

### Story 1 — The 88% query reduction

**Situation.** The main inventory dashboard in our multi-tenant SaaS had a p95 latency around 2.4 seconds and was the single largest source of database load. During peak hours, connection saturation on Postgres degraded unrelated endpoints.

**Task.** Bring the endpoint into an acceptable latency budget without changing the API contract, and make sure the problem couldn't silently return.

**Action.** I started by measuring rather than guessing — enabled query logging against production-shaped data and found roughly 520 queries for a 50-row page. Profiling showed five distinct causes: a lazy-loaded supplier relation inside the API Resource, a per-row `movements()->count()`, a per-row "latest movement" lookup, permission checks hydrating roles per row, and an uncached aggregate block. I fixed each with the appropriate tool — eager loading with explicit column selection, `withCount`, a correlated subquery select for the latest movement, eager-loading the permission graph once, and a tenant-namespaced Redis cache for the aggregates with observer-driven invalidation. Then I added guardrails: `Model::preventLazyLoading()` outside production so lazy loads throw in dev and CI, a query-count assertion in the endpoint's feature test, slow-query logging tagged by organization, and a per-request query-budget warning.

**Result.** Query count went from ~520 to ~62, an 88% reduction, and p95 latency dropped from 2.4s to about 310ms. Database CPU at peak fell noticeably, and the guardrails have caught two attempted regressions since.

**Reflection.** The fix was straightforward; the lasting value was the guardrails. If I were starting over I'd add the query-count assertions and `preventLazyLoading` on day one rather than after an incident, because the cost of preventing an N+1 is far lower than diagnosing one in production.

### Story 2 — Zero-downtime 15M-record migration

**Situation.** We needed to add a status column and supporting index to a 15M-row inventory table in a SaaS with customers across time zones, so there was no maintenance window.

**Task.** Ship the schema change with no downtime and no customer-visible errors, with a rollback path at every stage.

**Action.** I used an expand-migrate-contract approach across four deploys. First, added the column nullable, which on our Postgres version is a metadata-only operation rather than a table rewrite — I verified that against the version before relying on it. The application dual-wrote so new and updated rows were correct immediately. Then I wrote a backfill command using keyset pagination with a persisted checkpoint so it was resumable, 2,000-row batches with a 100ms pause between them, and backpressure that paused entirely when read-replica lag exceeded five seconds. The index was built with `CREATE INDEX CONCURRENTLY` so writes continued. To enforce NOT NULL I added a `NOT VALID` check constraint, validated it concurrently, then set NOT NULL — turning a long exclusive lock into a brief one. I also set `lock_timeout` with retries so a DDL statement could never queue behind a long analytics query and block the whole table.

**Result.** Zero downtime, zero customer-visible errors, and p99 latency was unchanged throughout the backfill window. A reconciliation query proved zero unmigrated rows before we advanced each stage.

**Reflection.** The most important property wasn't any individual technique — it was that until the final drop, rollback was a code deploy rather than a data restore, because the old shape was still present and populated. Early on I underestimated replica lag and had to add the backpressure mid-run; I'd build that in from the start now.

### Story 3 — Race condition on the trading platform

**Situation.** On a platform with about 20,000 daily active users, we saw occasional oversells: two concurrent requests could both pass an availability check and both commit.

**Task.** Eliminate the race without materially hurting throughput on the hot path.

**Action.** I reproduced it deterministically with a test that ran two concurrent transactions against a real Postgres instance — reproducing it first mattered, because it let me prove the fix rather than hope. The root cause was a classic read-modify-write: select the quantity, check it in PHP, then write. I replaced it with a single atomic conditional UPDATE where the availability check lives in the WHERE clause, so the check and the write are one operation with no window between them. Separately, we were seeing duplicate submissions from clients retrying after timeouts, which locking doesn't address at all — so I added an `Idempotency-Key` header enforced by middleware, backed by a unique index on the tenant and key, and returned the original response on replay rather than doing a check-then-insert, which races too. I also moved an external pricing call out of the transaction, since it was holding row locks for the duration of a network round trip.

**Result.** Oversells went to zero. Because the atomic UPDATE doesn't hold a lock across the transaction the way `SELECT FOR UPDATE` would, throughput on that path improved rather than degraded.

**Reflection.** The lesson I carry is that locking and idempotency solve different problems — one is about concurrent server-side operations, the other about duplicate client requests — and I'd previously conflated them. I also now add a reconciliation job that recomputes balances from the movement ledger and alerts on drift, because monitoring for correctness beats assuming it.

### Story 4 — Architecting the multi-tenant SaaS

**Situation.** Greenfield inventory management platform intended to serve thousands of small-to-mid organizations.

**Task.** Choose a tenancy model and build isolation that would hold up as the team grew and people who hadn't been in the original design meetings started shipping features.

**Action.** I chose row-level tenancy on `organization_id` because we needed thousands of tenants at near-zero marginal cost, single-run migrations, and cross-tenant analytics. The obvious risk is that isolation becomes application-enforced, so a single missing WHERE clause is a breach affecting every customer — so I built defense in depth rather than relying on one mechanism. Global scopes that fail closed, returning nothing rather than everything when tenant context is missing. Automatic tenant stamping on create plus a `saving` guard that refuses to write a record belonging to another tenant. Composite unique indexes and composite foreign keys so cross-tenant references are structurally impossible at the database level, not just validated. Tenant-namespaced cache keys. A base job class that re-establishes both tenant context and the spatie permission team ID, so no job author could forget. And an isolation test suite per resource plus an architecture test asserting every model with an `organization_id` column uses the tenant trait.

**Result.** No cross-tenant incidents. New engineers could add features without understanding every isolation mechanism, because the defaults were safe and the tests failed loudly when they weren't.

**Reflection.** The decision I'd revisit is offering a database-per-tenant tier earlier. We eventually had enterprise prospects with contractual isolation requirements that row-level tenancy couldn't satisfy on paper regardless of how good our controls were, and retrofitting that is far harder than designing for it.

### Story 5 — A production incident (adapt to a real one)

**Structure to follow:** what broke, how you detected it, how you contained it before fixing root cause, the root cause, the permanent fix, and the systemic change so the class of bug can't recur. Emphasize the containment-before-diagnosis instinct and the blameless follow-up. Interviewers weight the *process* far above the specific bug.

### Story 6 — Disagreement with a colleague

**Structure:** the technical disagreement, how you sought to understand their reasoning, what data or prototype you used to move from opinion to evidence, the outcome — including a case where you changed your own mind. Having a story where you were wrong and updated is more persuasive than one where you were right.

---

## 10. Questions to Ask the Interviewer

Asking good questions is part of the evaluation. These signal seniority.

**Engineering practice**
- How do you decide what gets built versus bought?
- What does your deployment pipeline look like, and how often do you deploy?
- How do you handle database migrations on large tables in production?
- What's your test strategy — where do you invest, and what do you deliberately not test?
- How is on-call structured, and what does a typical incident review look like?

**Architecture**
- Where is the system feeling strain right now?
- What's the biggest piece of technical debt, and what's blocking paying it down?
- How do services communicate — synchronous, events, or both — and how did you decide?
- If you could rebuild one part of the system, what would it be?

**Team and role**
- What would you want the person in this role to have accomplished in six months?
- How do senior engineers here influence technical direction?
- How are decisions documented — RFCs, ADRs, something else?
- What's the balance between feature work and platform/infrastructure work?

**Signals to listen for:** hesitancy about deploy frequency, no incident review process, "we don't really have time for tests," or an inability to name current architectural pain. Those tell you as much as the answers.

---

## 11. Red Flags to Avoid

Things that quietly cost you points in a senior loop:

| Red flag | What to do instead |
|----------|-------------------|
| "Laravel does that for you" as a complete answer | Explain *how* it does it, and when it doesn't |
| Claiming a technology solved everything | Name the trade-off you accepted |
| No numbers in your stories | Have baseline and after metrics ready |
| Describing team work as entirely your own | Be precise about your contribution; it's more credible, not less |
| "We use microservices because they scale" | Explain the specific coupling problem it solved and what it cost |
| Recommending the newest thing reflexively | Justify against boring, proven alternatives |
| Mocking everything in tests | Explain why you mock at boundaries and use a real DB otherwise |
| Never having been wrong | Have one genuine "I was wrong and changed course" story |
| Only optimistic answers about your architecture | Name the thing you'd redo — it reads as self-awareness |
| Reciting SOLID/patterns as definitions | Give a concrete instance from your own code and why it earned its place |
| Guessing when you don't know | "I haven't worked with that directly; here's how I'd approach finding out" |

**On the last one:** senior interviewers deliberately probe past the edge of your knowledge to see how you behave there. Confidently bluffing is far worse than reasoning out loud from first principles and being explicit about what you'd need to verify.

---

## Daily Drill Plan

| Day | Activity | Time |
|-----|----------|------|
| Mon | Rapid-fire PHP (1–60) out loud | 30 min |
| Tue | Rapid-fire Laravel (61–120) out loud | 30 min |
| Wed | Rapid-fire DB/Perf + Architecture (121–200) | 40 min |
| Thu | Code puzzles + one live-coding exercise, typed from scratch | 45 min |
| Fri | One system design prompt, whiteboarded and spoken | 45 min |
| Sat | Two STAR stories recorded and played back | 30 min |
| Sun | Re-read whichever tier file exposed the most gaps | 60 min |

**Record yourself for the STAR stories.** Playback exposes filler words, missing numbers, and rambling far better than self-assessment does.

---

**Back to:** [`README.md`](./README.md) · [`01-basic.md`](./01-basic.md) · [`02-oop-php.md`](./02-oop-php.md) · [`03-intermediate.md`](./03-intermediate.md) · [`04-senior.md`](./04-senior.md)
