# Go (Golang) / Gin — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies
> **Your anchors:** Chronos (Go distributed job scheduler, Raft leader election) · multi-tenant inventory SaaS · 88% query reduction · zero-downtime 15M+ record migration · 20K+ DAU trading platform
> **Written against Go 1.26** (current as of July 2026). Version-specific behaviour is flagged inline, because several of the most-asked interview questions changed answers in 1.22.

---

## How to use this material

Same method as the PHP/Laravel set: active recall, not passive reading.

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as if to an interviewer | 20 min/section |
| 2 | Type the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**Every Go example in these files has been compiled and vetted against Go 1.26.** Where a snippet is deliberately broken to demonstrate a bug, it is marked as such. If you find something that doesn't compile, it's a bug in the notes — fix it, because muscle memory from wrong code is worse than no notes.

---

## Why Go interviews are different from PHP interviews

This is worth internalising before you start, because it changes what you rehearse.

A Laravel interview mostly probes **architecture and data modelling** — you're asked how you'd structure a system, and the language is assumed. A Go interview probes the **language and runtime directly**. You will be asked what a slice is made of, whether a nil interface equals nil, what happens when two goroutines write to a map, and how the scheduler handles a blocking syscall. These aren't trivia; Go's whole value proposition is predictable concurrency and performance, so interviewers test whether you understand the machinery you're claiming to use.

The second difference: **concurrency is not an advanced topic in Go, it's a baseline one.** In PHP, concurrency questions are senior-tier. In Go, "write a worker pool" is a warm-up. That's why concurrency is Tier 2 here rather than buried in the senior file — it's the equivalent of the OOP tier in the PHP set, the make-or-break foundation everything else builds on.

The third: **your Chronos project is the single most valuable thing on your CV for these interviews.** A distributed job scheduler with Raft-based leader election touches consensus, leader election, distributed locking, idempotency, failure detection, and graceful shutdown — the exact topics a senior Go interview converges on. Several sections below are structured to feed that story.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Execution & build model, packages/modules, type system, untyped constants, strings/bytes/runes, **slice and map internals**, structs & embedding, pointers vs values, **method sets**, functions & closures, `defer`/`panic`/`recover`, interfaces & the **nil interface trap**, errors & wrapping, generics, standard library tour, tooling | 8–10 hours |
| [`02-concurrency.md`](./02-concurrency.md) | **The make-or-break tier.** Goroutines & the cost model, channels (buffered/unbuffered/nil/closed), `select`, `sync` primitives, `atomic`, `context` propagation, the **Go memory model** and happens-before, the race detector, worker pools, fan-in/fan-out, pipelines, `errgroup`, `singleflight`, semaphores, rate limiting, graceful shutdown, **goroutine leaks and deadlocks**, testing concurrent code with `synctest` | 12–15 hours |
| [`03-gin-web.md`](./03-gin-web.md) | `net/http` foundations, `http.Handler` and the server lifecycle, Gin's radix-tree router, `gin.Context` and the **copy-for-goroutine trap**, middleware pipeline, binding & validation, error-handling strategy, `database/sql` pool tuning, transactions with context, `pgx`/`sqlx`, migrations, Redis, gRPC, and a full testing strategy (`httptest`, table-driven, mocks, testcontainers) | 10–12 hours |
| [`04-senior.md`](./04-senior.md) | **GMP scheduler**, preemption, netpoller, stack growth, escape analysis, the **garbage collector** (tricolor, write barrier, `GOGC`, `GOMEMLIMIT`, Green Tea in 1.26), profiling with `pprof` and `runtime/trace`, benchmarking correctly, allocation reduction, **Raft and Chronos deep dive**, leader election, distributed locks, idempotency and exactly-once, resilience patterns, `log/slog` and OpenTelemetry, deployment and graceful shutdown | 14–18 hours |
| [`05-question-bank.md`](./05-question-bank.md) | 200+ questions with answers by tier · "what does this print" puzzles · live-coding exercises · debugging scenarios · system design prompts · STAR stories built from Chronos and your real work | Ongoing drill |

---

## Coverage map

Mark your own confidence 1–5 in a copy of this table. Anything below 4 goes back into the drill rotation.

### Language core
- Compilation and linking model, cross-compilation, build tags
- Modules, `go.mod`/`go.sum`, MVS version selection, vendoring, workspaces
- Package design, exported vs unexported, `init()` order, import cycles
- Type declarations vs aliases, zero values, no implicit conversion
- Untyped constants and `iota`
- Strings, bytes, runes, UTF-8, `strings.Builder`
- **Arrays vs slices**: header layout, `len`/`cap`, `append` growth and aliasing, three-index slices, `copy`, nil vs empty
- **Maps**: hashing, iteration randomisation, non-addressability, nil map reads vs writes, concurrent write panic, Swiss Tables (1.24+)
- Structs: embedding, promotion, tags, comparability, `struct{}`
- Pointers vs values, **method sets and addressability**
- Functions: multiple returns, named returns, variadics, closures, **the 1.22 loop variable change**
- `defer` evaluation order and argument capture, `panic`/`recover`
- **Interfaces**: implicit satisfaction, `itab` layout, type assertions and switches, **nil interface vs nil pointer**, interface pollution
- Errors: sentinels, wrapping with `%w`, `errors.Is`/`As`/`Join`, custom types
- Generics: type parameters, constraints, inference, and when they hurt
- Tooling: `go vet`, `gofmt`, `staticcheck`, `go test -race`, `go mod tidy`

### Concurrency
- Goroutine cost, stack growth, lifecycle, and why "cheap" is not "free"
- Channel semantics for every state: nil, empty, full, closed
- `select`: random selection, `default`, timeouts, disabling cases with nil
- `sync.Mutex`, `RWMutex` (and when it's slower), `WaitGroup` (+ `Go` in 1.25), `Once`, `Pool`, `sync.Map`
- `sync/atomic` typed values, compare-and-swap
- `context`: cancellation trees, deadlines, `context.AfterFunc`, values (and why they're a smell)
- **The Go memory model**: happens-before, why "it works on my machine" is meaningless
- Race detector: what it catches, what it cannot
- Patterns: worker pool, fan-in/fan-out, pipeline with cancellation, bounded parallelism, `errgroup`, `singleflight`, token bucket
- **Goroutine leaks**: the four causes, detection with pprof and `goleak`
- Deadlock diagnosis and the all-goroutines-asleep panic
- Testing concurrency: `-race`, `testing/synctest` (stable in 1.25)

### Gin & web services
- `net/http`: `Handler`, `ServeMux`, the 1.22 routing patterns, `Server` timeouts
- Gin engine, radix-tree routing, route conflicts, parameter binding
- `gin.Context`: request scope, `Copy()`, storing values, the goroutine trap
- Middleware: ordering, `Next`/`Abort`, recovery, structured request logging, auth, CORS, rate limiting, request IDs
- Binding: JSON/query/URI/form, `validator` tags, custom validators, error surfacing
- Consistent error envelopes and HTTP status mapping
- `database/sql`: pool sizing, `SetMaxOpenConns`/`SetMaxIdleConns`/`SetConnMaxLifetime`, statement caching, `QueryContext`
- Transactions: context-aware, rollback discipline, isolation levels
- `pgx` vs `database/sql` vs `sqlx` vs GORM — trade-offs
- gRPC: protobuf, streaming, interceptors, deadlines, error codes
- Testing: `httptest`, table-driven tests, interface fakes, `testcontainers`, golden files

### Runtime, performance & distributed systems
- **GMP scheduler**: work stealing, `GOMAXPROCS`, spinning threads, handoff
- Asynchronous preemption (1.14+) and why tight loops used to hang
- Netpoller and how blocking I/O doesn't block threads
- Goroutine stacks: growth, copying, and why pointers stay valid
- **Escape analysis**: reading `-gcflags='-m'`, common escape causes
- **GC**: tricolor mark-and-sweep, write barriers, assists, `GOGC` vs `GOMEMLIMIT`, Green Tea GC (default in 1.26)
- `pprof` (CPU, heap, block, mutex, goroutine), `runtime/trace`, flame graphs
- Benchmarking: `testing.B`, `b.Loop` (1.24+), avoiding dead-code elimination, `benchstat`
- **Raft**: leader election, log replication, safety, and exactly how Chronos uses it
- Distributed locks, fencing tokens, and why Redlock is contested
- Idempotency, at-least-once vs exactly-once, the outbox pattern
- Resilience: timeouts, retries with jitter, circuit breakers, bulkheads
- Observability: `log/slog`, OpenTelemetry tracing, Prometheus metrics
- Deployment: multi-stage Docker, distroless, `GOMAXPROCS` under cgroups, graceful shutdown, health checks

---

## Interview readiness checklist

Tick these only when you can do them **out loud, unprompted, in under 3 minutes each**.

**Language**
- [ ] Draw the slice header and explain why `append` inside a function sometimes mutates the caller's data and sometimes doesn't
- [ ] Explain why a nil `*MyError` stored in an `error` variable is not `nil`, and how to avoid it
- [ ] Explain method sets: why `T` doesn't satisfy an interface that `*T` does, and what addressability has to do with it
- [ ] Explain what changed about loop variables in Go 1.22 and what bug it fixed
- [ ] Explain `defer` argument evaluation, LIFO order, and interaction with named returns
- [ ] Say when generics make code worse

**Concurrency**
- [ ] Describe every channel operation's behaviour on a nil, closed, and full channel
- [ ] Write a bounded worker pool with context cancellation and error propagation, from scratch
- [ ] Name the four ways goroutines leak and how you'd detect each in production
- [ ] Explain happens-before and why the race detector finding nothing doesn't prove correctness
- [ ] Explain when `RWMutex` is slower than `Mutex`
- [ ] Explain why `context.Value` is discouraged for anything but request-scoped metadata

**Web / Gin**
- [ ] Explain why you must `c.Copy()` before using a `gin.Context` in a goroutine
- [ ] Size a `database/sql` connection pool and justify every number
- [ ] Explain the four `http.Server` timeouts and what each one protects against
- [ ] Design a consistent API error envelope and map domain errors to status codes

**Runtime & distributed**
- [ ] Explain the GMP model including what happens on a blocking syscall
- [ ] Explain escape analysis and give three things that force a heap allocation
- [ ] Explain `GOGC` vs `GOMEMLIMIT` and when you'd set each
- [ ] Walk through diagnosing a memory leak in production with pprof
- [ ] Explain Raft leader election and how Chronos avoids two schedulers firing the same job
- [ ] Explain why "exactly-once delivery" is impossible and what you built instead

---

## Study order recommendation

```
Week 1:  01-basic.md         + Basic Q&A drill
Week 2:  02-concurrency.md   (first half: goroutines, channels, select, sync)
Week 3:  02-concurrency.md   (second half: context, memory model, patterns, leaks)
Week 4:  03-gin-web.md       + Gin/HTTP Q&A drill
Week 5:  04-senior.md        (first half: runtime, GC, performance)
Week 6:  04-senior.md        (second half: Raft/Chronos, resilience, production)
Week 7+: 05-question-bank.md daily drill + STAR story rehearsal
```

Do **not** skip Tier 1 because you've written Go professionally. The slice-aliasing, nil-interface, and method-set questions are asked constantly, they have precise answers, and "I've never hit that in practice" reads as a gap rather than as experience.

---

**Previous topic:** [`../php-laravel/`](../php-laravel/) — completed
**Next topic in skill order:** to be set — see `.cursorrules`
