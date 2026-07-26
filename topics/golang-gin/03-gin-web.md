# Go — Tier 3: HTTP, Gin & Data Access

> Verified against **Go 1.26** and **Gin v1.12.0**.
>
> Gin is a thin layer over `net/http`. Interviewers know this, so questions usually start at the Gin API and immediately drill down to what the standard library is doing underneath. If you can only talk about `gin.Context`, that's a ceiling. Learn `net/http` first — this file is ordered that way deliberately.

---

## Table of Contents

1. [net/http Foundations](#1-nethttp-foundations)
2. [Server Configuration & Timeouts](#2-server-configuration--timeouts)
3. [Gin: Engine, Router & Context](#3-gin-engine-router--context)
4. [Middleware](#4-middleware)
5. [Binding & Validation](#5-binding--validation)
6. [Error Handling Strategy](#6-error-handling-strategy)
7. [Project Layout](#7-project-layout)
8. [database/sql](#8-databasesql)
9. [Transactions](#9-transactions)
10. [pgx, sqlx, GORM — Choosing](#10-pgx-sqlx-gorm--choosing)
11. [Redis & Caching](#11-redis--caching)
12. [gRPC](#12-grpc)
13. [Testing](#13-testing)
14. [Tier 3 Q&A Drill](#14-tier-3-qa-drill)

---

## 1. net/http Foundations

### The Handler interface

Everything in Go's HTTP stack is this one interface:

```go
type Handler interface {
    ServeHTTP(w http.ResponseWriter, r *http.Request)
}

type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) { f(w, r) }
```

`HandlerFunc` is the adapter pattern in four lines: it gives a plain function the method set of a `Handler`. Middleware, routers, and Gin itself are all just `Handler` implementations that wrap other `Handler`s. Being able to say *"Gin's `Engine` is an `http.Handler` — that's the whole integration story"* is the right level of understanding.

```go
func hello(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")   // headers BEFORE WriteHeader
    w.WriteHeader(http.StatusOK)                          // status BEFORE body
    json.NewEncoder(w).Encode(map[string]string{"msg": "hi"})
}

http.ListenAndServe(":8080", http.HandlerFunc(hello))
```

> **Trap — ordering on `ResponseWriter` is strict and silent.** Headers set after `WriteHeader` are discarded. Writing the body implicitly calls `WriteHeader(200)`, so a later explicit `WriteHeader(500)` logs `superfluous response.WriteHeader call` and does nothing. This is a real bug pattern: an error occurs mid-encoding, you try to switch to a 500, and the client already received a 200 with truncated JSON. The fix is to encode into a buffer first, or to decide the status before writing anything.

### One goroutine per connection

The server spawns a goroutine per connection, so **your handler runs concurrently with every other handler**. Any state a handler touches outside its own stack must be safe for concurrent use. This is the point where PHP habits break — there's no per-request isolation.

### Routing (Go 1.22+)

The standard `ServeMux` gained method matching and wildcards in 1.22, which narrowed the gap with third-party routers considerably:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /jobs/{id}", getJob)
mux.HandleFunc("POST /jobs", createJob)
mux.HandleFunc("GET /files/{path...}", serveFile)    // trailing wildcard

func getJob(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
}
```

> **Follow-up you should expect: "why use Gin at all now?"** A fair question since 1.22. The honest answer: `net/http` still lacks route groups, a middleware chain abstraction, binding/validation, and response helpers — so with the standard library you assemble those yourself or pull in `chi`. Gin gives you all of it plus a faster router, at the cost of a non-standard `Context` type that couples your handlers to the framework. For a small service I'd now consider `net/http` + `chi`; for a large API with lots of binding and validation, Gin still earns its place. Answering with a trade-off rather than "Gin is faster" is the senior move.

---

## 2. Server Configuration & Timeouts

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           router,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      15 * time.Second,
    IdleTimeout:       60 * time.Second,
    MaxHeaderBytes:    1 << 20,
}
```

> **`http.ListenAndServe(":8080", handler)` has NO timeouts at all.** Every one is infinite. A single slow client can hold a connection open forever, and enough of them exhaust your file descriptors. This is [Slowloris](https://en.wikipedia.org/wiki/Slowloris_(cyber_attack)) and it's the most commonly cited "production Go mistake." Never use the convenience function for a real server.

| Timeout | Covers | Protects against |
|---|---|---|
| `ReadHeaderTimeout` | Reading request headers | Slowloris — the most important one |
| `ReadTimeout` | Headers **+ body** | Slow or stalled uploads |
| `WriteTimeout` | Start of read → end of write | Slow clients, slow handlers |
| `IdleTimeout` | Between keep-alive requests | Idle connections hogging FDs |

> **Trap — `WriteTimeout` breaks streaming.** It's an absolute deadline from when the request is read, so server-sent events, long polling, and large downloads get cut off mid-stream. For those routes set `WriteTimeout: 0` and control the deadline per-request with `http.ResponseController` (1.20+) or `context.WithTimeout`.

### The client needs timeouts too

```go
// http.DefaultClient has NO timeout — a hung server hangs your service forever
client := &http.Client{
    Timeout: 10 * time.Second,           // total: dial + TLS + request + body read
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 100,        // default is 2 — a common throughput bug
        IdleConnTimeout:     90 * time.Second,
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout: 5 * time.Second,
    },
}
```

> **Two traps worth knowing:**
> - **`MaxIdleConnsPerHost` defaults to 2.** If your service calls one downstream host heavily, you'll constantly tear down and re-establish connections — including TLS handshakes. Raising it is often a dramatic latency win and it's an excellent "small change, big impact" story.
> - **You must always close and drain the response body**, or the connection can't be reused:
>   ```go
>   defer resp.Body.Close()
>   defer io.Copy(io.Discard, resp.Body)   // drain so the connection returns to the pool
>   ```
>   Closing without draining a partially-read body causes the transport to discard the connection.

---

## 3. Gin: Engine, Router & Context

```go
func main() {
    gin.SetMode(gin.ReleaseMode)          // gin.DebugMode is the default and is noisy + slower

    r := gin.New()                         // NO middleware
    // r := gin.Default()                  // = New() + Logger() + Recovery()

    r.Use(gin.Recovery())
    r.Use(RequestID(), StructuredLogger())

    v1 := r.Group("/api/v1")
    v1.Use(AuthRequired())
    {
        jobs := v1.Group("/jobs")
        jobs.GET("", listJobs)
        jobs.POST("", createJob)
        jobs.GET("/:id", getJob)
        jobs.DELETE("/:id", deleteJob)
    }

    srv := &http.Server{Addr: ":8080", Handler: r, ReadHeaderTimeout: 5 * time.Second}
    srv.ListenAndServe()
}
```

`gin.Engine` implements `http.Handler`, which is why it slots into a standard `http.Server` — and why you should always construct the `Server` yourself rather than calling `r.Run()`, which uses `ListenAndServe` and therefore has no timeouts.

### The router

Gin uses a **radix tree** (compressed prefix tree) per HTTP method. Lookup is O(length of path) with no allocations and no backtracking, which is where the "fast router" claim comes from — it doesn't test routes in sequence like a regex-based router.

Route precedence: static segments beat `:param` which beats `*wildcard`.

> **Trap — conflicting wildcards panic at startup, not at request time.** Registering both `/jobs/:id` and `/jobs/new` is fine (static wins), but `/jobs/:id` and `/jobs/:name` panics because a node can't have two different parameter names at the same position. Failing at boot is deliberate and good — it's a startup crash, not a mystery 404.

### gin.Context

`*gin.Context` bundles the request, response writer, path params, middleware chain state, and a per-request key/value store.

```go
func getJob(c *gin.Context) {
    id := c.Param("id")                          // path parameter
    page := c.DefaultQuery("page", "1")          // query string
    orgID := c.GetInt64("org_id")                // value set by middleware

    ctx := c.Request.Context()                   // the real context.Context

    job, err := svc.Get(ctx, orgID, id)
    if err != nil {
        c.Error(err)
        return
    }
    c.JSON(http.StatusOK, job)
}
```

> **`gin.Context` is not a `context.Context` you should pass to your domain layer.** It does implement the interface, but passing it downstream couples your service and repository packages to Gin — and its `Done()` behaviour has historically been surprising. **Always pass `c.Request.Context()`** to anything below the handler. Your service signatures should never mention Gin.

### The copy-for-goroutine trap

This is the Gin-specific question that gets asked most.

```go
// BROKEN — the context is recycled when the handler returns
func handler(c *gin.Context) {
    go func() {
        time.Sleep(5 * time.Second)
        log.Println(c.Param("id"))    // reading a recycled, possibly-reused Context
    }()
    c.JSON(200, gin.H{"status": "accepted"})
}

// CORRECT
func handler(c *gin.Context) {
    cc := c.Copy()                    // detached copy safe for use after the handler returns
    go func() {
        time.Sleep(5 * time.Second)
        log.Println(cc.Param("id"))
    }()
    c.JSON(200, gin.H{"status": "accepted"})
}
```

**Why:** Gin pools `Context` objects in a `sync.Pool` and reuses them across requests. Once your handler returns, that object may already be serving a different request. `Copy()` returns a detached copy with the keys and params preserved and the writer disabled.

> **The important follow-up, and the better answer:** `Copy()` fixes the data race, but the goroutine is *still* wrong in two other ways. The request context is cancelled when the handler returns, so any DB or HTTP call using it fails instantly; and a panic in that goroutine kills the whole process because Gin's `Recovery()` only wraps the handler's own goroutine.
>
> The genuinely correct pattern is to not spawn ad-hoc goroutines from handlers at all — hand the work to a queue or a long-lived worker pool with its own lifecycle and its own recover:
>
> ```go
> func handler(c *gin.Context) {
>     job := BuildJob(c)                       // extract plain data, no Gin types
>     if err := queue.Enqueue(c.Request.Context(), job); err != nil {
>         c.Error(err)
>         return
>     }
>     c.JSON(http.StatusAccepted, gin.H{"job_id": job.ID})
> }
> ```
>
> Saying "`c.Copy()`" answers the question. Saying "`c.Copy()`, but I'd actually enqueue it because the context is dead and a panic would take down the process" answers it at senior level.

---

## 4. Middleware

```go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := c.GetHeader("X-Request-ID")
        if id == "" {
            id = uuid.NewString()
        }

        c.Set("request_id", id)
        c.Header("X-Request-ID", id)

        // Put it in the real context so it propagates to the DB and downstream calls
        ctx := context.WithValue(c.Request.Context(), requestIDKey{}, id)
        c.Request = c.Request.WithContext(ctx)

        c.Next()
    }
}
```

### Ordering and control flow

`c.Next()` runs the rest of the chain and returns, so code after it runs on the way *out* — the same onion as Laravel middleware.

```go
func StructuredLogger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path

        c.Next()                          // ← everything downstream runs here

        slog.Info("request",
            "method", c.Request.Method,
            "path", path,
            "status", c.Writer.Status(),
            "duration_ms", time.Since(start).Milliseconds(),
            "size", c.Writer.Size(),
            "request_id", c.GetString("request_id"),
            "errors", c.Errors.String(),
        )
    }
}
```

> **Trap — capture the path *before* `c.Next()`.** Middleware or handlers can rewrite `c.Request.URL.Path`, so reading it afterwards can log the wrong route. Same for anything else you want to record as it was on the way in.

**`c.Abort()` vs `return`:**

```go
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims, err := parseToken(c.GetHeader("Authorization"))
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return                       // ← Abort alone does NOT stop this function
        }

        c.Set("user_id", claims.UserID)
        c.Set("org_id", claims.OrgID)
        c.Next()
    }
}
```

`Abort()` sets a flag that prevents *subsequent* handlers from running. It does **not** return from the current function — you still need `return`, or the rest of your middleware body executes. Forgetting the `return` after `Abort` is a classic bug that produces "response written twice" symptoms.

### Recovery

```go
r.Use(gin.CustomRecovery(func(c *gin.Context, recovered any) {
    slog.Error("panic recovered",
        "panic", recovered,
        "stack", string(debug.Stack()),
        "path", c.Request.URL.Path,
        "request_id", c.GetString("request_id"),
    )
    c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
}))
```

Remember from Tier 2: this protects the handler's goroutine only. Goroutines you spawn need their own recover.

### Tenant scoping — tying to your SaaS experience

```go
func TenantScope() gin.HandlerFunc {
    return func(c *gin.Context) {
        orgID := c.GetInt64("org_id")        // set by AuthRequired
        if orgID == 0 {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "no organization"})
            return
        }

        ctx := tenant.NewContext(c.Request.Context(), tenant.OrgID(orgID))
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}
```

> **The interesting comparison to draw.** In Laravel you enforced tenancy with an Eloquent global scope — the ORM silently added `WHERE organization_id = ?` to every query, which is convenient but means a raw query or a forgotten `withoutGlobalScope` bypasses it invisibly.
>
> Go has no ORM hook to lean on, so you need explicit enforcement, and the idiomatic approach is to make the *type system* carry it:
>
> ```go
> // The repository cannot be called without a tenant — it's in the signature.
> func (r *JobRepo) List(ctx context.Context, org tenant.OrgID, f Filter) ([]Job, error)
> ```
>
> Combined with `type OrgID int64` from Tier 1, a missing or transposed tenant ID becomes a **compile error** rather than a silently-wrong query. The trade-off is honest and worth stating: Laravel's approach is less boilerplate but fails open; Go's is more verbose but fails at build time. That's a genuinely strong multi-tenancy answer because it shows you've implemented it both ways and can compare.
>
> The belt-and-braces version is PostgreSQL Row-Level Security, which enforces it in the database regardless of application code — the same "the database is the only real enforcement point" reasoning as unique constraints in the PHP material.

---

## 5. Binding & Validation

```go
type CreateJobRequest struct {
    Name     string            `json:"name" binding:"required,min=3,max=100"`
    Schedule string            `json:"schedule" binding:"required,cron"`
    Timeout  int               `json:"timeout_seconds" binding:"required,min=1,max=3600"`
    Retries  int               `json:"retries" binding:"gte=0,lte=10"`
    Email    string            `json:"email" binding:"omitempty,email"`
    Payload  map[string]string `json:"payload" binding:"omitempty,max=50"`
}

func createJob(c *gin.Context) {
    var req CreateJobRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": validationDetails(err)})
        return
    }
    // req is now valid
}
```

### ShouldBind vs Bind

| | Behaviour on error |
|---|---|
| `c.Bind(&x)` | Aborts with 400 and writes a plain-text error **for you** |
| `c.ShouldBind(&x)` | Returns the error, writes nothing — **you** control the response |

**Always use `ShouldBind*` in a real API.** `Bind` produces an inconsistent error format that won't match your envelope.

Variants: `ShouldBindJSON`, `ShouldBindQuery`, `ShouldBindUri`, `ShouldBindHeader`, `ShouldBindWith`.

> **Trap — you cannot call `ShouldBindJSON` twice.** The request body is an `io.ReadCloser`, a stream that's consumed on first read. The second call sees EOF and returns an empty struct with no error, which is a silently wrong result. If you genuinely need to read it twice, use `c.ShouldBindBodyWith(&x, binding.JSON)`, which caches the body in the context.

### Readable validation errors

The default `validator` error is unusable in an API response. Convert it:

```go
type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func validationDetails(err error) []FieldError {
    var ve validator.ValidationErrors
    if !errors.As(err, &ve) {
        return []FieldError{{Field: "_", Message: "malformed request body"}}
    }

    out := make([]FieldError, 0, len(ve))
    for _, fe := range ve {
        out = append(out, FieldError{
            Field:   toSnakeCase(fe.Field()),
            Message: messageFor(fe),
        })
    }
    return out
}

func messageFor(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return "is required"
    case "min":
        return "must be at least " + fe.Param()
    case "max":
        return "must be at most " + fe.Param()
    case "email":
        return "must be a valid email address"
    default:
        return "is invalid"
    }
}
```

### Custom validators

```go
func init() {
    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        v.RegisterValidation("cron", func(fl validator.FieldLevel) bool {
            _, err := cron.ParseStandard(fl.Field().String())
            return err == nil
        })
    }
}
```

> **Trap — validation cannot enforce tenant ownership.** The same limitation as Laravel's `exists:` rule from the PHP material: a validator can check that a UUID is well-formed, but confirming the referenced row belongs to *this* organisation is a database query that must be tenant-scoped, and it's a point-in-time check that says nothing about contended state. Keep authorisation in the service layer and integrity in the database.

### Pointers for optional fields

```go
type UpdateJobRequest struct {
    Name    *string `json:"name"`         // nil = not provided; "" = explicitly cleared
    Enabled *bool   `json:"enabled"`      // nil = not provided; false = explicitly disabled
}
```

Without pointers you cannot distinguish "field absent" from "field set to the zero value" — which for a PATCH endpoint means you can never disable a boolean. This is the Go equivalent of the `$request->has()` vs `$request->input()` distinction, and it's a common interview question about partial updates.

---

## 6. Error Handling Strategy

### Domain errors, mapped once at the edge

```go
// Domain layer — no HTTP knowledge whatsoever
var (
    ErrNotFound     = errors.New("not found")
    ErrConflict     = errors.New("conflict")
    ErrUnauthorized = errors.New("unauthorized")
)

type ValidationError struct {
    Field  string
    Reason string
}

func (e *ValidationError) Error() string { return e.Field + ": " + e.Reason }
```

```go
// Transport layer — the ONLY place that knows about status codes
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) == 0 {
            return
        }
        err := c.Errors.Last().Err

        status, body := mapError(err)

        if status >= 500 {
            slog.Error("request failed",
                "err", err,
                "path", c.Request.URL.Path,
                "request_id", c.GetString("request_id"))
        }

        c.JSON(status, body)
    }
}

func mapError(err error) (int, gin.H) {
    var ve *ValidationError
    switch {
    case errors.Is(err, ErrNotFound):
        return http.StatusNotFound, gin.H{"error": "not found"}
    case errors.Is(err, ErrConflict):
        return http.StatusConflict, gin.H{"error": "conflict"}
    case errors.Is(err, ErrUnauthorized):
        return http.StatusUnauthorized, gin.H{"error": "unauthorized"}
    case errors.As(err, &ve):
        return http.StatusBadRequest, gin.H{"error": ve.Reason, "field": ve.Field}
    case errors.Is(err, context.DeadlineExceeded):
        return http.StatusGatewayTimeout, gin.H{"error": "upstream timeout"}
    default:
        return http.StatusInternalServerError, gin.H{"error": "internal error"}
    }
}
```

Handlers become trivial:

```go
func getJob(c *gin.Context) {
    job, err := svc.Get(c.Request.Context(), c.Param("id"))
    if err != nil {
        c.Error(err)      // record it; the middleware decides the response
        return
    }
    c.JSON(http.StatusOK, job)
}
```

> **Why this design wins points:** status-code mapping lives in exactly one function, the domain layer has no HTTP dependency, and it's `errors.Is`/`As` based so it works through wrapping. It's the Go equivalent of Laravel's exception handler, and drawing that parallel explicitly is a good way to show transferable thinking.

> **Trap — never leak internal errors to clients.** `c.JSON(500, gin.H{"error": err.Error()})` can expose SQL fragments, table names, internal hostnames, and file paths. Log the detail, return a generic message plus the request ID so support can correlate. This is OWASP "security misconfiguration" and interviewers do check for it.

---

## 7. Project Layout

```
cmd/
  api/main.go              # wiring only: config, deps, server, shutdown
  worker/main.go
internal/                  # ← the compiler ENFORCES this is private to the module
  job/
    job.go                 # domain types and errors
    service.go             # business logic; takes interfaces
    repository.go          # the interface the service needs
    postgres.go            # the implementation
  http/
    handler.go
    middleware.go
    router.go
  platform/
    database/
    cache/
pkg/                       # only if you genuinely publish something reusable
migrations/
```

`internal/` is a real language feature, not a convention: packages under it can only be imported by code rooted at its parent. It's the strongest encapsulation Go offers and it's the right default for a service.

> **Trap — don't cargo-cult "golang-standards/project-layout."** It's not official and it's widely criticised for pushing Java-style structure onto small services. A single `main.go` plus a couple of packages is perfectly idiomatic for a small service. Say *"I structure by domain, not by layer, and I only split when a package gets a second reason to change"* rather than reciting a directory tree.

Wiring in `main`:

```go
func main() {
    cfg := config.Load()

    db, err := sql.Open("pgx", cfg.DatabaseURL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    repo := job.NewPostgresRepo(db)          // concrete
    svc := job.NewService(repo)              // takes the interface
    h := httpapi.NewHandler(svc)
    r := httpapi.NewRouter(h)

    srv := &http.Server{Addr: cfg.Addr, Handler: r, ReadHeaderTimeout: 5 * time.Second}
    // ... graceful shutdown from Tier 2 §9
}
```

Explicit constructor wiring in `main` is idiomatic Go. There's no DI container, and reaching for one (`wire`, `fx`) on a small service is usually a step backwards — this is the biggest structural difference from Laravel and worth naming as a deliberate choice rather than a gap.

---

## 8. database/sql

### It is a pool, not a connection

```go
db, err := sql.Open("pgx", dsn)   // does NOT connect — lazy
if err != nil {
    return err
}
if err := db.PingContext(ctx); err != nil {   // this actually connects
    return err
}
```

`*sql.DB` is a **connection pool**, safe for concurrent use, and meant to be created once and shared for the process's lifetime. Opening one per request is a serious bug that shows up as connection exhaustion.

### Pool tuning — expect to justify every number

```go
db.SetMaxOpenConns(25)                   // hard ceiling on concurrent connections
db.SetMaxIdleConns(25)                   // keep them warm; match MaxOpenConns
db.SetConnMaxLifetime(5 * time.Minute)   // recycle: helps failover and load balancers
db.SetConnMaxIdleTime(1 * time.Minute)   // release idle capacity
```

> **How to reason about `MaxOpenConns` out loud:**
>
> Start from the *database's* limit, not the application's appetite. Postgres `max_connections` is commonly 100, and each connection costs several MB of server memory, so the total across all clients must fit. With 4 API replicas, 2 worker replicas, and headroom for migrations and a human with `psql`, roughly `(100 - 20) / 6 ≈ 13` per instance. Then check it against throughput: by Little's Law, `connections = throughput × latency`, so 500 req/s at 20 ms of database time needs about 10 concurrent connections. If the two numbers disagree, the database limit wins and you add PgBouncer.
>
> **Set `MaxIdleConns` equal to `MaxOpenConns`.** The default is 2, so a burst opens connections and then immediately closes them back down to 2 — you pay TCP and TLS setup repeatedly under exactly the load where you can least afford it. This mismatch is a genuinely common production issue and a good thing to mention.
>
> **`ConnMaxLifetime` matters for failover.** Without it, connections pinned to a demoted primary survive indefinitely. Five minutes bounds how long you can be talking to the wrong node after an RDS failover.

This maps directly onto your PgBouncer and connection-pooling work from the Laravel side — same reasoning, different client library.

### Querying

```go
func (r *Repo) GetJob(ctx context.Context, org OrgID, id string) (*Job, error) {
    const q = `SELECT id, name, schedule, next_run_at
               FROM jobs WHERE id = $1 AND organization_id = $2`

    var j Job
    err := r.db.QueryRowContext(ctx, q, id, int64(org)).
        Scan(&j.ID, &j.Name, &j.Schedule, &j.NextRunAt)

    switch {
    case errors.Is(err, sql.ErrNoRows):
        return nil, fmt.Errorf("job %s: %w", id, ErrNotFound)   // translate at the boundary
    case err != nil:
        return nil, fmt.Errorf("querying job %s: %w", id, err)
    }
    return &j, nil
}
```

```go
func (r *Repo) ListJobs(ctx context.Context, org OrgID) ([]Job, error) {
    rows, err := r.db.QueryContext(ctx, q, int64(org))
    if err != nil {
        return nil, fmt.Errorf("listing jobs: %w", err)
    }
    defer rows.Close()                     // ← MANDATORY: leaks a connection otherwise

    jobs := make([]Job, 0)                 // non-nil so JSON is [] not null
    for rows.Next() {
        var j Job
        if err := rows.Scan(&j.ID, &j.Name); err != nil {
            return nil, fmt.Errorf("scanning job: %w", err)
        }
        jobs = append(jobs, j)
    }
    return jobs, rows.Err()                // ← MANDATORY: Next() hides iteration errors
}
```

> **The two mandatory lines people forget, and both are asked about:**
> 1. **`defer rows.Close()`** — an unclosed `Rows` holds its connection out of the pool permanently. Enough of them and every request hangs waiting for a connection. This is the number one `database/sql` leak.
> 2. **`return rows.Err()`** — `rows.Next()` returns `false` both for "done" and "an error occurred." Without checking `Err()`, a network failure mid-iteration looks exactly like a successful short result set, so you silently return partial data.

> **Trap — always use the `Context` variants.** `db.Query` without a context ignores cancellation entirely, so a client disconnect leaves the query running to completion. `QueryContext` cancels it at the database.

### NULL handling

```go
var (
    description sql.NullString
    lastRun     sql.NullTime
    count       sql.NullInt64
)
rows.Scan(&description, &lastRun, &count)

if description.Valid {
    j.Description = description.String
}

// Go 1.22+ generic version — cleaner
var desc sql.Null[string]
```

The alternative is `COALESCE(description, '')` in SQL, which is often simpler when the zero value is a legitimate substitute.

### SQL injection

```go
// SAFE — parameterised; the driver never interpolates
db.QueryContext(ctx, "SELECT * FROM jobs WHERE name = $1", userInput)

// CATASTROPHIC
db.QueryContext(ctx, "SELECT * FROM jobs WHERE name = '" + userInput + "'")
```

Placeholders are sent separately from the query text, so there is no parse path from user data to SQL. For dynamic *identifiers* (a sortable column name) — which cannot be parameterised — validate against an allowlist:

```go
var sortable = map[string]string{
    "name":     "name",
    "created":  "created_at",
    "next_run": "next_run_at",
}

col, ok := sortable[c.Query("sort")]
if !ok {
    col = "created_at"
}
q := fmt.Sprintf("SELECT ... ORDER BY %s", col)   // safe: col came from OUR map, not the user
```

---

## 9. Transactions

```go
func (r *Repo) TransferStock(ctx context.Context, from, to string, qty int) error {
    tx, err := r.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
    if err != nil {
        return fmt.Errorf("begin: %w", err)
    }
    defer tx.Rollback()    // no-op after Commit; guarantees cleanup on every error path

    // Atomic conditional UPDATE — the guard is in the WHERE clause.
    // Same reasoning as the Laravel inventory work: never check-then-act.
    res, err := tx.ExecContext(ctx,
        `UPDATE inventory SET quantity = quantity - $1
         WHERE sku = $2 AND quantity >= $1`, qty, from)
    if err != nil {
        return fmt.Errorf("deducting: %w", err)
    }
    if n, _ := res.RowsAffected(); n == 0 {
        return ErrInsufficientStock
    }

    if _, err := tx.ExecContext(ctx,
        `UPDATE inventory SET quantity = quantity + $1 WHERE sku = $2`, qty, to); err != nil {
        return fmt.Errorf("crediting: %w", err)
    }

    return tx.Commit()
}
```

`defer tx.Rollback()` is the idiomatic safety net: after a successful `Commit` it returns `sql.ErrTxDone` and does nothing, so the single deferred call covers every early return and every panic.

### A reusable helper

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin: %w", err)
    }

    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback()
            panic(p)                       // re-panic after cleaning up
        }
        if err != nil {
            _ = tx.Rollback()
        }
    }()

    if err = fn(tx); err != nil {
        return err
    }
    return tx.Commit()
}
```

Note the **named return** — the deferred closure has to see `err` to decide whether to roll back. That's the Tier 1 §10 named-return mechanic doing real work.

> **Trap — a `*sql.Tx` is pinned to one connection and is NOT safe for concurrent use.** You cannot fan out queries within a transaction across goroutines. If you try, you'll get interleaved protocol traffic and corruption. Transactions are sequential by nature.

> **Trap — never hold a transaction open across an HTTP call.** The connection is held out of the pool for the duration, and a slow third party can drain your entire pool. Same discipline as not doing I/O under a mutex.

---

## 10. pgx, sqlx, GORM — Choosing

| | Style | Strengths | Costs |
|---|---|---|---|
| `database/sql` + driver | stdlib | Zero deps, portable, full control | Manual scanning, verbose |
| **`pgx` (native)** | Postgres-specific | Fastest, binary protocol, native JSONB/arrays/`LISTEN`, `COPY` | Postgres only |
| `sqlx` | thin extension | `StructScan`, `Get`, `Select`, named params | Still hand-written SQL |
| `sqlc` | codegen | **Type-safe from your SQL**, compile-time checked | Build step |
| GORM | full ORM | Fast CRUD, migrations, hooks | Hidden N+1, opaque SQL, reflection cost |

```go
// sqlx — the pragmatic middle ground
var jobs []Job
err := db.SelectContext(ctx, &jobs,
    `SELECT id, name, schedule FROM jobs WHERE organization_id = $1`, orgID)
```

> **How to answer "would you use an ORM in Go?"** — this is a real culture question and a flat "no" sounds dogmatic.
>
> The Go community leans away from full ORMs, and the reasons are concrete rather than aesthetic: GORM's fluent API hides the generated SQL, its association preloading reproduces exactly the N+1 problems you spent time eliminating in Laravel, and its reflection-heavy implementation costs measurably in hot paths. Meanwhile Go's static typing removes much of the value Eloquent provides, since you're writing explicit structs anyway.
>
> My preference is `sqlc` — you write the SQL, it generates type-safe Go, and a typo in a column name is a **build failure** rather than a runtime error. Failing at compile time is the same argument as `type OrgID int64` for tenancy.
>
> Then the honest caveat: coming from Eloquent, the productivity difference on simple CRUD is real and I'd not pretend otherwise. The trade is more upfront typing for SQL you can actually read in a query plan. Given that my strongest performance story is an 88% query reduction that depended on *seeing* the queries, that trade is one I'd make deliberately.

---

## 11. Redis & Caching

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         cfg.RedisAddr,
    PoolSize:     50,
    MinIdleConns: 10,
    DialTimeout:  2 * time.Second,
    ReadTimeout:  1 * time.Second,     // fail fast — a cache should never be the bottleneck
    WriteTimeout: 1 * time.Second,
})
```

### Cache-aside with stampede protection

```go
type Cache struct {
    rdb    *redis.Client
    group  singleflight.Group          // collapses concurrent misses in THIS process
}

func (c *Cache) GetJob(ctx context.Context, org OrgID, id string) (*Job, error) {
    key := fmt.Sprintf("org:%d:job:%s", org, id)      // ALWAYS tenant-scoped

    if b, err := c.rdb.Get(ctx, key).Bytes(); err == nil {
        var j Job
        if json.Unmarshal(b, &j) == nil {
            return &j, nil
        }
    }

    v, err, _ := c.group.Do(key, func() (any, error) {
        j, err := c.repo.GetJob(ctx, org, id)
        if err != nil {
            return nil, err
        }
        if b, err := json.Marshal(j); err == nil {
            c.rdb.Set(ctx, key, b, jitter(5*time.Minute))
        }
        return j, nil
    })
    if err != nil {
        return nil, err
    }
    return v.(*Job), nil
}

// Spread expiries so a batch of keys written together doesn't expire together
func jitter(d time.Duration) time.Duration {
    return d + time.Duration(rand.Int64N(int64(d/4)))
}
```

Three things an interviewer looks for and most candidates miss:
- **Tenant-scoped keys.** `job:%s` without the org prefix is a cross-tenant cache leak — the same class of bug as a missing `WHERE organization_id`, but harder to spot because it only manifests on a cache hit.
- **`singleflight`** for in-process stampede collapse (Tier 2 §9). Cross-process still needs a distributed lock.
- **TTL jitter.** Keys warmed together expire together and produce a thundering herd. This is the same reasoning as the cache-stampede discussion in the Laravel material.

> **Trap — a cache failure must not be a request failure.** If Redis is down, `Get` returns an error and you should fall through to the database, not 500. Notice the code above ignores the Redis error entirely on read. Degrading to slow-but-correct is almost always right for a cache.

---

## 12. gRPC

```protobuf
service JobService {
  rpc GetJob(GetJobRequest) returns (Job);
  rpc ListJobs(ListJobsRequest) returns (stream Job);       // server streaming
  rpc SubmitResults(stream JobResult) returns (Summary);    // client streaming
  rpc Watch(stream WatchRequest) returns (stream Event);    // bidirectional
}
```

```go
func (s *Server) GetJob(ctx context.Context, req *pb.GetJobRequest) (*pb.Job, error) {
    job, err := s.svc.Get(ctx, req.GetId())
    switch {
    case errors.Is(err, ErrNotFound):
        return nil, status.Errorf(codes.NotFound, "job %s not found", req.GetId())
    case err != nil:
        return nil, status.Error(codes.Internal, "internal error")
    }
    return toProto(job), nil
}
```

### Interceptors — the gRPC equivalent of middleware

```go
func UnaryLogger() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any,
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

        start := time.Now()
        resp, err := handler(ctx, req)

        slog.Info("grpc",
            "method", info.FullMethod,
            "code", status.Code(err),
            "duration_ms", time.Since(start).Milliseconds())

        return resp, err
    }
}
```

### REST vs gRPC — the comparison to have ready

| | REST/JSON | gRPC |
|---|---|---|
| Payload | Human-readable, verbose | Binary protobuf, compact and fast |
| Contract | OpenAPI (optional, often drifts) | `.proto` is mandatory and generates both sides |
| Streaming | SSE or WebSockets, bolted on | First-class, four modes |
| Browser | Native | Needs grpc-web + a proxy |
| Debugging | `curl` | `grpcurl`, less casual |
| Load balancing | Standard L7 | Needs L7-aware LB — HTTP/2 multiplexes over one connection |

> **The load-balancing point is the good one to raise unprompted.** gRPC holds a long-lived HTTP/2 connection and multiplexes every call over it, so a naive L4 load balancer pins a client to one backend forever and your traffic distribution collapses. You need client-side load balancing, an L7 proxy like Envoy, or a service mesh. Candidates who've only used gRPC in a demo never mention this.
>
> **The usual verdict:** gRPC for internal service-to-service where you control both ends and want a strict contract; REST at the public edge where clients are browsers and third parties. Chronos would use gRPC between scheduler nodes and REST for its management API.

---

## 13. Testing

### Handler tests with httptest

```go
func TestGetJob(t *testing.T) {
    gin.SetMode(gin.TestMode)

    tests := []struct {
        name       string
        jobID      string
        svc        *fakeService
        wantStatus int
        wantBody   string
    }{
        {
            name:       "found",
            jobID:      "abc",
            svc:        &fakeService{job: &Job{ID: "abc", Name: "nightly"}},
            wantStatus: http.StatusOK,
            wantBody:   `"name":"nightly"`,
        },
        {
            name:       "not found",
            jobID:      "missing",
            svc:        &fakeService{err: ErrNotFound},
            wantStatus: http.StatusNotFound,
        },
        {
            name:       "internal error",
            jobID:      "boom",
            svc:        &fakeService{err: errors.New("db exploded")},
            wantStatus: http.StatusInternalServerError,
            wantBody:   `"internal error"`,     // assert we do NOT leak "db exploded"
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()

            r := gin.New()
            r.Use(ErrorHandler())
            r.GET("/jobs/:id", NewHandler(tt.svc).GetJob)

            req := httptest.NewRequest(http.MethodGet, "/jobs/"+tt.jobID, nil)
            w := httptest.NewRecorder()

            r.ServeHTTP(w, req)      // no network, no port — just the Handler interface

            if w.Code != tt.wantStatus {
                t.Fatalf("status = %d, want %d", w.Code, tt.wantStatus)
            }
            if tt.wantBody != "" && !strings.Contains(w.Body.String(), tt.wantBody) {
                t.Errorf("body = %s, want it to contain %s", w.Body.String(), tt.wantBody)
            }
        })
    }
}
```

`httptest.NewRecorder()` works because the router is just an `http.Handler` — you call `ServeHTTP` directly with no server, no port, and no flakiness. That last test case is worth copying as a habit: **assert that internal error text does not reach the client.**

### Fakes over mocks

```go
type JobService interface {
    Get(ctx context.Context, id string) (*Job, error)
}

type fakeService struct {
    job *Job
    err error
}

func (f *fakeService) Get(context.Context, string) (*Job, error) { return f.job, f.err }
```

> Idiomatic Go prefers hand-written fakes to generated mocks. Interfaces are small (that's the point of Tier 1 §12), so a fake is usually five lines, and it produces far more readable tests than `EXPECT().Get(gomock.Any()).Return(...)`. Reach for `mockery`/`gomock` when you genuinely need call-order or argument assertions.

### Integration tests with testcontainers

```go
func TestRepo_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    ctx := context.Background()

    pg, err := postgres.Run(ctx, "postgres:17-alpine",
        postgres.WithDatabase("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2)),
    )
    if err != nil {
        t.Fatal(err)
    }
    defer pg.Terminate(ctx)

    dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
    db, _ := sql.Open("pgx", dsn)
    runMigrations(t, db)

    repo := NewPostgresRepo(db)
    // ... real queries against a real Postgres
}
```

A real database catches what a mock never will: constraint violations, transaction semantics, isolation-level behaviour, and SQL that's simply wrong. Run these with a build tag or `-short` so the unit suite stays fast.

### Tenant isolation tests

Directly transferable from your Laravel Pest suite, and worth having ready:

```go
func TestListJobs_TenantIsolation(t *testing.T) {
    repo := setupRepo(t)

    orgA, orgB := OrgID(1), OrgID(2)
    seedJobs(t, repo, orgA, 3)
    seedJobs(t, repo, orgB, 5)

    jobs, err := repo.ListJobs(ctx, orgA)
    if err != nil {
        t.Fatal(err)
    }
    if len(jobs) != 3 {
        t.Fatalf("got %d jobs, want 3 — possible cross-tenant leak", len(jobs))
    }
    for _, j := range jobs {
        if j.OrgID != orgA {
            t.Errorf("job %s belongs to org %d, not %d", j.ID, j.OrgID, orgA)
        }
    }
}
```

> **The strong framing:** *"Every repository method gets an isolation test with two seeded tenants. It's the cheapest possible insurance against the highest-severity bug class in a multi-tenant system — and unlike Laravel where a global scope silently covers you, in Go the tenant filter is written by hand in every query, so the test is the only thing that catches a missing one."*

### Coverage and fuzzing

```bash
go test ./... -race -cover -coverprofile=c.out
go tool cover -func=c.out | tail -1
```

```go
func FuzzParseCron(f *testing.F) {
    f.Add("*/5 * * * *")
    f.Add("0 0 * * MON")
    f.Fuzz(func(t *testing.T, expr string) {
        _, _ = ParseCron(expr)      // the assertion is "must not panic"
    })
}
```

Built-in fuzzing (1.18+) is excellent for parsers. For Chronos, fuzzing the cron expression parser is an obvious and impressive thing to have done.

---

## 14. Tier 3 Q&A Drill

### net/http

**1. What is `http.Handler`?**
A one-method interface, `ServeHTTP(ResponseWriter, *Request)`. Every router, middleware, and framework in Go — including Gin's `Engine` — is an implementation of it.

**2. What does `http.HandlerFunc` do?**
Adapts a plain function to the `Handler` interface by giving the function type a `ServeHTTP` method. It's the adapter pattern in four lines.

**3. What's wrong with `http.ListenAndServe(":8080", h)` in production?**
No timeouts of any kind. A slow client holds a connection indefinitely (Slowloris) until you exhaust file descriptors. Construct an `http.Server` with explicit timeouts.

**4. Name the four server timeouts and what each protects.**
`ReadHeaderTimeout` (Slowloris), `ReadTimeout` (headers + body, slow uploads), `WriteTimeout` (slow clients/handlers), `IdleTimeout` (keep-alive connections holding FDs).

**5. Why can `WriteTimeout` break a streaming endpoint?**
It's an absolute deadline from the start of the request, so long-lived responses get cut off. Set it to 0 for those routes and use per-request deadlines.

**6. What's the trap with `http.DefaultClient`?**
No timeout — a hung server hangs your goroutine forever. Also `MaxIdleConnsPerHost` defaults to 2, which causes constant reconnection to a heavily-used downstream.

**7. Why must you drain a response body, not just close it?**
An undrained body means the connection can't be returned to the keep-alive pool and gets discarded. `io.Copy(io.Discard, resp.Body)` before `Close`.

**8. What happens if you set a header after `WriteHeader`?**
It's silently ignored. Writing the body implicitly sends a 200, so a later `WriteHeader(500)` also does nothing and logs "superfluous".

**9. What changed in `net/http` routing in Go 1.22?**
Method matching and path wildcards in `ServeMux` (`GET /jobs/{id}`, `r.PathValue("id")`), which narrowed the gap with third-party routers considerably.

### Gin

**10. What data structure does Gin's router use?**
A radix tree per HTTP method — O(path length) lookup with no allocations and no sequential route matching.

**11. When does Gin panic at startup?**
On conflicting route registrations, such as two different parameter names at the same position (`/jobs/:id` and `/jobs/:name`). Failing at boot rather than at request time is deliberate.

**12. Why must you call `c.Copy()` before using a `gin.Context` in a goroutine?**
Gin pools and reuses `Context` objects across requests, so after the handler returns the object may already belong to another request.

**13. Is `c.Copy()` enough to make that goroutine correct?**
No. The request context is cancelled when the handler returns, so downstream calls fail immediately, and a panic there kills the process since `Recovery()` only wraps the handler's goroutine. Enqueue the work instead.

**14. Which context should you pass to your service layer?**
`c.Request.Context()`. Passing `*gin.Context` couples your domain to the framework.

**15. `c.Bind` vs `c.ShouldBind`?**
`Bind` writes a 400 with its own error format automatically; `ShouldBind` returns the error and lets you control the response. Always `ShouldBind` in a real API.

**16. Why can't you call `ShouldBindJSON` twice?**
The body is a stream, consumed on first read. The second call sees EOF and silently returns an empty struct. Use `ShouldBindBodyWith` if you must.

**17. Does `c.Abort()` stop the current function?**
No — it only prevents *subsequent* handlers from running. You still need an explicit `return`.

**18. Why capture the request path before `c.Next()` in logging middleware?**
Downstream code can rewrite `c.Request.URL.Path`, so reading it afterwards may log the wrong route.

**19. How do you distinguish "field absent" from "field set to zero" in a PATCH?**
Use pointer fields. `nil` means absent; a non-nil pointer to the zero value means explicitly set.

### database/sql

**20. Is `*sql.DB` a connection?**
No, it's a pool. Create one per process and share it — it's safe for concurrent use.

**21. Does `sql.Open` connect?**
No, it's lazy. `PingContext` forces an actual connection.

**22. How do you size `MaxOpenConns`?**
Work down from the database's `max_connections` divided across all clients with headroom, then sanity-check against Little's Law (`throughput × latency`). The database limit wins; if it's too tight, add PgBouncer.

**23. Why set `MaxIdleConns` equal to `MaxOpenConns`?**
The default of 2 means a burst opens connections then immediately closes them back to 2, so you repeatedly pay TCP and TLS setup under peak load.

**24. Why set `ConnMaxLifetime`?**
So connections are recycled — otherwise they stay pinned to a demoted primary after a failover, and load balancers never get to rebalance.

**25. What are the two mandatory lines when using `Query`?**
`defer rows.Close()` (otherwise the connection never returns to the pool) and `return rows.Err()` (otherwise a mid-iteration error is indistinguishable from a normal end).

**26. Why use `QueryContext` over `Query`?**
Without a context the query ignores cancellation, so a disconnected client leaves it running to completion.

**27. How do you handle NULL?**
`sql.NullString`/`NullTime`/`Null[T]`, or `COALESCE` in SQL when the zero value is an acceptable substitute.

**28. How do you safely order by a user-supplied column?**
You can't parameterise an identifier. Map the user's input through an allowlist and use only your own trusted string.

**29. Why `defer tx.Rollback()` even when you commit?**
After a successful commit it returns `ErrTxDone` and does nothing, so one deferred call safely covers every early return and panic.

**30. Can you use a `*sql.Tx` from multiple goroutines?**
No. It's pinned to a single connection and is not safe for concurrent use — transactions are inherently sequential.

**31. Why does the transaction helper need a named return?**
The deferred function must inspect `err` to decide whether to roll back, and only a named result is visible to the defer.

**32. Would you use GORM?**
Usually not. It hides the SQL, reproduces N+1 through association preloading, and costs reflection overhead — and Go's static typing removes much of the value an ORM adds. `sqlc` gives type safety with visible SQL and compile-time checking. The honest caveat is that simple CRUD is genuinely more work than Eloquent.

### Caching, gRPC, testing

**33. Why must cache keys include the tenant ID?**
Otherwise a cache hit serves one tenant's data to another — the same bug class as a missing `WHERE organization_id`, but only visible on a hit, so it's much harder to catch.

**34. What is TTL jitter and why do you need it?**
Randomising expiry so keys warmed together don't all expire simultaneously and stampede the database.

**35. Should a Redis outage fail the request?**
No. Fall through to the origin. A cache should degrade to slow-but-correct, never to an error.

**36. Name gRPC's four call types.**
Unary, server streaming, client streaming, bidirectional streaming.

**37. What's the gRPC load-balancing problem?**
It multiplexes all calls over one long-lived HTTP/2 connection, so an L4 load balancer pins a client to a single backend permanently. You need client-side LB, an L7 proxy, or a mesh.

**38. When do you choose gRPC over REST?**
Internal service-to-service where you own both ends and want a generated, enforced contract. REST at the public edge for browsers and third parties.

**39. Why does `httptest.NewRecorder` work without starting a server?**
Because the router is just an `http.Handler` — you invoke `ServeHTTP` directly, so there's no network, no port, and no flakiness.

**40. Why does idiomatic Go prefer fakes to generated mocks?**
Interfaces are small by design, so a fake is a few lines and reads better than mock expectation chains. Use generated mocks when you need call-order or argument assertions.

**41. What do integration tests with testcontainers catch that unit tests don't?**
Constraint violations, transaction and isolation semantics, and SQL that's simply wrong — all invisible to a mock.

**42. What's the one test every repository method in a multi-tenant system should have?**
An isolation test seeding two organisations and asserting only the requesting tenant's rows come back. In Go the tenant filter is hand-written per query, so this test is the only thing that catches a missing one.

---

> **The Tier 3 senior signal:** the questions here look like framework trivia but almost all of them are really about resource lifecycle — connections returning to pools, contexts being cancelled, bodies being drained, rows being closed, objects being recycled. Answer at that level rather than at the API level and you'll consistently be a layer deeper than the question.

---

**Next:** [`04-senior.md`](./04-senior.md) — the GMP scheduler, garbage collection, escape analysis, profiling with pprof, Raft and the Chronos deep dive, resilience patterns, and production deployment.

**Back to:** [`README.md`](./README.md) · [`01-basic.md`](./01-basic.md) · [`02-concurrency.md`](./02-concurrency.md)
