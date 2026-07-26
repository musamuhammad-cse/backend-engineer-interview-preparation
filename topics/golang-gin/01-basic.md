# Go — Tier 1: Language & Runtime Fundamentals

> Everything here is compiled and verified against **Go 1.26**. Outputs quoted in the text are real program output, not from memory.
>
> Do not skip this tier. Go interviews plant "simple" questions here that expose shallow foundations — slice aliasing, nil interfaces, method sets — and they're asked at every level from mid to staff. Coming from PHP, the specific risk is assuming Go's value semantics work like PHP's copy-on-write. They don't, and the differences are exactly what gets tested.

---

## Table of Contents

1. [Execution & Build Model](#1-execution--build-model)
2. [Packages, Modules & Initialisation](#2-packages-modules--initialisation)
3. [The Type System](#3-the-type-system)
4. [Untyped Constants & iota](#4-untyped-constants--iota)
5. [Strings, Bytes & Runes](#5-strings-bytes--runes)
6. [Arrays & Slices — The Most-Tested Topic](#6-arrays--slices--the-most-tested-topic)
7. [Maps](#7-maps)
8. [Structs & Embedding](#8-structs--embedding)
9. [Pointers, Values & Method Sets](#9-pointers-values--method-sets)
10. [Functions & Closures](#10-functions--closures)
11. [defer, panic & recover](#11-defer-panic--recover)
12. [Interfaces](#12-interfaces)
13. [Errors](#13-errors)
14. [Generics](#14-generics)
15. [Standard Library Tour](#15-standard-library-tour)
16. [Tooling](#16-tooling)
17. [Tier 1 Q&A Drill](#17-tier-1-qa-drill)

---

## 1. Execution & Build Model

Coming from PHP, this is the first thing to reframe. There is no interpreter, no opcode cache, no `php-fpm` pool, and no per-request bootstrap.

| | PHP / Laravel | Go |
|---|---|---|
| Execution | Source → Zend opcodes → VM (OPcache caches opcodes) | Source → machine code, ahead of time |
| Deployment unit | Source tree + runtime + extensions | One static binary |
| Process model | One request per worker process, shared-nothing | One long-lived process, many goroutines |
| Per-request state | Destroyed after each request | **Persists** — this is the big mental shift |
| Startup cost | Framework boot on every request (mitigated by OPcache/Octane) | Once, at process start |
| Concurrency | Multiple processes (php-fpm children) | Goroutines inside one process |

```bash
go build ./...            # compile, produce binary
go run ./cmd/api          # compile to temp dir and execute
go test ./... -race       # run tests with the race detector
go vet ./...              # static analysis for likely bugs
GOOS=linux GOARCH=arm64 go build ./cmd/api    # cross-compile, no toolchain juggling
```

**The static binary matters in interviews.** `FROM scratch` or `FROM gcr.io/distroless/static` containers with a ~15 MB image, no runtime to patch, no `composer install` at deploy time. When asked "why Go for this service?", deployment simplicity and predictable memory are stronger answers than "it's fast."

> **Trap — cgo silently breaks static linking.** If you import a package that uses cgo (the classic offender is `net` with the default resolver, or `os/user`), you may get a dynamically linked binary that fails in a `scratch` container with a confusing error. `CGO_ENABLED=0 go build` forces pure-Go implementations. Being able to say *"I set `CGO_ENABLED=0` so the binary is genuinely static and works in distroless"* is a small, credible production detail.

> **Follow-up:** *Why is the binary so much bigger than a C equivalent?* It embeds the runtime — the scheduler, garbage collector, and type metadata for reflection. Strip debug info with `-ldflags="-s -w"` if size matters, but that removes symbols from panic traces, which is usually a bad trade in production.

### The shared-process shift

This is the single biggest source of bugs for engineers moving from PHP to Go, and it's worth stating explicitly in an interview if you're asked about the transition:

```go
// In PHP this would be harmless — the process dies after the request.
// In Go this is a data race and a memory leak.
var requestCache = map[string]User{}   // package-level, shared by every request

func handler(w http.ResponseWriter, r *http.Request) {
    requestCache[r.URL.Query().Get("id")] = User{}   // concurrent map write -> crash
}
```

Package-level mutable state is shared by every concurrent request forever. In PHP, global state is merely bad style; in Go it's a correctness bug and an unbounded memory growth path.

---

## 2. Packages, Modules & Initialisation

### Modules

```bash
go mod init github.com/you/chronos
go get github.com/gin-gonic/gin@latest
go mod tidy        # add missing, remove unused
go mod why golang.org/x/net    # explain why a dependency is in the graph
go mod graph       # full dependency graph
```

`go.mod` declares the module path, Go version, and direct/indirect requirements. `go.sum` holds cryptographic hashes of every module version in the graph — it's a lockfile *and* a tamper check, and it must be committed.

> **Trap — Go uses Minimal Version Selection (MVS), not "latest wins."** If your module needs `foo v1.2.0` and a dependency needs `foo v1.4.0`, Go picks `v1.4.0` — the minimum version that satisfies *everyone*, not the newest available. This is deliberately different from npm/Composer and makes builds reproducible without a lockfile resolution step. Composer resolves to the newest version matching your constraints; Go resolves to the oldest that works. Being able to contrast the two is a good answer.

> **Follow-up:** *How does Go handle a breaking change?* Semantic import versioning: `v2+` must change the import path (`github.com/foo/bar/v2`). That means `v1` and `v2` are *different packages* and can coexist in one build — which solves the diamond dependency problem that plagues other ecosystems.

### Visibility

There is no `public`/`private`/`protected` keyword. **Capitalisation is the access modifier.**

```go
package inventory

type Item struct {
    SKU      string  // exported — visible outside the package
    Quantity int     // exported
    orgID    int64   // unexported — package-private
}

func New(sku string) *Item { return &Item{SKU: sku} }   // exported
func validate(sku string) error { return nil }          // unexported
```

The unit of encapsulation is the **package**, not the type. Any code in package `inventory` can read `orgID` on any `Item`. There is no per-type privacy.

> **Trap:** unexported struct fields are invisible to `encoding/json`, `reflect`-based ORMs, and most validators. `json.Marshal` on a struct with only unexported fields returns `{}`. This bites people who make fields lowercase for "encapsulation" and then wonder why their API returns empty objects.

### Initialisation order

```go
package main

import "fmt"

var a = b + 1        // 3 — Go resolves dependency order, not source order
var b = c * 2        // 2
var c = 1

func init() { fmt.Println("init 1") }
func init() { fmt.Println("init 2") }   // multiple init() allowed, run in source order

func main() { fmt.Println(a, b, c) }
```

Order: imported packages fully initialise first (depth-first), then package-level variables in dependency order, then `init()` functions in source order, then `main()`.

> **Trap — `init()` is a testing and clarity hazard.** It runs on import, cannot fail gracefully (only `panic`), cannot be called explicitly, and makes ordering implicit. Prefer explicit constructors called from `main()`. The one legitimate use is driver registration (`sql.Register`, `image.RegisterFormat`), which is exactly what the blank import `_ "github.com/lib/pq"` is for.

> **Trap — import cycles are a hard compile error.** Go has no forward declarations and no lazy resolution. If `service` imports `repository` and `repository` imports `service`, it will not build. The fix is to define the interface in the *consumer's* package — which conveniently pushes you toward dependency inversion whether you wanted it or not.

---

## 3. The Type System

### No implicit conversion. At all.

```go
var i int = 42
var f float64 = i      // compile error: cannot use i (variable of type int)
var f2 float64 = float64(i)   // explicit conversion required

var i32 int32 = 1
var i64 int64 = i32    // compile error — even between integer types
```

Even `int` and `int64` are distinct types requiring conversion, on a 64-bit platform where they have identical representation. This is intentional: conversions are where precision is lost, so Go makes every one of them visible.

### Named types vs aliases

```go
type UserID int64      // NEW TYPE — distinct, has its own method set
type Alias = int64     // ALIAS — literally the same type, interchangeable

func process(id UserID) {}

var raw int64 = 5
process(raw)            // compile error: int64 is not UserID
process(UserID(raw))    // fine
process(Alias(raw))     // still an error — Alias IS int64
```

> **Why this matters for your multi-tenant work:** this is Go's answer to primitive obsession, and it's a genuinely good story to tell. In the PHP SaaS, `int $organizationId` and `int $userId` are the same type, so transposing them at a call site compiles fine and silently queries the wrong tenant. In Go:
>
> ```go
> type OrgID int64
> type UserID int64
>
> func ItemsFor(org OrgID, user UserID) ([]Item, error) { /* ... */ }
>
> ItemsFor(userID, orgID)   // compile error — arguments transposed
> ```
>
> The tenant boundary becomes a compile-time guarantee rather than a code-review convention. Volunteering this when asked "how would you prevent cross-tenant leaks in Go?" is much stronger than "add a WHERE clause."

### Zero values

Every type has a zero value, and variables are **always** initialised. There is no undefined, no null-by-default surprise.

| Type | Zero value |
|------|-----------|
| numeric | `0` |
| `bool` | `false` |
| `string` | `""` (not nil — strings are never nil) |
| pointer, slice, map, channel, func, interface | `nil` |
| struct | struct with all fields at their zero values |
| array | array with all elements at their zero values |

**The design goal is "make the zero value useful."** The best examples in the standard library:

```go
var buf bytes.Buffer      // ready to use, no constructor
buf.WriteString("hello")

var mu sync.Mutex         // ready to use
mu.Lock()

var wg sync.WaitGroup     // ready to use
```

> **Trap — nil maps and nil slices behave differently.** A nil slice supports `append`, `len`, `range`, and everything except indexing. A nil map supports reads and `len`, but **writing to it panics**. This asymmetry is asked constantly:
>
> ```go
> var s []int
> s = append(s, 1)          // fine — append allocates
>
> var m map[string]int
> _ = m["missing"]           // fine — returns 0
> m["key"] = 1               // PANIC: assignment to entry in nil map
> ```
>
> The reason: `append` returns a new slice header, so it can allocate and hand it back. Map writes mutate in place and there's nowhere to store a newly allocated map.

---

## 4. Untyped Constants & iota

Constants in Go are more interesting than they look, and they're a favourite "do you actually know the spec" question.

```go
const big = 1 << 40        // untyped — no overflow even though it exceeds int32
const pi = 3.14159         // untyped float

var x int64 = big          // fine: untyped constant adopts the target type
var y float64 = big        // also fine
// var z int32 = big       // compile error: overflows int32 — caught at COMPILE time
```

Untyped constants have **arbitrary precision** at compile time and only take a type when assigned. This is why `math.MaxUint64` can be declared as a constant even though it doesn't fit in an `int`.

```go
const huge = 1 << 100      // legal as a constant
// fmt.Println(huge)       // compile error: constant overflows int
fmt.Println(huge >> 90)    // fine: 1024, fits once shifted
```

### iota

```go
type Status int

const (
    StatusPending Status = iota   // 0
    StatusRunning                 // 1
    StatusSucceeded               // 2
    StatusFailed                  // 3
)

func (s Status) String() string {
    switch s {
    case StatusPending:
        return "pending"
    case StatusRunning:
        return "running"
    case StatusSucceeded:
        return "succeeded"
    case StatusFailed:
        return "failed"
    default:
        return fmt.Sprintf("Status(%d)", int(s))
    }
}
```

Bit flags with `iota`:

```go
type Permission uint8

const (
    PermRead Permission = 1 << iota   // 1
    PermWrite                          // 2
    PermDelete                         // 4
    PermAdmin                          // 8
)

func (p Permission) Has(q Permission) bool { return p&q != 0 }
```

> **Trap — Go has no real enums, and this has consequences.** `Status` is just an `int`. Nothing stops `Status(99)`, and there's no exhaustiveness check on a `switch` — the compiler will not tell you that you forgot a case when you add one. This is a genuine weakness versus PHP 8.1 backed enums, and saying so demonstrates judgement rather than fanboyism.
>
> The mitigations, in order of practicality:
> 1. Always include a `default` that surfaces the unexpected value (note the `String()` above returns `Status(99)` rather than lying).
> 2. Add a `Valid() bool` method and call it at every deserialisation boundary.
> 3. Use the `exhaustive` linter in CI, which *does* check switch coverage over `iota` constant groups.
> 4. Generate `String()` with `stringer` rather than hand-writing it (`//go:generate stringer -type=Status`).

> **Trap — `iota` resets per `const` block and increments per *line*, not per constant.** Blank identifiers skip values:
>
> ```go
> const (
>     _  = iota             // skip 0
>     KB = 1 << (10 * iota) // 1 << 10
>     MB                     // 1 << 20
>     GB                     // 1 << 30
> )
> ```

---

## 5. Strings, Bytes & Runes

A Go `string` is an **immutable read-only slice of bytes**, conventionally UTF-8. It is *not* a sequence of characters.

```go
s := "héllo"
fmt.Println(len(s))              // 6  — BYTES, not characters
fmt.Println(s[1])                // 195 — a byte (0xC3), not 'é'
fmt.Println(len([]rune(s)))      // 5  — actual character count
```

That output is real: `é` is two bytes in UTF-8, so `len` reports 6 for a 5-character string, and indexing lands mid-character.

### Indexing vs ranging

```go
s := "héllo"

for i := 0; i < len(s); i++ {
    fmt.Printf("%d:%d ", i, s[i])    // iterates BYTES
}

for i, r := range s {
    fmt.Printf("%d:%c ", i, r)       // iterates RUNES; i is the BYTE offset
}
// prints: 0:h 1:é 3:l 4:l 5:o
//              ^ index jumps 1 -> 3 because é occupied two bytes
```

**`range` over a string decodes UTF-8 and yields runes, but the index is a byte offset.** That index discontinuity is the detail interviewers look for.

> **Trap — invalid UTF-8 does not panic.** `range` yields `utf8.RuneError` (`\uFFFD`) and advances one byte. Converting `[]byte` from an untrusted source to a string does no validation. If correctness matters, `utf8.ValidString(s)` explicitly.

### Building strings efficiently

Strings are immutable, so `+=` in a loop allocates a new string every iteration — O(n²) copying.

```go
// BAD — quadratic
var s string
for i := 0; i < 10000; i++ {
    s += "x"
}

// GOOD — amortised linear
var b strings.Builder
b.Grow(10000)                     // pre-size when you know the length
for i := 0; i < 10000; i++ {
    b.WriteString("x")
}
s := b.String()
```

`strings.Builder` avoids a copy in `String()` by using `unsafe` internally — it hands out the backing array directly, which is safe only because `Builder` refuses to be copied after first use (it panics via a self-pointer check).

> **Follow-up:** *Why does `strings.Builder` panic if you copy it?* It stores a pointer to itself on first write. If the struct is copied, that pointer no longer matches the new address, and the check fires. Without this, a copied builder could hand out a backing array that another builder is still writing to.

### Conversions allocate

```go
b := []byte(s)     // ALLOCATES and copies — strings are immutable, slices are not
s2 := string(b)    // ALLOCATES and copies
```

The compiler optimises a few specific cases (`for i, r := range []rune(s)`, map lookups with `m[string(b)]`, comparisons), but assume conversion copies. In a hot path this is a common, easily-missed allocation source.

---

## 6. Arrays & Slices — The Most-Tested Topic

If you prepare one thing thoroughly, make it this. Slice questions appear in essentially every Go interview because they test whether you understand memory rather than just syntax.

### Arrays are values

```go
a := [3]int{1, 2, 3}
b := a          // FULL COPY — arrays are values
b[0] = 99
fmt.Println(a[0], b[0])    // 1 99

func modify(arr [3]int) { arr[0] = 100 }   // receives a copy; caller unaffected
```

The length is **part of the type**: `[3]int` and `[4]int` are different types. This is why arrays are rare in Go code — you almost always want a slice.

### The slice header

A slice is a three-word struct. This is the mental model everything else follows from:

```go
type slice struct {
    ptr *T   // pointer to the backing array
    len int  // number of elements accessible
    cap int  // elements from ptr to the end of the backing array
}
```

Verified: `unsafe.Sizeof([]int{})` is **24 bytes** on 64-bit (three 8-byte words).

```go
s := make([]int, 3, 5)     // len 3, cap 5
fmt.Println(len(s), cap(s))    // 3 5
```

**Passing a slice to a function copies the header, not the data.** So the function can modify existing elements (both headers point at the same array) but cannot change the caller's length.

```go
func fill(s []int)   { s[0] = 99 }          // caller SEES this
func grow(s []int)   { s = append(s, 4) }   // caller does NOT see this
```

### The aliasing trap — the single most-asked slice question

```go
base := make([]int, 3, 5)
base[0], base[1], base[2] = 1, 2, 3
view := base[0:2]              // len 2, cap 5 — SHARES the backing array
view = append(view, 99)        // len 2 < cap 5, so it writes IN PLACE at index 2

fmt.Println(base)   // [1 2 99]  ← base was silently modified
fmt.Println(view)   // [1 2 99]
```

That output is verified. `append` only allocates a new array when capacity is exhausted; otherwise it writes into the existing one, and any other slice sharing that array sees the change.

The nastiest version, because it looks completely safe:

```go
full := []int{1, 2, 3}      // len 3, cap 3
v2 := full[0:2]             // len 2, cap 3  ← cap comes from the ORIGINAL, not the slice length
v2 = append(v2, 99)         // 2 < 3, so it fits — writes into full[2]

fmt.Println(full)   // [1 2 99]  ← mutated
```

**Slicing does not reduce capacity.** `full[0:2]` still has capacity 3 because capacity is measured from the start offset to the end of the backing array.

### Three-index slicing — the fix

```go
v3 := full[0:2:2]           // len 2, CAP 2 — capacity explicitly limited
v3 = append(v3, 99)         // cap exhausted -> allocates a new array
fmt.Println(full)           // [1 2 3]  ← untouched
```

`s[low:high:max]` sets `cap = max - low`. Use it whenever you hand a sub-slice to code you don't control. This is the idiomatic "defensive copy" for slices and knowing it is a strong signal.

> **The classic real-world version of this bug:**
>
> ```go
> func (s *Server) Handle(buf []byte) {
>     header := buf[:8]
>     go s.process(header)   // shares the buffer; the caller may reuse buf for the next request
> }
> ```
>
> If `buf` comes from a `sync.Pool` or is reused across requests, `header` sees another request's data. The fix is an explicit copy, not a three-index slice, because you want independent memory:
>
> ```go
> header := append([]byte(nil), buf[:8]...)   // or slices.Clone(buf[:8])
> ```

### Growth

Verified capacity sequence in Go 1.26 for `append` on an `[]int` starting from nil:

```
4, 8, 16, 32, 64, 128, 256, 512, 848, 1280, 1792, 2560, ...
```

Doubling up to 256 elements, then roughly 1.25× growth, with the result rounded up to a memory size class. The exact numbers change between releases — **do not memorise them**. What matters in an interview is the shape: *doubling while small, tapering to ~1.25× when large, then rounded to an allocator size class.* Say that and you're demonstrably ahead of someone who says "it doubles."

```go
// Always pre-size when you know the count — this is one of the highest-value
// easy optimisations in Go, and it maps directly to your query-reduction story.
items := make([]Item, 0, len(rows))    // one allocation instead of ~12
for _, r := range rows {
    items = append(items, toItem(r))
}
```

### nil vs empty

```go
var a []int              // nil:   a == nil, len 0, cap 0
b := []int{}             // empty: b != nil, len 0, cap 0
c := make([]int, 0)      // empty: c != nil, len 0, cap 0

len(a) == len(b)         // true — behave identically for len/range/append
a == nil                 // true
b == nil                 // false
```

They behave identically for every operation *except* comparison to nil and JSON encoding:

```go
json.Marshal(a)   // "null"
json.Marshal(b)   // "[]"
```

> **Trap with real API consequences:** returning a nil slice from a handler produces `{"items": null}` instead of `{"items": []}`, which breaks JavaScript clients doing `data.items.length`. Prefer returning `[]Item{}` from API layers, or initialise with `make([]Item, 0)`. This is a genuinely common production bug and a good thing to mention when discussing API design.

### Deleting and copying

```go
// Modern (Go 1.21+ `slices` package) — prefer these
s = slices.Delete(s, i, i+1)        // remove index i, preserving order
s = slices.Insert(s, i, v)
clone := slices.Clone(s)            // shallow copy
idx, found := slices.BinarySearch(s, target)
slices.Sort(s)
slices.SortFunc(s, func(a, b Item) int { return cmp.Compare(a.SKU, b.SKU) })

// Manual delete, order-preserving
s = append(s[:i], s[i+1:]...)

// Manual delete, order NOT preserved — O(1)
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```

> **Trap — deleting from a slice of pointers leaks memory.** `s = s[:len(s)-1]` shrinks the length but the backing array still holds the pointer at the old position, so the pointee can't be collected. Zero it first:
>
> ```go
> s[len(s)-1] = nil
> s = s[:len(s)-1]
> ```
>
> `slices.Delete` handles this for you as of Go 1.22 — it zeroes the vacated tail. This is exactly the kind of subtle leak that shows up as "our memory grows slowly over days" in production, and it's a great pprof story.

---

## 7. Maps

```go
m := make(map[string]int)          // preferred
m := map[string]int{"a": 1}        // literal
m := make(map[string]int, 1000)    // pre-sized — avoids rehashing
```

### Comma-ok

```go
v := m["missing"]           // 0 — indistinguishable from a stored zero
v, ok := m["missing"]       // 0, false — this is the distinction you need
```

If your value type can legitimately be the zero value (an `int` counter, a `bool` flag), you **must** use comma-ok or you'll conflate "absent" with "zero."

### Iteration order is deliberately randomised

```go
for k, v := range m {   // order differs on EVERY run, even for the same map
    fmt.Println(k, v)
}
```

This is not "unspecified but stable" — the runtime injects a random start offset specifically so you cannot accidentally depend on order. To iterate deterministically:

```go
keys := slices.Sorted(maps.Keys(m))    // Go 1.23+: iterator-based
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

> **Follow-up:** *Why did they do that?* Because Go's authors saw code depend on the incidental ordering of an earlier implementation, then break when the implementation changed. Randomising forces the bug to surface immediately in development rather than in production after a runtime upgrade. It's a good example of a language deliberately making a hazard *louder*.

### Map elements are not addressable

```go
type Counter struct{ N int }
m := map[string]Counter{"a": {}}

m["a"].N++          // COMPILE ERROR: cannot assign to struct field in map
&m["a"]             // COMPILE ERROR: cannot take address
```

The reason: maps rehash and move entries in memory, so any pointer you took could dangle. Two fixes:

```go
// 1. Read, modify, write back
c := m["a"]
c.N++
m["a"] = c

// 2. Store pointers (usually better for mutable values)
m2 := map[string]*Counter{"a": {}}
m2["a"].N++         // fine — the map value is a pointer, and we're not moving it
```

### Concurrent access

```go
// This CRASHES the program — not a data race warning, a hard fatal error
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()
// fatal error: concurrent map writes
```

The runtime has an explicit check for this and calls `throw`, which **cannot be recovered**. It is not a `panic`.

> **Trap that catches people:** this is deliberately different from a data race on an `int`, which silently corrupts. The map check exists because concurrent map writes corrupt the hash table in ways that produce baffling failures later, so the runtime chose to fail loudly and immediately. Note that concurrent *reads* are safe; it's read-with-write and write-with-write that are fatal.

The three options, and knowing when to use each is the real question:

```go
// 1. Mutex — the default, correct answer for most cases
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Get(k string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[k]
    return v, ok
}

func (s *SafeMap) Set(k string, v int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[k] = v
}

// 2. sync.Map — ONLY for its two specific workloads (see Tier 2)
// 3. Sharded map — many independent mutexes, when lock contention is measured
```

> **Trap:** `sync.Map` is not "the concurrent map you should use." It's optimised for exactly two cases: write-once-read-many, and disjoint key sets per goroutine. For general read/write workloads it is **slower** than a plain map with an `RWMutex`, and it loses type safety (`any` keys and values). Reaching for it reflexively is a mild red flag; knowing its two use cases is a green one.

### Implementation note

Go 1.24 replaced the map implementation with **Swiss Tables**, giving roughly 30% faster lookups and lower memory for large maps. You don't need internals, but knowing the term and that maps are open-addressed with SIMD-assisted probing (rather than the old bucket-chain design) is a nice depth signal if map performance comes up.

---

## 8. Structs & Embedding

```go
type Job struct {
    ID        string    `json:"id" db:"id"`
    Schedule  string    `json:"schedule" db:"cron_expr"`
    OrgID     int64     `json:"-" db:"organization_id"`   // "-" excludes from JSON
    NextRun   time.Time `json:"next_run" db:"next_run_at"`
    Payload   []byte    `json:"payload,omitempty"`
}
```

Struct tags are just strings read via reflection at runtime. A typo in a tag is **not** a compile error — it silently does nothing. `go vet` catches malformed tags, which is one reason to run it in CI.

### Embedding is composition, not inheritance

```go
type Base struct {
    ID        string
    CreatedAt time.Time
}

func (b *Base) Age() time.Duration { return time.Since(b.CreatedAt) }

type Job struct {
    Base                  // embedded — fields and methods are PROMOTED
    Schedule string
}

j := Job{}
j.ID = "abc"              // promoted field — shorthand for j.Base.ID
fmt.Println(j.Age())      // promoted method
```

**This is not inheritance.** There is no subtyping, no virtual dispatch, no `super`. `Job` is not a `Base` and cannot be passed where a `Base` is expected. Promotion is purely syntactic sugar for `j.Base.ID`.

```go
func process(b Base) {}
process(j)          // COMPILE ERROR: Job is not Base
process(j.Base)     // fine — explicit
```

> **Trap — embedding an interface, and why it's a footgun.**
>
> ```go
> type Store interface {
>     Get(id string) (*Job, error)
>     Put(j *Job) error
>     Delete(id string) error
> }
>
> type ReadOnlyStore struct {
>     Store          // embedded INTERFACE — satisfies Store automatically
> }
> ```
>
> `ReadOnlyStore` now satisfies `Store` even though it implements nothing. Calling `Delete` forwards to the embedded value — and if that's nil, you get a nil pointer panic at runtime rather than a compile error. This is occasionally used deliberately (to implement a large interface partially in a test mock) but it silently disables the compiler's completeness check. If you use it in a mock, you're trading a compile error for a runtime panic; that's sometimes worth it, and you should say so knowingly.

> **Trap — ambiguous promotion.** If two embedded types have the same method, calling it on the outer type is a compile error unless the outer type defines its own. Depth wins: a shallower method shadows a deeper one silently.

### Comparability

```go
type Point struct{ X, Y int }
Point{1, 2} == Point{1, 2}       // true — structs of comparable fields are comparable

type Bad struct{ Data []int }
// Bad{} == Bad{}                // COMPILE ERROR: slices are not comparable
```

Slices, maps, and functions are not comparable, so any struct containing one isn't either. This matters because **map keys must be comparable** — a struct with a slice field cannot be a map key.

### The empty struct

```go
struct{}{}          // occupies ZERO bytes

set := map[string]struct{}{}       // a set, using no memory for values
set["key"] = struct{}{}
_, exists := set["key"]

done := make(chan struct{})        // signal-only channel, no payload
close(done)
```

`map[string]bool` costs one byte per entry; `map[string]struct{}` costs zero. The idiom also documents intent: the value is meaningless. Using it correctly is a small fluency signal.

---

## 9. Pointers, Values & Method Sets

This section contains the question most likely to catch you, so read it twice.

### Receivers

```go
type Counter struct{ n int }

func (c Counter) Get() int  { return c.n }    // VALUE receiver — operates on a copy
func (c *Counter) Inc()     { c.n++ }         // POINTER receiver — operates on the original
```

**Use a pointer receiver when:** the method mutates, the struct is large enough that copying costs, or the type contains a `sync.Mutex` (copying a mutex is a bug — `go vet` catches it).

**Be consistent.** If any method needs a pointer receiver, give them all pointer receivers. Mixing them is the source of the confusion below.

### The method set rule

| Receiver declared on | Method set of `T` | Method set of `*T` |
|---|---|---|
| value receiver `func (t T)` | ✅ included | ✅ included |
| pointer receiver `func (t *T)` | ❌ **not** included | ✅ included |

**`*T` has all methods. `T` has only the value-receiver methods.**

```go
type Speaker interface{ Speak() string }

type Dog struct{}
func (d *Dog) Speak() string { return "woof" }   // POINTER receiver

var s Speaker = Dog{}     // COMPILE ERROR: Dog does not implement Speaker
                          //   (method Speak has pointer receiver)
var s Speaker = &Dog{}    // fine
```

This error message is one of the most common in Go, and being able to explain *why* rather than just fixing it is the differentiator.

### Why the asymmetry? Addressability.

The subtle part, and the follow-up you should expect:

```go
c := Counter{}
c.Inc()          // WORKS — even though Inc needs *Counter
```

Go rewrites this to `(&c).Inc()` automatically, because `c` is an **addressable** variable. So why doesn't the interface assignment work the same way?

Because interface values store a **copy**. If Go auto-took the address when boxing a value into an interface, it would take the address of a temporary copy — mutations would silently apply to a copy the caller can never see. Rather than allow that silent bug, the compiler refuses.

The same rule explains these:

```go
m := map[string]Counter{"a": {}}
m["a"].Inc()                    // ERROR — map elements are not addressable

var i interface{} = Counter{}
// i.(Counter).Inc()            // ERROR — assertion result is not addressable

Counter{}.Inc()                 // ERROR — a literal is not addressable

s := []Counter{{}}
s[0].Inc()                      // WORKS — slice elements ARE addressable
```

> **The one-sentence version to have ready:** *"A value type's method set excludes pointer-receiver methods because interface values hold a copy — auto-addressing would let you mutate a temporary. Direct calls on addressable variables work because the compiler can safely insert the `&`."*

### Pointers vs values in practice

Go has no reference types the way PHP has objects. **Everything is passed by value** — including pointers, which are values that happen to hold an address.

```go
func rename(j Job)  { j.Schedule = "x" }    // caller unaffected
func rename2(j *Job) { j.Schedule = "x" }   // caller sees the change
```

Slices, maps, and channels *feel* like references because their headers contain pointers, but the header itself is copied. That's exactly why `append` inside a function doesn't affect the caller's length: you mutated a copy of the header.

> **Trap:** "should I always use pointers to avoid copying?" No. Small structs (a few words) copy faster than they dereference, and value receivers keep data on the stack where the GC never sees it. Pointers force heap allocation when they escape. The default is a value receiver for small immutable types and a pointer receiver for anything mutable or large — and "large" means benchmark it, not guess.

---

## 10. Functions & Closures

### Multiple returns and the error convention

```go
func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := Divide(10, 2)
if err != nil {
    return fmt.Errorf("dividing: %w", err)
}
```

The convention is absolute: **error is the last return value, and you check it immediately.** Ignoring it with `_` is legal and is exactly what `errcheck`/`golangci-lint` will flag.

### Named returns

```go
func fetch(id string) (job *Job, err error) {
    defer func() {
        if err != nil {
            metrics.Inc("fetch.error")   // deferred func can SEE and MODIFY named returns
        }
    }()
    // ...
    return nil, errors.New("not found")
}
```

Verified behaviour — the difference matters:

```go
func namedReturn() (n int) {
    defer func() { n++ }()
    return 5              // sets n = 5, then defer increments -> returns 6
}

func plainReturn() int {
    n := 5
    defer func() { n++ }()
    return n              // return value copied FIRST, then defer increments local -> returns 5
}
// Output: 6 5
```

**`return x` in a function with named results assigns to the named variable, then runs defers, then returns.** With unnamed results the value is copied to the return slot before defers run, so a defer mutating the local has no effect.

> **Style note:** use named returns for the deferred-error-handling pattern above, and for documenting which of two same-typed returns is which. Avoid "naked returns" (`return` with no values) in anything longer than a few lines — they're genuinely hard to read.

### Closures capture by reference

```go
func counter() func() int {
    count := 0                  // escapes to the heap — the closure outlives the function
    return func() int {
        count++                 // captures the VARIABLE, not its value
        return count
    }
}

c := counter()
c(); c(); fmt.Println(c())    // 3
```

### The loop variable change — Go 1.22

This is essential to know because the correct answer *changed*, and interviewers who learned Go earlier may still expect the old one.

```go
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()
}
```

- **Go ≤ 1.21:** one shared `i`; all goroutines usually print `3`. The fix was `i := i` or passing `i` as an argument.
- **Go ≥ 1.22:** `i` is a **new variable each iteration**; prints `0 1 2` in some order.

The `x := x` shadowing idiom is now unnecessary in modern Go. If you see it in a codebase, it's either pre-1.22 or defensive habit.

> **How to handle this in an interview:** don't just say "it prints 0 1 2." Say *"Since 1.22 the loop variable is per-iteration, so it prints 0, 1, 2 in nondeterministic order. Before 1.22 it was a single shared variable and this was the single most common Go bug — you'd typically see 3, 3, 3."* Naming the version and the old behaviour demonstrates you've tracked the language, and it protects you if the interviewer is running an older mental model.
>
> The `go.mod` `go` directive controls this per-module, so an old module compiled with a new toolchain keeps the old semantics.

Also new in 1.22 — ranging over an integer:

```go
for i := range 10 {    // 0..9
    fmt.Println(i)
}
for range 3 {          // repeat 3 times, no variable
    doThing()
}
```

### Variadics

```go
func Sum(nums ...int) int {          // nums is []int inside the function
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

Sum(1, 2, 3)
Sum(existing...)      // spread an existing slice
```

> **Trap:** `Sum(existing...)` passes the **same backing array**, not a copy. If `Sum` appends to `nums` within capacity, it mutates the caller's slice. Same aliasing rule as §6.

### First-class functions

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {   // reverse: first listed runs outermost
        h = mws[i](h)
    }
    return h
}
```

This is the same onion-pipeline idea as Laravel middleware, built from plain function composition instead of a `Pipeline` class.

---

## 11. defer, panic & recover

### defer semantics

Three rules, all of which get tested:

**1. Arguments are evaluated immediately; execution is deferred.**

```go
func main() {
    i := 0
    defer fmt.Println("deferred:", i)   // prints 0 — i evaluated NOW
    i = 99
    fmt.Println("immediate:", i)        // 99
}
// immediate: 99
// deferred: 0
```

**2. LIFO order.**

```go
for i := 0; i < 3; i++ {
    defer fmt.Println(i)     // prints 2, 1, 0
}
```

**3. Defers run when the *function* returns, not when the block ends.**

```go
// BUG — accumulates open file handles until the function ends
func processAll(paths []string) error {
    for _, p := range paths {
        f, err := os.Open(p)
        if err != nil {
            return err
        }
        defer f.Close()      // 10,000 files -> 10,000 open handles
        process(f)
    }
    return nil
}

// FIX — give each iteration its own function scope
func processAll(paths []string) error {
    for _, p := range paths {
        if err := func() error {
            f, err := os.Open(p)
            if err != nil {
                return err
            }
            defer f.Close()      // runs at the end of THIS iteration
            return process(f)
        }(); err != nil {
            return err
        }
    }
    return nil
}
```

This "defer in a loop" bug is a standard interview question and a real production failure (file descriptor exhaustion).

> **Trap — `defer` on a `Close()` that returns an error.** `defer f.Close()` silently discards the error. For a file you're *writing*, `Close` is when buffered data is flushed, so discarding it can lose data silently. Capture it:
>
> ```go
> func write(path string) (err error) {
>     f, err := os.Create(path)
>     if err != nil {
>         return err
>     }
>     defer func() {
>         if cerr := f.Close(); cerr != nil && err == nil {
>             err = cerr        // named return — report Close's error if nothing else failed
>         }
>     }()
>     // ... write
>     return nil
> }
> ```

### panic and recover

`panic` unwinds the stack running defers; `recover` inside a deferred function stops the unwinding.

```go
func safe() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    panic("boom")
}
```

**`recover` only works when called directly inside a deferred function.** Not in a function the defer calls, not outside a defer.

**Panics are for programmer errors, not expected failures.** A missing database row is an `error`; a nil map write is a panic. Library code should almost never panic across an API boundary.

> **Critical trap — a panic in a goroutine crashes the entire process.**
>
> ```go
> func main() {
>     defer func() { recover() }()   // does NOT protect the goroutine below
>     go func() { panic("boom") }()  // whole program dies
>     select {}
> }
> ```
>
> Each goroutine has its own stack, so a `recover` in `main` cannot catch a panic in a child goroutine. Every goroutine you spawn that runs untrusted work needs its own recovery:
>
> ```go
> func Go(fn func()) {
>     go func() {
>         defer func() {
>             if r := recover(); r != nil {
>                 slog.Error("goroutine panic", "panic", r, "stack", string(debug.Stack()))
>             }
>         }()
>         fn()
>     }()
> }
> ```
>
> **This is directly relevant to Chronos.** A scheduler that spawns a goroutine per job execution will die entirely if one job panics — taking every other in-flight job with it. A per-job recover that marks the job failed and keeps the scheduler alive is exactly the kind of production reasoning senior interviews reward. Gin's `Recovery()` middleware does this for HTTP handlers, but it only covers the handler's own goroutine, not ones you spawn inside it.

---

## 12. Interfaces

### Implicit satisfaction

```go
type Notifier interface {
    Notify(ctx context.Context, msg string) error
}

type SlackNotifier struct{ webhook string }

// No "implements" keyword — satisfying the method set IS implementing it
func (s *SlackNotifier) Notify(ctx context.Context, msg string) error { return nil }
```

The consequence is architecturally significant: **the consumer defines the interface, not the producer.** You can write an interface for a third-party type you don't control, and interfaces can be added after the fact without touching implementations.

```go
// In YOUR package — you define exactly the 1 method you need from a 40-method client
type ItemIndexer interface {
    Index(ctx context.Context, item Item) error
}

func NewSyncer(idx ItemIndexer) *Syncer { return &Syncer{idx: idx} }
```

> **The Go proverb:** *"Accept interfaces, return structs."* Take an interface as a parameter so callers can substitute; return a concrete type so callers get the full API and you don't force a lowest common denominator.

> **Interface pollution — the anti-pattern to name.** Coming from PHP/Laravel, the instinct is to define an interface for every service and bind it in a container. In Go that's considered poor style. An interface with exactly one implementation, defined in the same package as that implementation, adds indirection for nothing — you can't jump to the definition, and the compiler can't inline through it. Define the interface at the point of *consumption*, when there are two implementations or when you need a test double, and keep it small.

> **The other proverb:** *"The bigger the interface, the weaker the abstraction."* `io.Reader` has one method and is implemented by hundreds of types.

### Interface internals

A non-empty interface value is two words: a pointer to an `itab` (which holds the dynamic type and method pointers) and a pointer to the data. `unsafe.Sizeof(any(nil))` is **16 bytes**, verified.

```go
var s Speaker = &Dog{}
//  ┌──────────┬──────────┐
//  │ itab     │ data ptr │      itab = (type=*Dog, iface=Speaker, methods=[Speak])
//  └──────────┴──────────┘
```

This matters for two reasons: an interface value costs two words, and calling through an interface is an indirect call the compiler usually can't inline. In hot loops that's measurable.

### The nil interface trap — the highest-value question in this file

```go
type MyErr struct{ Code int }
func (e *MyErr) Error() string { return fmt.Sprintf("err %d", e.Code) }

func mayFail(fail bool) error {
    var e *MyErr          // typed nil pointer
    if fail {
        e = &MyErr{Code: 1}
    }
    return e              // ← THE BUG: always returns a non-nil interface
}

err := mayFail(false)
fmt.Println(err == nil)   // false  ← verified
fmt.Println(err)          // <nil>  ← and it PRINTS as nil, which is why it's so confusing
```

That output is real and it's what makes this bug so vicious: `err != nil` is true, but printing `err` shows `<nil>`, so the log tells you nothing is wrong while the code takes the error branch.

**Why:** an interface is nil only when **both** the type word and the data word are nil. Returning a typed nil pointer sets the type word to `*MyErr`, so the interface is non-nil even though the pointer inside it is nil.

**The fix — never declare a concrete error type as a variable you might return:**

```go
func mayFail(fail bool) error {
    if fail {
        return &MyErr{Code: 1}
    }
    return nil            // explicit untyped nil
}
```

> **Follow-ups to expect:**
> - *How do you detect it?* `go vet` catches some cases; `staticcheck` catches more. `errors.Is(err, nil)` does **not** help. In a debugger or with `%#v` you'd see `(*main.MyErr)(nil)`.
> - *Where does this bite in real code?* Any function returning a concrete error type through an `error` interface — including the common pattern of a struct method returning `*ValidationError`. Also `io.Writer` fields holding a typed nil.
> - *Does it apply to non-error interfaces?* Yes, identically. `error` is just the most common case.

### Type assertions and switches

```go
// Assertion — the comma-ok form never panics
if slack, ok := n.(*SlackNotifier); ok {
    slack.webhook = "..."
}

s := n.(*SlackNotifier)   // PANICS if the assertion fails — avoid unless invariant

// Type switch
switch v := err.(type) {
case *ValidationError:
    return 400, v.Fields
case *NotFoundError:
    return 404, nil
case nil:
    return 200, nil
default:
    return 500, nil
}
```

> **Prefer `errors.As` over a type switch for errors** — a type switch only matches the outermost error and fails if the error has been wrapped. See §13.

### `any`

`any` is an alias for `interface{}` (Go 1.18+). Purely cosmetic — identical type, nicer to read.

---

## 13. Errors

### The interface

```go
type error interface {
    Error() string
}
```

That's it. Errors are values, not control flow.

### Sentinel errors

```go
var ErrNotFound = errors.New("not found")     // package-level, comparable by identity

func Get(id string) (*Job, error) {
    if missing {
        return nil, ErrNotFound
    }
}

if errors.Is(err, ErrNotFound) {  }
```

Use `errors.Is`, not `==`, so wrapped errors still match.

### Wrapping

```go
if err := db.QueryRowContext(ctx, q, id).Scan(&j.ID); err != nil {
    return nil, fmt.Errorf("fetching job %s: %w", id, err)   // %w WRAPS
}
```

`%w` preserves the chain so `errors.Is`/`errors.As` can unwrap it. `%v` formats the message but **breaks the chain** — the wrapped error becomes unreachable.

**Message convention:** lowercase, no trailing punctuation, no "failed to" prefix. Each layer adds context, and they concatenate into a readable trace:

```
fetching job abc123: querying database: dial tcp 10.0.1.5:5432: connect: connection refused
```

### Custom error types

```go
type ValidationError struct {
    Field  string
    Reason string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Reason)
}

// Optional: implement Unwrap to participate in a chain
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }
func (e *QueryError) Unwrap() error { return e.Err }
```

### errors.Is vs errors.As

```go
errors.Is(err, ErrNotFound)      // IDENTITY: is this specific error in the chain?

var ve *ValidationError
if errors.As(err, &ve) {         // TYPE: is there an error of this TYPE in the chain?
    fmt.Println(ve.Field)        // and give me it, so I can read its fields
}
```

**`Is` for sentinels, `As` for typed errors you need data from.** Note `As` takes a **pointer to** the target variable — passing `ve` instead of `&ve` panics.

### errors.Join (Go 1.20+)

```go
var errs []error
for _, item := range items {
    if err := validate(item); err != nil {
        errs = append(errs, err)
    }
}
return errors.Join(errs...)    // nil if the slice is empty or all-nil
```

`errors.Is`/`As` traverse joined errors too. Useful for validation where you want every failure, not just the first.

### Error handling strategy

Three choices at every call site, and being deliberate about which is what separates good Go from noisy Go:

```go
// 1. Handle it
if err != nil {
    metrics.Inc("cache.miss")
    return fetchFromOrigin(ctx, id)
}

// 2. Add context and propagate — the common case
if err != nil {
    return fmt.Errorf("loading job %s: %w", id, err)
}

// 3. Deliberately ignore — make it explicit and say why
_ = resp.Body.Close()   // nothing useful to do; the request already succeeded
```

> **Trap — don't wrap at every level.** Wrapping in a five-layer call stack produces `a: b: c: d: e: connection refused`. Add context where it's *informative* (an ID, an operation name), not reflexively. A good rule: wrap when crossing a package boundary, return bare within one.

> **Trap — don't log and return.** Logging an error and also returning it means it gets logged again by the caller, and again above that. Pick one: handle it (log, don't return) or propagate it (return, don't log). Log once, at the top, where you have full context.

> **The interview framing coming from PHP:** you'll be asked to compare with exceptions. The honest answer is a trade-off, not a sermon. Exceptions keep the happy path clean and make it impossible to forget a failure, but they're invisible in signatures and can unwind through code that wasn't ready. Go's errors are explicit and typed into the signature so you can't miss that a call can fail — at the cost of real verbosity, and the compiler still doesn't force you to check them. Saying *"Go's approach makes failure visible but relies on linters for enforcement, which is a weaker guarantee than checked exceptions"* is a balanced, senior answer.

---

## 14. Generics

Added in Go 1.18. Understand them, use them sparingly.

```go
type Number interface {
    ~int | ~int64 | ~float64          // ~ means "any type whose UNDERLYING type is this"
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

Sum([]int{1, 2, 3})            // T inferred as int
Sum([]float64{1.5, 2.5})
```

The `~` matters: without it, `type Celsius float64` would not satisfy the constraint. With it, any named type with that underlying type does.

### Useful built-in constraints

```go
import "cmp"

func Max[T cmp.Ordered](a, b T) T {    // cmp.Ordered: any type supporting < > <= >=
    if a > b {
        return a
    }
    return b
}

any                    // no constraint
comparable             // supports == and != — required for map keys
```

### A genuinely useful generic

```go
func MapSlice[T, U any](in []T, fn func(T) U) []U {
    out := make([]U, 0, len(in))
    for _, v := range in {
        out = append(out, fn(v))
    }
    return out
}

skus := MapSlice(items, func(i Item) string { return i.SKU })
```

### Generic types

```go
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{items: make(map[K]V)}
}

func (c *Cache[K, V]) Get(k K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[k]
    return v, ok
}
```

Note the zero-value idiom for a generic return:

```go
func (c *Cache[K, V]) MustGet(k K) V {
    v, ok := c.Get(k)
    if !ok {
        var zero V      // you cannot write `return nil` — V might be int
        return zero
    }
    return v
}
```

### When generics make things worse

> **This is the actual interview question** — anyone can write a generic `Map`. The judgement is knowing the costs:
>
> 1. **Methods cannot have type parameters.** `func (c *Cache[K,V]) MapTo[U any](...)` is illegal. This kills a lot of fluent APIs and is the most common wall people hit.
> 2. **No specialisation on type.** You cannot write "if T is a string do X." You need a constraint with a method, or reflection, or just two functions.
> 3. **Implementation is GC-shape stenciling** — one instantiation per pointer-shaped type group, with a dictionary passed at runtime. So generics are *not* reliably faster than `any`+assertion, and can be slower than a hand-written monomorphic version.
> 4. **Readability cost is real.** `func Process[T Constraint, U any, K comparable](...)` is genuinely harder to read than three concrete functions.
>
> The rule the Go team publishes: **write the concrete version first; introduce a type parameter only when you're writing the same function for the third time.** Interfaces remain the right tool when you need *behaviour* polymorphism; generics are for *container and algorithm* polymorphism where the types are otherwise unrelated.

---

## 15. Standard Library Tour

Knowing the standard library well is a distinguishing Go signal, because the ecosystem norm is to reach for it before a dependency.

| Package | Use | Interview-worthy detail |
|---------|-----|------------------------|
| `context` | Cancellation, deadlines | First parameter, always. Never store in a struct |
| `errors` | `Is`/`As`/`Join`/`Unwrap` | `As` needs a pointer to the target |
| `fmt` | Formatting | `%w` wraps, `%v` doesn't; `%+v` prints struct field names |
| `sync` | Mutex, WaitGroup, Once, Pool | `WaitGroup.Go` added in 1.25 |
| `sync/atomic` | Lock-free counters | Prefer typed `atomic.Int64` over the function forms |
| `time` | Time, durations, timers | `time.Time` comparison — use `Equal`, not `==` |
| `net/http` | HTTP client & server | Enhanced routing patterns in 1.22 |
| `encoding/json` | JSON | Unexported fields are invisible; `json/v2` is experimental in 1.26 |
| `database/sql` | DB abstraction | It's a **pool**, not a connection |
| `io` | `Reader`/`Writer` | The two most important interfaces in Go |
| `log/slog` | Structured logging | Added 1.21 — use this, not `log` |
| `slices`, `maps` | Generic helpers | Added 1.21; `slices.Sorted(maps.Keys(m))` |
| `cmp` | Ordering | `cmp.Compare`, `cmp.Or` |
| `testing` | Tests, benchmarks, fuzzing | `b.Loop()` from 1.24 is the correct benchmark form |
| `testing/synctest` | Deterministic concurrency tests | Stable in 1.25 — see Tier 2 |

### `time` gotchas worth knowing

```go
t1 == t2                      // WRONG — compares monotonic clock and location too
t1.Equal(t2)                  // correct

time.Now().Sub(start)         // fine
time.Since(start)             // clearer

// Timers must be stopped or they leak until they fire
timer := time.NewTimer(5 * time.Minute)
defer timer.Stop()

// time.After in a select inside a loop leaks a timer per iteration until it fires
for {
    select {
    case <-ch:
    case <-time.After(time.Second):   // allocates a new timer EVERY loop iteration
    }
}
```

> As of Go 1.23 the garbage collector can collect unstopped timers, so `time.After` leaks are much less severe than they used to be. Still worth knowing the historical answer, because interviewers ask it and the "old" answer shows you understand *why* it was a problem.

### `io` composition

```go
var r io.Reader = strings.NewReader("hello")
r = io.LimitReader(r, 100)                  // cap the read
h := sha256.New()
r = io.TeeReader(r, h)                      // hash while reading
io.Copy(io.Discard, r)
```

Small interfaces composing into pipelines is Go's core design idea in one example — a good thing to point at when asked "what do you like about Go?"

---

## 16. Tooling

```bash
gofmt -l .                 # list unformatted files — formatting is NOT a debate in Go
go vet ./...               # correctness heuristics: printf args, lock copies, bad tags
staticcheck ./...          # much deeper static analysis
golangci-lint run          # meta-linter, what most teams run in CI

go test ./... -race                          # race detector — run in CI, always
go test ./... -cover -coverprofile=c.out
go tool cover -html=c.out

go build -gcflags='-m' ./...     # escape analysis decisions
go test -bench=. -benchmem       # benchmarks with allocation counts
go tool pprof                    # profiling — see Tier 4
```

> **`go vet` findings to know by name**, because interviewers ask "what does vet catch?":
> - `printf`: format string doesn't match arguments
> - `copylocks`: copying a value containing a `sync.Mutex` (a real bug — you get two independent locks)
> - `lostcancel`: `context.WithCancel` whose `cancel` is never called (a goroutine leak)
> - `structtag`: malformed struct tags
> - `loopclosure`: pre-1.22 loop variable capture

> **The cultural point worth making:** `gofmt` ended formatting arguments permanently, and the standard toolchain ships testing, benchmarking, profiling, race detection, and fuzzing. Contrast with PHP where you assemble PHPUnit/Pest, PHP-CS-Fixer, PHPStan, and Xdebug yourself. When asked "what do you like about Go," the integrated toolchain is a more thoughtful answer than "goroutines."

---

## 17. Tier 1 Q&A Drill

### Types, constants, strings

**1. Why does Go require explicit conversion between `int` and `int64`?**
They're distinct named types. Conversions can lose precision, and Go makes every one of them visible at the call site rather than silent.

**2. What's the difference between `type A int` and `type A = int`?**
The first is a new distinct type with its own method set. The second is an alias — literally the same type, freely interchangeable.

**3. What is an untyped constant?**
A constant with arbitrary precision and no type until assigned. `1 << 40` is legal even on a 32-bit target as long as you never assign it to something too small — overflow is a compile-time error.

**4. `len("héllo")` — what and why?**
6. `len` counts bytes and `é` is two bytes in UTF-8. Rune count is `len([]rune(s))` or `utf8.RuneCountInString(s)`.

**5. What does `for i, r := range s` give you for a string?**
`r` is a decoded rune; `i` is the **byte** offset, so it can jump by more than one.

**6. Why is `s += "x"` in a loop bad?**
Strings are immutable, so each `+=` allocates and copies — O(n²). Use `strings.Builder` with `Grow`.

**7. Does `[]byte(s)` allocate?**
Yes, it copies. The compiler elides it in a few specific patterns (map lookup, comparison, range) but assume a copy.

### Slices and maps

**8. What are the three fields of a slice header?**
Pointer to the backing array, length, capacity. 24 bytes on 64-bit.

**9. Why can a function modify a slice's elements but not its length?**
The header is passed by value. The pointer inside it is shared (so element writes are visible), but reassigning `len` changes only the local copy.

**10. `full := []int{1,2,3}; v := full[0:2]; v = append(v, 99)` — what is `full`?**
`[1 2 99]`. Slicing doesn't reduce capacity, so `v` has cap 3, `append` fits in place, and it overwrites `full[2]`.

**11. How do you prevent that?**
Three-index slicing `full[0:2:2]` to cap the capacity, forcing `append` to allocate. Or `slices.Clone` if you want fully independent memory.

**12. Describe slice growth.**
Doubling while small (under 256 elements), tapering toward ~1.25× for large slices, then rounded up to an allocator size class. Exact numbers vary by release — don't memorise them.

**13. nil slice vs empty slice?**
Identical for `len`, `range`, and `append`. Differ for `== nil` and JSON: nil marshals to `null`, empty to `[]`. Return empty from API layers.

**14. How do you delete from a slice without leaking?**
`slices.Delete`, which zeroes the vacated tail. Manually, zero the last element before reslicing, or the backing array keeps the pointer alive.

**15. Why is map iteration order random?**
Deliberately randomised by the runtime so code can't accidentally depend on it. Sort keys for deterministic output.

**16. Why can't you do `m["k"].Field = v`?**
Map values aren't addressable — the map may rehash and move them. Read-modify-write, or store pointers.

**17. What happens on a concurrent map write?**
`fatal error: concurrent map writes` — a runtime `throw`, not a panic, and it **cannot be recovered**. Concurrent reads alone are safe.

**18. When would you use `sync.Map`?**
Only two cases: write-once-read-many, or goroutines touching disjoint key sets. Otherwise a plain map with `RWMutex` is faster and type-safe.

**19. Can a struct containing a slice be a map key?**
No. Map keys must be comparable, and slices aren't, so neither is a struct containing one.

**20. Why does `map[string]struct{}` beat `map[string]bool` for a set?**
`struct{}` is zero bytes, so the values cost nothing. It also documents that the value is meaningless.

### Structs, pointers, methods

**21. Is embedding inheritance?**
No. It's composition with automatic field/method promotion. There's no subtyping — `Job` embedding `Base` cannot be passed where `Base` is expected.

**22. What's the risk of embedding an interface in a struct?**
The outer type satisfies the interface without implementing anything. Unimplemented methods forward to the embedded value, and if it's nil you get a runtime panic instead of a compile error.

**23. State the method set rule.**
`*T`'s method set includes both value- and pointer-receiver methods. `T`'s includes only value-receiver methods.

**24. Why does `c.Inc()` work on a value but `var s Speaker = Dog{}` not?**
`c` is addressable, so the compiler inserts `&c`. Boxing into an interface copies the value, so auto-addressing would take the address of a temporary and mutations would be invisible. The compiler refuses rather than allow that.

**25. Name three non-addressable things.**
Map elements, the result of a type assertion, composite literals, and return values. Slice elements *are* addressable.

**26. When do you use a pointer receiver?**
When the method mutates, when the struct is large, or when it contains a mutex. Be consistent across the type.

**27. Is everything in Go passed by value?**
Yes, including pointers and slice/map/channel headers. There are no reference parameters.

### Functions, defer, panic

**28. When are deferred function arguments evaluated?**
At the point the `defer` statement executes, not when the deferred call runs.

**29. In what order do defers run?**
LIFO, when the enclosing *function* returns — not at end of block.

**30. Why is `defer f.Close()` in a loop a bug?**
Defers accumulate until the function returns, so you hold every file handle open at once. Wrap the body in a closure.

**31. `func f() (n int) { defer func(){ n++ }(); return 5 }` returns?**
6. `return 5` assigns to the named result, then defers run and can modify it. With an unnamed result it would return 5.

**32. What changed about loop variables in Go 1.22?**
The loop variable is now per-iteration rather than shared, fixing the classic `go func(){ print(i) }()` bug. Pre-1.22 you'd typically see `3 3 3`; now you get `0 1 2` in some order. Controlled by the `go` directive in `go.mod`.

**33. Where must `recover` be called?**
Directly inside a deferred function of the panicking goroutine. Not nested deeper, not in another goroutine.

**34. What happens if a goroutine panics and nothing recovers?**
The whole process dies. A `recover` in `main` does not protect child goroutines — each needs its own.

**35. When should you panic?**
Programmer errors and truly unrecoverable states. Expected failures are errors. Library code shouldn't panic across its API boundary.

### Interfaces and errors

**36. How does a type implement an interface in Go?**
Implicitly, by having the methods. There's no declaration, which lets consumers define interfaces for types they don't own.

**37. "Accept interfaces, return structs" — why?**
Accepting an interface lets callers substitute implementations; returning a concrete type gives callers the full API instead of a lowest common denominator.

**38. What is interface pollution?**
Defining an interface for every type, especially single-implementation interfaces next to their implementation. Idiomatic Go defines interfaces at the point of consumption.

**39. Why is a nil `*MyErr` returned as `error` not equal to nil?**
An interface is nil only when both its type word and data word are nil. A typed nil sets the type word, so the interface is non-nil — and it still *prints* as `<nil>`, which is what makes it so confusing.

**40. How do you avoid that bug?**
Never return a concrete-typed error variable through an `error` return. Return a literal `nil` on the success path.

**41. `errors.Is` vs `errors.As`?**
`Is` tests identity against a sentinel through the wrap chain. `As` finds an error of a given type in the chain and assigns it, so you can read its fields. `As` needs a pointer to the target.

**42. `%w` vs `%v` in `fmt.Errorf`?**
`%w` wraps and keeps the chain traversable by `Is`/`As`. `%v` just formats the message and severs the chain.

**43. Should you log and return an error?**
No — you'll get duplicate logs at every level. Either handle it (log, don't return) or propagate it (return, don't log).

**44. Compare Go errors with exceptions honestly.**
Errors are visible in the signature so you can't miss that a call fails, and they're ordinary values you can wrap and inspect. The costs are verbosity and that nothing forces you to check them — enforcement is a linter, not the compiler, which is weaker than checked exceptions.

### Generics and tooling

**45. What does `~int` mean in a constraint?**
Any type whose *underlying* type is `int`, so `type Celsius int` satisfies it. Without `~`, only `int` itself does.

**46. Can methods have type parameters?**
No. Only functions and types. This is the most common limitation people hit.

**47. Are generics faster than `interface{}`?**
Not reliably. Go uses GC-shape stenciling with runtime dictionaries, so pointer-shaped types share one instantiation. Benchmark rather than assume.

**48. When should you not use generics?**
Until you've written the same function three times. Interfaces are still right for behaviour polymorphism; generics are for containers and algorithms over unrelated types.

**49. Name four things `go vet` catches.**
`printf` argument mismatches, `copylocks` (copying a mutex), `lostcancel` (uncalled context cancel), and malformed struct tags.

**50. Why is `CGO_ENABLED=0` common in Dockerfiles?**
It forces pure-Go implementations so the binary is genuinely static and runs in `scratch`/distroless images without libc.

---

> **The Tier 1 senior signal:** none of these are trivia. Slice aliasing is a real data-corruption bug, the nil interface is a real production incident, and method sets explain a compile error you'll hit weekly. Interviewers ask them because the answers reveal whether you understand Go's memory model or have been pattern-matching syntax. Being able to say *why* the language behaves this way — addressability, two-word interfaces, headers passed by value — is what separates a Tier 1 pass from a Tier 1 distinction.

---

**Next:** [`02-concurrency.md`](./02-concurrency.md) — goroutines, channels, `select`, the `sync` package, `context`, the memory model, concurrency patterns, and how to find a goroutine leak.

**Back to:** [`README.md`](./README.md)
