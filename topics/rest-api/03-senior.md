# REST API — Tier 3: Senior (Advanced Patterns & Operations)

> **Target:** Senior Backend Engineer (8+ years, PHP/Laravel, Go, JS)  
> **Focus:** HATEOAS, backward compatibility, idempotency, webhooks, API gateways, security, governance  
> **Prerequisites:** REST API Tier 1 (basic) and Tier 2 (intermediate) knowledge assumed

At the senior level, you design **systems of APIs** that evolve independently, scale to millions of requests, and remain safe under failure. This tier covers patterns that distinguish a senior engineer: knowing when to apply hypermedia, how to evolve APIs without breaking clients, why idempotency is an API contract not just a DB constraint, and how to govern API ecosystems at scale.

Every section includes **Traps** (what interviewers watch for) and **Follow-ups** (the second and third questions they ask).

---

## Table of Contents

1. [HATEOAS & Richardson Maturity Model](#1-hateoas--richardson-maturity-model)
2. [Backward Compatibility](#2-backward-compatibility)
3. [Idempotency — The Idempotency-Key Pattern](#3-idempotency--the-idempotency-key-pattern)
4. [Bulk Operations & Batch APIs](#4-bulk-operations--batch-apis)
5. [Webhooks](#5-webhooks)
6. [API Gateways](#6-api-gateways)
7. [GraphQL vs REST — The Real Tradeoffs](#7-graphql-vs-rest--the-real-tradeoffs)
8. [API Performance](#8-api-performance)
9. [API Security — OWASP API Top 10](#9-api-security--owasp-api-top-10)
10. [Contract Testing & API Governance](#10-contract-testing--api-governance)
11. [Tier 3 Q&A Drill](#11-tier-3-qa-drill)

---

## 1. HATEOAS & Richardson Maturity Model

### Richardson Maturity Model (RMM)

| Level | Name | Description |
|-------|------|-------------|
| 0 | The Swamp of POX | HTTP as tunnel (XML-RPC, SOAP) |
| 1 | Resources | URI hierarchy (`/orders/123`) |
| 2 | HTTP Verbs | Proper GET/POST/PUT/PATCH/DELETE + status codes |
| 3 | Hypermedia Controls | Responses include links for next actions |

### Level 3 — Hypermedia as the Engine of Application State

At Level 3, the server drives state transitions through links. The client does not hardcode URLs.

**Without HATEOAS (Level 2):**

```http
GET /orders/123 HTTP/1.1
Accept: application/json
```

```json
{
  "id": 123,
  "status": "pending",
  "total": 29.99
}
```

Client must know from docs to `POST /orders/123/pay`.

**With HATEOAS (Level 3):**

```http
GET /orders/123 HTTP/1.1
Accept: application/hal+json
```

```json
{
  "id": 123,
  "status": "pending",
  "total": 29.99,
  "_links": {
    "self": { "href": "/orders/123" },
    "pay": { "href": "/orders/123/pay" },
    "cancel": { "href": "/orders/123/cancel" }
  }
}
```

After paying, the `pay` link disappears and `refund` appears — the client navigates the state machine through links.

### Hypermedia Media Types

| Format | Link Style | Key Feature |
|--------|------------|-------------|
| **HAL** | `_links` + `_embedded` | Simplest, widely adopted |
| **JSON:API** | `relationships` + `links` | Sparse fields, compound docs |
| **Siren** | `actions` + `links` | Action-oriented (method, fields, type) |
| **Collection+JSON** | `links` + `queries` + `template` | Collection read/write |

**Siren example with actions:**

```json
{
  "class": ["order"],
  "properties": { "status": "pending", "total": 29.99 },
  "links": [ { "rel": ["self"], "href": "/orders/123" } ],
  "actions": [
    {
      "name": "pay",
      "method": "POST",
      "href": "/orders/123/pay",
      "type": "application/json",
      "fields": [ { "name": "payment_method", "type": "text" } ]
    }
  ]
}
```

### When HATEOAS Is Valuable

**API discoverability** — a generic client explores the entire API from a root URL:

```http
GET /api HTTP/1.1
Accept: application/hal+json
```

```json
{
  "_links": {
    "self": { "href": "/api" },
    "orders": { "href": "/orders{?page,limit}", "templated": true },
    "customers": { "href": "/customers{?page,limit}", "templated": true }
  }
}
```

**Client/server decoupling** — server changes URLs without breaking clients (URLs are opaque). **Multi-client ecosystems** — web, mobile, third-party all follow the same links.

### When HATEOAS Is Overhead

**Internal microservices** with fixed contracts — add serialization cost for zero benefit. **Mobile apps** with pre-defined UI flows — links are ignored. **High-throughput low-latency APIs** — link serialization adds payload size and CPU cost at 100K+ RPS.

> **Trap:** Calling your API Level 3 because you put `self` links, but clients hardcode URLs anyway. Real HATEOAS means the client navigates solely by links and breaks if you remove one.
>
> **Follow-up:** "Your mobile app uses HATEOAS. The user sees a 'Pay' button. How does HATEOAS help?" — It doesn't. Button position, label, and action are in native code. HATEOAS adds complexity for no mobile benefit. Use it where it matters (web, third-party) and skip it where it doesn't.

> **Trap:** Building hypermedia for every endpoint — order transitions benefit, a simple KV lookup does not. Apply HATEOAS where state machines exist.

> **Trap:** Links that change without versioning — URL decoupling doesn't replace schema versioning. Schema evolution still requires additive changes or media type versioning.

### Implementing HATEOAS

```
function addOrderLinks(order, request):
    links["self"] = { "href": "/orders/" + order.id }

    if order.status == "pending":
        links["pay"]    = { "href": "/orders/" + order.id + "/pay" }
        links["cancel"] = { "href": "/orders/" + order.id + "/cancel" }
    if order.status == "paid":
        links["refund"]  = { "href": "/orders/" + order.id + "/refund" }
        links["receipt"] = { "href": "/orders/" + order.id + "/receipt" }

    return { ...order, "_links": links }
```

Templated links (RFC 6570):

```json
{
  "_links": {
    "self": { "href": "/orders{?page,limit,sort}", "templated": true },
    "next": { "href": "/orders?page=2&limit=20" }
  }
}
```

---

## 2. Backward Compatibility

### Postel's Law (Robustness Principle)

> *"Be conservative in what you send, be liberal in what you accept."*

**Sending:** follow schema strictly — no removing fields, no type changes. **Accepting:** tolerate extra fields, accept multiple formats (`"true"` / `true`), map legacy field names.

```
function parseActive(value):
    if is_string(value): return to_lower(value) == "true"
    return bool(value)
```

> **Trap:** Being too liberal — silently ignoring misspelled fields means the client thinks it set a field when it didn't. Log/warn on unknown fields in dev, enforce strict mode in prod with a grace period.

### Additive Changes (Always Safe)

| Change | Example | Safe? |
|--------|---------|-------|
| Add new field | Add `discount_total` | Yes |
| Add new endpoint | `POST /orders/{id}/split` | Yes |
| Add new optional param | `?include=items` | Yes |
| Add new enum value | `on_hold` status | Maybe (check client switch) |
| Relax constraint | Required → optional | Maybe |

### Breaking Changes (Never Without Deprecation)

| Change | Why It Breaks |
|--------|---------------|
| Remove a field | Client crashes on `undefined` |
| Rename a field | Client gets null for expected key |
| Change field type | `"123"` → `123` breaks parse |
| Make optional → required | Client requests now 400 |
| Remove an endpoint | 404 on working calls |
| Reorder an array | Client expects sorted order |

### Field Expansion Pattern

Instead of breaking a field, add a new one and deprecate the old:

```
Before: { "name": "Widget", "category": "electronics" }

Migration:
{
  "name": "Widget",
  "category": "electronics",
  "category_id": "el-001",
  "category_name": "Electronics"
}

After: { "name": "Widget", "category_id": "el-001", "category_name": "Electronics" }
```

Timeline: add new fields → mark old as deprecated → remove after window.

### Deprecation Headers

```http
HTTP/1.1 200 OK
Sunset: Sat, 31 Jan 2026 23:59:59 GMT
Deprecated: true
Link: </docs/migration-v2>; rel="deprecation"
```

For per-field deprecation:

```http
X-Deprecated-Fields: legacy_field, old_tax_rate
Warning: 299 - "Field 'legacy_field' is deprecated. Use 'new_field'."
```

> **Trap:** No deprecation notice — just removing fields hoping clients noticed the changelog. They won't. It will be an incident.

> **Trap:** Renaming without aliasing — return both fields for two release cycles, then remove old.

### Client Tolerance

**Server side (liberal accept):** map legacy field names, ignore unknown fields (with warning log), coerce types where safe.

**Client side (conservative send):** always send canonical field names and types.

> **Trap:** Thinking all clients upgrade immediately — mobile apps have 2-6 week rollout, IoT devices may never upgrade. Always support at least one previous version.

### Migration Strategies

**Parallel run:** old and new endpoints coexist (`/v1/orders` and `/v2/orders`).

**Adapter layer:** translate between API versions and internal model:

```
Controller(v1) → AdapterV1 → Service
Controller(v2) → AdapterV2 → Service
```

The service never changes; only the adapter changes.

---

## 3. Idempotency — The Idempotency-Key Pattern

### Why Idempotency Matters

Payment APIs and any financially-significant operation. Classic failure: client sends `POST /orders` → server processes + charges → network timeout → client retries → **double charge**. Idempotency prevents this.

### The `Idempotency-Key` Header

```
POST /orders
Idempotency-Key: 7c9e6679-7425-40df-9c7e-4b8b9f3a5c2e

{ "items": [...], "total": 29.99 }
```

**Key generation:**

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| Client UUID | Client generates, sends with request | Simple, no RTT, offline | Bad clients reuse keys |
| Server pre-create | `POST /idempotency-keys` | Server controls uniqueness | Extra RTT |
| Request hash | Hash method + path + body | No client cooperation | Body must be identical |

### Key Scope

| Scope | Example | Behavior |
|-------|---------|----------|
| Per user | `user_42:7c9e6679...` | Different users can reuse same UUID |
| Per resource | `order_ctx:7c9e6679...` | Same key, different resource = different request |
| Global | `7c9e6679...` | Must be globally unique forever |

**Recommended:** per-user + per-resource scope.

### Storage (Redis)

```
SET idempotency:{key} "{status}:{response}" EX 86400 NX
```

- `NX` — only set if absent (prevents concurrent duplicates)
- `EX 86400` — 24h TTL

### Processing Flow

```
function handlePostOrder(request):
    key = request.header("Idempotency-Key")
    if not key: return 400

    existing = redis.get("idempotency:" + key)
    if existing:
        res = parse(existing)
        res.header("X-Idempotent-Replay", "true")
        return 200 res

    // Lock to prevent concurrent duplicates
    lock = redis.set("idempotency:" + key + ":lock",
                     "processing", NX, EX, 30)
    if not lock:
        sleep(100ms)
        return handlePostOrder(request)

    try:
        order = createOrder(request.body)
        result = { "id": order.id, "status": "created" }
        redis.set("idempotency:" + key,
                  "completed:" + json(result), EX, 86400)
        return 201 result
    catch e:
        redis.set("idempotency:" + key,
                  "failed:" + json(e), EX, 86400)
        return 500
    finally:
        redis.del("idempotency:" + key + ":lock")
```

> **Trap:** Using idempotency keys as a replacement for DB unique constraints — use both. Idempotency keys catch retries within TTL. Unique constraints (`UNIQUE(merchant_ref)`) catch TTL expiry + late duplicate.
>
> **Follow-up:** "TTL expires and client retries with same key?" — Key is gone from Redis, treated as new request → duplicate. The unique constraint on the business identifier catches this.

> **Trap:** Too-short TTL — client retries for 2 hours but TTL is 30 min. After expiry, retry creates a duplicate. Set TTL based on real retry behavior (Stripe uses 24h).

> **Trap:** Returning different status on replay — idempotency means same response + same status. Use `X-Idempotent-Replay: true` header, not a different status code.

---

## 4. Bulk Operations & Batch APIs

### Why Batch

Without batching: 100 orders = 100 HTTP round trips, 100 TLS handshakes, 100 auth checks. With batch: one request.

### Synchronous Batch

```
POST /items/batch

{ "items": [
    { "sku": "A001", "name": "Widget", "price": 9.99 },
    { "sku": "A002", "name": "Gadget", "price": 14.99 }
]}
```

```http
HTTP/1.1 200 OK
```

```json
{
  "results": [
    { "status": 201, "id": "ord-001", "sku": "A001" },
    { "status": 422, "sku": "A002", "error": "Invalid price" }
  ],
  "summary": { "total": 2, "succeeded": 1, "failed": 1 }
}
```

### Asynchronous Batch

For large payloads or long processing:

```
POST /items/batch
{ "items": [...5000 items...], "callback_url": "https://..." }
```

```http
HTTP/1.1 202 Accepted
Location: /jobs/batch-987
```

```json
{ "job_id": "batch-987", "status": "processing", "estimated_seconds": 120 }
```

Poll `GET /jobs/batch-987` or receive webhook.

### Partial Success — Critical Pattern

A batch of 1000 items must not fail entirely because one item is invalid. Per-item status codes:

```json
{
  "results": [
    { "index": 0, "status": 201, "id": "ord-001" },
    { "index": 1, "status": 422, "error": { "code": "INVALID_SKU", "message": "..." } },
    { "index": 2, "status": 409, "error": { "code": "DUPLICATE", "message": "..." } }
  ],
  "summary": { "total": 3, "succeeded": 1, "failed": 1, "conflict": 1 }
}
```

> **Trap:** Assuming atomicity — design for partial success unless the business requires all-or-nothing (financial transfers). Use an `"atomic": true` flag for transactional batches.

### Best Practices

- **Limit batch size** (e.g., 100 items) — return 422 with `BATCH_SIZE_EXCEEDED`
- **Validate all items before processing any** — collect errors, reject early if invalid
- **Detect duplicates within batch** — same SKU appearing twice should fail early
- **Progress tracking** for long batches:

```json
{ "progress": { "total": 5000, "processed": 1234, "failed": 2, "percent": 24.68 } }
```

> **Trap:** Unlimited batch sizes → OOM or timeout. Enforce and document limits.

> **Trap:** Not returning per-item error details — a single "batch failed" status is useless. Include index, code, and message per item.

---

## 5. Webhooks

### When to Use

Server-to-server async notifications: order placed, payment received, shipment updated. Use when the consumer should not poll.

### Registration

```
POST /webhooks
{ "url": "https://consumer.com/webhooks/orders",
  "events": ["order.placed", "order.paid", "order.refunded"] }
```

Events should be granular: `order.placed`, `order.paid`, `order.shipped` — not `order.*`.

### Payload Envelope

```json
{
  "id": "evt_9abc8d7e-6f42-4a8c-9b3d-1e2f4a6b8c0d",
  "type": "order.placed",
  "created": "2026-07-26T14:30:00Z",
  "data": { "order_id": "ord_789", "total": 2999, "currency": "USD" }
}
```

The `id` field is the consumer's deduplication key.

### Delivery Guarantees

**At-least-once:** delivered at least once, possibly more. Consumer deduplicates:

```
function handleWebhook(payload):
    if redis.exists("dedup:" + payload.id): return 200
    processEvent(payload)
    redis.set("dedup:" + payload.id, "1", EX, 604800)
    return 200
```

**Exactly-once:** at-least-once + consumer dedup = effectively once.

> **Trap:** Not signing payloads — anyone who discovers the webhook URL can forge events. Always sign and verify.

### Retry Strategy

```
Attempt 1: 0s       Attempt 2: +10s    Attempt 3: +30s
Attempt 4: +90s     Attempt 5: +270s   ... Max: 24h
```

```
function getRetryDelay(attempt):
    delay = min(10 * 2^attempt, 3600)  // max 1 hour
    return delay + random(0, delay * 0.1)  // jitter
```

**Dead letter:** after N retries exhausted, move to DLQ. Notify consumer via dashboard/email. Don't retry forever.

### Signing (HMAC-SHA256)

**Sender:**

```
function signPayload(payload, secret):
    rawBody = JSON.stringify(payload)  // exact bytes, no re-serialize
    sig = HMAC_SHA256(rawBody, secret)
    return base64(sig)
```

```
X-Signature-256: t=1722004200,v1=abc123def456...
```

**Verification (consumer):**

```
function verifySignature(payload, header, secret):
    parts = parseSignatureHeader(header)
    if abs(now() - parts["t"]) > 300: return false  // replay protection

    signedPayload = parts["t"] + "." + JSON.stringify(payload)
    expected = HMAC_SHA256(signedPayload, secret)

    return hash_equals(expected, base64_decode(parts["v1"]))
```

**Critical:** use `hash_equals` (constant-time comparison). Never `==` or `===`.

### Replay Protection

Timestamp tolerance window (5 min) + idempotency key per event. Store processed event IDs in Redis with 7-day TTL.

> **Trap:** Not handling consumer unavailability — never deliver webhooks synchronously in the request path. Use a queue:

```
// BAD: blocks request
function onOrderPlaced(order):
    for webhook in getWebhooks("order.placed"):
        http.post(webhook.url, payload)  // sync!

// GOOD: async
function onOrderPlaced(order):
    queue.dispatch("deliver_webhook", { event: "order.placed", data: order })
```

> **Trap:** JSON canonicalization — parsing then re-serializing JSON changes whitespace/key order, breaking the signature. Sign the **exact raw bytes** of the body.

> **Trap:** Blocking on webhook delivery or not setting a timeout — a slow consumer can hold connections indefinitely. Set a reasonable timeout (5-10s).

### Retry Queue Architecture

```
Event occurs → Event Bus → Webhook Queue → Retry Logic
                                        ↓
                                Consumer endpoint
                                        ↓
                        Success → mark delivered
                        Failure → queue retry | max → dead letter
```

### Delivery Logs and Monitoring

**Log every delivery attempt:**

```json
{
  "webhook_id": "wh_xyz456",
  "event_id": "evt_9abc8d7e",
  "event_type": "order.placed",
  "consumer_url": "https://consumer.com/webhooks/orders",
  "attempt": 1,
  "status_code": 200,
  "latency_ms": 145,
  "timestamp": "2026-07-26T14:30:01Z"
}
```

**Key metrics:**

| Metric | Purpose |
|--------|---------|
| `webhook.delivery.count` | Total deliveries |
| `webhook.delivery.success` | Successful deliveries |
| `webhook.delivery.failure` | Failed deliveries |
| `webhook.delivery.latency_p99` | p99 delivery latency |
| `webhook.queue.depth` | Current queue depth |
| `webhook.dead_letter.count` | Messages in dead letter |
| `webhook.delivery.retry_rate` | % of deliveries that required retry |

**Alert thresholds:**
- Dead letter queue size > 0 (immediately investigate)
- Delivery failure rate > 5% over 5 minutes
- Queue depth growing continuously (consumer likely offline)
- Signature verification failures (potential attack or misconfigured secret)

### Signing Key Rotation

Support multiple signatures in the header so consumers can rotate secrets without downtime:

```
X-Signature-256: t=1722004200,v1=abc123...,v1=def456...
```

Publish new secret to consumer, then remove old secret after consumer confirms rotation. The consumer verifies against all `v1` entries until it switches to the new secret.

### Rate Limiting Outbound Webhooks

Per-webhook rate limiting prevents hammering a slow consumer:

```
function deliverWebhook(webhook, payload):
    if rateLimiter.isRateLimited("webhook:" + webhook.id):
        queue.retryLater(payload, delay="1m")
        return

    response = http.post(webhook.url, payload, timeout="10s")
    // ...
```

---

## 6. API Gateways

### What an API Gateway Provides

| Feature | Description |
|---------|-------------|
| Single entry point | One hostname, routes to services |
| Rate limiting | Per client, endpoint, tenant |
| Authentication | Validate tokens, API keys, OAuth |
| Request transformation | Rewrite paths, add/remove headers |
| Response aggregation | Combine responses from multiple services |
| Caching | Gateway-level response cache |
| Circuit breaking | Stop routing to unhealthy services |
| Protocol translation | REST → gRPC, HTTP → WebSocket |
| Logging & monitoring | Unified access logs, metrics |
| TLS termination | HTTPS, certificate management |

```
Client → API Gateway → [Orders Service, Payments Service, Users Service]
```

### When Needed

Microservices (different auth per service), multitenant APIs (per-tenant rate limits), external-facing APIs (auth, caching, monitoring).

### When Not Needed

Single monolith (the app handles these), internal-only APIs (service mesh handles sidecar concerns).

### Common Gateways

| Gateway | Language | Key Features |
|---------|----------|--------------|
| Kong | Lua/Nginx | Plugin ecosystem, declarative config |
| AWS API Gateway | Managed | AWS integration, Lambda authorizers |
| Envoy | C++ | L7 proxy, service mesh, gRPC |
| NGINX | C | High perf, simple routing, OpenResty |
| Tyk | Go | API management, analytics, multi-cloud |

### Gateway vs Service Mesh

| | API Gateway | Service Mesh |
|---|---|---|
| Traffic | North-south (external → internal) | East-west (service → service) |
| Placement | Edge | Sidecar per pod |
| Features | Auth, rate limiting, caching, API keys | mTLS, retry, circuit break, observability |

**Common pattern:** gateway for north-south, mesh for east-west. Gateway authenticates external request; mesh handles internal calls with mTLS.

> **Trap:** Putting business logic in the gateway — the gateway should only handle cross-cutting concerns. Order validation, payment processing, inventory checks belong in services.

> **Trap:** Gateway as SPOF — deploy multiple replicas with a load balancer. Use health checks and auto-scaling.

> **Trap:** Bypassing gateway internally — internal services calling each other through the gateway adds latency. Use service mesh or direct discovery for east-west.

### Rate Limiting at Gateway

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1627200000
Retry-After: 18
```

Per-tenant limits: `tenant_free: 10/min`, `tenant_pro: 100/min`, `tenant_enterprise: 1000/min`.

**Distributed rate limiting** uses Redis to maintain counters across gateway instances:

```
// Token bucket per user in Redis
function checkRateLimit(userId, maxRequests, windowMs):
    key = "ratelimit:" + userId + ":" + currentWindow()
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, windowMs / 1000)
    return count <= maxRequests
```

Fault-tolerant mode: if Redis is down, allow the request (log the failure). Better to temporarily allow excess than to block all traffic.

### Request Transformation

```
// Kong request transformer config
request_transformer:
  config:
    add:
      headers:
        X-Forwarded-Proto: "{scheme}"
        X-Real-IP: "{remote_addr}"
        X-Internal-Request-ID: "{uuid}"
    remove:
      headers:
        X-Internal-Token
        X-Debug-Info
```

**Path rewriting:**

```
// Incoming: GET /api/v2/orders/123
// Target:   GET /orders-service/orders/123
rewrite:
  pattern: "^/api/v2/(.*)$"
  replacement: "/orders-service/$1"
```

### Response Aggregation

Gateway fan-out to multiple services in parallel:

```
function aggregateOrderDetail(orderId):
    order    = http.get("http://orders/orders/" + orderId)
    payment  = http.get("http://payments/orders/" + orderId)
    shipping = http.get("http://shipping/orders/" + orderId)

    // Sequential per-service timeout: 2s each
    results = await Promise.allSettled([
        order, payment, shipping
    ])

    return {
        order: results[0].status == "fulfilled" ? results[0].value.body : null,
        payment: results[1].status == "fulfilled" ? results[1].value.body : null,
        shipping: results[2].status == "fulfilled" ? results[2].value.body : null,
        _warnings: getFailures(results)
    }
```

### Caching at Gateway

```
// Cache GET /orders for 60s, keyed by URL + auth
cache:
  - endpoint: GET /orders
    ttl: 60
    scope: per_user  // Don't serve user A's cached response to user B
  - endpoint: GET /products
    ttl: 300
    scope: global    // Products are public, cache globally
```

**Cache invalidation via purge:**

```
// Call from service when data changes
POST /gateway/cache/purge
{ "paths": ["/orders/123", "/orders?page=1"] }
```

### Circuit Breaking

States: **Closed** (normal) → **Open** (failing, reject immediately) → **Half-open** (test with few requests) → **Closed** (recovered) or **Open** (still failing). Prevents cascading failures.

```
circuit_breaker:
  threshold: 5           // failures before opening
  timeout: 30000          // 30s half-open wait
  half_open_trials: 3     // requests to test recovery
  error_codes: [500, 502, 503]
```

**Trap:** Gateway becoming a single point of failure — deploy multiple replicas with a load balancer. Use health checks and auto-scaling. The gateway must be at least as available as the services behind it.

**Trap:** Bypassing the gateway internally — internal services calling each other through the gateway adds unnecessary latency and a bottleneck. Internal calls should go through the service mesh or direct service discovery, not the gateway.

---

## 7. GraphQL vs REST — The Real Tradeoffs

### GraphQL Strengths

**Single endpoint** — all ops through `POST /graphql`. No URL design decisions.

```http
POST /graphql
Content-Type: application/json

{
  "query": "query { order(id: \"123\") { id status total items { name price } } }"
}
```

**Client-driven queries** — client specifies exactly the fields it needs. No over-fetching (getting 50 fields when you need 3) or under-fetching (needing multiple requests):

```graphql
query {
  order(id: "123") {
    id
    status
    total
    customer { name email }
    items {
      product { name }
      quantity
    }
  }
}
```

Compare REST: `GET /orders/123` + `GET /customers/456` + `GET /orders/123/items` + multiple `GET /products/{id}`. Four round trips vs one.

**Strong typing** — schema as source of truth. Introspection enables powerful tooling (GraphiQL, auto-complete, schema validation):

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  total: Float!
  customer: Customer!
  items: [OrderItem!]!
  createdAt: DateTime!
}

enum OrderStatus {
  PENDING  PAID  SHIPPED  DELIVERED  CANCELLED
}
```

**Codegen** — generate TypeScript types, React hooks, iOS models from schema. Eliminates manual type mapping.

### GraphQL Weaknesses

**Caching complexity** — all POST to one endpoint, HTTP caching doesn't work (POST responses are not cacheable by default). Three caching strategies:

1. **Automatic Persisted Queries (APQ):** Server stores hashed queries; client sends hash via GET request. GET responses are cacheable at CDN level.

```
// First: POST to register query
POST /graphql
{ "query": "...", "extensions": { "persistedQuery": { "version": 1, "sha256Hash": "abc123" } } }

// Subsequent: GET with hash (cacheable)
GET /graphql?extensions={"persistedQuery":{"version":1,"sha256Hash":"abc123"}}
```

2. **Normalized client cache (Apollo Client):** Cache by type+id, not by query. When data for `Order:123` updates, all queries referencing it automatically update.

3. **Resolver-level caching:** Cache expensive resolver results with DataLoader or Redis:

```
const loader = new DataLoader(async (ids) => {
    const cached = redis.mget(ids.map(id => "product:" + id))
    // ... fallback to DB for cache misses
})
```

**N+1 problem** — 100 orders each with items calls the items resolver 100 times. Solution: DataLoader batches and caches resolver calls:

```
const itemsLoader = new DataLoader(async (orderIds) => {
    const items = await db.query(
        "SELECT * FROM order_items WHERE order_id IN (?)", [orderIds])
    return orderIds.map(id => items.filter(i => i.order_id === id))
})
```

> **Trap:** DataLoader solves N+1 but adds latency — the batch waits for `load` calls to accumulate (event loop tick), adding ~10-50ms. Matters for p99 < 50ms APIs.

**Query complexity attacks** — `orders { items { product { supplier { address { country } } } } }` at 5 levels. Protect with depth limiting and cost analysis:

```
graphqlDepthLimit(5)
costAnalysis({ maxCost: 1000, costMap: { "Order.items": 10, "Product.supplier": 5 } })
```

**Rate limiting is harder** — all queries hit the same endpoint. Rate limit by query cost, not request count.

**File uploads are awkward** — no native support. Use multipart request spec or pre-signed upload URLs.

### REST Strengths

**Simple caching** — HTTP caching (Cache-Control, ETag, CDN) works natively. **Easy rate limiting** — per-endpoint limits. **Well-understood** — curl, browser, Postman. **Great for hypermedia** — HATEOAS. **Bulk operations** — natural batch endpoints.

### REST Weaknesses

**Over-fetching** — get 50 fields when you need 3. **Under-fetching** — need 4 API calls for related data. **Multiple round trips** — mobile suffers. **Versioning overhead** — new version = new endpoint.

### Hybrid Approaches

**REST for CRUD + GraphQL for queries:**

```
POST /v1/orders       // REST mutation (idempotent, HTTP cache)
PATCH /v1/orders/123  // REST mutation
POST /graphql          // GraphQL for complex reads
```

**GraphQL BFF** — each client gets a tailored GraphQL schema that translates to microservice calls.

> **Trap:** GraphQL as REST replacement — they solve different problems. GraphQL = client-driven queries. REST = resource-oriented architecture. Use GraphQL for flexible queries, REST for caching/hypermedia/bulk.

> **Trap:** Ignoring caching in GraphQL — use persisted queries, normalized cache, and DataLoader. Don't skip caching just because it's harder.

---

## 8. API Performance

### Response Compression

| Algorithm | Ratio | Notes |
|-----------|-------|-------|
| gzip (level 6) | 4-5x | Fast, general purpose |
| brotli (level 5) | 5-6x | Better ratio, moderate speed |
| brotli (level 11) | 6-8x | Best ratio, slow encode — use for static |

Accept-Encoding negotiation: `Accept-Encoding: gzip, br` → `Content-Encoding: br`.

> **Trap:** Compressing already-compressed data (JPEG, PNG, WebP) — wastes CPU. Check content type before compressing.

### Connection Keep-Alive

```
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

Reuses TCP connection, avoids TLS handshake overhead. Timeout: 5-15s. Max: 100-1000 requests.

### HTTP/2 & HTTP/3

**HTTP/2:** multiplexing (one TCP connection), HPACK header compression, server push (use sparingly).

> **Trap:** HTTP/2 server push pushing uncached resources — can slow down clients. Use `103 Early Hints` or wait for cache-aware push.

**HTTP/3 (QUIC):** UDP-based, zero RTT, no head-of-line blocking (transport level), connection migration for mobile. Adoption increasing via CDNs.

### Pagination

Never return unbounded collections.

**Offset pagination:** `?page=1&per_page=20` — simple but inefficient for large datasets; unstable under writes.

**Cursor pagination:** `?cursor=base64(lastId)&limit=20` — stable, efficient with indexed lookups. Use for real-time data.

```
Link: </orders?page=1>; rel="first",
      </orders?page=3>; rel="prev",
      </orders?page=5>; rel="next",
      </orders?page=10>; rel="last"
```

### Field Selection

```
GET /orders/123?fields=id,status,total
```

```
function getOrder(id, fields):
    order = db.query("SELECT * FROM orders WHERE id = ?", [id])
    return fields ? pick(order, fields.split(",")) : order
```

### Async Processing

Expensive operations → 202 Accepted + job polling or webhook:

```http
HTTP/1.1 202 Accepted
Location: /jobs/rpt-987
Retry-After: 30
```

### Multi-Level Caching

```
Client → CDN (edge cache) → Gateway (response cache) → App (in-memory) → DB
```

Event-driven invalidation:

```
function onOrderUpdated(orderId):
    cdn.purge("/orders/" + orderId)
    redis.del("order:" + orderId)
```

### Database Optimization

- Indexes matching query patterns
- N+1 detection: DataLoader, eager loading
- Slow query logging (>100ms)
- Cursor pagination (index-based) over offset (scan-based)
- Read replicas for read traffic

### Profiling

| Metric | Target | Measure |
|--------|--------|---------|
| p50 latency | < 50ms | Histogram |
| p99 latency | < 200ms | Histogram |
| Error rate | < 0.1% | Status codes |
| Payload size | < 100KB | Response size |
| DB queries | < 10 per request | Query logging |

> **Trap:** Premature optimization — measure first. A 10ms improvement on 10 RPM endpoint is wasted effort. Profile to find real bottleneck.

---

## 9. API Security — OWASP API Top 10

### API1: Broken Object Level Authorization (BOLA)

\#1 API security risk. API doesn't verify user owns the requested object. Most common vulnerability in real-world APIs.

**Vulnerable:**

```
function getOrder(orderId):
    return db.query("SELECT * FROM orders WHERE id = ?", [orderId])
    // No check: does user own this order?

// Scenario: User A requests GET /orders/456 (order belongs to User B)
// Returns full order data — BOLA breach
```

**Fixed:**

```
function getOrder(userId, orderId):
    order = db.query("SELECT * FROM orders WHERE id = ? AND customer_id = ?",
                     [orderId, userId])
    if order is null:
        return 404  // Don't reveal IF the object exists
    return order
```

**Authorization check patterns:**

```
// Pattern 1: Ownership check
SELECT * FROM orders WHERE id = ? AND customer_id = ?

// Pattern 2: Role-based
if request.auth.role != "admin" and order.customer_id != userId:
    return 403

// Pattern 3: Scope-based (OAuth)
if not request.auth.hasScope("orders:read"):
    return 403
```

> **Trap:** Authentication ≠ authorization. You can have perfect JWT validation and still be vulnerable to BOLA. Check authorization on every object-level operation, not just at the endpoint level.

> **Trap:** Returning 403 for BOLA — this reveals the object exists. Return 404 consistently for both "not found" and "not authorized to see" to prevent ID enumeration.

### API2: Broken Authentication

**Common vulnerabilities:**

- Weak JWT secrets (`"secret"`, `"password123"`)
- No MFA for sensitive operations (password change, payout)
- Credential stuffing (no rate limiting on login)
- Token leakage in URLs or logs
- No token rotation (refresh tokens)
- Weak password policies

**JWT best practices:**

```
JWT_SECRET = env("JWT_SECRET")  // Minimum 32 bytes, random

{
  "sub": "user_456",
  "iat": 1722004200,
  "exp": 1722090600,     // Short-lived: 15-60 min
  "scope": "orders:read orders:write"
}

function validateToken(token):
    try:
        payload = jwt.verify(token, env("JWT_SECRET"), { algorithms: ["HS256"] })
        if payload.exp < now(): return null
        return payload
    except: return null
```

**Refresh token flow:** short-lived access token (15 min) + long-lived refresh token (7 days) stored in HTTP-only cookie. Refresh endpoint issues new access token. If refresh token is compromised, rotate it immediately.

### API3: Excessive Data Exposure

Returning full DB rows instead of client-needed fields. Relies on the client to filter — which the attacker can ignore.

**Vulnerable:**

```
function getUsers():
    users = db.query("SELECT * FROM users")
    return users
```

```json
// Response includes password_hash, ssn, internal_notes
{ "id": 1, "name": "John", "email": "john@example.com",
  "password_hash": "$2y$10$...", "ssn": "123-45-6789",
  "internal_notes": "Frequent complainer" }
```

**Fixed:**

```
function getUsers():
    return db.query("SELECT id, name, email FROM users")
```

**GraphQL exacerbates this** — a client can request `User.ssn` if the schema exposes it, even if the UI doesn't use it. Schema design must consider field-level authorization.

> **Trap:** Returning all fields and filtering client-side — this is both a security risk (sensitive data in transit, attacker ignores client-side filter) and a performance issue (unnecessary bandwidth). The server controls what's returned.

### API4: Lack of Resources & Rate Limiting

**Vulnerabilities:**

- No rate limiting → brute force attacks, DoS, billing abuse, scraping
- No pagination limits → client requests 100,000 items → OOM or timeout
- No request size limits → large payload attacks (XML bomb, JSON depth)
- No complexity limits in GraphQL → expensive queries crash the server

**Rate limiting strategies:**

```
// Per-authenticated-user rate limit
rateLimiter.check("user:" + userId, { max: 100, windowMs: 60000 })

// Per-endpoint rate limit (different limits for different costs)
rateLimiter.check("endpoint:" + method + path, { max: 1000, windowMs: 60000 })

// Per-IP rate limit for unauthenticated endpoints
rateLimiter.check("ip:" + ip, { max: 10, windowMs: 60000 })

// Return headers
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1627200000
Retry-After: 18
```

**Request size limits:**

```
// Limit JSON body to 1MB
app.use(bodyParser.json({ limit: "1mb" }))

// Limit pagination
if (request.query.per_page > 100):
    return 422 "per_page max is 100"
```

### API5: Broken Function Level Authorization

Admin endpoints accessible to regular users. Deny by default — check role on every controller action.

### API6: Mass Assignment

Clients set fields they shouldn't (role, status, owner, balance) via POST/PUT.

```
// WHITELIST approach, never blacklist
ALLOWED_FIELDS = ["name", "email", "avatar_url"]
filtered = pick(body, ALLOWED_FIELDS)
db.query("UPDATE users SET ? WHERE id = ?", [filtered, userId])
```

### API7: Security Misconfiguration

CORS `*`, verbose errors (stack traces), missing HSTS/X-Content-Type-Options, default credentials.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Cache-Control: no-store
```

### API8: Injection

SQL injection — always use parameterized queries, never string interpolation.

NoSQL injection — sanitize types (`parseInt`), reject operators in user input.

### API9: Improper Assets Management

Old API versions still running, undocumented endpoints, shadow APIs.

Countermeasures: OpenAPI spec as inventory, sunset headers, network segmentation (internal endpoints not routable from public internet).

### API10: Insufficient Logging & Monitoring

No audit trail for API access or anomalies.

Log: timestamp, method, path, user_id, ip, status, latency, request_id.

Alert on: multiple 401/403 (brute force), high request rate (scraping), access to deprecated endpoints, BOLA attempts (user accessing many different IDs).

> **Trap:** Rate limiting by IP only — a malicious user on shared network (office, university) bypasses IP-based limits. Always rate limit by authenticated user ID too.

---

## 10. Contract Testing & API Governance

### Consumer-Driven Contracts (CDC) with Pact

**Concept:** Consumer defines expected interaction (request + response). Provider verifies it satisfies all contracts.

**Workflow:**

```
Consumer writes Pact test → publishes to Pact Broker → Provider CI verifies
```

**Consumer test (JS):**

```
const provider = new PactV3({ consumer: "OrderWebApp", provider: "OrderApi" })

test("returns order by ID", async () => {
    provider
        .given("an order with ID ord_123 exists")
        .uponReceiving("a request for order ord_123")
        .withRequest({ method: "GET", path: "/orders/ord_123" })
        .willRespondWith({
            status: 200,
            body: { id: "ord_123", status: "paid" },
        })

    await provider.executeTest(async (mockServer) => {
        const res = await fetch(`${mockServer.url}/orders/ord_123`)
        expect(await res.json()).toEqual({ id: "ord_123", status: "paid" })
    })
})
```

**Provider verification (Laravel):**

```
class OrderApiProviderTest extends TestCase
{
    public function test_satisfies_contracts()
    {
        (new Verifier(new PactConfig))
            ->setProviderState("/pact/state")
            ->verify(PactVerifyConfig::create()
                ->setProviderBaseUrl("http://localhost:8000")
                ->setPactBrokerUrl("https://pact-broker.example.com"));
    }
}
```

**Provider state handler:**

```
app.post("/pact/state", (req, res) => {
    if (req.body.state === "an order with ID ord_123 exists")
        db.seed("orders", { id: "ord_123", status: "paid" })
    res.json({ success: true })
})
```

**Benefits:** early breakage detection, independent deployments, documented contracts.

> **Trap:** Contract tests ≠ integration tests — they verify shape (request → response), not behavior (business logic, side effects, persistence). They complement integration tests.

> **Trap:** Over-specifying contracts — exact values make brittle contracts. Use Pact matchers:

```
body: {
    id: like("ord_123"),
    status: term({ generate: "paid", matcher: "^(pending|paid|shipped)$" }),
    total: decimal(29.99)
}
```

### OpenAPI Linting with Spectral

Enforce naming, required fields, error format, security:

```
# spectral.yml
rules:
  path-kebab-case:
    message: "Paths must be kebab-case"
    given: "$.paths[*]~"
    then:
      function: pattern
      functionOptions: { match: "^/v[0-9]+/[a-z0-9-]+(/[a-z0-9-]+)*$" }

  error-response-schema:
    message: "Error responses must have code and message"
    given: "$.paths[*][*].responses[4??].content.application.json.schema"
    then:
      function: schema
      functionOptions:
        schema: { type: object, required: ["code", "message"] }

  require-auth:
    message: "All endpoints must have security"
    given: "$.paths[*][*]"
    then: { function: truthy, field: security }
```

```
npx spectral lint openapi.yaml
```

### Breaking Change Detection

```
npx openapi-diff openapi-v1.yaml openapi-v2.yaml
```

Block PRs with breaking changes unless a deprecation policy is followed.

### API Consistency

**Naming:**

```
Resources: plural nouns — GET /orders, POST /orders, GET /orders/{id}
Nested:    GET /orders/{id}/items
Query:     ?sort_by=created_at&filter[status]=active
Errors:    { "code": "ORDER_NOT_FOUND", "message": "..." }
```

**Status codes:** 200 (GET success), 201 (POST), 204 (DELETE), 400 (validation), 401 (no auth), 403 (no permission), 404 (not found), 409 (conflict), 422 (validation), 429 (rate limit).

**Response structure:**

```
// Collection
{ "data": [...], "meta": { "total": 200, "page": 1, "per_page": 20 },
  "links": { "self": "...", "next": "...", "last": "..." } }

// Single
{ "data": { "id": "123", ... } }

// Error
{ "error": { "code": "VALIDATION_ERROR", "message": "...",
    "details": [{ "field": "email", "message": "required" }] } }
```

### API Changelog

```
## 2026-07-26
### Added
- POST /orders/{id}/split
- Field discount_total on GET /orders/{id}
### Deprecated
- Field legacy_total — Use total. Removes 2026-10-26.
```

> **Trap:** Not versioning contract tests — Pact files should be tagged with consumer and provider versions.

---

## 11. Tier 3 Q&A Drill

Close the file, answer out loud, then check your answer for vagueness.

### HATEOAS & RMM

**Q1:** What is the Richardson Maturity Model and what distinguishes Level 2 from Level 3?

<details>
<summary>Answer</summary>

RMM: Level 0 (HTTP as tunnel, SOAP), Level 1 (resource URIs), Level 2 (HTTP verbs + status codes), Level 3 (hypermedia controls — responses include links for next actions). At Level 2 the client needs docs to know `POST /orders/{id}/pay` is valid. At Level 3 the `pay` link only appears when the order is `pending` — the client discovers actions dynamically.
</details>

**Q2:** When would you NOT use HATEOAS?

<details>
<summary>Answer</summary>

Internal microservices (fixed contracts), mobile apps with pre-defined UI, high-throughput low-latency APIs (link serialization overhead), single first-party apps (decoupling benefit doesn't justify complexity).
</details>

**Q3:** How do you version hypermedia links?

<details>
<summary>Answer</summary>

Through media type versioning, not URL versioning. Use vendor-specific media types: `application/vnd.myapi.v2.hal+json`. URLs are opaque to the client in true HATEOAS. When link semantics or available actions change, create a new media type version.
</details>

**Q4:** What's the difference between HAL and Siren as hypermedia formats?

<details>
<summary>Answer</summary>

HAL is simpler — uses `_links` and `_embedded` objects. Good for read-heavy APIs where you just need to link to related resources. Siren is more expressive — includes `actions` with method, href, type, and fields. A Siren client can render a form (e.g., "Pay" button with payment_method field) without prior documentation. Choose Siren when clients need to discover available actions dynamically; choose HAL when you just need resource linking.
</details>

### Backward Compatibility

**Q5:** Explain Postel's Law applied to API evolution.

<details>
<summary>Answer</summary>

Conservative in what you send (follow schema, don't remove fields, don't change types). Liberal in what you accept (tolerate extra fields, accept multiple formats, map legacy field names). Allows clients to evolve at their own pace without breaking.
</details>

**Q6:** A field `order_total` must be renamed to `total`. Walk through the migration.

<details>
<summary>Answer</summary>

1. Add `total`, keep `order_total`. Both populated with same value. (Month 1)
2. Mark `order_total` deprecated with `Deprecation: true` and `Sunset:` headers. Include `Link` header to migration docs. (Month 2)
3. Return `order_total` as null, include `X-Deprecated-Fields: order_total`. (Month 3)
4. Remove `order_total`. (Month 4+)
During the entire migration, accept `order_total` in requests and map it to `total` internally.
</details>

**Q7:** You discover a field has a typo — `"descripton"` instead of `"description"`. Do you fix it?

<details>
<summary>Answer</summary>

No — fixing it breaks any client reading `descripton`. Instead, add the correct field `description` and mark `descripton` as deprecated using the field expansion pattern. Return both for two release cycles, then remove the old one. Even though it was a bug, it's now a contract.
</details>

**Q8:** What headers communicate deprecation, and when do you use each?

<details>
<summary>Answer</summary>

- `Deprecation: true` — signals the endpoint or response is deprecated (draft standard)
- `Sunset: Sat, 31 Jan 2026 23:59:59 GMT` — tells the client when the feature will be removed
- `Link: </docs/migration-v2>; rel="deprecation"` — URL to migration guide
- `Warning: 299 - "..."` — human-readable deprecation message
- `X-Deprecated-Fields: field1, field2` — per-field deprecation (custom header)
Use all that apply. Clients monitoring these headers can plan migrations proactively.
</details>

### Idempotency

**Q9:** How does `Idempotency-Key` prevent duplicate payments?

<details>
<summary>Answer</summary>

Client sends UUID. Server checks Redis — if absent, processes and stores `{key → response}` with 24h TTL. If same key arrives (retry), server returns stored response. `NX` flag on initial SET prevents concurrent duplicates from two simultaneous retries.
</details>

**Q10:** What if TTL expires and client retries with same key?

<details>
<summary>Answer</summary>

Key is gone from Redis → treated as new → duplicate. That's why you also need a DB unique constraint on the business identifier (e.g., `UNIQUE(merchant_ref, amount)`). Idempotency key is a performance optimization within the TTL window; unique constraint is the safety net.
</details>

**Q11:** How do you distinguish an idempotent replay from a new response?

<details>
<summary>Answer</summary>

`X-Idempotent-Replay: true` header. The response body and status code must be identical to the original response. A 201 Created on first request should return 201 (with the replay header) on retry, not 200 OK. Changing the status code breaks the idempotency contract.
</details>

**Q12:** What scope do you use for idempotency keys? Why?

<details>
<summary>Answer</summary>

Per-user + per-resource scope: `idempotency:{user_id}:{resource_type}:{client_key}`. This allows different users to reuse the same UUID (they're in different scopes) and different resources (e.g., creating an order vs creating a payment) to have independent idempotency. Global scope forces clients to generate globally unique UUIDs, which is harder to get right.
</details>

**Q13:** Your idempotency system uses Redis. What happens if Redis goes down?

<details>
<summary>Answer</summary>

Without Redis, idempotency checks fail open — all requests are treated as new, potentially creating duplicates. Mitigations: (1) Redis replication with automatic failover, (2) degrade gracefully with a DB-backed idempotency check as fallback (but slower), (3) have the unique DB constraint as the ultimate safety net. Document that idempotency guarantees degrade during Redis outage.
</details>

### Bulk Operations

**Q14:** Design a batch API for 1000 orders. Error handling?

<details>
<summary>Answer</summary>

Synchronous batch with per-item status codes (201 success, 422 validation error, 409 duplicate). Validate all items before processing any — reject with batch-level error if pre-validation fails. Enforce max batch size (100), return 422 if exceeded. For >100 items, use async pattern (202 + job ID + webhook callback). Each item processes independently — partial success model. Return a summary with total/succeeded/failed counts.
</details>

**Q15:** When use transactional (all-or-nothing) vs partial-success batch?

<details>
<summary>Answer</summary>

Transactional: financial transfers, inventory adjustments where consistency across items is critical — one failure rolls back all. Partial-success: order creation, data import — some invalid SKUs shouldn't block valid ones. Document with a flag like `"atomic": true`. The API should default to partial-success and require explicit opt-in for transactional mode.
</details>

### Webhooks

**Q16:** How do you sign and verify webhook payloads?

<details>
<summary>Answer</summary>

Sign raw JSON body bytes (never parse and re-serialize) with HMAC-SHA256 + per-webhook secret. Header: `X-Signature-256: t=timestamp,v1=base64sig`. Consumer splits header, extracts timestamp, reconstructs `timestamp + "." + rawBody`, computes HMAC, compares with `hash_equals` (constant-time). Reject if timestamp is more than 5 minutes from current time (replay protection). Store processed event IDs in Redis for deduplication.
</details>

**Q17:** At-least-once delivery + consumer dedup?

<details>
<summary>Answer</summary>

Retry with exponential backoff on non-2xx responses or network errors (10s → 24h max). Consumer deduplicates via event `id` in Redis with 7-day TTL. If the ID exists, return 200 without processing. This creates effectively-once delivery.
</details>

**Q18:** Consumer down for 2 hours. What happens in the webhook system?

<details>
<summary>Answer</summary>

Retry queue grows. Exponential backoff: 10s, 30s, 90s, 270s... Each attempt recorded in delivery logs. After 24h of consecutive failures, message moves to dead letter queue. Operations alerted: DLQ size > 0. Team investigates (misconfigured URL, consumer outage, auth failure?). After consumer recovers, replay events from DLQ manually or programmatically.
</details>

**Q19:** Your webhook consumer wants exactly-once delivery. Can you guarantee it?

<details>
<summary>Answer</summary>

No — exactly-once is impossible in distributed systems (FLP impossibility, network partitions). What you can provide is **at-least-once delivery with idempotent consumers** (effectively once). The webhook system retries on failure, and the consumer deduplicates by event ID. The consumer must implement deduplication using the `id` field and a store with sufficient TTL (7 days).
</details>

### API Gateways

**Q20:** API gateway vs service mesh — when do you need each?

<details>
<summary>Answer</summary>

Gateway: north-south traffic (external → internal), provides auth, rate limiting, caching, request transformation, TLS termination. Mesh: east-west traffic (service → service), sidecar proxies providing mTLS, retry, circuit breaking, observability. They complement each other. In a typical deployment: gateway at the edge authenticates external requests, then internal calls between services use mesh with mTLS and automatic retry.
</details>

**Q21:** What should NOT be in an API gateway? Give examples.

<details>
<summary>Answer</summary>

Business logic — order validation, payment processing, inventory checks, user registration, email sending. Service-specific configuration — per-service error handling, service-specific caching TTLs, service-specific authentication logic. The gateway handles cross-cutting concerns only. If you need to modify the gateway when adding a new feature to a service, the gateway has too much business logic.
</details>

**Q22:** How do you handle gateway failure in production?

<details>
<summary>Answer</summary>

Multiple gateway replicas behind a load balancer (HA mode). Health check endpoints for load balancer probes. Auto-scaling based on CPU/memory/request rate. Rate limiting per client prevents one noisy tenant from starving others. Circuit breaking to unhealthy downstream services prevents cascading failures. Graceful degradation: if a downstream service is slow, the gateway returns a partial response or 503 instead of hanging.
</details>

### GraphQL vs REST

**Q23:** When do you choose GraphQL over REST?

<details>
<summary>Answer</summary>

GraphQL: complex data graphs with nested relationships, multiple frontend clients with different data needs, mobile apps (avoid over/under-fetching on slow networks), rapid frontend iteration without backend changes, when you need strong typing and introspection. REST: simple CRUD APIs, heavy caching requirements (HTTP caching with CDN), bulk operations, hypermedia/HATEOAS, public APIs with diverse unknown consumers, when you need HTTP semantics (ETags, conditional requests, caching headers).
</details>

**Q24:** How do you protect a GraphQL API from expensive or malicious queries?

<details>
<summary>Answer</summary>

1. **Query depth limiting** — max 5-7 levels deep. Catches deeply nested queries.
2. **Query cost analysis** — assign cost per field (1 for scalars, 5-10 for list relationships), sum the total, reject if > threshold.
3. **Rate limiting by cost** — limit by accumulated query cost per user per window, not by request count.
4. **Query timeouts** — kill queries exceeding N seconds.
5. **Persisted queries** in production — only allow pre-registered queries, reject ad-hoc queries.
6. **Alias limiting** — prevent a single query from making the same field request 1000 times via aliases.
</details>

**Q25:** You have a REST API and a team wants to add GraphQL. How do you approach the hybrid?

<details>
<summary>Answer</summary>

Start with a GraphQL BFF (Backend For Frontend) layer — a thin GraphQL server that sits between the client and the existing REST API. The BFF translates GraphQL queries into REST calls. This keeps the REST API untouched while providing GraphQL benefits to the frontend. The REST API remains the source of truth for internal services, third-party integrations, and clients that need HTTP caching/bulk operations. Over time, consider moving read-heavy endpoints to direct GraphQL resolvers if the REST translation layer becomes a bottleneck.
</details>

### API Performance

**Q26:** gzip vs brotli — which to use for API responses?

<details>
<summary>Answer</summary>

Brotli at quality 5 for JSON/text responses (better compression ratio than gzip, typically 5-6x vs 4-5x). Gzip for latency-sensitive scenarios (brotli encoding is slower, especially at higher quality levels). Set brotli quality 11 only for static assets (slow encode, best ratio). Don't compress already-compressed content (images, video). Set a minimum size threshold (1KB) to avoid compressing trivial responses.
</details>

**Q27:** Offset pagination vs cursor pagination — tradeoffs?

<details>
<summary>Answer</summary>

Offset (`?page=3&per_page=20`): simple to implement and understand, supports random access ("go to page 7"), but inefficient for large datasets (database must scan offset rows) and unstable under concurrent writes (insertions shift page boundaries). Cursor (`?cursor=base64(lastId)&limit=20`): stable under writes, efficient with indexed lookups (WHERE id > cursor), but no random access and more complex client logic. Use cursor for real-time feeds, activity logs, and any data with ongoing writes. Use offset for admin panels where users need to jump to arbitrary pages.
</details>

**Q28:** What metrics do you track to measure API performance?

<details>
<summary>Answer</summary>

p50/p95/p99 latency per endpoint, error rate (5xx) per endpoint, request rate per endpoint, payload size distribution, database query count per request, database query time as % of total latency, cache hit ratio (CDN + app-level), upstream dependency latency (if the API calls other services). Build dashboards with these sliced by endpoint, user tier, and deployment version.
</details>

### API Security

**Q29:** What is BOLA (API1) and how do you prevent it?

<details>
<summary>Answer</summary>

Broken Object Level Authorization — when an API doesn't verify the authenticated user owns the requested object. E.g., `GET /orders/123` returns order 123 even if it belongs to a different user. Prevention: every object-level operation checks ownership — `SELECT * FROM orders WHERE id = ? AND customer_id = ?`. Return 404 (not 403) to prevent ID enumeration. The check must be in the data access layer, not just in the controller.
</details>

**Q30:** Authentication vs authorization — what's the difference and why does it matter for APIs?

<details>
<summary>Answer</summary>

Authentication verifies identity (who you are — JWT validation, API key check, password verification). Authorization verifies permissions (what you can do — can this user access this order? can this user delete products?). APIs must implement both. BOLA is the #1 OWASP API risk specifically because many APIs implement authentication properly but forget to check authorization on every object-level operation. A perfect JWT doesn't prevent User A from reading User B's orders.
</details>

**Q31:** What is mass assignment (API6) and how do you prevent it in a Laravel API?

<details>
<summary>Answer</summary>

Mass assignment occurs when an attacker sends extra fields in a POST/PUT payload that they shouldn't be able to set — e.g., `{ "role": "admin", "is_verified": true, "balance": 999999 }`. In Laravel, use `$fillable` to whitelist writable fields:

```php
class User extends Model
{
    protected $fillable = ['name', 'email', 'avatar_url'];
    // role, is_verified, balance are NOT in fillable — they're protected
}
```

Never use `$guarded = []` or `$guarded = ['id']` without whitelisting. The whitelist approach is always safer than blacklist.
</details>

### Contract Testing & Governance

**Q32:** How does Pact CDC work and what does it NOT test?

<details>
<summary>Answer</summary>

Consumer writes a Pact test defining the expected HTTP interaction (request → response). This produces a Pact JSON file published to a Pact Broker. The provider verifies it can satisfy all consumer contracts in CI. Pact tests verify the **shape** of the interaction — they don't test business logic correctness, side effects (did the order actually get created?), data persistence, performance, security, or error handling beyond the defined responses. They complement integration tests, they don't replace them.
</details>

**Q33:** How do you detect breaking changes in an OpenAPI spec?

<details>
<summary>Answer</summary>

Use `openapi-diff` to compare the old and new spec. It detects: removed endpoints, removed fields, changed field types, tightened constraints (optional → required), renamed fields, changed response status codes. Run this in CI on every PR that modifies the OpenAPI spec (`.yaml` or `.json`). Block merges with breaking changes unless the PR includes a deprecation policy (Sunset headers, parallel run plan, migration guide).
</details>

**Q34:** What rules would you enforce with Spectral linting on an OpenAPI spec?

<details>
<summary>Answer</summary>

(1) Path naming: kebab-case, plural nouns. (2) Error responses: must have `code` and `message` fields on 4xx/5xx. (3) Security: every endpoint must have a security scheme defined. (4) Pagination: paginated endpoints must return `{ data, meta: { cursor, has_more } }`. (5) Versioning: paths must start with `/v1/` or similar. (6) Response structure: collections wrapped in `data`, errors wrapped in `error`. (7) No `anyOf`/`oneOf` without discriminators. These ensure consistency across dozens of microservices.
</details>

### System Design

**Q35:** Design a payment API that supports idempotency, webhooks, and batch reconciliation.

<details>
<summary>Answer</summary>

**Endpoints:** `POST /payments` with `Idempotency-Key` header, `GET /payments/{id}`, `POST /payments/batch` (max 100 items). **Webhooks:** `payment.succeeded`, `payment.failed` events delivered with HMAC-SHA256 signing and exponential backoff retry. **Idempotency:** Redis-backed with 24h TTL, NX lock for concurrency, `X-Idempotent-Replay` header on replay. **Batch:** validate all items before processing, per-item status codes, partial success model, summary with counts. **Security:** BOLA check on every payment (user must own payment), rate limiting per user (100/min), request logging for audit trail. **Monitoring:** payment success rate, webhook delivery rate, dead letter queue alerts.
</details>

**Q36:** 50 microservices and growing. How do you govern the API ecosystem?

<details>
<summary>Answer</summary>

1. API Gateway (Kong/Envoy) as single external entry point — all auth, rate limiting, TLS at the edge.
2. OpenAPI spec per service as source of truth — published to a central registry.
3. Spectral linting in CI — enforce naming conventions, security schemes, error format consistency.
4. Pact contract testing — consumers define expectations, provider CI verifies; catches breaking changes before deployment.
5. Breaking change detection with `openapi-diff` — no breaking changes without deprecation policy.
6. Automated deprecation headers (`Sunset`, `Deprecation`) — tracked centrally, removed on schedule.
7. API changelog — maintained by platform team, published to all developers.
8. API metrics dashboard — latency, error rate, request count per endpoint per service.
9. Quarterly API review — audit endpoints for usage, deprecation candidates, design issues.
10. API design review process — every new endpoint goes through a design review with the platform team before implementation.
</details>

**Q37:** You're adding a new endpoint that returns sensitive financial data. Walk through your security checklist.

<details>
<summary>Answer</summary>

(1) Authentication: JWT with expiration, short-lived access tokens. (2) Authorization: check user owns the resource (`WHERE customer_id = ?`), return 404 if not. (3) Rate limiting: stricter limit for this endpoint (e.g., 10/min per user). (4) Audit logging: log every access with user_id, timestamp, IP, resource_id. (5) Response: return only the fields the client needs (no full DB row). (6) Encryption in transit: enforce HTTPS with HSTS. (7) Idempotency: if it's a mutation, require `Idempotency-Key`. (8) Input validation: strict schema validation, reject unknown fields. (9) CORS: restrict origins if browser-facing. (10) Penetration testing: before going live, test for BOLA, mass assignment, injection.
</details>

**Q38:** A client reports that their system breaks after your API deployment. What do you do?

<details>
<summary>Answer</summary>

(1) Immediately check if the deployment introduced a breaking change — diff the OpenAPI spec before and after. (2) If it did, roll back the deployment. (3) Identify the breaking change: was it a removed field, type change, or new requirement? (4) Add the field back with a deprecation notice. (5) Follow the proper deprecation process: announce, set Sunset header, give migration timeline. (6) Root cause analysis: why wasn't this caught? Was the API governance process bypassed? Did contract tests miss it? (7) Improve the process: add openapi-diff to CI, enforce Pact verification, require design review for all endpoint changes.
</details>

---

> **Next:** Move to the question bank (`04-question-bank.md`) for 160+ interview questions, system design prompts, code puzzles, and STAR story frameworks.
