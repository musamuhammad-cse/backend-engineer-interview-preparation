# Go — Tier 2: Concurrency

> **This is the make-or-break tier.** In a PHP interview, concurrency is a senior topic. In a Go interview it's the baseline — "write a worker pool" is a warm-up question, and every system design answer you give will be probed for race conditions, cancellation, and leak behaviour.
>
> All examples compiled and verified against **Go 1.26**. Where a snippet is deliberately broken, it's labelled.

---

## Table of Contents

1. [Goroutines](#1-goroutines)
2. [Channels](#2-channels)
3. [select](#3-select)
4. [The sync Package](#4-the-sync-package)
5. [sync/atomic](#5-syncatomic)
6. [context](#6-context)
7. [The Go Memory Model](#7-the-go-memory-model)
8. [The Race Detector](#8-the-race-detector)
9. [Concurrency Patterns](#9-concurrency-patterns)
10. [Goroutine Leaks](#10-goroutine-leaks)
11. [Deadlocks & Livelocks](#11-deadlocks--livelocks)
12. [Testing Concurrent Code](#12-testing-concurrent-code)
13. [Anti-Patterns](#13-anti-patterns)
14. [Tier 2 Q&A Drill](#14-tier-2-qa-drill)

---

## 1. Goroutines

```go
go doWork()                     // spawn; returns immediately
go func() { doWork() }()        // spawn a closure
```

### The cost model

A goroutine starts with a **2 KB stack** that grows and shrinks on demand (up to 1 GB default on 64-bit). An OS thread typically reserves 1–8 MB. That's the ~1000× difference that makes "spawn one per request" viable in Go and insane in a thread-per-request model.

| | OS thread | Goroutine |
|---|---|---|
| Initial stack | 1–8 MB (reserved) | 2 KB (grows) |
| Creation cost | ~microseconds, kernel call | ~nanoseconds, user space |
| Context switch | Kernel, ~1–2 µs | Runtime, ~100–200 ns |
| Scheduled by | OS kernel | Go runtime (GMP — see Tier 4) |
| Practical count | thousands | millions |

> **Trap — "goroutines are cheap" is the setup for a follow-up, not the answer.** Cheap is not free. Each one costs at minimum 2 KB plus a runtime descriptor, and more importantly it usually holds *other* resources: a database connection, an open socket, a buffer. Spawning one per item over a million-row result set will exhaust your connection pool long before it exhausts memory. The senior answer is: *"goroutines are cheap enough that the limiting factor is whatever they hold, not the goroutines themselves — so I bound concurrency based on the scarce resource, usually the DB pool or the downstream service's rate limit."*

### They are not free-running

```go
// BUG — main exits immediately; the goroutine may never run
func main() {
    go fmt.Println("hello")
}
```

When `main` returns the process exits, killing every goroutine without running their defers. You need explicit synchronisation — a `WaitGroup`, a channel, or `errgroup`.

There is also **no goroutine ID, no way to kill a goroutine from outside, and no priority.** A goroutine stops when its function returns. Everything else — cancellation, timeouts, shutdown — is cooperative, which is exactly why `context` exists.

> **Follow-up you should expect:** *"How do you kill a goroutine?"* You don't. You signal it via a channel or a cancelled context and it returns on its own. If it's blocked on something that doesn't select on cancellation (a blocking syscall, a channel nobody sends to), it leaks — permanently. That constraint drives most of Go's concurrency design.

---

## 2. Channels

```go
unbuffered := make(chan int)        // synchronous rendezvous
buffered   := make(chan int, 10)    // asynchronous up to 10 elements

ch <- v          // send
v := <-ch        // receive
v, ok := <-ch    // ok is false if the channel is closed AND drained
close(ch)
```

### Behaviour in every state — memorise this table

This is the single most-asked channel question, and it's worth being able to recite.

| Operation | nil channel | empty (open) | full (open) | closed |
|---|---|---|---|---|
| **send** | blocks forever | proceeds/blocks until received | blocks | **panic** |
| **receive** | blocks forever | blocks | proceeds | returns zero value immediately, `ok == false` |
| **close** | **panic** | ok | ok | **panic** |

Verified output:

```go
ch := make(chan int, 2)
ch <- 1
close(ch)
v1, ok1 := <-ch     // 1, true   — buffered values are still drained
v2, ok2 := <-ch     // 0, false  — then zero value forever
```

```
1 closed chan: 1 true | 0 false
8 recovered: send on closed channel
9 recovered: close of closed channel
```

**Closing does not discard buffered values** — receivers drain the buffer first, then get zero values. This is why `range` over a closed channel finishes the remaining items before terminating.

### Who closes?

> **The rule: the sender closes, never the receiver.** A receiver closing means a sender can panic. With multiple senders, no single one can safely close — you need either a coordinating goroutine that closes after a `WaitGroup.Wait()`, or a separate `done` channel that senders select on.

```go
// Correct pattern for fan-in with N senders
var wg sync.WaitGroup
out := make(chan Result)

for _, src := range sources {
    wg.Go(func() {                 // Go 1.25+: wg.Add(1) + go func(){ defer wg.Done() ... }
        out <- fetch(src)
    })
}

go func() {
    wg.Wait()
    close(out)      // exactly one closer, after all senders are done
}()

for r := range out {
    process(r)
}
```

`sync.WaitGroup.Go` was added in **Go 1.25** and replaces the `Add(1)` / `defer Done()` boilerplate. Verified working. Mentioning it signals you track the language.

### Closed channels as broadcast

A closed channel makes every receiver return immediately, which makes it the standard broadcast primitive:

```go
done := make(chan struct{})
// ... later
close(done)          // ALL goroutines selecting on <-done wake up at once
```

Sending a value would only wake one receiver. Closing wakes all of them. This is why shutdown signals are always "close a channel," never "send a value."

### Buffered vs unbuffered

**Unbuffered is a synchronisation point**, not just a queue of size zero. The send completes only when a receiver takes the value, so both goroutines are at a known point in time — it's a rendezvous and it gives you a happens-before edge in both directions.

**Buffered decouples** sender and receiver up to the buffer size. Use it when:
- You know the exact count (`make(chan Result, len(jobs))`) — prevents senders blocking
- You're implementing a semaphore (see §9)
- You've *measured* that handoff overhead matters

> **Trap — buffer size is not a performance knob.** A common mistake is "add a buffer to make it faster." A buffer hides backpressure: the producer runs ahead, memory grows, and when the buffer fills you're back to blocking but now with a queue of stale work. Unbuffered gives you backpressure for free. Choose a buffer for a *semantic* reason, not a speed hope.

### Direction types

```go
func produce(out chan<- int)    // send-only  — compiler enforces
func consume(in  <-chan int)    // receive-only
```

Always narrow the direction in function signatures. It documents intent and makes "receiver accidentally closes the channel" a compile error.

---

## 3. select

```go
select {
case v := <-ch1:
    use(v)
case ch2 <- x:
    // sent
case <-ctx.Done():
    return ctx.Err()
case <-time.After(time.Second):
    return ErrTimeout
default:
    // runs immediately if NO other case is ready
}
```

### Three rules

1. **If multiple cases are ready, one is chosen uniformly at random.** Not in source order — the randomisation prevents starvation. This is asked often.
2. **`default` makes the select non-blocking.** Without it, `select` blocks until a case is ready.
3. **A `select` with no ready cases and no `default` blocks;** with zero cases (`select {}`) it blocks forever, which is a deadlock unless other goroutines are running.

### nil channels disable cases — the underused trick

Since receiving from a nil channel blocks forever, setting a channel variable to nil **removes that case from the select**:

```go
// Merge two streams, handling each finishing independently
func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
            case v, ok := <-a:
                if !ok {
                    a = nil          // disable this case; don't spin on a closed channel
                    continue
                }
                out <- v
            case v, ok := <-b:
                if !ok {
                    b = nil
                    continue
                }
                out <- v
            }
        }
    }()
    return out
}
```

Without setting `a = nil`, a closed channel is *always ready* (returning zero values), so the select would spin at 100% CPU. Verified: nil cases are correctly skipped.

> **This is a great answer to "have you hit a busy-loop bug in Go?"** — a closed channel in a select that you keep receiving from is the classic one.

### Timeouts

```go
select {
case res := <-resultCh:
    return res, nil
case <-ctx.Done():
    return nil, ctx.Err()          // PREFER this over time.After
}
```

Prefer `ctx.Done()` to `time.After` because the deadline then propagates to everything downstream. A `time.After` timeout abandons the operation locally while the work keeps running — you've timed out the *caller*, not the *work*.

---

## 4. The sync Package

### Mutex

```go
type Scheduler struct {
    mu   sync.Mutex
    jobs map[string]*Job
}

func (s *Scheduler) Add(j *Job) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.jobs[j.ID] = j
}
```

Rules that matter:
- **`sync.Mutex` is not reentrant.** Locking twice in the same goroutine deadlocks. There is no recursive mutex, deliberately.
- **Never copy a locked mutex.** Put the mutex in the struct and use pointer receivers; `go vet`'s `copylocks` catches violations.
- **Keep the critical section small.** Never do I/O — an HTTP call or DB query — while holding a lock.

```go
// BAD — holds the lock across a network call; every other goroutine queues behind it
func (s *Scheduler) Sync() {
    s.mu.Lock()
    defer s.mu.Unlock()
    resp, _ := http.Get(url)      // could take 30 seconds
    s.jobs = parse(resp)
}

// GOOD — do the slow work outside, hold the lock only for the swap
func (s *Scheduler) Sync() error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    fresh := parse(resp)

    s.mu.Lock()
    s.jobs = fresh
    s.mu.Unlock()
    return nil
}
```

### RWMutex — and when it's slower

```go
func (s *Scheduler) Get(id string) (*Job, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    j, ok := s.jobs[id]
    return j, ok
}
```

Many concurrent readers, or one writer with exclusive access.

> **Trap — `RWMutex` is not a free upgrade over `Mutex`.** It has a larger struct, more bookkeeping, and higher uncontended cost. For short critical sections (a map lookup taking nanoseconds), the extra overhead exceeds the parallelism gained, and a plain `Mutex` measures faster. `RWMutex` wins when reads are *long* and vastly outnumber writes. "I'd start with `Mutex` and only move to `RWMutex` if profiling showed read contention" is the right answer.

> **Trap — writer starvation.** Go's `RWMutex` blocks new readers once a writer is waiting, specifically to prevent starvation. Nice property, but it means a long read can stall every subsequent reader behind one waiting writer.

### WaitGroup

```go
var wg sync.WaitGroup

// Go 1.25+ — preferred
for _, job := range jobs {
    wg.Go(func() { process(job) })
}
wg.Wait()

// Classic form
for _, job := range jobs {
    wg.Add(1)
    go func() {
        defer wg.Done()      // ALWAYS deferred — a panic must still decrement
        process(job)
    }()
}
wg.Wait()
```

> **Trap — `Add` must be called before the goroutine starts**, not inside it. Calling `wg.Add(1)` inside the goroutine races with `Wait`, which may see a zero counter and return early. `wg.Go` eliminates this class of bug entirely, which is why it was added.

> **Trap — `WaitGroup` gives you no error path.** If any goroutine fails, `Wait` doesn't know. Use `errgroup` (§9) when work can fail.

### Once

```go
type Client struct {
    once sync.Once
    conn *grpc.ClientConn
}

func (c *Client) Conn() *grpc.ClientConn {
    c.once.Do(func() { c.conn = dial() })
    return c.conn
}
```

`Do` blocks all callers until the first completes, so no one sees a half-initialised value.

> **Trap — `Once` does not retry on failure.** If `dial()` fails and you store nil, every future caller gets nil forever. For fallible initialisation you need your own retry-capable primitive, or `sync.OnceValue`/`OnceValues` (Go 1.21+) combined with explicit error handling:
>
> ```go
> var getConn = sync.OnceValues(func() (*grpc.ClientConn, error) { return dial() })
> ```
>
> This still caches the error permanently — which is often correct for config parsing and wrong for network dialling. Knowing the distinction is the follow-up.

### Pool

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func handle(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()                 // ALWAYS reset — you get a dirty object
    defer bufPool.Put(buf)
    buf.Write(data)
}
```

`sync.Pool` reduces GC pressure for high-churn temporary objects.

> **Traps, all of which get asked:**
> - **Items are removed at every GC.** It's a cache with no guarantees, not a resource pool. Never use it for connections.
> - **You must reset the object.** `Get` returns whatever was put back, dirty.
> - **Don't pool objects with wildly varying sizes** — a pool of `[]byte` where one request needs 10 MB keeps that 10 MB alive for every subsequent small request. Cap the size on `Put`.
> - **Only use it when profiling shows allocation pressure.** It's a measurable-win optimisation, not a default.

### sync.Map

Covered in Tier 1 §7. Recap: only for write-once-read-many or disjoint key sets. Otherwise `map` + `RWMutex` is faster and type-safe.

---

## 5. sync/atomic

```go
var requests atomic.Int64        // typed — preferred over atomic.AddInt64(&n, 1)

requests.Add(1)
n := requests.Load()
requests.Store(0)

ok := requests.CompareAndSwap(5, 10)    // CAS: set to 10 only if currently 5
old := requests.Swap(0)
```

The typed forms (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`, `atomic.Value`) were added in 1.19 and are strictly better: they can't be misaligned, and you can't accidentally do a non-atomic read of the same variable.

### Atomic vs mutex

Atomics are for **a single word**. The moment you need two fields consistent with each other, you need a mutex:

```go
// BROKEN — each operation is atomic, but the PAIR is not
var count atomic.Int64
var total atomic.Int64

count.Add(1)
total.Add(price)      // another goroutine can observe count updated but total not

// A reader computing total/count sees an inconsistent average.
```

> **The rule:** atomics protect a *variable*; mutexes protect an *invariant*.

### Copy-on-write with atomic.Pointer

A genuinely useful pattern for read-heavy configuration — reads become completely lock-free:

```go
type Config struct{ Timeout time.Duration }

var cfg atomic.Pointer[Config]

func Get() *Config { return cfg.Load() }       // no lock at all

func Reload(c *Config) { cfg.Store(c) }        // swap the whole thing atomically
```

Readers never block, and writers replace the pointer wholesale. This is how you'd hold hot-reloadable config or a routing table in a high-throughput service. The constraint is that the pointed-to value must be treated as **immutable** after publishing.

---

## 6. context

`context` is how Go does cancellation, deadlines, and request-scoped values. It's in essentially every function signature in a real Go service.

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}          // closed when cancelled
    Err() error                     // Canceled or DeadlineExceeded
    Value(key any) any
}
```

### Creating contexts

```go
ctx := context.Background()                              // root, never cancelled
ctx := context.TODO()                                    // placeholder; you haven't decided yet

ctx, cancel := context.WithCancel(parent)
defer cancel()                                            // ALWAYS — even on the success path

ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

ctx, cancel := context.WithDeadline(parent, t)
defer cancel()

ctx := context.WithValue(parent, requestIDKey{}, id)

ctx, cancel := context.WithCancelCause(parent)           // 1.20+: attach a reason
cancel(ErrTooSlow)
context.Cause(ctx)                                        // retrieves ErrTooSlow

stop := context.AfterFunc(ctx, func() { cleanup() })      // 1.21+: run fn on cancellation
defer stop()
```

> **Trap — `defer cancel()` is mandatory, even for `WithTimeout`.** Not calling it leaks the timer and the child context's goroutine until the deadline fires. `go vet`'s `lostcancel` check exists specifically for this. It's the most common context mistake.

### Cancellation is a tree

Cancelling a parent cancels every descendant, but not the other way round. That's what makes "client disconnected, abandon all the work this request started" a single operation.

```
context.Background()
  └── request ctx (cancelled when the HTTP client disconnects)
        ├── DB query ctx (5s timeout)
        └── downstream API ctx (2s timeout)
```

### Using it correctly

```go
func (s *Scheduler) Run(ctx context.Context) error {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()          // return promptly on cancellation
        case <-ticker.C:
            if err := s.tick(ctx); err != nil {
                return err
            }
        }
    }
}
```

For CPU-bound loops with no channel operation, poll periodically:

```go
for i, row := range rows {
    if i%1000 == 0 {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
    }
    process(row)
}
```

### The rules

1. **`ctx` is the first parameter, always named `ctx`.** Never a struct field, never a return value.
2. **Never pass a nil context.** Use `context.TODO()` if you genuinely don't have one yet.
3. **`context.Value` is for request-scoped metadata only** — request ID, trace span, authenticated user. Not for optional parameters, not for dependencies.
4. **Use an unexported key type**, never a string:

```go
type ctxKey struct{}      // unexported, unique — no collisions possible

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, ctxKey{}, id)
}

func RequestID(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(ctxKey{}).(string)
    return id, ok
}
```

A `string` key can collide with another package's key silently. `struct{}{}` types are unique per package and cost zero bytes.

> **Trap — why is `context.Value` discouraged?** It's untyped (`any` in, `any` out), invisible in the signature so callers can't tell what a function needs, and impossible to check at compile time. It turns explicit dependencies into implicit runtime lookups — service location, essentially. The test: if the function *cannot work* without the value, it should be a parameter. If it's cross-cutting metadata that every layer might want to log, context is right.

> **Trap — a cancelled context does not stop anything by itself.** It closes a channel. If your code never selects on `ctx.Done()`, cancellation does nothing at all. Everything is cooperative.

---

## 7. The Go Memory Model

This is what separates people who've *used* goroutines from people who understand them, and it's the thing to reach for when asked "how do you know this code is correct?"

### The problem

Without synchronisation, there is **no guarantee** one goroutine ever observes another's writes — and no guarantee about ordering. The compiler reorders instructions and the CPU reorders memory operations, both aggressively.

```go
// BROKEN — may loop forever, even though `done` is obviously set
var done bool
var msg string

go func() {
    msg = "hello"
    done = true
}()

for !done {          // the compiler may hoist this read out of the loop entirely
}
fmt.Println(msg)     // and msg may still be ""
```

Two independent failures here: the compiler can cache `done` in a register (making the loop infinite), and even if it re-reads, there's no guarantee `msg` was written before `done` from this goroutine's perspective.

### Happens-before

The memory model defines a partial order. If event A *happens-before* B, then B observes A's writes. The edges you can rely on:

| Establishes happens-before | Detail |
|---|---|
| **Channel send → corresponding receive** | The send happens-before the receive completes |
| **Channel receive → corresponding send completing** | For unbuffered channels, in both directions |
| **`close(ch)` → a receive that returns zero** | Standard shutdown-signal edge |
| **`mu.Unlock()` → subsequent `mu.Lock()`** | The classic critical-section edge |
| **`once.Do(f)` returning → any later `Do` returning** | `f` completes before any caller proceeds |
| **`go` statement → the goroutine's first instruction** | Setup before `go` is visible inside |
| **Goroutine exit → nothing** | ⚠️ *No* edge. You need a `WaitGroup` or channel |
| **`atomic` store → a load observing it** | Sequentially consistent since Go 1.19 |

**Fixed:**

```go
var msg string
done := make(chan struct{})

go func() {
    msg = "hello"
    close(done)       // everything before this is visible after <-done
}()

<-done
fmt.Println(msg)      // guaranteed "hello"
```

> **The critical insight for interviews:** *"it works when I run it"* proves nothing about a concurrent program. Correctness comes from establishing a happens-before edge, not from observing that the timing happened to work out on your laptop. Racy code can work for years and then break because you upgraded Go, changed CPU architecture, or added load. Being able to say this — and name the specific edge your code relies on — is a strong senior signal.

> **Follow-up:** *Is `var x int32; x++` from two goroutines safe if you don't care about the exact count?* No. It's undefined behaviour, not "approximately right." Torn reads and lost updates are the benign outcomes; the memory model gives you no guarantees at all, and the race detector will flag it. Use `atomic.Int64`.

---

## 8. The Race Detector

```bash
go test ./... -race
go build -race && ./app          # also works on binaries, ~10x slower
```

The race detector instruments memory accesses and reports when two goroutines touch the same address without synchronisation, at least one writing.

```
WARNING: DATA RACE
Write at 0x00c000018050 by goroutine 7:
  main.main.func1()
      /app/main.go:12 +0x3c
Previous read at 0x00c000018050 by main goroutine:
  main.main()
      /app/main.go:15 +0x8c
```

### What it cannot do

> **The crucial limitation: the race detector only reports races it actually observes at runtime.** It is not static analysis. If two goroutines never happen to interleave during your test, nothing is reported — and the race is still there. A clean `-race` run is evidence, not proof.
>
> Practical consequences worth stating:
> - Run `-race` in CI on the full suite, not just when investigating.
> - Concurrency tests need enough iterations and parallelism to interleave. A test that spawns two goroutines once will rarely trip it.
> - ~10× CPU and ~5× memory overhead, so it's a CI/staging tool, not production. (Though running one canary instance with `-race` is a real technique.)
> - It detects data races, **not logic races**. A correctly-mutexed check-then-act is race-detector-clean and still wrong — which is exactly the same TOCTOU class you know from the PHP inventory work.

---

## 9. Concurrency Patterns

### Worker pool — the one to be able to write from memory

```go
func WorkerPool[T, R any](
    ctx context.Context,
    inputs []T,
    workers int,
    fn func(context.Context, T) (R, error),
) ([]R, error) {
    jobs := make(chan int)
    results := make([]R, len(inputs))

    g, ctx := errgroup.WithContext(ctx)

    for range workers {
        g.Go(func() error {
            for i := range jobs {
                r, err := fn(ctx, inputs[i])
                if err != nil {
                    return fmt.Errorf("item %d: %w", i, err)
                }
                results[i] = r          // safe: each index written by exactly one goroutine
            }
            return nil
        })
    }

    // Feed jobs, aborting if the group is cancelled
    g.Go(func() error {
        defer close(jobs)
        for i := range inputs {
            select {
            case jobs <- i:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

Points an interviewer looks for:
- **Bounded** worker count, not one goroutine per item
- **Context cancellation** honoured in the feeder, so a failure stops the producer
- **Results by index** — no mutex needed because each slot has exactly one writer
- **Errors propagate**, and `errgroup.WithContext` cancels siblings on the first failure
- **`close(jobs)`** by the sole sender, so workers' `range` terminates

> **Follow-up: "why not one goroutine per item?"** Because the goroutines aren't the constraint — what they hold is. A million goroutines each wanting a DB connection will exhaust a 25-connection pool and queue anyway, just with a million stack allocations of overhead and no backpressure. Bound to the scarce resource.

### errgroup

```go
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error { return fetchUser(ctx) })
g.Go(func() error { return fetchOrders(ctx) })
g.Go(func() error { return fetchInventory(ctx) })

if err := g.Wait(); err != nil {     // first non-nil error; ctx cancelled for the rest
    return err
}
```

`g.SetLimit(n)` bounds concurrency, which makes `errgroup` alone sufficient for most worker pools:

```go
g.SetLimit(10)
for _, job := range jobs {
    g.Go(func() error { return process(ctx, job) })   // blocks when 10 are in flight
}
```

> **Trap:** `Wait` returns only the **first** error; the rest are discarded. If you need all of them, collect into a slice under a mutex and `errors.Join` them.

### Semaphore with a buffered channel

```go
sem := make(chan struct{}, 10)      // 10 permits

for _, job := range jobs {
    sem <- struct{}{}               // acquire (blocks when full)
    go func() {
        defer func() { <-sem }()    // release
        process(job)
    }()
}
```

Worth knowing because it's the idiom people expect, but `errgroup.SetLimit` or `golang.org/x/sync/semaphore` (which supports weighted acquisition and context cancellation) are better in real code.

### Pipeline with cancellation

```go
func gen(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return              // ← without this, this goroutine leaks
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Compose; cancelling ctx tears down the whole pipeline
for v := range square(ctx, gen(ctx, 1, 2, 3, 4)) {
    fmt.Println(v)
}
```

**Every stage must select on `ctx.Done()` when sending.** A consumer that stops reading early (a `break` in the range) leaves every upstream stage blocked forever on a send. This is *the* canonical goroutine leak.

### singleflight — deduplicating concurrent work

```go
var group singleflight.Group

func (c *Cache) Get(ctx context.Context, key string) (*Item, error) {
    v, err, _ := group.Do(key, func() (any, error) {
        return c.fetchFromDB(ctx, key)
    })
    if err != nil {
        return nil, err
    }
    return v.(*Item), nil
}
```

1000 concurrent requests for the same missing key produce **one** database query; the rest wait and share the result.

> **This is the Go answer to cache stampede**, and it maps directly onto the `Cache::lock()` / atomic-lock discussion from the Laravel material. Worth drawing that parallel out loud — it shows you recognise the same problem across stacks. The difference: `singleflight` is in-process only, so with N replicas you get N queries, not one. For cross-process deduplication you still need a distributed lock in Redis.

### Rate limiting

```go
limiter := rate.NewLimiter(rate.Limit(100), 20)   // 100/sec sustained, burst 20

if err := limiter.Wait(ctx); err != nil {          // blocks, respects cancellation
    return err
}
// or
if !limiter.Allow() {                              // non-blocking
    return ErrRateLimited
}
```

`golang.org/x/time/rate` is a token bucket. `Wait` respects context cancellation, which is why it's preferable to a hand-rolled ticker.

### Graceful shutdown

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer stop()

    srv := &http.Server{Addr: ":8080", Handler: router}

    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("server failed", "err", err)
            os.Exit(1)
        }
    }()

    <-ctx.Done()                     // SIGTERM received
    slog.Info("shutting down")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("forced shutdown", "err", err)
    }

    workers.Wait()                   // drain background work too
    slog.Info("stopped cleanly")
}
```

Points that matter:
- **Use a fresh context for shutdown.** The signal context is already cancelled, so passing it to `Shutdown` would abort immediately — a very common bug.
- `Shutdown` stops accepting new connections and waits for in-flight requests.
- `ErrServerClosed` is the normal return from `ListenAndServe` after shutdown, not a failure.
- **Kubernetes context:** you have `terminationGracePeriodSeconds` (default 30) before SIGKILL. Your timeout must be under it, and you should stop passing readiness checks *before* you stop accepting connections, so the load balancer drains you first.

---

## 10. Goroutine Leaks

A leaked goroutine is one that never returns. It holds its stack, everything its closure references, and any resource it acquired — forever. This is the most common serious bug in production Go services and a guaranteed interview topic.

### The four causes

**1. Blocked forever on a channel send (no receiver)**

```go
func leak() {
    ch := make(chan int)          // unbuffered
    go func() { ch <- expensive() }()
    return                        // nobody ever receives — that goroutine is stuck for the process's life
}
```

The classic real version: a timeout abandons the caller but the worker still tries to send.

```go
// LEAKS on every timeout
func fetch(ctx context.Context) (int, error) {
    ch := make(chan int)
    go func() { ch <- slowCall() }()      // blocks forever if we time out
    select {
    case v := <-ch:
        return v, nil
    case <-ctx.Done():
        return 0, ctx.Err()               // goroutine still blocked on the send
    }
}

// FIXED — buffer of 1 means the send always completes
func fetch(ctx context.Context) (int, error) {
    ch := make(chan int, 1)               // ← the entire fix
    go func() { ch <- slowCall() }()
    select {
    case v := <-ch:
        return v, nil
    case <-ctx.Done():
        return 0, ctx.Err()               // goroutine sends into the buffer and exits
    }
}
```

> **This one-character fix is worth memorising.** "Use a buffered channel of size 1 so an abandoned producer can still complete its send and exit" is a precise, senior answer.

**2. Blocked forever on a channel receive (no sender)**

```go
go func() {
    for v := range ch {     // if nobody ever closes ch, this never returns
        process(v)
    }
}()
```

**3. Ignoring context cancellation** — an infinite loop with no `ctx.Done()` case.

**4. `time.Ticker` never stopped**

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()          // without this, the runtime timer stays alive
```

### Detection

```go
// In production: expose pprof and watch the trend
import _ "net/http/pprof"
go func() { http.ListenAndServe("localhost:6060", nil) }()
```

```bash
curl localhost:6060/debug/pprof/goroutine?debug=1     # counts grouped by stack
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

**A steadily rising `runtime.NumGoroutine()` is the signature.** Export it as a Prometheus gauge and alert on the trend — that's the practical answer to "how would you catch this before a customer does."

In tests:

```go
func TestNoLeaks(t *testing.T) {
    defer goleak.VerifyNone(t)      // go.uber.org/goleak
    // ... exercise the code
}
```

Go 1.26 also ships an experimental **goroutine leak profiler** (`GOEXPERIMENT=goroutineleakprofile`), which identifies goroutines blocked with no possible way to proceed. Worth naming as something you're tracking.

---

## 11. Deadlocks & Livelocks

### The runtime detector, and its big limitation

```go
func main() {
    ch := make(chan int)
    ch <- 1        // no receiver, and no other goroutine exists
}
// fatal error: all goroutines are asleep - deadlock!
```

> **Critical caveat:** the runtime only detects a **global** deadlock — every goroutine asleep. If one goroutine is spinning, sleeping, or blocked on a network read, the runtime sees a runnable program and reports nothing, even though the other 500 goroutines are deadlocked. In a real server there's always an accept loop running, so **you will essentially never see this error in production.** Partial deadlocks show up as rising goroutine counts and timeouts instead. Making this point is a good depth signal, because most candidates cite the detector as though it's a safety net.

### Lock ordering

```go
// DEADLOCK — two goroutines acquire in opposite orders
// G1: mu1.Lock(); mu2.Lock()
// G2: mu2.Lock(); mu1.Lock()
```

Same problem, same fix as the database deadlocks in your Laravel work: **establish a global lock order and always acquire in that order.** For dynamic sets, order by a stable key:

```go
func TransferStock(a, b *Warehouse, qty int) {
    first, second := a, b
    if a.ID > b.ID {
        first, second = b, a         // deterministic order by ID
    }
    first.mu.Lock()
    defer first.mu.Unlock()
    second.mu.Lock()
    defer second.mu.Unlock()
    // ...
}
```

Being able to say *"it's the same consistent-lock-ordering rule I used to fix MySQL deadlocks — the mechanism differs, the discipline is identical"* connects your existing experience to Go concretely.

### Diagnosing a stuck process

```bash
kill -QUIT <pid>      # SIGQUIT dumps all goroutine stacks and exits
```

Or `curl localhost:6060/debug/pprof/goroutine?debug=2` for a full stack dump without killing the process. Look for many goroutines parked in `sync.runtime_SemacquireMutex` or `chan send`.

---

## 12. Testing Concurrent Code

### The old problem

Concurrent tests were flaky and slow because you had to `time.Sleep` and hope.

```go
// BAD — flaky and slow. Fails on a loaded CI runner, passes locally.
go worker()
time.Sleep(100 * time.Millisecond)
assert.Equal(t, 1, counter.Load())
```

### testing/synctest — stable in Go 1.25

`synctest` runs a test in an isolated "bubble" with a **fake clock**. Time only advances when every goroutine in the bubble is blocked, so timing is deterministic and instant.

```go
func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
        defer cancel()

        start := time.Now()
        <-ctx.Done()                       // returns INSTANTLY in real time

        if elapsed := time.Since(start); elapsed != time.Hour {
            t.Errorf("fake clock elapsed = %v, want 1h", elapsed)
        }
    })
}
```

A one-hour timeout test runs in microseconds and is fully deterministic. `synctest.Wait()` blocks until every other goroutine in the bubble is durably blocked, which lets you assert on intermediate states without sleeping.

> **This is a strong thing to mention.** It's recent (experimental in 1.24, stable in 1.25), it solves a problem every Go team has, and knowing it signals you follow the language. For Chronos — a scheduler that is fundamentally about time — it's transformative: you can test "what happens when a job's deadline passes during leader election" deterministically instead of with sleeps.

### Table-driven tests with parallelism

```go
func TestProcess(t *testing.T) {
    tests := []struct {
        name    string
        input   int
        want    int
        wantErr bool
    }{
        {"zero", 0, 0, false},
        {"positive", 5, 25, false},
        {"negative", -1, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()                 // safe: since 1.22, tt is per-iteration
            got, err := Process(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

Pre-1.22 this needed `tt := tt` before the closure. It doesn't any more — same loop-variable change from Tier 1 §10.

### Forcing races to surface

```go
func TestConcurrentIncrement(t *testing.T) {
    c := NewCounter()
    var wg sync.WaitGroup
    for range 100 {                  // enough goroutines to actually interleave
        wg.Go(func() {
            for range 1000 {
                c.Inc()
            }
        })
    }
    wg.Wait()
    if got := c.Value(); got != 100_000 {
        t.Errorf("got %d, want 100000", got)
    }
}
```

Run with `-race -count=10`. Two goroutines once will not reliably trip the detector; a hundred doing a thousand operations each will.

---

## 13. Anti-Patterns

**Starting a goroutine without knowing how it ends.** Every `go` statement should have an answer to "what makes this return?" If you can't say, it's a leak.

**Using channels where a mutex is clearer.** *"Share memory by communicating"* is a guideline, not a law. Protecting a counter or a map with a mutex is simpler and faster than routing every access through a channel and a manager goroutine. The Go team's own advice: use whichever is more obvious; channels for transferring ownership and orchestrating, mutexes for guarding state.

**Unbounded goroutine spawning.** `for _, item := range millionItems { go process(item) }`. Bound it.

**`time.Sleep` as synchronisation.** It's a race with a timer attached. Use channels, `WaitGroup`, or `synctest`.

**Passing context in a struct.** `type Server struct { ctx context.Context }` breaks per-request cancellation — every request shares one lifetime. The exception is a long-lived worker whose context genuinely *is* the object's lifetime, and even then take it in `Run(ctx)`.

**Ignoring `ctx.Done()` in loops.** Cancellation is cooperative; a loop that never checks is uncancellable.

**Naked `go` in an HTTP handler.** The request context is cancelled when the handler returns, so anything the goroutine does with it fails immediately — and any panic kills the process. See Tier 3 §3 for the Gin-specific version of this.

---

## 14. Tier 2 Q&A Drill

### Goroutines and channels

**1. How big is a goroutine's initial stack, and why does it matter?**
2 KB, grown by copying as needed. It's what makes hundreds of thousands of concurrent goroutines viable where OS threads (1–8 MB reserved) would not be.

**2. How do you kill a goroutine?**
You can't. You signal it via a channel or cancelled context and it returns voluntarily. If it's blocked on something that doesn't watch for cancellation, it leaks permanently.

**3. What happens when `main` returns with goroutines still running?**
The process exits immediately and their defers do not run.

**4. Send on a closed channel?**
Panic.

**5. Receive from a closed channel?**
Returns immediately: buffered values first, then the zero value with `ok == false`.

**6. Send or receive on a nil channel?**
Blocks forever. Useful deliberately — setting a channel to nil disables its case in a `select`.

**7. Close a nil or already-closed channel?**
Panic in both cases.

**8. Who should close a channel?**
The sender, always. With multiple senders, a coordinator closes after `wg.Wait()`.

**9. Why close a channel instead of sending a sentinel value for shutdown?**
Closing wakes *every* receiver; a send wakes exactly one. Broadcast requires close.

**10. Unbuffered vs buffered — what's the real difference?**
Unbuffered is a synchronisation point: the send completes only when a receiver takes it, giving a happens-before edge both ways. Buffered decouples them and hides backpressure.

**11. Should you add a buffer to make things faster?**
No. Choose a buffer for a semantic reason (known count, semaphore). A buffer added for speed just lets the producer build a queue of stale work before blocking anyway.

**12. If two `select` cases are ready, which runs?**
One chosen uniformly at random, to prevent starvation.

**13. What does `default` do in a `select`?**
Makes it non-blocking — runs immediately if no other case is ready.

**14. Why would a `select` with a closed channel spin at 100% CPU?**
A closed channel is always ready, so the case fires continuously. Set the variable to nil to disable that case.

**15. `ctx.Done()` or `time.After` for a timeout?**
`ctx.Done()`, so the deadline propagates downstream. `time.After` only times out the caller while the work continues.

### sync and atomic

**16. Is `sync.Mutex` reentrant?**
No. Locking twice in one goroutine deadlocks. Deliberate — reentrant locks hide broken invariants.

**17. Why must you not copy a mutex?**
You get two independent locks protecting nothing. `go vet`'s `copylocks` catches it.

**18. Is `RWMutex` always better for read-heavy workloads?**
No. It has higher uncontended overhead, so for very short critical sections a plain `Mutex` is faster. It wins when reads are long and greatly outnumber writes.

**19. What's writer starvation and how does Go's `RWMutex` handle it?**
Continuous readers could block a writer forever. Go blocks new readers once a writer is waiting, at the cost of stalling readers behind that writer.

**20. Why must `wg.Add` be outside the goroutine?**
Calling it inside races with `Wait`, which may observe zero and return early. `wg.Go` (1.25+) removes the hazard.

**21. What does `sync.Once` do if the function fails?**
Nothing special — it never runs again. A failed initialisation is cached forever, which is usually wrong for network operations.

**22. When should you use `sync.Pool`?**
Only when profiling shows allocation pressure from high-churn temporary objects. Never for connections — the pool is cleared at every GC.

**23. What must you always do with a pooled object?**
Reset it. `Get` returns a dirty object exactly as it was put back.

**24. Atomics or mutex?**
Atomics protect a single variable; mutexes protect an invariant across several. Two atomics updated separately can be observed inconsistently.

**25. What's `atomic.Pointer` good for?**
Lock-free copy-on-write for read-heavy immutable data — hot-reloadable config, routing tables. Readers never block.

### context

**26. Why is `defer cancel()` required even with `WithTimeout`?**
Otherwise the timer and the child context leak until the deadline fires. `go vet`'s `lostcancel` checks for it.

**27. Does cancelling a context stop the work?**
No. It closes a channel. Code that never selects on `ctx.Done()` is unaffected — cancellation is entirely cooperative.

**28. Why use an unexported struct type as a context key?**
String keys can silently collide across packages. An unexported `struct{}` type is unique per package and costs zero bytes.

**29. Why is `context.Value` discouraged?**
Untyped, invisible in the signature, unverifiable at compile time — it's service location. If the function can't work without it, make it a parameter.

**30. Where should a context live?**
First parameter of the function. Never a struct field, never a return value.

**31. What's `context.WithCancelCause` for?**
Attaching a reason to cancellation, retrievable with `context.Cause(ctx)`, so you can distinguish "client hung up" from "deadline exceeded" from "shed load."

### Memory model and races

**32. Why might `for !done {}` loop forever even after `done = true`?**
Without a happens-before edge the compiler can hoist the read into a register, and there's no guarantee the write is visible at all. Not a timing issue — an absence-of-guarantee issue.

**33. Name four things that establish happens-before.**
Channel send→receive, `Unlock`→`Lock`, `close`→receive, `once.Do` completing, and the `go` statement→goroutine start.

**34. Does a goroutine exiting establish happens-before with anything?**
No. You need a `WaitGroup`, channel, or other explicit synchronisation.

**35. If `-race` reports nothing, is the code race-free?**
No. The detector reports only races it observes at runtime. A clean run is evidence, not proof.

**36. What's the race detector's overhead?**
Roughly 10× CPU, 5× memory. CI and staging tool, not a production default — though a single `-race` canary is a real technique.

**37. Data race vs logic race?**
A data race is unsynchronised concurrent access to memory. A logic race (check-then-act) can be perfectly synchronised and still wrong — the detector won't see it.

### Patterns and leaks

**38. Name the four ways goroutines leak.**
Blocked forever on a send with no receiver; blocked on a receive with no sender or close; a loop ignoring `ctx.Done()`; an unstopped `Ticker`.

**39. A worker sends to an unbuffered channel but the caller timed out. What happens, and what's the fix?**
The worker blocks on the send forever. Make the channel buffered with capacity 1 so the send always completes and the goroutine exits.

**40. How do you find a goroutine leak in production?**
Export `runtime.NumGoroutine()` as a gauge and alert on the upward trend, then pull `/debug/pprof/goroutine?debug=2` and look for many goroutines parked at the same stack frame.

**41. Why doesn't the runtime's deadlock detector help in production?**
It only fires when *every* goroutine is asleep. A server always has an accept loop running, so partial deadlocks are invisible to it and show up as rising goroutine counts and timeouts.

**42. How do you prevent lock-ordering deadlocks?**
Establish a global acquisition order; for dynamic sets, sort by a stable key before locking. Same discipline as avoiding database deadlocks.

**43. What does `errgroup.WithContext` give you over `WaitGroup`?**
Error propagation, and automatic cancellation of siblings when the first goroutine fails.

**44. What's the limitation of `errgroup.Wait`?**
It returns only the first error. Collect them yourself and `errors.Join` if you need all.

**45. What problem does `singleflight` solve?**
Cache stampede — N concurrent requests for the same key collapse into one call. In-process only, so with N replicas you get N calls; cross-process needs a distributed lock.

**46. Why use a buffered channel as a semaphore?**
Acquiring is a send that blocks when the buffer is full, so the buffer size is the concurrency limit. `errgroup.SetLimit` is usually cleaner.

**47. What's the bug in passing the signal context to `srv.Shutdown`?**
It's already cancelled, so shutdown aborts immediately instead of draining. Use a fresh context with its own timeout.

**48. In Kubernetes, what constrains your shutdown timeout?**
`terminationGracePeriodSeconds` (default 30) before SIGKILL. Fail readiness first so the load balancer drains you before you stop accepting.

**49. How do you test a one-hour timeout without waiting an hour?**
`testing/synctest` (stable in 1.25) — a fake clock that advances only when all bubbled goroutines are blocked, so the test is instant and deterministic.

**50. When is a mutex better than a channel?**
When you're guarding state rather than transferring ownership. A counter or cache behind a mutex is simpler and faster than a manager goroutine and a channel per operation.

---

> **The Tier 2 senior signal:** anyone can write `go func()`. What's tested is whether you know how it *ends*, what happens when the caller gives up, and how you'd prove it correct. Every concurrency answer you give should mention its cancellation path and its failure mode unprompted — that single habit distinguishes a senior Go answer from a competent one.

---

**Next:** [`03-gin-web.md`](./03-gin-web.md) — `net/http` foundations, Gin's router and context, middleware, binding and validation, `database/sql` pool tuning, gRPC, and testing.

**Back to:** [`README.md`](./README.md) · [`01-basic.md`](./01-basic.md)
