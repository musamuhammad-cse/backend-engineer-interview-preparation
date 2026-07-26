# PHP / Laravel — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS (Laravel, Passport, spatie/permission teams mode, PostgreSQL, Redis, Pest) · 88% query reduction · zero-downtime 15M+ record migration · 20K+ DAU trading platform · Chronos (Go distributed scheduler, Raft)

---

## How to use this material

This is not a skim-read. Each tier builds on the previous one and is designed for **active recall**, not passive reading.

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as if to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**The senior signal is not knowing definitions.** It's knowing trade-offs, failure modes, and what you'd do at 3am when it breaks. Every section below flags **Traps** (what interviewers use to catch you) and **Follow-ups** (the second and third question they will ask).

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | PHP language & engine fundamentals, types, OOP, closures, generators, exceptions, autoloading, PHP 8.x features · Laravel request lifecycle, service providers, facades, routing, controllers, validation, Eloquent basics, migrations, Artisan, baseline security | 6–8 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Service container internals & contextual binding, middleware pipeline, all Eloquent relationships, advanced queries & serialization, collections, transactions, queues/jobs/batching, events & observers, caching & locks, API design, auth guards, full testing strategy with Pest | 10–12 hours |
| [`03-senior.md`](./03-senior.md) | Multi-tenancy architecture & leak vectors, spatie teams internals, Passport/OAuth2 deep dive, concurrency & race conditions, isolation levels, performance engineering (your 88% story), zero-downtime migrations (your 15M story), Clean Architecture/DDD/CQRS, scaling & deployment, observability, OWASP in Laravel, Octane & PHP internals | 14–18 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 200+ interview questions with answers, grouped by tier and theme · System design prompts · STAR stories built from your real experience · Live-coding exercises | Ongoing drill |

---

## Coverage map

Use this to find gaps fast. Mark your own confidence 1–5 in a copy of this table.

### PHP language
- Execution model (SAPI, php-fpm, Zend VM, OPcache, JIT)
- Type system: scalar hints, union, intersection, nullable, `never`, `void`, `static`, `mixed`
- `declare(strict_types=1)` and coercion rules
- Value vs reference semantics, copy-on-write, refcounting, GC cycles
- Arrays as ordered hash maps, packed vs hashed
- Closures, `use` by value/reference, arrow functions, first-class callables, `Closure::bind`
- OOP: visibility, `static`, abstract, interfaces, traits + conflict resolution, enums, `readonly`, constructor promotion, late static binding, magic methods, `__clone` deep copy
- Exceptions: `Throwable`/`Error`/`Exception`, chaining, `finally` semantics
- Generators, `yield from`, memory characteristics
- SPL interfaces: `ArrayAccess`, `IteratorAggregate`, `Countable`, `JsonSerializable`, `Stringable`
- PSR standards (PSR-4, PSR-7, PSR-11, PSR-12, PSR-15)
- PHP 8.0 → 8.4 feature timeline

### Laravel framework
- Full request lifecycle end to end
- Service container: autowiring, contextual binding, tagging, extending, scoped bindings
- Service providers: `register` vs `boot`, deferred providers
- Facades: `__callStatic`, real-time facades, faking in tests
- Middleware pipeline internals, priority, terminable middleware, L11 `bootstrap/app.php`
- Routing: binding (implicit/explicit/scoped), caching, signed URLs, rate limiting
- Validation: Form Requests, custom rules, conditional rules, `Rule` objects
- Eloquent: all relationship types, casts, accessors/mutators, events/observers, scopes, serialization
- Query performance: eager loading, subquery selects, aggregates, `chunkById`, `cursor`, `lazyById`
- Collections & lazy collections
- Queues: workers, retries, backoff, unique jobs, batches, chains, Horizon
- Events, listeners, subscribers, `ShouldDispatchAfterCommit`
- Cache: drivers, tags, atomic locks, stampede protection
- Auth: guards, providers, Sanctum vs Passport, policies, gates, spatie/permission teams
- Testing: Pest, factories, fakes, mocks, time control, architecture tests
- Octane, Horizon, Telescope, Pulse
- Deployment, config/route caching, graceful worker shutdown

### Systems & architecture (Laravel-flavored)
- Row-level multi-tenancy: implementation, isolation testing, leak vectors
- Concurrency: pessimistic/optimistic locking, atomic SQL, advisory locks, idempotency
- Transaction isolation levels and anomalies
- Zero-downtime schema evolution (expand → migrate → contract)
- PostgreSQL DDL locking, `CREATE INDEX CONCURRENTLY`, `NOT VALID` constraints
- Indexing strategy, `EXPLAIN ANALYZE`, keyset pagination
- Read replicas, connection pooling (PgBouncer), sticky reads
- Clean Architecture, DDD, CQRS, transactional outbox, saga/compensation
- SOLID with real Laravel code
- Observability: structured logs, correlation IDs, OpenTelemetry, Prometheus
- OWASP Top 10 mapped to Laravel controls

---

## Interview readiness checklist

Tick these only when you can do them **out loud, unprompted, in under 3 minutes each**.

- [ ] Walk the Laravel request lifecycle from `index.php` to response, naming real classes
- [ ] Explain how the service container resolves a constructor dependency it has never seen
- [ ] Explain how a Facade turns a static call into a container resolution
- [ ] Describe the middleware pipeline implementation (why it's an onion, not a list)
- [ ] Diagnose an N+1 and give three different fixes with trade-offs
- [ ] Design row-level multi-tenancy and name six ways it leaks
- [ ] Solve "two users buy the last item" four different ways and pick one with justification
- [ ] Compare READ COMMITTED vs REPEATABLE READ vs SERIALIZABLE with a concrete anomaly
- [ ] Give the full expand → migrate → contract playbook for adding a NOT NULL column to 15M rows
- [ ] Tell the 88% query reduction story with measurement, hypothesis, fix, verification, numbers
- [ ] Explain when you would *not* use the repository pattern
- [ ] Explain why Octane breaks code that works under php-fpm
- [ ] Explain idempotency and design an idempotent order-submission endpoint
- [ ] Explain how you'd add OpenTelemetry tracing across a Laravel API and its queue workers

---

## Study order recommendation

Even with 8+ years of experience, run the tiers in order. The Basic tier is where interviewers plant "simple" questions that expose shallow foundations (`==` vs `===` type juggling rules, copy-on-write, `self` vs `static`). Skipping it is the most common senior-candidate mistake.

```
Week 1:  01-basic.md        + Basic Q&A drill
Week 2:  02-intermediate.md + Intermediate Q&A drill
Week 3:  03-senior.md (first half: multi-tenancy, concurrency, performance)
Week 4:  03-senior.md (second half: architecture, scaling, security, internals)
Week 5+: 04-question-bank.md daily drill + STAR story rehearsal
```

**Next topic in skill order:** Go (Golang) — after PHP/Laravel is solid.
