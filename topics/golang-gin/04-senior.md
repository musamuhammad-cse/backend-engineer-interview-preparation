# Go — Tier 4: Runtime, Performance & Distributed Systems

> Verified against **Go 1.26**. Escape-analysis output, allocation counts, and compiler thresholds in this file are real measurements from this toolchain, not recalled figures.
>
> This is the tier where a senior loop is won or lost. Tiers 1–3 establish that you can write Go; this one establishes that you understand what it does at runtime and that you've operated it under load. **Your Chronos project is the spine of §9–11** — those sections are written to give you a defensible, detailed story.

---

## Table of Contents

1. [The GMP Scheduler](#1-the-gmp-scheduler)
2. [Preemption & the Netpoller](#2-preemption--the-netpoller)
3. [Goroutine Stacks](#3-goroutine-stacks)
4. [Escape Analysis](#4-escape-analysis)
5. [Garbage Collection](#5-garbage-collection)
6. [GOGC, GOMEMLIMIT & GOMAXPROCS](#6-gogc-gomemlimit--gomaxprocs)
7. [Profiling with pprof](#7-profiling-with-pprof)
8. [Benchmarking Correctly](#8-benchmarking-correctly)
9. [Raft & Chronos](#9-raft--chronos)
10. [Distributed Locks & Fencing](#10-distributed-locks--fencing)
11. [Idempotency & Delivery Semantics](#11-idempotency--delivery-semantics)
12. [Resilience Patterns](#12-resilience-patterns)
13. [Observability](#13-observability)
14. [Deployment & Production](#14-deployment--production)
15. [Tier 4 Q&A Drill](#15-tier-4-qa-drill)

---

## 1. The GMP Scheduler

Go multiplexes many goroutines onto few OS threads. Three entities:

| | Name | What it is |
|---|---|---|
| **G** | Goroutine | A function with its own stack and state. Cheap, millions possible |
| **M** | Machine | An OS thread. Actually executes code |
| **P** | Processor | A scheduling context holding a run queue. **`GOMAXPROCS` of them** |

**To run Go code, an M must hold a P.** The number of Ps caps how many goroutines run *simultaneously*; the number of Ms can be much larger because threads blocked in syscalls don't hold a P.

```
   ┌── P0 ──┐        ┌── P1 ──┐         Global run queue
   │ runq:  │        │ runq:  │         [ G G G ]
   │ G G G  │        │ G G    │
   └───┬────┘        └───┬────┘         netpoller: [G waiting on I/O]
       │                 │
      M0                M1
```

### Work stealing

Each P has a local run queue (256 slots). When a P empties its queue it looks for work in this order:

1. Its own local run queue
2. The global run queue (checked every 61st scheduling tick regardless, to prevent starvation)
3. The netpoller, for goroutines whose I/O completed
4. **Steal half** of a randomly chosen other P's queue

Work stealing keeps cores busy without a central lock on every scheduling decision. The "every 61 ticks" detail is a nice one to know: without it, two goroutines that keep spawning each other into a local queue could starve the global queue forever.

### The blocking-syscall handoff

This is the mechanism to be able to explain, because it's what makes Go's I/O model work:

When a goroutine makes a **blocking syscall** (a file read, a `cgo` call), its M is stuck in the kernel. Rather than lose a scheduling slot, the runtime **detaches the P from that M** and hands it to another M (reusing a parked one or creating a new one). The original M stays blocked with just its goroutine; the P and its remaining run queue carry on elsewhere.

When the syscall returns, the M tries to reacquire a P. If none is free, its goroutine goes to the global run queue and the M parks.

> **The consequence:** you can have thousands of OS threads if you have thousands of goroutines blocked in syscalls, even with `GOMAXPROCS=8`. The default thread limit is 10,000, and hitting it is a fatal error. This is the answer to "can Go run out of threads?" — yes, typically via `cgo` calls or heavy blocking file I/O, and it's rare because network I/O doesn't work this way at all (§2).

---

## 2. Preemption & the Netpoller

### Asynchronous preemption (Go 1.14+)

Before 1.14, Go was **cooperatively** scheduled: a goroutine yielded only at function calls, channel operations, or allocations. A tight loop with none of those could hog a P forever.

```go
// Pre-1.14: this could hang the whole program at GOMAXPROCS=1.
// The GC waits for all goroutines to reach a safe point — and this one never does.
go func() { for {} }()
```

Since 1.14 the runtime sends a `SIGURG` to a thread running a goroutine for more than ~10 ms, and the signal handler preempts it at an asynchronous safe point.

> **Why this matters beyond trivia:** GC needs every goroutine to reach a safe point to complete its stop-the-world phases. Pre-emption made those phases reliably sub-millisecond regardless of what user code does. If asked "what was the biggest runtime improvement in Go's history," the GC latency work (1.5 concurrent collector → 1.8 sub-millisecond STW → 1.14 async preemption) is a well-informed answer.

### The netpoller

**Network I/O does not block a thread.** This is the single most important performance property of Go servers.

When a goroutine reads from a socket with no data available:
1. The fd is registered with the OS event mechanism (`epoll` on Linux, `kqueue` on BSD, IOCP on Windows).
2. The goroutine is parked — it costs nothing but its stack.
3. **The M is released to run other goroutines.**
4. When the fd becomes ready, the netpoller returns the goroutine to a run queue.

```go
// This looks like blocking, synchronous code...
n, err := conn.Read(buf)
// ...but the runtime turned it into epoll registration + goroutine park.
```

> **This is the whole answer to "why is Go good for network services?"** You write straightforward synchronous code and get event-driven I/O performance, without callbacks, promises, or an async/await colouring problem. 100,000 idle connections cost ~100,000 parked goroutines (a few hundred MB of stacks) and *zero* blocked threads. The Node.js comparison is instructive: Node gets the same I/O efficiency but forces async syntax on you and can't use multiple cores without extra processes; Go gives you both.
>
> The contrast with `php-fpm` is even sharper and worth drawing: there, one blocking database call occupies an entire worker process for its duration, so your concurrency ceiling is your worker count. In Go, a goroutine waiting on the database costs a parked stack and the thread moves on.

---

## 3. Goroutine Stacks

- Start at **2 KB**, contiguous.
- Grow by **copying**: allocate a stack twice the size, copy the frames, adjust pointers, free the old.
- Shrink during GC when using less than a quarter of the space.
- Max 1 GB on 64-bit (`debug.SetMaxStack` to change), and exceeding it is a fatal `stack overflow`.

Every function has a prologue that checks whether the stack has room; if not, it calls `morestack`.

> **Follow-up: "if the stack moves, don't pointers to stack variables break?"** No — the runtime knows the exact layout of every frame from compiler-emitted metadata and rewrites every pointer that points into the old stack. This is only possible because Go is garbage collected and fully type-aware, and it's why Go uses contiguous growable stacks rather than the segmented stacks it had before 1.4 (which suffered a "hot split" problem where a loop calling a function across a segment boundary thrashed).

---

## 4. Escape Analysis

The compiler decides at compile time whether a value can live on the stack (free, reclaimed on return) or must go on the heap (costs an allocation and GC work).

**The rule: if the compiler cannot prove a value stops being referenced when the function returns, it escapes.**

```bash
go build -gcflags='-m' ./...
```

### Real, verified output

These are actual results from Go 1.26 on this machine:

```go
//go:noinline
func dynamicSlice(n int) byte { buf := make([]byte, n); buf[0] = 1; return buf[0] }

//go:noinline
func smallConst() byte { buf := make([]byte, 64); buf[0] = 1; return buf[0] }

//go:noinline
func bigConst() byte { buf := make([]byte, 1<<20); buf[0] = 1; return buf[0] }
```

```
main.go:7:13:  make([]byte, n) does not escape
main.go:14:13: make([]byte, 64) does not escape
main.go:21:13: make([]byte, 1048576) escapes to heap
```

```
BenchmarkDynamic-12   323210820   3.361 ns/op     0 B/op   0 allocs/op
BenchmarkSmall-12     581205402   1.943 ns/op     0 B/op   0 allocs/op
BenchmarkBig-12           10000  238502 ns/op   1048579 B/op   1 allocs/op
```

> **This is worth internalising because the common interview answer is wrong.** People routinely say "a slice with a runtime-determined size must heap-allocate." Measured: `make([]byte, n)` with a dynamic `n` allocated **zero bytes**. Modern Go emits a variable-size stack allocation with a runtime size check, falling back to the heap only if the value is too large or actually escapes.
>
> Saying *"I'd check with `-gcflags=-m` rather than assume, because the compiler stack-allocates more than people expect — I've seen dynamically-sized `make` stay on the stack"* is a much stronger answer than reciting a rule of thumb.

### What actually forces the heap

Verified causes, in rough order of how often they bite:

```go
// 1. Returning a pointer that outlives the frame
func New() *Job { return &Job{} }           // &Job{...} escapes to heap

// 2. Interface boxing — including every fmt call
func log(j Job) { fmt.Println(j) }           // j escapes to heap

// 3. Capture by a goroutine closure
func spawn() { j := Job{}; go func() { _ = j }() }   // func literal escapes to heap

// 4. Exceeding the size threshold
buf := make([]byte, 1<<20)                   // escapes to heap
```

**Exact thresholds** (from `cmd/compile/internal/ir/cfg.go` in this toolchain):

| Threshold | Value | Applies to |
|---|---|---|
| `MaxStackVarSize` | **128 KB** | Explicit declarations: `var x T`, `x := ...` |
| `MaxImplicitStackVarSize` | **64 KB** | Implicit: `new(T)`, `&T{}`, `make([]T, n)` |

Quoting these precisely, with where they come from, is a genuine depth signal.

> **Trap — `fmt.Println(x)` forces `x` to the heap, always.** The variadic `...any` boxes every argument into an interface, and because `fmt` uses reflection the compiler can't prove the value doesn't escape. This is why debug logging in a hot loop is disproportionately expensive, and why `slog` with typed attributes (`slog.Int("n", n)`) is cheaper than `slog.Any`.

> **Trap — inlining changes escape results.** In my measurements, `func heapAlloc() *Job { return &Job{} }` reported `escapes to heap` at its definition but `does not escape` at the inlined call site in `main`. Escape analysis runs after inlining, so a function that allocates in isolation may not allocate when inlined. **Never reason about allocations from source alone — measure with `-benchmem`.**

---

## 5. Garbage Collection

Go uses a **concurrent, tri-colour, mark-and-sweep, non-generational, non-compacting** collector. Each of those words is a design decision an interviewer may probe.

### The tri-colour algorithm

- **White** — not yet examined; candidates for collection
- **Grey** — reachable, but its references haven't been scanned yet
- **Black** — reachable, and fully scanned

1. Everything starts white.
2. Roots (stacks, globals) go grey.
3. Repeatedly: take a grey object, blacken it, and grey everything it points to.
4. When no grey remain, all white objects are unreachable and get swept.

**The invariant: a black object must never point to a white object.** If mutator code broke that during the concurrent mark, a live object could be collected.

### Write barriers

Since the application runs *concurrently* with marking, it can break the invariant:

```go
black.field = white   // black object now references a white one
```

Go inserts a **write barrier** — a small piece of compiler-generated code on every pointer write during the mark phase — which greys the object being stored. Go uses a hybrid Dijkstra/Yuasa barrier that removes the need to rescan stacks.

> **This is why pointer-heavy data structures cost more than value-heavy ones.** Every pointer write during marking pays the barrier, and every pointer is another edge to trace. It's the concrete argument for `[]Item` over `[]*Item`, and for a `map[string]int` over a `map[string]*int`.

### The GC cycle

```
1. Sweep termination     — STW, brief
2. Mark                  — CONCURRENT with your code; write barrier on
3. Mark termination      — STW, sub-millisecond
4. Sweep                 — CONCURRENT, lazy
```

Only steps 1 and 3 stop the world, and both are **sub-millisecond** and bounded by goroutine count, not heap size. Go deliberately optimises for **latency over throughput** — it will burn more CPU than a generational collector to keep pauses tiny.

### Mark assists

If your program allocates faster than the collector marks, the allocating goroutine is conscripted to do marking work proportional to what it allocated. **This is why a service under allocation pressure gets slower rather than running out of memory** — the cost is charged back to the allocator. Seeing high `GC assist` time in a trace is a strong signal to reduce allocations.

### Green Tea GC — default in Go 1.26

Verified in this toolchain: `GreenTeaGC: true` in the baseline experiment configuration, so it is **on by default in 1.26** (it was opt-in in 1.25).

Green Tea changes the marking algorithm to scan memory **span-at-a-time rather than object-at-a-time**, improving locality and enabling SIMD-assisted scanning. The reported benefit is a 10–40% reduction in GC overhead for pointer-heavy, allocation-heavy workloads. Knowing this is current, correctly attributed, and off-by-default-in-1.25/on-in-1.26 is exactly the kind of thing that reads as "keeps up with the ecosystem."

### Why not generational?

> **A great question to be ready for.** The generational hypothesis (most objects die young) does hold for Go. But Go doesn't need a generational collector as much as a JVM does, because escape analysis puts most short-lived objects on the **stack**, where they're free — the generational nursery's job is partly already done by the compiler. Combined with a non-compacting design (Go has interior pointers and `unsafe`, so objects can't be moved freely), the added complexity wasn't worth it. The Go team has experimented with it repeatedly and concluded the gains didn't justify the cost.

---

## 6. GOGC, GOMEMLIMIT & GOMAXPROCS

### GOGC

```bash
GOGC=100    # default: collect when the heap has grown 100% since the last mark
GOGC=200    # let it grow 200% — fewer GCs, more memory, more throughput
GOGC=off    # disable entirely (only sane with GOMEMLIMIT set)
```

With `GOGC=100`, a 100 MB live heap triggers the next GC at 200 MB. It's a pure **time/space trade**: higher means less CPU on GC and more memory used.

### GOMEMLIMIT (Go 1.19+)

```bash
GOMEMLIMIT=1GiB
```

A **soft** limit on total memory. As the heap approaches it, the GC runs more aggressively — continuously if needed — to stay under it.

> **The killer combination for containers, and a strong thing to volunteer:**
>
> ```bash
> GOGC=off GOMEMLIMIT=900MiB       # in a 1 GiB container
> ```
>
> This gives you: no GC at all until you approach 900 MiB, so minimal CPU overhead in steady state; and a hard-ish ceiling that prevents the OOM killer.
>
> **The problem it solves is real.** Before `GOMEMLIMIT`, a Go service in a memory-limited container could get OOM-killed *while having plenty of garbage to collect*, because `GOGC` is relative to live heap and knows nothing about the container limit. A brief spike in live data raised the trigger point above the cgroup limit, and the kernel killed the process before the next GC ran. Being able to describe that failure mode concretely is a strong "I've operated Go in Kubernetes" signal.
>
> **The caveat:** it's soft. If your *live* heap genuinely exceeds the limit, the GC will thrash — burning most of your CPU on collection while making no progress — rather than OOM. That's arguably worse, since a thrashing pod stays "healthy" to Kubernetes while serving nothing. Leave headroom (~90% of the container limit) and alert on GC CPU fraction.

### GOMAXPROCS — container-aware since Go 1.25

Sets the number of Ps: how many goroutines execute Go code simultaneously. Defaults to the number of logical CPUs.

> **This answer changed recently and knowing the new version is a differentiator.**
>
> **Historically (Go ≤ 1.24):** `GOMAXPROCS` read the *host's* CPU count and ignored cgroup limits entirely. A pod limited to 500m CPU on a 64-core node got `GOMAXPROCS=64`. The result: 64 Ps competing for half a core, heavy context switching, and CFS throttling that showed up as brutal p99 latency. The standard fix was `go.uber.org/automaxprocs`, imported for its side effect.
>
> **Since Go 1.25:** the runtime is **cgroup-aware on Linux** and derives `GOMAXPROCS` from the container's CPU limit, updating it if the limit changes at runtime. I verified this toolchain ships `runtime/cgroup_linux.go` with a `containermaxprocs` GODEBUG toggle. **`automaxprocs` is no longer needed on 1.25+.**
>
> Saying *"that used to require `automaxprocs`, but the runtime became cgroup-aware in 1.25 so it's handled natively now"* demonstrates you track the language rather than repeating a blog post from 2019.

---

## 7. Profiling with pprof

```go
import _ "net/http/pprof"       // registers handlers on DefaultServeMux

func main() {
    go func() {
        // NEVER expose this publicly — it leaks source paths and allows CPU-heavy requests
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
}
```

### The profiles and what each is for

| Profile | Endpoint | Answers |
|---|---|---|
| CPU | `/debug/pprof/profile?seconds=30` | Where is CPU time spent? |
| Heap | `/debug/pprof/heap` | What's using memory *now* (live) |
| Allocs | `/debug/pprof/allocs` | What has allocated *since start* (churn) |
| Goroutine | `/debug/pprof/goroutine` | How many, and blocked where — **leak hunting** |
| Block | `/debug/pprof/block` | Time blocked on channels/mutexes (needs `SetBlockProfileRate`) |
| Mutex | `/debug/pprof/mutex` | Lock contention (needs `SetMutexProfileFraction`) |
| Trace | `/debug/pprof/trace?seconds=5` | Full timeline: scheduling, GC, syscalls |

```bash
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/heap
go tool trace trace.out
```

Inside `pprof`: `top`, `list <func>` (line-level costs), `web` (call graph), `peek`.

### Heap vs allocs — the distinction that matters

- **`inuse_space`** — live memory right now. Use this for **leaks**.
- **`alloc_space`** — everything ever allocated. Use this for **GC pressure**.

A function can dominate `alloc_space` while contributing nothing to `inuse_space` — that's high churn, not a leak, and the fix is different (reduce allocation rate vs find the retaining reference).

### A memory leak walkthrough — have this ready as a narrative

> "Memory climbs steadily and the pod OOMs every few days. Here's how I'd work it:
>
> **1. Confirm it's the heap, not goroutines.** Check `runtime.NumGoroutine()` first — a rising goroutine count is a far more common cause than genuine heap growth, and it presents identically. If goroutines are flat, it's the heap.
>
> **2. Take two heap profiles**, hours apart, and diff them:
> ```bash
> go tool pprof -base heap1.pb.gz heap2.pb.gz
> ```
> The diff shows exactly what grew, which is much more useful than a single snapshot.
>
> **3. Look at `inuse_space`, not `alloc_space`.** Churn is not a leak.
>
> **4. Find who's retaining it.** In Go a 'leak' is always a live reference: an unbounded map used as a cache with no eviction, a slice appended to forever, a `sync.Pool` holding oversized buffers, or goroutines parked with their stacks and captured variables alive.
>
> **5. Confirm the fix by diffing profiles again**, and add a `NumGoroutine` and `heap_inuse` gauge to Grafana so the next one is caught by an alert instead of an OOM."

That structure — confirm the category, diff rather than snapshot, distinguish live from churn, then verify — is what an interviewer wants. Most candidates jump straight to "I'd use pprof."

---

## 8. Benchmarking Correctly

```go
func BenchmarkParseCron(b *testing.B) {
    expr := "*/5 * * * *"
    b.ReportAllocs()

    for b.Loop() {              // Go 1.24+ — preferred over `for i := 0; i < b.N; i++`
        ParseCron(expr)
    }
}
```

```bash
go test -bench=. -benchmem -count=10 ./... | tee new.txt
benchstat old.txt new.txt
```

> **`b.Loop()` (Go 1.24+) is the correct modern form** and fixes two real problems with `for i := 0; i < b.N; i++`:
> 1. **It prevents the compiler from optimising away the loop body**, so you don't need `runtime.KeepAlive` or a package-level sink variable.
> 2. **Setup before the loop is excluded from timing automatically**, so `b.ResetTimer()` is usually unnecessary.
>
> Using `b.Loop` is a small but unmistakable signal that you write Go in 2026 rather than 2019.

**Three benchmarking mistakes to name:**

```go
// 1. Dead code elimination — the compiler removes an unused result
for b.Loop() { Fib(20) }                  // safe with b.Loop; with b.N you needed a sink

// 2. Timing your setup
data := generateHugeDataset()             // b.Loop excludes this automatically
for b.Loop() { Process(data) }

// 3. Trusting a single run
// Always -count=10 and compare with benchstat. A single number is noise.
```

> **The framing that matters most:** benchmarks measure microseconds; users experience p99 latency under concurrent load with a cold cache. A 30% faster function that isn't on the critical path is worth nothing. **Profile first to find where time actually goes, then benchmark the specific thing you're changing, then verify in production with the same metric you optimised.** Tying this to your 88% query reduction is natural: that win came from eliminating N+1 queries, not from making any single function faster — an algorithmic and I/O win, which is where the real wins nearly always are.

---

## 9. Raft & Chronos

Your distributed job scheduler is the strongest asset you have for a senior Go interview. This section is written so you can go three questions deep on it.

### The problem it solves

A job scheduler across N nodes must fire each scheduled job **exactly once**, survive node failure, and never double-fire. Two schedulers both deciding "the nightly billing job is due" means customers get billed twice. That's the core tension, and stating it that concretely is a strong opening.

### Raft in the terms you need

Raft is a consensus algorithm chosen over Paxos because it's decomposed into three understandable pieces:

**Leader election.** Every node is a Follower, Candidate, or Leader. Followers expect a heartbeat from the Leader within a randomised election timeout (typically 150–300 ms). On timeout a Follower increments the term, becomes a Candidate, votes for itself, and requests votes. A majority makes it Leader.

> **Why randomised timeouts?** With fixed timeouts, every node would time out simultaneously, all become candidates, split the vote, and repeat — potentially forever. Randomisation makes one node reliably time out first. This is the detail that shows you understand Raft rather than having read its summary.

**Log replication.** All writes go to the Leader, which appends to its log and replicates to Followers. Once a **majority** has persisted an entry, it's committed and applied to the state machine.

**Safety.** A candidate can only win if its log is at least as up-to-date as the voter's, which guarantees a new Leader never lacks a committed entry.

### Quorum and split brain

With `2f+1` nodes you tolerate `f` failures. Three nodes tolerate one; five tolerate two.

> **Why a majority specifically?** Any two majorities of the same set must intersect in at least one node, so the new leader's quorum necessarily includes someone who saw the last committed entry. That intersection property *is* the safety proof, and stating it that way is much better than "majority means more than half."
>
> **Split brain:** if a 5-node cluster partitions 3/2, only the 3-node side can form a majority, so the 2-node side cannot elect a leader and stops accepting writes. **Availability is sacrificed for consistency — Raft is CP in CAP terms.** For a job scheduler that's the correct trade: a scheduler that pauses is inconvenient, a scheduler that double-fires billing is a customer incident.

### How Chronos uses it

```go
type Scheduler struct {
    raft     *raft.Raft
    store    JobStore
    executor Executor
}

func (s *Scheduler) Run(ctx context.Context) error {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()

        case isLeader := <-s.raft.LeaderCh():
            slog.Info("leadership changed", "leader", isLeader)

        case <-ticker.C:
            // Only the leader decides what is due. Followers stay hot and ready.
            if s.raft.State() != raft.Leader {
                continue
            }
            if err := s.tick(ctx); err != nil {
                slog.Error("tick failed", "err", err)
            }
        }
    }
}

func (s *Scheduler) tick(ctx context.Context) error {
    due, err := s.store.DueJobs(ctx, time.Now())
    if err != nil {
        return fmt.Errorf("fetching due jobs: %w", err)
    }

    for _, job := range due {
        // Replicate the DECISION through Raft before executing.
        // If we crash mid-tick, the new leader replays from the committed log.
        cmd := Command{Type: CmdClaimJob, JobID: job.ID, RunID: uuid.NewString(),
                       ScheduledFor: job.NextRun}

        b, err := json.Marshal(cmd)
        if err != nil {
            return err
        }
        if err := s.raft.Apply(b, 5*time.Second).Error(); err != nil {
            return fmt.Errorf("replicating claim for %s: %w", job.ID, err)
        }

        s.executor.Submit(ctx, job, cmd.RunID)   // bounded worker pool, own recover
    }
    return nil
}
```

### The four follow-ups you should expect

**1. "Why not just use a database lock or Redis?"**

> A perfectly reasonable alternative and I'd say so. `SELECT ... FOR UPDATE SKIP LOCKED` on Postgres is simpler, well-understood, and correct — it's what I'd reach for in a system that already has a database. I used Raft because I wanted the scheduler to have no external dependency and to own its own replicated state, and because leader election with a proper consensus algorithm was the thing I actually wanted to learn. The honest trade-off: Raft means operating a stateful quorum-based cluster, which is meaningfully harder than adding a row lock.

That answer is strong because it shows you know the simpler option exists and made a deliberate choice, rather than reaching for consensus because it sounds impressive.

**2. "The leader claims a job, then crashes before executing it. What happens?"**

> The claim is committed to the Raft log, so the new leader sees it on replay. But it can't know whether execution *started*. This is the fundamental at-least-once vs at-most-once decision and it has no clever answer — you pick based on the job. Chronos defaults to at-least-once and requires job handlers to be idempotent via the `RunID`, because a job that occasionally runs twice is usually recoverable and a job that silently never runs is not. For jobs where duplicate execution is genuinely unacceptable, the handler writes a `(job_id, run_id)` row with a unique constraint before doing work, so the second attempt fails the insert and exits.

**3. "Clock skew — node A thinks the job is due, node B doesn't."**

> Only the leader evaluates schedules, so there's exactly one clock making the decision at any moment. That converts a distributed clock problem into a single-node one. It doesn't eliminate skew across *leadership changes* — a new leader with a clock 5 seconds behind might re-evaluate a job as not-yet-due — which is why the committed claim records `ScheduledFor` rather than "now": deduplication is against the *scheduled* time, not wall-clock. For anything tighter I'd need bounded clock uncertainty like Spanner's TrueTime, which isn't available without special hardware.

**4. "How do you test this?"**

> Three layers. Unit tests for the state machine — apply a log, assert the resulting state, since it's a deterministic function. Integration tests with an in-memory three-node cluster where I kill the leader mid-tick and assert exactly-once semantics hold. And `testing/synctest` for the timing-dependent paths, since it gives a deterministic fake clock — election timeouts and job deadlines become instant and reproducible instead of `time.Sleep` and hope. Plus `-race` on everything, always.

---

## 10. Distributed Locks & Fencing

```go
// Redis lock — SET NX PX is atomic
func (l *Lock) Acquire(ctx context.Context, key string, ttl time.Duration) (string, error) {
    token := uuid.NewString()                     // unique per acquisition
    ok, err := l.rdb.SetNX(ctx, key, token, ttl).Result()
    if err != nil {
        return "", err
    }
    if !ok {
        return "", ErrLockHeld
    }
    return token, nil
}

// Release MUST verify ownership atomically, or you release someone else's lock
var releaseScript = redis.NewScript(`
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
`)
```

> **Why the Lua script is mandatory:** a naive `GET`, compare, `DEL` sequence has a gap. If your lock expires between the `GET` and the `DEL`, and another process acquires it in that window, your `DEL` deletes *their* lock. This is exactly the check-then-act race from the Laravel inventory work, just distributed — same bug, same fix, which is to make the compare-and-delete atomic.

### The fundamental limitation — and the answer that impresses

> **A distributed lock with a TTL cannot guarantee mutual exclusion.** Not "is hard to" — cannot.
>
> The failure: process A acquires a 30-second lock and starts work. A GC pause, a CFS throttle, or a network partition stalls it for 35 seconds. The lock expires. Process B acquires it legitimately and starts work. Process A wakes up, has no idea time passed, and continues — **both are now in the critical section.** No TTL tuning fixes this, because you can't bound a pause.
>
> **The fix is fencing tokens.** The lock service issues a monotonically increasing token with each grant. Every write to the protected resource includes the token, and the resource **rejects any token lower than the highest it has seen**:
>
> ```
> A acquires, token 33.  A stalls.
> Lock expires. B acquires, token 34. B writes with 34.  → accepted, storage now at 34
> A wakes, writes with 33.                               → REJECTED, 33 < 34
> ```
>
> This moves enforcement to the resource, which is the only place that can actually order the writes. Redis alone can't provide this (`INCR` isn't safe across failover); ZooKeeper's `zxid`, etcd's revision numbers, or a database sequence can.
>
> **This is also the honest answer on Redlock.** Martin Kleppmann's critique is that Redlock relies on bounded clock drift and bounded pauses, neither of which holds in practice; Salvatore Sanfilippo's response is that it's fine for efficiency use cases. My position: use a distributed lock when duplicate work is merely *wasteful* (avoiding a duplicate cache rebuild), and use fencing tokens or a database constraint when duplicate work is *incorrect* (charging a customer). For Chronos, the unique constraint on `(job_id, run_id)` is the real safety mechanism; Raft leadership is an optimisation that avoids the wasted attempt.

That answer — naming the researchers, stating the failure precisely, and giving a decision rule — is a staff-level response to a very common senior question.

---

## 11. Idempotency & Delivery Semantics

> **"Exactly-once delivery is impossible in a distributed system."** Be able to say this and immediately explain why, because the follow-up is always "so what do you do?"
>
> The proof sketch: a sender transmits and waits for an acknowledgement. If no ack arrives, it cannot distinguish "the message was lost" from "the message arrived and the ack was lost." Retrying risks a duplicate; not retrying risks a loss. There is no third option, and adding more acknowledgements just moves the same problem one round trip later — it's the Two Generals Problem.
>
> **What you build instead is at-least-once delivery plus idempotent processing**, which together produce exactly-once *effects*. The distinction between delivery and effects is the crux, and stating it that way is what interviewers are listening for.

### Idempotency in practice

```go
func (s *Service) ProcessPayment(ctx context.Context, key string, req PaymentRequest) (*Payment, error) {
    // The unique constraint IS the concurrency control — not a SELECT-then-INSERT.
    var p Payment
    err := s.db.QueryRowContext(ctx, `
        INSERT INTO payments (idempotency_key, org_id, amount, status)
        VALUES ($1, $2, $3, 'pending')
        ON CONFLICT (idempotency_key) DO NOTHING
        RETURNING id, status, amount`,
        key, req.OrgID, req.Amount,
    ).Scan(&p.ID, &p.Status, &p.Amount)

    if errors.Is(err, sql.ErrNoRows) {
        // Conflict: someone already created it. Return the existing result.
        return s.getByKey(ctx, key)
    }
    if err != nil {
        return nil, fmt.Errorf("creating payment: %w", err)
    }

    return s.charge(ctx, &p)
}
```

> **The key insight, and it's the same one from the Laravel material:** the uniqueness guarantee must come from a **database constraint**, not from application logic. A `SELECT` to check existence followed by an `INSERT` is a check-then-act race — two concurrent requests both see nothing and both insert. `ON CONFLICT DO NOTHING` pushes the decision into the single point that can serialise it. Being able to say *"I use the constraint as the concurrency primitive rather than checking first"* connects your PHP concurrency experience directly to Go and shows the principle transferred, not just the syntax.

### The transactional outbox

The other classic: you need to update the database *and* publish an event, atomically. You can't have a transaction spanning Postgres and Kafka.

```go
// Inside ONE transaction: the state change and the intent to publish
func (r *Repo) CreateJobWithEvent(ctx context.Context, j *Job) error {
    return WithTx(ctx, r.db, func(tx *sql.Tx) error {
        if _, err := tx.ExecContext(ctx,
            `INSERT INTO jobs (id, org_id, name) VALUES ($1,$2,$3)`,
            j.ID, j.OrgID, j.Name); err != nil {
            return err
        }
        _, err := tx.ExecContext(ctx,
            `INSERT INTO outbox (id, topic, payload) VALUES ($1,$2,$3)`,
            uuid.NewString(), "job.created", mustJSON(j))
        return err
    })
}
```

A separate relay polls `outbox` (or reads the WAL via CDC) and publishes to Kafka, marking rows sent. If publishing fails it retries; if it publishes twice, consumers deduplicate on the event ID. **Atomicity comes from the single database transaction; delivery is at-least-once; consumers make it idempotent.** Same pattern you'd use in Laravel with `afterCommit`, but explicit.

---

## 12. Resilience Patterns

### Retry with exponential backoff and jitter

```go
func Retry(ctx context.Context, attempts int, base time.Duration, fn func() error) error {
    var err error
    for i := range attempts {
        if err = fn(); err == nil {
            return nil
        }

        var perm *PermanentError
        if errors.As(err, &perm) {
            return err                     // don't retry a 400 — it will never succeed
        }

        if i == attempts-1 {
            break
        }

        // Full jitter: random in [0, base*2^i). Prevents synchronised retry storms.
        backoff := time.Duration(rand.Int64N(int64(base) << i))

        select {
        case <-time.After(backoff):
        case <-ctx.Done():
            return ctx.Err()               // honour cancellation while sleeping
        }
    }
    return fmt.Errorf("after %d attempts: %w", attempts, err)
}
```

> **Jitter is the part people omit, and it's the part that matters.** Without it, every client that failed during an outage retries at exactly the same moments, so the recovering service is hit by synchronised waves and knocked over again. AWS's "Exponential Backoff and Jitter" article is the canonical reference and full jitter is generally the best-performing variant. Citing it is a good signal.
>
> **Also: only retry idempotent operations.** Retrying a non-idempotent POST after a timeout can double-charge a customer — you don't know whether the first attempt succeeded. This is where §11's idempotency keys become load-bearing rather than nice-to-have.

### Circuit breaker

Three states: **Closed** (normal), **Open** (fail fast without calling), **Half-Open** (allow a probe).

```go
type Breaker struct {
    mu          sync.Mutex
    state       State
    failures    int
    threshold   int
    openedAt    time.Time
    cooldown    time.Duration
}

func (b *Breaker) Do(fn func() error) error {
    if !b.allow() {
        return ErrCircuitOpen              // fail fast — no waiting on a dead service
    }
    err := fn()
    b.record(err)
    return err
}
```

> **What a breaker actually protects.** It's not primarily about the failing dependency — it's about *you*. Without one, every request to a dead downstream occupies a goroutine and a connection for the full timeout. At 1000 req/s with a 30-second timeout, that's 30,000 goroutines piling up holding connections, and your service dies of resource exhaustion caused by someone else's outage. The breaker converts a slow failure into a fast one, which is what stops the cascade. Framing it as protecting the caller rather than the callee is the more sophisticated answer.

### Timeouts and bulkheads

```go
// Every downstream call gets a budget, always shorter than the caller's
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()
```

**Timeout budgets must decrease as you go down the stack.** If the client has 5 seconds and your handler calls three services with 5-second timeouts each, you'll blow the budget while still waiting on the first. Allocate a share and pass the shrinking deadline via context — that propagation is exactly what `context.WithTimeout` on an inherited context gives you for free.

**Bulkhead:** isolate resource pools per dependency so one saturated downstream can't consume every worker. A separate bounded pool (or `errgroup.SetLimit`) per dependency.

---

## 13. Observability

### Structured logging with slog

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
        if a.Key == "password" || a.Key == "token" || a.Key == "authorization" {
            return slog.Attr{Key: a.Key, Value: slog.StringValue("[REDACTED]")}
        }
        return a
    },
}))
slog.SetDefault(logger)

slog.InfoContext(ctx, "job executed",
    slog.String("job_id", job.ID),
    slog.Int64("org_id", int64(job.OrgID)),
    slog.Duration("duration", elapsed),
)
```

`log/slog` (Go 1.21+) is the standard-library answer — use it rather than `logrus` or `zap` in new code unless you've measured that you need zap's allocation profile.

> **Use typed attributes (`slog.String`, `slog.Int64`) rather than loose key-value pairs.** The typed form avoids boxing into `any`, which per §4 forces a heap allocation per field. In a hot path that's a measurable difference, and knowing *why* connects logging back to escape analysis.

**Correlation IDs are what make logs usable at scale.** Put the request ID in the context at the edge (Tier 3 §4), attach it to every log line, and return it in the response so a support ticket maps to a log query.

### OpenTelemetry

```go
tracer := otel.Tracer("chronos/scheduler")

func (s *Scheduler) tick(ctx context.Context) error {
    ctx, span := tracer.Start(ctx, "scheduler.tick")
    defer span.End()

    span.SetAttributes(attribute.Int("jobs.due", len(due)))

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "tick failed")
    }
    return err
}
```

Because the span lives in the context, it propagates through every function that already takes `ctx` — which is why "context first parameter, always" pays off structurally rather than just stylistically.

### The metrics that matter

**RED for services:** Rate, Errors, Duration. **USE for resources:** Utilisation, Saturation, Errors.

Go-specific gauges worth exporting from day one:

```go
func collectRuntimeMetrics(db *sql.DB) {
    goroutines.Set(float64(runtime.NumGoroutine()))   // leak detection — the highest-value Go metric

    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    heapInuse.Set(float64(m.HeapInuse))
    heapObjects.Set(float64(m.HeapObjects))
    gcPauseTotal.Set(float64(m.PauseTotalNs) / 1e9)   // GC pause time
    gcCount.Set(float64(m.NumGC))                     // GC frequency

    s := db.Stats()
    dbInUse.Set(float64(s.InUse))                     // connections currently checked out
    dbWaitCount.Set(float64(s.WaitCount))             // ← pool saturation: see below
    dbWaitDuration.Set(s.WaitDuration.Seconds())
}
```

> **`WaitCount` from `db.Stats()` is the underrated one.** A non-zero and growing `WaitCount` means goroutines are queuing for a database connection — your pool is undersized or queries are too slow. It's the earliest warning of the failure mode that eventually presents as timeouts, and most people never export it.

**Always histograms, never averages.** An average latency of 50 ms is consistent with everyone getting 50 ms or with 95% getting 10 ms and 5% getting 850 ms. Only percentiles tell you which.

---

## 14. Deployment & Production

### Multi-stage Docker

```dockerfile
FROM golang:1.26-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                       # cached layer — deps change rarely
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=${VERSION}" \
    -trimpath -o /app ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Every line earns its place, and you should be able to justify each:
- **`go mod download` before `COPY . .`** — dependencies form a cached layer that survives source changes.
- **`CGO_ENABLED=0`** — genuinely static binary, works in distroless (Tier 1 §1).
- **`-ldflags="-s -w"`** — strips debug info and the symbol table, roughly 30% smaller.
- **`-trimpath`** — removes local filesystem paths, for reproducible builds and to avoid leaking your directory structure.
- **`-X main.version=...`** — inject the build version at link time; expose it on `/healthz`.
- **distroless `nonroot`** — no shell, no package manager, minimal CVE surface, non-root by default.

Result: ~15 MB image with essentially no attack surface.

### Kubernetes essentials

```yaml
resources:
  requests: { memory: "256Mi", cpu: "250m" }
  limits:   { memory: "512Mi", cpu: "1000m" }
env:
  - name: GOMEMLIMIT
    value: "460MiB"                       # ~90% of the memory limit
livenessProbe:                             # is the process wedged? restart it
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
readinessProbe:                            # can it serve traffic? remove from LB
  httpGet: { path: /readyz, port: 8080 }
  periodSeconds: 5
terminationGracePeriodSeconds: 30
```

> **Liveness and readiness must not be the same check, and this is a common interview probe.**
> - **Readiness** should include dependencies (can I reach the database?), because if you can't serve, you should leave the load balancer.
> - **Liveness** must **not** check dependencies. If it does, a brief database blip fails liveness on *every* pod simultaneously and Kubernetes restarts your entire fleet — turning a recoverable dependency outage into a full outage. Liveness answers only "is this process wedged?"
>
> That failure mode is real, memorable, and shows operational scar tissue.

### Graceful shutdown in Kubernetes — the ordering people get wrong

```go
<-ctx.Done()                              // SIGTERM

health.SetNotReady()                      // 1. fail readiness FIRST
time.Sleep(5 * time.Second)               // 2. wait for the LB to notice and stop routing
srv.Shutdown(shutdownCtx)                 // 3. now drain in-flight requests
workers.Wait()                            // 4. finish background work
db.Close()                                // 5. release resources last
```

> **Step 2 is the one everyone omits.** Kubernetes sends SIGTERM and removes the pod from Endpoints *concurrently*, and kube-proxy/ingress propagation takes a few seconds. If you shut down immediately you reject requests that were routed to you during that window — visible as a burst of 502s on every deploy. The deliberate sleep before draining is what makes deploys actually zero-downtime, and it connects nicely to your zero-downtime migration experience: same principle of sequencing changes so no window exists where a client can observe an inconsistent state.

### Configuration

```go
type Config struct {
    Addr        string        `env:"ADDR" envDefault:":8080"`
    DatabaseURL string        `env:"DATABASE_URL,required"`
    LogLevel    slog.Level    `env:"LOG_LEVEL" envDefault:"info"`
    Timeout     time.Duration `env:"TIMEOUT" envDefault:"30s"`
}
```

**Fail fast at startup on missing or invalid configuration.** A service that boots with a bad config and fails on the first request is far worse than one that refuses to start — the latter fails the deployment and rolls back automatically.

---

## 15. Tier 4 Q&A Drill

### Runtime and scheduler

**1. What are G, M and P?**
Goroutine (the work and its stack), Machine (an OS thread), Processor (a scheduling context with a run queue). An M must hold a P to run Go code, and there are `GOMAXPROCS` Ps.

**2. How does work stealing work?**
An idle P checks its local queue, then the global queue, then the netpoller, then steals half of a random P's queue. It also checks the global queue every 61st tick to prevent starvation.

**3. What happens on a blocking syscall?**
The P detaches from the blocked M and is handed to another M, so the remaining goroutines keep running. The original M reacquires a P when the syscall returns, or parks.

**4. Can Go run out of threads?**
Yes — the default limit is 10,000 and exceeding it is fatal. It happens with many concurrent blocking syscalls or cgo calls, not with network I/O.

**5. What is asynchronous preemption and why was it added?**
Since 1.14 the runtime signals (`SIGURG`) goroutines running longer than ~10 ms so they can be preempted at a safe point. Before it, a tight loop with no function calls could block GC indefinitely.

**6. How does the netpoller make blocking code non-blocking?**
On an unready socket the fd is registered with epoll/kqueue, the goroutine parks, and the M is released. The netpoller requeues the goroutine when the fd is ready. You write synchronous code and get event-driven performance.

**7. How big is a goroutine stack and how does it grow?**
2 KB initially, grown by allocating a larger stack and copying frames, with all pointers into the stack rewritten using compiler-emitted metadata.

**8. If stacks move, why don't pointers break?**
The runtime knows every frame's layout and rewrites pointers during the copy. Only possible because Go is GC'd and fully type-aware.

### Memory and GC

**9. What is escape analysis?**
A compile-time determination of whether a value can stay on the stack. If the compiler can't prove it stops being referenced at return, it goes on the heap.

**10. Does `make([]byte, n)` with a runtime `n` heap-allocate?**
Not necessarily — I measured 0 allocs/op for a non-escaping dynamic `make` in Go 1.26. The compiler emits a variable-size stack allocation with a runtime size check. Check with `-gcflags=-m` rather than assuming.

**11. What are the stack allocation size thresholds?**
128 KB for explicit variables (`var x T`), 64 KB for implicit ones (`new(T)`, `&T{}`, `make([]T, n)`).

**12. Why does `fmt.Println(x)` force a heap allocation?**
Boxing into `...any` plus reflection means the compiler can't prove `x` doesn't escape. It's why debug logging in hot paths is disproportionately costly.

**13. Can you determine allocations by reading the source?**
No. Escape analysis runs after inlining, so the same function can allocate standalone and not allocate when inlined. Measure with `-benchmem`.

**14. Describe Go's GC.**
Concurrent, tri-colour, mark-and-sweep, non-generational, non-compacting. Two sub-millisecond stop-the-world phases; marking and sweeping run alongside your code.

**15. What's the tri-colour invariant and why do you need write barriers?**
A black object must never point to a white one. Since the mutator runs during marking it can violate that, so a compiler-inserted write barrier greys objects on pointer writes.

**16. Why are pointer-heavy structures more expensive?**
More edges to trace and a write-barrier cost on every pointer write during marking. `[]Item` beats `[]*Item` for GC.

**17. What are mark assists?**
When a goroutine allocates faster than the GC marks, it's conscripted into marking proportional to its allocation. That's why allocation pressure shows up as latency, not OOM.

**18. Why isn't Go's GC generational?**
Escape analysis already keeps most short-lived objects on the stack, so the nursery's main benefit is largely captured. Combined with the non-compacting design, the complexity wasn't judged worthwhile.

**19. What's new about GC in Go 1.26?**
Green Tea GC is on by default (opt-in in 1.25). It marks span-at-a-time rather than object-at-a-time for better locality and SIMD-assisted scanning, cutting GC overhead 10–40% on pointer-heavy workloads.

**20. `GOGC` vs `GOMEMLIMIT`?**
`GOGC` is a growth percentage relative to live heap and knows nothing about container limits. `GOMEMLIMIT` is a soft total-memory ceiling that makes the GC more aggressive as you approach it.

**21. What problem does `GOMEMLIMIT` solve?**
Getting OOM-killed while having collectable garbage: a live-heap spike pushed the `GOGC` trigger above the cgroup limit and the kernel killed the process before the next GC.

**22. What's the risk of `GOMEMLIMIT`?**
It's soft. If the live heap genuinely exceeds it, the GC thrashes instead of OOMing — arguably worse, since the pod looks healthy while serving nothing. Leave headroom and alert on GC CPU fraction.

**23. What's the `GOMAXPROCS`-in-containers problem, and is it still a problem?**
Through Go 1.24 the runtime read the host CPU count and ignored cgroup limits, so a 500m pod on a 64-core node got `GOMAXPROCS=64` and suffered CFS throttling. **Since 1.25 the runtime is cgroup-aware**, so `automaxprocs` is no longer needed.

### Profiling and benchmarking

**24. Which pprof profile for a memory leak?**
Heap, and specifically `inuse_space` — `alloc_space` shows churn, not retention. Diff two profiles taken hours apart with `-base`.

**25. First thing to check when memory grows steadily?**
`runtime.NumGoroutine()`. A goroutine leak is more common than a heap leak and presents identically.

**26. What does a "leak" mean in a GC'd language?**
Something still holds a live reference: an unbounded cache map, an ever-growing slice, an oversized pooled buffer, or parked goroutines holding their stacks.

**27. Why `b.Loop()` over `for i := 0; i < b.N; i++`?**
It prevents dead-code elimination of the loop body and automatically excludes setup from timing, removing the need for a sink variable and `b.ResetTimer()`.

**28. Why `-count=10` and benchstat?**
A single benchmark run is noise. `benchstat` reports whether the difference is statistically meaningful.

**29. What's the limitation of microbenchmarks?**
They measure isolated function cost, not p99 under concurrent load with cold caches. Profile to find where time goes; benchmark only the thing you're changing.

### Distributed systems

**30. Why does Raft randomise election timeouts?**
Fixed timeouts make every node time out together, split the vote, and repeat. Randomisation ensures one node reliably goes first.

**31. Why a majority quorum specifically?**
Any two majorities of the same set intersect in at least one node, so a new leader's quorum always includes someone who saw the last committed entry. That intersection is the safety proof.

**32. What happens to a 5-node Raft cluster partitioned 3/2?**
Only the 3-node side can reach quorum. The 2-node side can't elect a leader and stops accepting writes — Raft chooses consistency over availability.

**33. Why Raft over a database lock for a scheduler?**
Both work. `SELECT ... FOR UPDATE SKIP LOCKED` is simpler and appropriate if you already have a database. Raft avoids an external dependency and gives you replicated state, at the cost of operating a stateful quorum cluster.

**34. Leader claims a job then dies. Duplicate execution?**
Possibly. The claim is in the committed log but you can't know whether execution began. Choose at-least-once with idempotent handlers, keyed on a run ID, rather than pretending exactly-once is achievable.

**35. Why can't a TTL-based distributed lock guarantee mutual exclusion?**
The holder can be paused (GC, CFS throttle, partition) beyond the TTL. The lock expires, another process acquires it, and the first resumes still believing it holds the lock. No TTL value fixes an unbounded pause.

**36. What's a fencing token?**
A monotonically increasing number issued with each lock grant, included in every write, with the resource rejecting tokens lower than the highest seen. It moves enforcement to the resource, the only place that can order the writes.

**37. What's your position on Redlock?**
Use a distributed lock when duplication is merely wasteful; use fencing tokens or a database constraint when duplication is incorrect. Kleppmann's critique — that it assumes bounded clock drift and bounded pauses — is right for correctness-critical use.

**38. Why is exactly-once delivery impossible?**
A sender that gets no ack can't distinguish a lost message from a lost ack. Retrying risks duplication, not retrying risks loss, and more acks just move the problem. It's the Two Generals Problem.

**39. So what do you build?**
At-least-once delivery plus idempotent processing, which yields exactly-once *effects*. The distinction between delivery and effects is the whole point.

**40. How do you implement an idempotency key correctly?**
A unique constraint with `INSERT ... ON CONFLICT DO NOTHING`, returning the existing record on conflict. Never `SELECT` then `INSERT` — that's a check-then-act race.

**41. What's the transactional outbox for?**
Atomically changing state and publishing an event when you can't have a transaction across the database and the broker. Write both to the database in one transaction; a relay publishes the outbox rows at-least-once.

### Resilience and production

**42. Why is jitter essential in retry backoff?**
Without it, all clients that failed during an outage retry in synchronised waves and re-topple the recovering service. Full jitter (random in `[0, base·2^i)`) generally performs best.

**43. What must you check before retrying?**
That the operation is idempotent and the error is transient. Retrying a non-idempotent POST after a timeout can double-charge, because you don't know whether the first attempt landed.

**44. What does a circuit breaker actually protect?**
Primarily the caller. Without it, each request to a dead downstream ties up a goroutine and a connection for the full timeout, and you exhaust your own resources because of someone else's outage.

**45. Why must timeout budgets shrink down the stack?**
Otherwise a handler with a 5-second client budget calling three 5-second dependencies blows the budget on the first. Allocate a share and propagate the shrinking deadline via context.

**46. Why use typed `slog` attributes?**
`slog.Any` boxes into an interface and forces a heap allocation per field; typed attributes avoid it. Same escape-analysis reasoning as §4.

**47. Which Go metrics do you always export?**
`NumGoroutine` (leaks), heap in-use, GC pause and frequency, and `db.Stats().WaitCount` — a growing wait count is the earliest sign of pool saturation and almost nobody exports it.

**48. Why must liveness not check dependencies?**
A database blip would fail liveness on every pod at once and Kubernetes would restart the whole fleet, turning a recoverable dependency outage into a total one. Readiness checks dependencies; liveness only asks whether the process is wedged.

**49. What's the missing step in most graceful shutdowns?**
Failing readiness and then *waiting* before draining. Kubernetes removes the pod from Endpoints concurrently with SIGTERM, so shutting down immediately rejects requests still being routed to you — the burst of 502s on every deploy.

**50. Justify three flags in your production Dockerfile.**
`CGO_ENABLED=0` for a genuinely static binary that runs in distroless; `-ldflags="-s -w"` to strip debug info for a ~30% smaller image; `-trimpath` for reproducible builds that don't leak local paths.

---

> **The Tier 4 senior signal:** every answer here should carry evidence. Not "Go has a fast GC" but "two sub-millisecond stop-the-world phases, and mark assists mean allocation pressure shows up as latency." Not "I'd use pprof" but "I'd diff two heap profiles on `inuse_space` after ruling out a goroutine leak." Not "we used Raft" but "Raft is CP, so a minority partition stops scheduling — which is the right trade for a biller." Specificity is the whole game at this tier, and the fastest way to sound specific is to have actually measured something. You have: use the numbers.

---

**Next:** [`05-question-bank.md`](./05-question-bank.md) — the full drill: 200+ questions by tier, "what does this print" puzzles, live-coding exercises, debugging scenarios, system design prompts, and STAR stories.

**Back to:** [`README.md`](./README.md) · [`01-basic.md`](./01-basic.md) · [`02-concurrency.md`](./02-concurrency.md) · [`03-gin-web.md`](./03-gin-web.md)
