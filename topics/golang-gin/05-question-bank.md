# Go / Gin — Question Bank & Drill Material

> **Every "what does this print" answer in §1 was produced by actually running the code on Go 1.26.** They are not recalled or reasoned-out — they're captured program output. If one surprises you, that's the point.
>
> Use this file for active recall. Cover the answer, say yours out loud, then compare. Vagueness is what fails senior loops, so grade yourself on precision, not on whether you were "basically right."

---

## Table of Contents

1. [What Does This Print? — 25 Verified Puzzles](#1-what-does-this-print--25-verified-puzzles)
2. [Rapid-Fire: Language](#2-rapid-fire-language)
3. [Rapid-Fire: Concurrency](#3-rapid-fire-concurrency)
4. [Rapid-Fire: Web, Gin & Data](#4-rapid-fire-web-gin--data)
5. [Rapid-Fire: Runtime & Distributed](#5-rapid-fire-runtime--distributed)
6. [Live-Coding Exercises](#6-live-coding-exercises)
7. [Debugging Scenarios](#7-debugging-scenarios)
8. [System Design Prompts](#8-system-design-prompts)
9. [STAR Stories](#9-star-stories)
10. [Questions to Ask Them](#10-questions-to-ask-them)
11. [Final Week Checklist](#11-final-week-checklist)

---

## 1. What Does This Print? — 25 Verified Puzzles

### P1 — Slice aliasing

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]
b = append(b, 99)
fmt.Println(a, b, len(b), cap(b))
```

<details><summary>Answer</summary>

```
[1 2 3 99 5] [2 3 99] 3 4
```

`b` has len 2 but cap 4 (from index 1 to the end of the backing array). `append` fits, so it writes into `a[3]`, overwriting the 4. `cap(b)` is 4, not 5, because capacity is measured from the slice's start offset.
</details>

### P2 — Two appends from the same base

```go
s1 := make([]int, 0, 2)
s1 = append(s1, 1)
s2 := append(s1, 2)
s3 := append(s1, 3)
fmt.Println(s1, s2, s3)
```

<details><summary>Answer</summary>

```
[1] [1 3] [1 3]
```

**This is the nastiest slice puzzle there is.** `s1` has len 1, cap 2. Both appends write to index 1 of the *same* backing array, so `s3`'s write clobbers `s2`'s. `s2` and `s3` are separate headers pointing at identical memory. `s1` still prints `[1]` because its length is still 1.

The lesson: `append` returning a value does not mean it didn't mutate shared state.
</details>

### P3 — Map of pointers

```go
type P struct{ X int }
m := map[string]*P{"a": {1}}
m["a"].X = 99
fmt.Println(m["a"].X)
```

<details><summary>Answer</summary>

```
99
```

Legal because the map value is a *pointer* — we're not assigning to a map element, we're dereferencing one. With `map[string]P` this would be a compile error, since map elements aren't addressable.
</details>

### P4 — Method sets on an addressable value

```go
type T struct{ n int }
func (t T) ValInc()  { t.n++ }
func (t *T) PtrInc() { t.n++ }

t := T{n: 1}
t.ValInc()
fmt.Print(t.n, " ")
t.PtrInc()
fmt.Println(t.n)
```

<details><summary>Answer</summary>

```
1 2
```

`ValInc` operates on a copy, so nothing changes. `PtrInc` works on a value because `t` is addressable — the compiler rewrites it to `(&t).PtrInc()`. That auto-addressing is exactly what does *not* happen when boxing into an interface.
</details>

### P5 — defer argument evaluation

```go
func main() {
    i := 0
    defer fmt.Println("deferred i =", i)
    i = 42
    fmt.Println("immediate i =", i)
}
```

<details><summary>Answer</summary>

```
immediate i = 42
deferred i = 0
```

Deferred *arguments* are evaluated when the `defer` statement runs, not when the call executes. To capture the final value, defer a closure instead.
</details>

### P6 — The nil interface

```go
type MyErr struct{}
func (e *MyErr) Error() string { return "my error" }

func f() error {
    var e *MyErr
    return e
}

err := f()
fmt.Println(err == nil, err)
```

<details><summary>Answer</summary>

```
false my error
```

The interface holds type `*MyErr` with a nil data pointer, so it isn't nil.

The *printed* part is the subtle bit, and it's worth knowing because it changes how the bug presents. `Error()` here never dereferences `e`, so it returns normally and prints `my error`. In the Tier 1 version the method reads `e.Code`, which panics on a nil receiver — and `fmt` **recovers that panic internally and substitutes `<nil>`**. Verified:

```
A (no dereference):   == nil: false | print: my error
B (dereferences):     == nil: false | print: <nil>
direct call on B:     panic: invalid memory address or nil pointer dereference
```

So the same bug either prints a plausible error message or prints `<nil>` — and the `<nil>` case is far more dangerous, because your logs claim nothing is wrong while the code takes the error branch. The invariant holds in both: the interface is never nil.
</details>

### P7 — defer order

```go
for j := 0; j < 3; j++ {
    defer fmt.Print(j, " ")
}
```

<details><summary>Answer</summary>

```
2 1 0
```

LIFO, and each captured its `j` at defer time.
</details>

### P8 — Named vs plain return with defer

```go
func a() (result int) {
    defer func() { result *= 2 }()
    return 5
}

func b() int {
    x := 5
    defer func() { x *= 2 }()
    return x
}

fmt.Println(a(), b())
```

<details><summary>Answer</summary>

```
10 5
```

In `a`, `return 5` assigns to the named result, then the defer doubles it. In `b`, the value is copied to the return slot first, so doubling the local has no effect.
</details>

### P9 — Loop variable capture (Go 1.22+)

```go
fns := make([]func() int, 3)
for k := range 3 {
    fns[k] = func() int { return k }
}
fmt.Println(fns[0](), fns[1](), fns[2]())
```

<details><summary>Answer</summary>

```
0 1 2
```

Since Go 1.22 each iteration gets a fresh `k`. **Before 1.22 this printed `3 3 3`** and needed `k := k`. Always state both the current answer and the historical one — the interviewer may be testing the old behaviour.
</details>

### P10 — String length and iteration

```go
str := "日本語"
fmt.Println(len(str), len([]rune(str)))
for idx := range str {
    fmt.Print(idx, " ")
}
```

<details><summary>Answer</summary>

```
9 3
0 3 6
```

Three characters, three bytes each in UTF-8. `range` yields byte offsets, so the indices jump by 3.
</details>

### P11 — Type assertion on a value receiver

```go
type Animal interface{ Speak() string }
type Dog struct{}
func (d Dog) Speak() string { return "woof" }

var an Animal = Dog{}
_, isDog := an.(Dog)
_, isPtr := an.(*Dog)
fmt.Println(isDog, isPtr)
```

<details><summary>Answer</summary>

```
true false
```

The interface holds a `Dog` value, not a pointer. Type assertions match the **exact dynamic type** — `*Dog` is a different type, even though `*Dog`'s method set includes `Speak`.
</details>

### P12 — nil slice vs nil map

```go
var ns []int
var nm map[string]int
fmt.Println(len(ns), len(nm), ns == nil, nm == nil)
ns = append(ns, 1)
fmt.Println(ns)
```

<details><summary>Answer</summary>

```
0 0 true true
[1]
```

`append` on a nil slice works and allocates. Writing to a nil map would panic — that's the asymmetry.
</details>

### P13 — Arrays are values

```go
arr := [3]int{1, 2, 3}
arr2 := arr
arr2[0] = 99
fmt.Println(arr, arr2, arr == [3]int{1, 2, 3})
```

<details><summary>Answer</summary>

```
[1 2 3] [99 2 3] true
```

Assignment copies the whole array, and arrays of comparable elements are comparable.
</details>

### P14 — Embedding has no virtual dispatch

```go
type Base struct{}
func (b Base) Who() string  { return "Base" }
func (b Base) Call() string { return "called by " + b.Who() }

type Derived struct{ Base }
func (d Derived) Who() string { return "Derived" }

d := Derived{}
fmt.Println(d.Who(), "|", d.Call())
```

<details><summary>Answer</summary>

```
Derived | called by Base
```

**This is the best puzzle for proving embedding isn't inheritance.** `d.Who()` resolves to `Derived`'s method by shallower-depth promotion. But `Call()` is promoted from `Base`, so inside it the receiver is a `Base` — which knows nothing about `Derived` and calls `Base.Who()`.

In Java or PHP this would print "called by Derived" via virtual dispatch. Go has none. If you want that behaviour you need an interface and explicit delegation.
</details>

### P15 — errors.Is through wrapping

```go
base := errors.New("base")
wrapped := fmt.Errorf("layer1: %w", fmt.Errorf("layer2: %w", base))
fmt.Println(errors.Is(wrapped, base), wrapped)

notWrapped := fmt.Errorf("layer1: %v", base)
fmt.Println(errors.Is(notWrapped, base))
```

<details><summary>Answer</summary>

```
true layer1: layer2: base
false
```

`%w` preserves the chain through any depth. `%v` formats the text identically but severs it, so `errors.Is` fails — a silent bug, since the message looks right.
</details>

### P16 — select on a closed channel

```go
c := make(chan int)
close(c)
n := 0
for range 3 {
    select {
    case <-c:
        n++
    }
}
fmt.Println("fired", n, "times")
```

<details><summary>Answer</summary>

```
fired 3 times
```

Instantly. A closed channel is **always ready**, so it never blocks. This is why a closed channel left in a `select` inside a loop spins at 100% CPU — set the variable to nil to disable the case.
</details>

### P17 — Draining a closed buffered channel

```go
bc := make(chan int, 3)
bc <- 1
bc <- 2
close(bc)
total := 0
for v := range bc {
    total += v
}
fmt.Println(total)
```

<details><summary>Answer</summary>

```
3
```

Closing doesn't discard buffered values. `range` drains 1 and 2, then sees the channel closed and empty and terminates.
</details>

### P18 — Concurrent writes to distinct slice indices

```go
res := make([]int, 5)
var wg sync.WaitGroup
for k := range 5 {
    wg.Go(func() { res[k] = k * k })
}
wg.Wait()
fmt.Println(res)
```

<details><summary>Answer</summary>

```
[0 1 4 9 16]
```

**Race-detector clean, and correct.** Each goroutine writes a distinct index, so there's no overlapping memory access. The slice header is only read. This is the standard way to collect results without a mutex — and knowing *why* it's safe matters, because the same code with `append` instead of index assignment would be a genuine race.
</details>

### P19 — time.Time comparison

```go
n1 := time.Now()
n2 := n1.Round(0)                  // strips the monotonic reading
fmt.Println(n1 == n2, n1.Equal(n2))

utc := time.Date(2026, 7, 26, 12, 0, 0, 0, time.UTC)
other := utc.In(dhaka)             // same instant, different location
fmt.Println(utc == other, utc.Equal(other))
```

<details><summary>Answer</summary>

```
false true
false true
```

`==` compares the struct, which includes the wall clock, the **monotonic reading**, and the `*Location` pointer. Two values representing the same instant compare unequal if either differs — which also means a `time.Time` that has been JSON round-tripped never `==` the original. **Always use `Equal`.**
</details>

### P20 — Shadowing

```go
x := 1
{
    x := 2
    _ = x
}
if x := 3; x > 0 {
    _ = x
}
fmt.Println(x)
```

<details><summary>Answer</summary>

```
1
```

Both inner `x` declarations are new variables in new scopes. The `if` statement's initialiser scopes to the `if`. This is why accidental shadowing (`err :=` inside a block instead of `err =`) silently discards errors — the classic version of this bug.
</details>

### P21 — Three-index slicing

```go
orig := make([]int, 10, 20)
sub := orig[2:5]
sub3 := orig[2:5:6]
fmt.Println(len(sub), cap(sub), "|", len(sub3), cap(sub3))
```

<details><summary>Answer</summary>

```
3 18 | 3 4
```

`cap(sub)` is 18 = 20 − 2 (start offset to end of array). The three-index form sets cap to `max − low` = 6 − 2 = 4.
</details>

### P22 — copy truncates

```go
src := []int{1, 2, 3}
dst := make([]int, 2)
n := copy(dst, src)
fmt.Println(n, dst)
```

<details><summary>Answer</summary>

```
2 [1 2]
```

`copy` moves `min(len(dst), len(src))` elements and returns the count. It never grows the destination — a common source of silent truncation when someone forgets to size `dst`.
</details>

### P23 — Struct comparability

```go
type Q struct{ A, B int }
fmt.Println(Q{1, 2} == Q{1, 2})
```

<details><summary>Answer</summary>

```
true
```

Field-wise comparison. Add a slice, map, or func field and it becomes a compile error — and the struct can no longer be a map key.
</details>

### P24 — Interface nil check on a nil map value

```go
var m map[string]int
v, ok := m["missing"]
fmt.Println(v, ok, len(m))
```

<details><summary>Answer</summary>

```
0 false 0
```

Reading from a nil map is safe and returns the zero value. Only writing panics.
</details>

### P25 — Goroutine with no synchronisation

```go
func main() {
    go fmt.Println("hello")
}
```

<details><summary>Answer</summary>

Usually prints **nothing**. `main` returns immediately and the process exits, killing the goroutine before it's scheduled. Occasionally it may print if the scheduler happens to run it first — which makes it worse than deterministically broken, because it'll work on your laptop and fail in CI.
</details>

---

## 2. Rapid-Fire: Language

Cover the right column. Aim for under 20 seconds each.

| # | Question | Answer |
|---|---|---|
| 1 | Three fields of a slice header? | Pointer to backing array, len, cap — 24 bytes on 64-bit |
| 2 | Why can't a function change a caller's slice length? | The header is passed by value; only the pointed-to array is shared |
| 3 | How do you stop `append` mutating a shared array? | Three-index slicing `s[a:b:b]`, or `slices.Clone` for full independence |
| 4 | Slice growth pattern? | Doubling under 256 elements, ~1.25× above, rounded to a size class |
| 5 | nil vs empty slice? | Same for len/range/append; differ for `== nil` and JSON (`null` vs `[]`) |
| 6 | Why randomised map iteration? | So code can't depend on order and break on a runtime upgrade |
| 7 | Why aren't map elements addressable? | Rehashing moves them, so any pointer could dangle |
| 8 | Concurrent map write? | `fatal error` — a runtime throw, unrecoverable. Reads alone are safe |
| 9 | When is `sync.Map` right? | Write-once-read-many, or disjoint key sets. Otherwise `map` + `RWMutex` |
| 10 | `len("héllo")`? | 6 — bytes, not characters |
| 11 | What does `range` over a string yield? | Runes, indexed by **byte** offset |
| 12 | Why is `s += x` in a loop bad? | Strings are immutable → O(n²) copying. Use `strings.Builder` |
| 13 | Is embedding inheritance? | No — composition with promotion. No subtyping, no virtual dispatch |
| 14 | Method set of `T` vs `*T`? | `*T` has all methods; `T` has only value-receiver methods |
| 15 | Why does `t.PtrInc()` work on a value? | `t` is addressable, so the compiler inserts `&t` |
| 16 | Why doesn't that work for interfaces? | Interfaces hold a copy; auto-addressing would mutate a temporary |
| 17 | Name three non-addressable things | Map elements, type-assertion results, composite literals |
| 18 | When are deferred args evaluated? | At the `defer` statement, not at execution |
| 19 | Defer order? | LIFO, at *function* return — not end of block |
| 20 | Why is `defer f.Close()` in a loop a bug? | Handles accumulate until the function returns. Wrap in a closure |
| 21 | Named return + defer? | The defer can modify the result. Unnamed returns are copied first |
| 22 | What changed in 1.22 for loops? | Per-iteration loop variables; fixed the `3 3 3` closure bug |
| 23 | Where must `recover` be? | Directly in a deferred function of the panicking goroutine |
| 24 | Goroutine panics, nothing recovers? | The whole process dies. `main`'s recover doesn't help |
| 25 | How does a type implement an interface? | Implicitly, by having the methods |
| 26 | "Accept interfaces, return structs" — why? | Callers can substitute inputs and get the full concrete API back |
| 27 | What's interface pollution? | Single-implementation interfaces next to their implementation. Define at consumption |
| 28 | Why is a typed nil in an `error` not nil? | The interface's type word is set, so only the data word is nil |
| 29 | `errors.Is` vs `As`? | `Is` = identity vs a sentinel; `As` = find a type in the chain and bind it |
| 30 | `%w` vs `%v`? | `%w` keeps the chain traversable; `%v` severs it silently |
| 31 | Should you log and return an error? | No — duplicate logs at every level. Do one or the other |
| 32 | What does `~int` mean? | Any type with underlying type `int`, e.g. `type Celsius int` |
| 33 | Can methods have type parameters? | No. Functions and types only |
| 34 | Are generics faster than `any`? | Not reliably — GC-shape stenciling shares instantiations. Benchmark |
| 35 | When not to use generics? | Until you've written the same function three times |
| 36 | `type A int` vs `type A = int`? | New distinct type vs a pure alias |
| 37 | Why does Go have no enums? | `iota` constants are just ints — no exhaustiveness check. Use the `exhaustive` linter |
| 38 | What's MVS? | Minimal Version Selection — pick the minimum satisfying everyone, not the newest |
| 39 | Why must `v2+` change the import path? | Semantic import versioning lets v1 and v2 coexist in one build |
| 40 | Four things `go vet` catches? | printf mismatches, copylocks, lostcancel, malformed struct tags |

---

## 3. Rapid-Fire: Concurrency

| # | Question | Answer |
|---|---|---|
| 1 | Goroutine initial stack? | 2 KB, grown by copying |
| 2 | How do you kill a goroutine? | You can't — signal it and it returns voluntarily |
| 3 | Send on closed channel? | Panic |
| 4 | Receive on closed channel? | Buffered values first, then zero value with `ok == false` |
| 5 | Send/receive on nil channel? | Blocks forever — useful to disable a `select` case |
| 6 | Close a nil or closed channel? | Panic |
| 7 | Who closes a channel? | The sender. With N senders, a coordinator after `wg.Wait()` |
| 8 | Why close rather than send a shutdown value? | Close wakes *all* receivers; a send wakes one |
| 9 | Unbuffered vs buffered? | Unbuffered is a rendezvous with a two-way happens-before edge |
| 10 | Should you buffer for speed? | No — it hides backpressure and queues stale work |
| 11 | Two ready `select` cases? | Uniformly random, to prevent starvation |
| 12 | What does `default` do? | Makes the select non-blocking |
| 13 | Why would a select busy-loop? | A closed channel is always ready. Nil the variable to disable it |
| 14 | `ctx.Done()` or `time.After`? | `ctx.Done()` — the deadline propagates downstream |
| 15 | Is `sync.Mutex` reentrant? | No. Double-lock deadlocks |
| 16 | Why never copy a mutex? | Two independent locks guarding nothing. `go vet` copylocks |
| 17 | Is `RWMutex` always better for reads? | No — higher uncontended cost. Wins only for long, frequent reads |
| 18 | Writer starvation in Go's `RWMutex`? | New readers block once a writer waits |
| 19 | Why must `wg.Add` precede the goroutine? | Otherwise it races with `Wait`, which may see zero |
| 20 | What's `wg.Go`? | Go 1.25+ — `Add(1)` + `go` + `defer Done()` in one call |
| 21 | `sync.Once` when the function fails? | Never runs again. The failure is cached forever |
| 22 | When to use `sync.Pool`? | Measured allocation pressure only. Never for connections — cleared at every GC |
| 23 | What must you do with a pooled object? | Reset it — you get it back dirty |
| 24 | Atomics or mutex? | Atomics guard a variable; mutexes guard an invariant across variables |
| 25 | Use for `atomic.Pointer`? | Lock-free copy-on-write config; readers never block |
| 26 | Why is `defer cancel()` mandatory? | Otherwise the timer and child context leak. `go vet` lostcancel |
| 27 | Does cancelling stop the work? | No — it closes a channel. Entirely cooperative |
| 28 | Why unexported struct context keys? | String keys collide silently across packages |
| 29 | Why avoid `context.Value`? | Untyped, invisible in the signature — it's service location |
| 30 | Where does a context live? | First parameter. Never a struct field |
| 31 | Why can `for !done {}` never exit? | No happens-before edge — the compiler may hoist the read to a register |
| 32 | Four happens-before edges? | send→receive, Unlock→Lock, close→receive, `once.Do` completing |
| 33 | Does goroutine exit create an edge? | No. You need a WaitGroup or channel |
| 34 | Clean `-race` = race-free? | No. It only reports races it observed at runtime |
| 35 | Race detector overhead? | ~10× CPU, ~5× memory. CI/staging, or a single canary |
| 36 | Data race vs logic race? | Data = unsynchronised memory access; logic = check-then-act, invisible to `-race` |
| 37 | Four goroutine leak causes? | Blocked send, blocked receive, ignored cancellation, unstopped ticker |
| 38 | Worker times out — one-char fix? | Buffer the result channel with capacity 1 so the send always completes |
| 39 | Find a leak in production? | Alert on rising `NumGoroutine`, then `/debug/pprof/goroutine?debug=2` |
| 40 | Why won't the deadlock detector help? | It needs *all* goroutines asleep; a server always has an accept loop |
| 41 | Prevent lock-order deadlocks? | Global acquisition order; sort by a stable key for dynamic sets |
| 42 | `errgroup` over `WaitGroup`? | Error propagation plus sibling cancellation on first failure |
| 43 | `errgroup.Wait` limitation? | Returns only the first error |
| 44 | What does `singleflight` solve? | Cache stampede — in-process only, so N replicas still make N calls |
| 45 | Buffered channel as semaphore? | Buffer size = concurrency limit; acquire is a send |
| 46 | Bug in passing the signal ctx to `Shutdown`? | It's already cancelled — shutdown aborts instead of draining |
| 47 | Testing a 1-hour timeout instantly? | `testing/synctest` fake clock, stable in Go 1.25 |
| 48 | When is a mutex better than a channel? | Guarding state, rather than transferring ownership |

---

## 4. Rapid-Fire: Web, Gin & Data

| # | Question | Answer |
|---|---|---|
| 1 | What is `http.Handler`? | One method: `ServeHTTP(ResponseWriter, *Request)`. Everything implements it |
| 2 | What does `HandlerFunc` do? | Adapter — gives a function type a `ServeHTTP` method |
| 3 | What's wrong with `http.ListenAndServe`? | No timeouts at all → Slowloris and FD exhaustion |
| 4 | Four server timeouts? | ReadHeader (Slowloris), Read (body), Write (slow clients), Idle (keep-alive) |
| 5 | Why does `WriteTimeout` break SSE? | Absolute deadline from request start — cuts long responses off |
| 6 | Trap with `http.DefaultClient`? | No timeout, and `MaxIdleConnsPerHost` defaults to 2 |
| 7 | Why drain a response body? | An undrained body means the connection can't be reused |
| 8 | Header set after `WriteHeader`? | Silently ignored |
| 9 | What changed in 1.22 routing? | Method matching and wildcards in `ServeMux` + `r.PathValue` |
| 10 | Gin's router data structure? | Radix tree per method — O(path length), no allocations |
| 11 | When does Gin panic at boot? | Conflicting routes, e.g. `/j/:id` and `/j/:name` |
| 12 | Why `c.Copy()` before a goroutine? | Contexts are pooled and reused across requests |
| 13 | Is `c.Copy()` sufficient? | No — the request context is dead and a panic kills the process. Enqueue instead |
| 14 | Which context goes to your service? | `c.Request.Context()`, never `*gin.Context` |
| 15 | `Bind` vs `ShouldBind`? | `Bind` writes its own 400; `ShouldBind` returns the error. Use ShouldBind |
| 16 | Why can't you bind JSON twice? | The body is a consumed stream. Use `ShouldBindBodyWith` |
| 17 | Does `c.Abort()` return? | No — it only stops *later* handlers. You still need `return` |
| 18 | Capture the path before or after `Next()`? | Before — downstream code can rewrite it |
| 19 | Absent vs zero in a PATCH? | Pointer fields: nil = absent, non-nil zero = explicitly set |
| 20 | Is `*sql.DB` a connection? | No, a pool. One per process, safe for concurrent use |
| 21 | Does `sql.Open` connect? | No, lazy. `PingContext` forces it |
| 22 | How do you size `MaxOpenConns`? | From the DB's `max_connections` split across clients, sanity-checked with Little's Law |
| 23 | Why `MaxIdleConns == MaxOpenConns`? | Default 2 means repeated TCP/TLS setup under exactly peak load |
| 24 | Why `ConnMaxLifetime`? | Recycles connections after a failover or rebalance |
| 25 | Two mandatory lines with `Query`? | `defer rows.Close()` and `return rows.Err()` |
| 26 | Why `QueryContext`? | Otherwise a client disconnect leaves the query running |
| 27 | Order by a user-supplied column? | Allowlist map — identifiers can't be parameterised |
| 28 | Why `defer tx.Rollback()` when you commit? | It's a no-op post-commit and covers every early return and panic |
| 29 | Concurrent use of `*sql.Tx`? | Not allowed — pinned to one connection |
| 30 | Why does the tx helper need a named return? | The defer must inspect `err` to decide on rollback |
| 31 | Would you use GORM? | Usually not — hidden SQL, N+1 via preloading, reflection cost. Prefer `sqlc` |
| 32 | Why tenant-scope cache keys? | Otherwise a hit serves another tenant's data — only visible on a hit |
| 33 | What's TTL jitter for? | Stops keys warmed together from expiring together and stampeding |
| 34 | Should Redis being down fail the request? | No — fall through to the origin. Degrade to slow, not broken |
| 35 | gRPC's four call types? | Unary, server stream, client stream, bidirectional |
| 36 | gRPC load-balancing problem? | One long-lived HTTP/2 connection pins you to a backend under an L4 LB |
| 37 | gRPC or REST? | gRPC internally where you own both ends; REST at the public edge |
| 38 | Why does `httptest.NewRecorder` need no server? | The router is an `http.Handler` — call `ServeHTTP` directly |
| 39 | Fakes or generated mocks? | Fakes — Go interfaces are small, so they're a few lines and read better |
| 40 | The one test every multi-tenant repo method needs? | Two seeded orgs, assert only the requester's rows return |

---

## 5. Rapid-Fire: Runtime & Distributed

| # | Question | Answer |
|---|---|---|
| 1 | G, M, P? | Goroutine, OS thread (Machine), scheduling context. An M needs a P to run Go |
| 2 | Work stealing order? | Local queue → global (also every 61st tick) → netpoller → steal half from a random P |
| 3 | Blocking syscall? | The P detaches and moves to another M; the blocked M keeps only its goroutine |
| 4 | Can Go run out of threads? | Yes — 10,000 default limit, fatal. Via cgo or blocking file I/O, not network I/O |
| 5 | Asynchronous preemption? | SIGURG after ~10 ms (1.14+). Before it, tight loops could block GC forever |
| 6 | How does the netpoller work? | Register fd with epoll, park the goroutine, release the M, requeue when ready |
| 7 | Why is Go good for network services? | Synchronous-looking code with event-driven I/O and no async colouring |
| 8 | Stack growth? | Copy to a bigger stack and rewrite every pointer using frame metadata |
| 9 | What is escape analysis? | Compile-time proof of whether a value outlives its frame |
| 10 | Does `make([]byte, n)` heap-allocate? | Not necessarily — measured 0 allocs in 1.26. Check `-gcflags=-m` |
| 11 | Stack size thresholds? | 128 KB explicit vars, 64 KB implicit (`new`, `&T{}`, `make`) |
| 12 | Why does `fmt.Println(x)` allocate? | `...any` boxing plus reflection — the compiler can't prove non-escape |
| 13 | Can you tell allocations from source? | No — escape analysis runs after inlining. Measure with `-benchmem` |
| 14 | Describe Go's GC | Concurrent tri-colour mark-and-sweep, non-generational, non-compacting |
| 15 | Tri-colour invariant? | Black must never point to white — hence write barriers during marking |
| 16 | Why are pointer-heavy structures costly? | More edges to trace plus a write-barrier cost per pointer write |
| 17 | Mark assists? | Allocating goroutines are conscripted into marking — pressure becomes latency |
| 18 | Why not generational? | Escape analysis already stack-allocates most short-lived objects |
| 19 | GC news in 1.26? | Green Tea on by default — span-at-a-time marking, 10–40% less GC overhead |
| 20 | `GOGC` vs `GOMEMLIMIT`? | Growth % of live heap vs a soft total-memory ceiling |
| 21 | What does `GOMEMLIMIT` fix? | OOM-kill while holding collectable garbage in a container |
| 22 | Risk of `GOMEMLIMIT`? | Soft — if live heap exceeds it, the GC thrashes instead of OOMing |
| 23 | `GOMAXPROCS` in containers? | Was host-CPU-count and needed `automaxprocs`; **cgroup-aware since 1.25** |
| 24 | Profile for a memory leak? | Heap `inuse_space`, diffed with `-base` across two snapshots |
| 25 | First check when memory grows? | `NumGoroutine` — a goroutine leak looks identical and is more common |
| 26 | What's a "leak" in a GC'd language? | A live reference: unbounded map, growing slice, oversized pool, parked goroutines |
| 27 | Why `b.Loop()`? | Prevents dead-code elimination and excludes setup from timing (1.24+) |
| 28 | Why `-count=10` + benchstat? | A single run is noise; benchstat tests significance |
| 29 | Why randomised Raft election timeouts? | Fixed timeouts cause simultaneous candidacy and split votes forever |
| 30 | Why a majority quorum? | Any two majorities intersect, so a new leader saw the last committed entry |
| 31 | 5-node cluster split 3/2? | Only the 3-side has quorum; the 2-side stops writing. Raft is CP |
| 32 | Raft or a DB lock for a scheduler? | Both valid. `FOR UPDATE SKIP LOCKED` is simpler; Raft avoids a dependency |
| 33 | Leader claims a job then dies? | Can't know if execution started → at-least-once with idempotent handlers |
| 34 | Why can't a TTL lock guarantee exclusion? | An unbounded pause outlives the TTL; two holders result |
| 35 | Fencing token? | Monotonic number per grant; the resource rejects anything lower than its max |
| 36 | Position on Redlock? | Fine for efficiency; use fencing or a DB constraint for correctness |
| 37 | Why is exactly-once delivery impossible? | A missing ack is indistinguishable from a lost message — Two Generals |
| 38 | So what do you build? | At-least-once delivery + idempotent processing = exactly-once *effects* |
| 39 | Idempotency key implementation? | Unique constraint + `ON CONFLICT DO NOTHING`. Never SELECT-then-INSERT |
| 40 | Transactional outbox? | State change and event row in one transaction; a relay publishes at-least-once |
| 41 | Why jitter in backoff? | Prevents synchronised retry waves re-toppling a recovering service |
| 42 | What must be true before retrying? | The operation is idempotent and the error is transient |
| 43 | What does a circuit breaker protect? | Primarily *you* — it stops goroutines and connections piling up on a dead dependency |
| 44 | Why must timeout budgets shrink? | Otherwise three 5s dependencies blow a 5s client budget on the first call |
| 45 | Why typed `slog` attributes? | `slog.Any` boxes and heap-allocates per field |
| 46 | Metrics you always export? | `NumGoroutine`, heap in-use, GC pause/count, `db.Stats().WaitCount` |
| 47 | Why must liveness skip dependencies? | A DB blip would fail liveness fleet-wide and restart everything |
| 48 | Missing step in most graceful shutdowns? | Fail readiness, then *wait* before draining — else 502s on every deploy |
| 49 | Justify `CGO_ENABLED=0` | Genuinely static binary that runs in distroless/scratch |
| 50 | Justify `-trimpath` and `-ldflags="-s -w"` | Reproducible builds without local paths; ~30% smaller image |

---

## 6. Live-Coding Exercises

Do these on a whiteboard or in a plain editor with no completion. Time yourself.

### E1 — Bounded worker pool (15 min) ⭐ most likely

Process N items with at most K concurrent workers, propagate the first error, cancel siblings, and preserve result order.

> Solution and the reasoning to narrate: [`02-concurrency.md` §9](./02-concurrency.md#9-concurrency-patterns). Say the four checkpoints out loud as you write: bounded, cancellable, errors propagate, exactly one closer.

### E2 — Rate limiter (15 min)

Implement a token bucket: `Allow() bool` plus a context-aware `Wait(ctx) error`. Discuss token bucket vs leaky bucket vs sliding window, and what changes when you need it distributed (Redis + Lua, per §10 of the senior file).

### E3 — LRU cache (20 min)

`Get`/`Put` in O(1), thread-safe, with a capacity bound. Map plus a doubly-linked list (`container/list`). Follow-ups: how would you add TTL? How would you shard it to reduce lock contention? What breaks under concurrent `Get` promotion?

### E4 — Merge N sorted channels (15 min)

Fan-in with correct termination. The nil-channel trick from `02-concurrency.md` §3 is the elegant answer for two; for N, use a `WaitGroup` plus a single closer goroutine.

### E5 — Concurrent-safe counter, three ways (10 min)

Mutex, `atomic.Int64`, and a channel-owning goroutine. Then benchmark them and explain the ordering — atomic fastest, mutex close, channel slowest by an order of magnitude. The point is knowing *why* channels lose here: every operation is a scheduling event.

### E6 — Retry with exponential backoff and jitter (10 min)

Must honour context cancellation while sleeping, and must not retry permanent errors. See senior §12.

### E7 — Parse and validate a cron expression (20 min)

Direct Chronos tie-in. Then: "how would you test it?" — table-driven plus `FuzzParseCron`, where the assertion is that it never panics.

### E8 — Deduplicate a stream with a sliding window (15 min)

Given a channel of events with IDs, emit each ID at most once within a 5-minute window. Tests map usage, time handling, and memory bounding (the interesting follow-up: how do you stop the dedup set growing forever?).

### E9 — Implement `errgroup` from scratch (15 min)

`Go(func() error)`, `Wait() error`, first error wins, context cancelled on failure. Tests `sync.Once`, mutexes, and context composition together.

### E10 — Graceful HTTP server (10 min)

Signal handling, drain with timeout, correct shutdown ordering. Senior §14. Everyone claims to know this; few sequence readiness before drain.

---

## 7. Debugging Scenarios

These are "here's a symptom, walk me through it" questions. Practise the *method*, not just the answer.

### D1 — Memory grows steadily, pod OOMs every three days

> **Method:** Check `NumGoroutine` first — a goroutine leak is more common and looks identical. If flat, take two heap profiles hours apart and diff with `pprof -base`, reading `inuse_space` not `alloc_space`. A "leak" in Go is always a live reference: an unbounded cache map, a slice appended to forever, a `sync.Pool` holding oversized buffers, or a deleted slice element still pointed to by the backing array. Fix, verify with another diff, then add a gauge and an alert so the next one is caught before it pages anyone.

### D2 — p99 latency spikes every 30 seconds, p50 is fine

> **Method:** A periodic p99 spike with a healthy p50 is a GC signature. Confirm with `GODEBUG=gctrace=1` or the GC pause metric, then check whether it's actually the pauses (sub-millisecond, so rarely the culprit) or **mark assists** — where allocating goroutines are conscripted into marking and slow down proportionally. If it's assists, the fix is reducing allocation rate, found via `alloc_space` profiling. If pauses are genuinely long, look for a huge number of goroutines (stack scanning) or raise `GOGC`.
>
> Rule out the alternatives out loud: a cron job, cache expiry stampede (hence TTL jitter), connection recycling at `ConnMaxLifetime`, or a downstream's own periodicity.

### D3 — Service works locally, times out under load in production

> **Method:** Most likely resource exhaustion invisible at low concurrency. Check in order: `db.Stats().WaitCount` (pool too small — goroutines queuing for connections), `MaxIdleConnsPerHost` on the HTTP client (defaults to 2, so downstream calls serialise), `GOMAXPROCS` versus the cgroup CPU limit (fixed natively in Go 1.25+, but check the Go version), and goroutine count for a leak that only manifests under traffic.

### D4 — Occasional wrong data returned to the wrong tenant

> **Method:** This is the highest-severity bug class in a multi-tenant system, so start with the highest-probability causes. First: cache keys missing the org prefix — it only manifests on a hit, so it's load-dependent and invisible in tests. Second: a query missing its `WHERE organization_id`, which in Go is hand-written per query and therefore easy to omit. Third: state shared across requests — a package-level variable or a `gin.Context` used in a goroutine after the handler returned.
>
> Then the systemic fixes, which is what they're really asking for: `type OrgID int64` so a transposed argument is a compile error, tenant-isolation tests on every repository method, and Postgres Row-Level Security so the database enforces it regardless of application code.

### D5 — CPU pinned at 100% with no traffic

> **Method:** A busy loop. Take a CPU profile — it'll point straight at it. The usual cause is a `select` on a closed channel that's never nilled, so the case fires continuously. Others: a `for {}` polling loop with no sleep or blocking operation, and a `time.Ticker` with too small an interval.

### D6 — Deploys cause a burst of 502s

> **Method:** Shutdown ordering. Kubernetes removes the pod from Endpoints and sends SIGTERM concurrently, and propagation takes seconds. If you call `srv.Shutdown` immediately you reject requests still being routed to you. Fix: fail readiness first, sleep ~5 s, *then* drain. Also verify `terminationGracePeriodSeconds` exceeds your drain timeout, and that you're not passing the already-cancelled signal context to `Shutdown`.

### D7 — A test passes locally and fails in CI, intermittently

> **Method:** Almost always a real race that CI's different core count and load exposes. Run `go test -race -count=100`. If it's timing-based, the test is the bug — replace `time.Sleep` synchronisation with channels or `testing/synctest`. Also suspect map iteration order (randomised per run), test interdependence via shared state, and `t.Parallel()` with a shared fixture.

---

## 8. System Design Prompts

Structure every one of these the same way: **clarify requirements → estimate scale → sketch the high-level design → go deep where they push → name the trade-offs and failure modes you accepted.**

### S1 — Design a distributed job scheduler ⭐ your home turf

Steer here whenever you get a choice — you've built it. Cover: job storage and the due-jobs query (indexed on `next_run_at`, partial index on enabled jobs), leader election (Raft vs `SKIP LOCKED`, and why), execution (bounded worker pool with per-job recover), exactly-once (impossible — at-least-once plus `(job_id, run_id)` uniqueness), retries with backoff and a dead-letter queue, and observability (schedule lag as the key SLI, not just success rate).

### S2 — Design a rate limiter for a public API

Per-user, per-IP, per-endpoint tiers. Token bucket in Redis with a Lua script for atomicity. Discuss: fixed window's boundary burst problem, sliding window log's memory cost, sliding window counter as the pragmatic middle. Then the distributed question — local counters with periodic sync trade accuracy for latency. Response headers (`X-RateLimit-*`, `Retry-After`) and what to do when Redis is down (fail open for availability, fail closed for abuse protection — and this is a product decision, not a technical one).

### S3 — Design a URL shortener

Classic. Key insight for the Go framing: ID generation strategy (counter with base62 vs hash with collision handling vs Snowflake), read-heavy so cache aggressively, and 301 vs 302 (301 is cached by browsers forever, which kills your analytics).

### S4 — Migrate a Laravel monolith to Go microservices

**Ask this to be scoped down** — and note that the right first answer is often "I'd push back on the premise." Cover the strangler fig pattern, choosing the first service by bounded context (highest-load, fewest dependencies), the shared-database anti-pattern and how to break it, dual-write versus CDC for data migration, and how you'd keep auth working across both stacks during the transition.

> **The senior move here is to question whether it should happen at all.** A monolith with a performance problem usually has a query problem, and you've got direct evidence: your 88% query reduction. Say that. "Before splitting, I'd want to know we've exhausted the cheaper wins, because microservices trade a performance problem for a distributed-systems problem."

### S5 — Design a real-time notification system

WebSocket connection management in Go (a goroutine per connection is genuinely fine — this is where the netpoller shines), horizontal scaling with Redis pub/sub, fanout strategies, offline delivery and a message queue, and delivery guarantees.

### S6 — Design the inventory system you already built, but in Go

Direct compare-and-contrast with the Laravel version, which is a gift: you can talk about the same problem with two implementations. Multi-tenancy enforcement (compile-time types vs runtime scopes), concurrency (atomic conditional UPDATE — identical in both), caching, and the reservation ledger pattern.

---

## 9. STAR Stories

Prepare these written out, then rehearse until you can tell each in **two minutes**. Lead with the result.

### ST1 — The 88% query reduction ⭐ your strongest

- **Situation:** [endpoint/page], [latency], [traffic], and the business impact.
- **Task:** Your specific ownership and the target.
- **Action:** How you *found* it (profiling/query log/APM — be specific, this is where credibility lives), what the root cause was (N+1, missing index, over-fetching), what you changed, and how you validated it.
- **Result:** 88% fewer queries, [latency before → after], [infrastructure or cost effect].
- **Go framing:** "The same class of problem exists in Go but the tooling is different — I'd use pprof rather than an APM query log, and `sqlc` would have made the over-fetching visible at review time because the generated types show exactly what's selected."
- **Expect:** *"How did you prevent regression?"* Have an answer — a query-count assertion in tests is the strong one.

### ST2 — Zero-downtime 15M-record migration

- **Situation:** The schema change, the table size, the uptime constraint.
- **Action:** Expand → migrate → contract. Backfill in batches with throttling. Dual writes during transition. `CREATE INDEX CONCURRENTLY`. The rollback plan at each phase.
- **Result:** Zero downtime, and how you verified data integrity.
- **Go framing:** Batched backfill with a bounded worker pool and context cancellation — you can sketch the exact code from `02-concurrency.md` §9.
- **Expect:** *"What would you do differently?"* and *"What if the backfill had to be aborted at 80%?"* Both are testing whether you actually ran it or read about it.

### ST3 — Chronos: the distributed scheduler ⭐ your Go story

- **Situation:** Why you built it, what problem it models.
- **Task:** Exactly-once execution across N nodes with failure tolerance.
- **Action:** Raft for leader election, the claim-then-execute flow, idempotency via run IDs, bounded execution pool with per-job panic recovery, `synctest` for deterministic time-dependent tests.
- **Result:** What it demonstrably does; any benchmark numbers you have.
- **The honest reflection that lands well:** "Raft was the right choice for what I wanted to learn but it's over-engineered for most real scheduling needs — `SELECT ... FOR UPDATE SKIP LOCKED` would have been simpler and correct. Knowing that is the actual takeaway."
- **Expect:** all four follow-ups from senior §9. Rehearse them.

### ST4 — Multi-tenant SaaS architecture

- Row-level tenancy via `organization_id`, Passport, `spatie/laravel-permission` in teams mode, the leak vectors you had to close, and how you tested isolation.
- **Go framing:** the compile-time `OrgID` type argument from `03-gin-web.md` §4 — this is a genuinely strong point because it shows the design improving with the language.

### ST5 — 20K+ DAU trading platform

- Scale, latency requirements, concurrency and consistency challenges, what broke and how you fixed it.
- **Go framing:** where goroutines and channels would change the design — particularly the real-time price feed, which is a natural fit for the netpoller.

### ST6 — A production incident you caused or fixed

**Have one ready.** Senior interviews ask this, and "I can't think of one" is a bad answer. Structure: what broke, blast radius, how you detected it (and how long that took — be honest), how you mitigated, root cause, and the systemic fix that prevents the *class* of problem rather than the instance. Own your mistakes plainly; the maturity is the point, not the incident.

### ST7 — A technical disagreement

How you handled it, what changed your mind or theirs, and how you'd handle it now. They're assessing collaboration, not who was right. A story where you were persuaded is often stronger than one where you prevailed.

---

## 10. Questions to Ask Them

These signal seniority. Pick three or four that you actually want answered.

**On engineering practice**
- What does your Go code review process look like — do you run `-race` in CI, and what's your linter setup?
- How do you handle database migrations, and have you done zero-downtime schema changes?
- What's your test strategy — the unit/integration balance, and do you use testcontainers?

**On operations**
- What's your on-call rotation and how noisy is it?
- Walk me through your last significant incident — what changed afterwards?
- What are your p99 latency targets and do you currently meet them?

**On the codebase**
- What's the oldest part of the system, and what would you rewrite if you could?
- How do you decide between adding to an existing service and creating a new one?
- What Go version are you on, and how do you handle upgrades?

**On the role**
- What does success look like in the first 90 days?
- What's the biggest technical challenge the team is facing this quarter?
- How are technical decisions made — RFCs, ADRs, or informally?

---

## 11. Final Week Checklist

**Seven days out**
- [ ] Re-read all five files, focusing only on your marked-weak sections
- [ ] Do all 25 puzzles from §1 cold; anything you miss goes into daily rotation
- [ ] Write E1 (worker pool) from memory, three times, on three different days

**Three days out**
- [ ] Rehearse all seven STAR stories out loud, timed to two minutes each
- [ ] Do two system design prompts out loud with a whiteboard
- [ ] Re-read the traps: nil interface, slice aliasing, method sets, goroutine leaks, `c.Copy()`

**Day before**
- [ ] Skim the rapid-fire tables only — do not learn anything new
- [ ] Re-read your Chronos follow-ups (senior §9) and the fencing-token answer (§10)
- [ ] Prepare your questions for them
- [ ] Sleep. Genuinely more valuable than another hour of revision.

**On the day**
- [ ] Think out loud — silence is scored as being stuck
- [ ] State assumptions before coding, and ask about scale before designing
- [ ] When you don't know something, say so and reason toward it. "I don't know, but here's how I'd find out" scores well; bluffing scores very badly
- [ ] For every concurrency answer, mention the cancellation path and the failure mode unprompted
- [ ] Give numbers wherever you have them — you have real ones, use them

---

> **The single highest-leverage habit for this loop:** for every answer, add one sentence of *evidence* or *trade-off* that the question didn't ask for. Not "I'd use a worker pool," but "I'd use a worker pool bounded by the DB pool size, since goroutines aren't the scarce resource — the connections are." That one extra clause, applied consistently, is the difference between a hire and a strong hire.

---

**Back to:** [`README.md`](./README.md) · [`01-basic.md`](./01-basic.md) · [`02-concurrency.md`](./02-concurrency.md) · [`03-gin-web.md`](./03-gin-web.md) · [`04-senior.md`](./04-senior.md)
