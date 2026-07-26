# REST API — Tier 1: Basic (Fundamentals & HTTP)

REST is everywhere. Every job posting for a backend engineer expects you to know it. Yet most engineers — even senior ones — have only a shallow understanding. They conflate REST with "any HTTP API that returns JSON." They've never read Roy Fielding's dissertation. They can't explain what makes an API RESTful vs simply HTTP-based.

This tier covers the absolute fundamentals. No framework. No PHP. No Laravel. No Go. Just HTTP, resources, and the constraints that define REST.

> **Target:** Senior Backend Engineer (8+ years)
> **Tier:** 1 — Basic
> **Theme:** Language-agnostic REST & HTTP fundamentals

---

## Table of Contents

1. [What REST Actually Is](#1-what-rest-actually-is)
2. [HTTP Methods — The Complete Semantics](#2-http-methods--the-complete-semantics)
3. [HTTP Status Codes — The Complete Reference](#3-http-status-codes--the-complete-reference)
4. [Headers — Request & Response](#4-headers--request--response)
5. [URL Design — Resources & Naming](#5-url-design--resources--naming)
6. [Content Negotiation](#6-content-negotiation)
7. [REST vs the Alternatives](#7-rest-vs-the-alternatives)
8. [Idempotence & Safety — Deep Dive](#8-idempotence--safety--deep-dive)
9. [Request/Response Structure](#9-requestresponse-structure)
10. [Tier 1 Q&A Drill](#10-tier-1-qa-drill)

---

## 1. What REST Actually Is

REST stands for **RE**presentational **S**tate **T**ransfer. It was defined by Roy Fielding in Chapter 5 of his 2000 PhD dissertation at UC Irvine. If you call yourself a senior backend engineer, you should have at least skimmed it.

REST is an **architectural style** — not a protocol, not a framework, not a library. It defines six constraints:

| # | Constraint | Description |
|---|-----------|-------------|
| 1 | **Client-Server** | Separation of concerns. Clients don't care about data storage. Servers don't care about UI. Both can evolve independently. |
| 2 | **Stateless** | Every request from the client contains all the information the server needs to understand and process it. No client context stored server-side between requests. Sessions are a violation. |
| 3 | **Cacheable** | Responses must implicitly or explicitly define themselves as cacheable or not. Caching improves performance and reduces latency. |
| 4 | **Uniform Interface** | The defining constraint. Four sub-constraints: identification of resources, manipulation through representations, self-descriptive messages, and HATEOAS (hypermedia as the engine of application state). |
| 5 | **Layered System** | Intermediaries (proxies, gateways, load balancers) can be inserted between client and server without either knowing. Each layer only knows about the immediate layer. |
| 6 | **Code on Demand (optional)** | Servers can temporarily extend client functionality by transferring executable code (e.g., JavaScript). This is the only optional constraint. |

### The Richardson Maturity Model

Leonard Richardson proposed a maturity model in 2008 that grades APIs from Level 0 to Level 3:

```
Level 0: The Swamp of POX
    Single URL, single HTTP method (typically POST). Everything is an XML-RPC or SOAP-style
    tunnel. Think legacy enterprise SOAP-over-HTTP.

Level 1: Resources
    Multiple URLs, each representing a resource, but still using only POST.
    Example: POST /orders, POST /users, POST /products

Level 2: HTTP Verbs
    Proper use of GET, POST, PUT, PATCH, DELETE.
    HTTP status codes for errors. This is where most "REST APIs" live.

Level 3: Hypermedia (HATEOAS)
    Responses include links that guide clients through the API.
    The client navigates the API entirely through these links — no hardcoded URLs.
    Almost no one actually does this. Even well-known "REST APIs" are Level 2.
```

> **Trap:** Calling every HTTP API "RESTful"
>
> Just because you return JSON over HTTP doesn't mean you're RESTful. If you use sessions, you're violating statelessness. If your client hardcodes URLs (and it does), you're violating HATEOAS. Most APIs claiming to be REST are Level 2 at best — "HTTP APIs with proper verbs." Call them that. Senior engineers distinguish REST from HTTP API.
>
> **Follow-up:** "We have sessions in our REST API — is that OK?"
> No. Sessions break statelessness. The server should not store client state between requests. If the user is authenticated, send credentials (token, API key) with every request. That's what the Authorization header is for. Statelessness is what makes REST scalable — any server can handle any request because there's no session affinity.

> **Trap:** Conflating REST with CRUD
>
> REST is about resources and their representations. CRUD is a database operations pattern (Create, Read, Update, Delete). They map roughly to POST/GET/PUT/DELETE — but the mapping is loose. REST operations don't have to be CRUD. You can have operations that don't map to CRUD (e.g., `POST /payments` is not "creating a payment resource" — it's executing a payment). Don't say "REST is just CRUD over HTTP."
>
> **Follow-up:** "What do you do when an operation doesn't map to CRUD?"
> Two approaches: (1) Create a resource that represents the operation (e.g., `POST /payment-intents`), or (2) Use a sub-resource with a semantically meaningful name (`POST /orders/{id}/cancel`). The first is more RESTful. The second is pragmatic. Both are better than `POST /cancelOrder`.

---

## 2. HTTP Methods — The Complete Semantics

HTTP defines a set of methods (also called verbs) that indicate the desired action on a resource. You must know what each means, which are safe, which are idempotent, and what the body conventions are.

### The Methods

| Method | Semantics | Safe | Idempotent | Request Body | Response Body |
|--------|-----------|------|------------|-------------|---------------|
| GET | Retrieve a resource | Yes | Yes | No (body ignored) | Resource representation |
| POST | Create a resource or submit data | No | No | Yes | Created resource (201) or status |
| PUT | Replace a resource entirely | No | Yes | Yes | Updated or created resource |
| PATCH | Partially modify a resource | No | Not by default | Yes (diff) | Updated resource |
| DELETE | Remove a resource | No | Yes | Optional (convention: no) | Usually 204 No Content |
| HEAD | Same as GET but no response body | Yes | Yes | No | Headers only |
| OPTIONS | Describe communication options | Yes | Yes | No | Allowed methods, CORS headers |
| CONNECT | Establish a tunnel (e.g., HTTPS proxy) | No | No | No | Tunnel established |
| TRACE | Message loop-back for debugging | Yes | Yes | No | Received request |

### Safe vs Idempotent — Distinction

**Safe:** The method causes no side effects on the server. The resource state doesn't change. GET, HEAD, OPTIONS, TRACE are safe.

**Idempotent:** Making the same request N times produces the same server state as making it once. The response may differ (e.g., the second DELETE returns 404 instead of 200), but the server state is the same.

```http
DELETE /users/42

# First call: 204 No Content — user deleted
# Second call: 404 Not Found — user already gone, server state is the same
# DELETE is idempotent because after the first call, the resource is gone.
# The second call has no further effect on server state.
```

```http
POST /users

# First call: 201 Created — user created with id=1
# Second call: 201 Created — another user created with id=2
# POST is NOT idempotent. Each call creates a new resource.
```

> **Trap:** Using POST for everything (not RESTful)
>
> I've seen APIs where every operation is POST — including reads. `POST /getUser`, `POST /deleteOrder`. This is Level 0 on the Richardson Maturity Model (the "Swamp of POX"). It breaks cacheability, breaks idempotency guarantees, and makes the API impossible for intermediaries to understand. GET requests can be cached by browsers and proxies. POST requests cannot. If you're building an API that only uses POST, you're building RPC over HTTP, not REST.
>
> **Follow-up:** "But POST can do everything — why use multiple methods?"
> HTTP methods exist so intermediaries (proxies, caches, browsers) can understand the intent. A proxy knows a GET is safe and cacheable. A proxy knows a PUT is idempotent and can be retried safely. When you POST everything, you lose all of that. Also, you lose HTTP-level caching, bookmarkability of resources, and the ability to use standard HTTP tooling.

> **Trap:** Thinking PATCH is always idempotent
>
> Whether PATCH is idempotent depends entirely on the patch format:
>
> - **JSON Patch (RFC 6902):** Not idempotent. Operations like "add item to array" applied twice produce different results.
> - **JSON Merge Patch (RFC 7396):** Idempotent. Setting a field to a value is the same whether you do it once or twice.
> - **Custom patch formats:** Depends entirely on semantics.
>
> The HTTP spec (RFC 7231) says PATCH is *not* guaranteed to be idempotent. Don't assume it is. If you need idempotent updates, use PUT (full replace) or require an `If-Match`/`If-Unmodified-Since` header with PATCH.
>
> **Follow-up:** "How do you make PATCH idempotent?"
> Use conditional requests with `If-Match` (ETag) or `If-Unmodified-Since`. The PATCH request includes the expected current state. If the resource changed between reads, the request fails with 412 Precondition Failed. This gives you optimistic concurrency control. Alternatively, use a JSON Merge Patch format which is inherently idempotent.

> **Trap:** Sending a body with DELETE
>
> HTTP semantics say DELETE *may* have a body, but servers are not required to parse it. Many proxies and servers strip the body from DELETE requests. If you need to send additional information with a DELETE (e.g., a reason for deletion), encode it in headers or query parameters. Better yet, design your API so DELETE doesn't need extra data. If you must — use `POST /orders/{id}/cancel` instead of trying to body-encode a DELETE.
>
> **Follow-up:** "What about bulk DELETE?"
> Don't use DELTE with a body body for bulk operations. Use `POST /bulk/delete` with a body, or pass IDs as query parameters: `DELETE /users?id=1&id=2&id=3`. Some APIs accept comma-separated IDs: `DELETE /users/1,2,3`. The most RESTful approach is to create a "deletion job" resource: `POST /deletion-jobs` with a list of IDs.

---

## 3. HTTP Status Codes — The Complete Reference

HTTP status codes are three-digit numbers grouped into five classes. A senior engineer should know the most common ones and, more importantly, use them correctly.

### 1xx — Informational

These are provisional responses. The client should continue or the server is switching protocols.

| Code | Name | Usage |
|------|------|-------|
| 100 | Continue | Server received headers, client should send body. Rarely used in REST APIs. |
| 101 | Switching Protocols | Server agrees to upgrade protocols (e.g., WebSocket upgrade via `Upgrade` header). Used in WebSocket handshake. |
| 102 | Processing | WebDAV only. Server is processing but no response yet. |
| 103 | Early Hints | Server hints to client about resources to preload while the full response is generated. |

### 2xx — Success

The request was received, understood, and accepted.

| Code | Name | Usage |
|------|------|-------|
| 200 | OK | Standard success. GET requests return 200 with the resource body. POST/PUT/PATCH operations that return the modified resource. |
| 201 | Created | Resource was created. Used with POST (and sometimes PUT). Include `Location` header pointing to the new resource. |
| 202 | Accepted | Request accepted for processing but not completed. Used for async operations (e.g., report generation, batch processing). No resource created yet. |
| 204 | No Content | Request succeeded but no body to return. DELETE typically returns 204. PUT/PATCH can return 204 if no body needed. |
| 205 | Reset Content | Client should reset the document view. Rare. |
| 206 | Partial Content | Partial GET (Range header). Used for resumable downloads, video streaming. |

```http
POST /users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}

---

HTTP/1.1 201 Created
Location: /users/42
Content-Type: application/json

{
  "data": {
    "id": 42,
    "name": "Alice",
    "email": "alice@example.com"
  }
}
```

```http
DELETE /users/42

---

HTTP/1.1 204 No Content
```

### 3xx — Redirection

The client must take additional action to complete the request.

| Code | Name | Usage |
|------|------|-------|
| 301 | Moved Permanently | Resource has a new permanent URL. Search engines update their index. Browser follows redirect. |
| 302 | Found (Temporary Redirect) | Resource temporarily at a different URL. HTTP/1.0 original. Ambiguous — many clients change POST to GET. |
| 303 | See Other | Redirect to a different resource (often a status page). Forces GET on the target. Used after POST to avoid re-submission. |
| 304 | Not Modified | Conditional GET — resource hasn't changed. Used with `If-None-Match` (ETag) or `If-Modified-Since`. Response has no body. |
| 307 | Temporary Redirect | Like 302 but guarantees method and body are not changed. POST stays POST. |
| 308 | Permanent Redirect | Like 301 but guarantees method and body are not changed. |

```http
GET /users/42

---

HTTP/1.1 301 Moved Permanently
Location: /v2/users/42
```

```http
GET /users/42
If-None-Match: "abc123"

---

HTTP/1.1 304 Not Modified
ETag: "abc123"
```

### 4xx — Client Error

The request contains bad syntax or cannot be fulfilled.

| Code | Name | Usage |
|------|------|-------|
| 400 | Bad Request | Malformed request syntax, invalid message framing, or deceptive request routing. Use for validation errors (though 422 is more precise). |
| 401 | Unauthorized | Authentication required. The client is not authenticated. MUST include `WWW-Authenticate` header. |
| 403 | Forbidden | The client is authenticated but does not have permission. Do NOT use for "not authenticated" — that's 401. |
| 404 | Not Found | Resource does not exist. Don't reveal *why* — could be "never existed" or "existed and was deleted." |
| 405 | Method Not Allowed | HTTP method not supported for this resource. MUST include `Allow` header listing allowed methods. |
| 406 | Not Acceptable | Cannot generate a response matching the `Accept` header. Returned when content negotiation fails. |
| 407 | Proxy Authentication Required | Client must authenticate with a proxy. |
| 408 | Request Timeout | Server timed out waiting for the request. |
| 409 | Conflict | Request conflicts with current server state. Used for: concurrent edits, duplicate resources, state machine violations. |
| 410 | Gone | Resource is gone and won't come back. Like 404 but more permanent. Search engines remove from index. |
| 411 | Length Required | `Content-Length` header required but missing. |
| 412 | Precondition Failed | One or more conditions in request headers evaluated to false. Used with `If-Match`, `If-Unmodified-Since`. |
| 413 | Payload Too Large | Request entity larger than server is willing to process. |
| 415 | Unsupported Media Type | `Content-Type` not supported by the server. |
| 422 | Unprocessable Entity | Request body is syntactically correct (valid JSON) but semantically invalid (missing required fields, wrong types, business rule violations). WebDAV extension, universally used. |
| 429 | Too Many Requests | Rate limit exceeded. Include `Retry-After` header. |
| 451 | Unavailable For Legal Reasons | Resource blocked due to legal demands (e.g., DMCA takedown). |

```http
POST /users
Content-Type: application/json

{ "name": "" }

---

HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The given data was invalid.",
    "details": {
      "name": ["The name field is required."]
    }
  }
}
```

```http
POST /orders
Content-Type: application/json

{
  "product_id": 5,
  "quantity": 1
}

---

HTTP/1.1 409 Conflict
Content-Type: application/json

{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Product 5 has only 0 units in stock."
  }
}
```

```http
GET /admin/secrets

---

HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
```

```http
GET /admin/secrets
Authorization: Bearer valid_token_but_not_admin

---

HTTP/1.1 403 Forbidden
```

### 5xx — Server Error

The server failed to fulfill a valid request.

| Code | Name | Usage |
|------|------|-------|
| 500 | Internal Server Error | Generic server error. Unhandled exception, unexpected condition. Never leak stack traces. |
| 501 | Not Implemented | Server does not support the functionality. Method not recognized. |
| 502 | Bad Gateway | Server (as gateway/proxy) received an invalid response from an upstream server. Common with reverse proxies (nginx, haproxy). |
| 503 | Service Unavailable | Server temporarily unable to handle request. Overloaded or under maintenance. Include `Retry-After` header. |
| 504 | Gateway Timeout | Server (as gateway/proxy) did not receive a timely response from upstream. |

> **Trap:** Returning 403 when you mean 401
>
> This is **the most common status code error** in professional APIs.
>
> - **401 Unauthorized:** "I don't know who you are." No authentication credentials, or invalid/expired credentials. Response MUST include `WWW-Authenticate` header.
> - **403 Forbidden:** "I know who you are, but you can't do this." Valid authentication, insufficient permissions.
>
> When a request comes in without a token, return **401**, not 403. When a token is valid but the user isn't an admin, return **403**. Mixing them up makes debugging miserable for API consumers.
>
> **Follow-up:** "What should the WWW-Authenticate header look like?"
> ```
> WWW-Authenticate: Bearer realm="api", error="invalid_token", error_description="The access token expired"
> ```
> The format depends on the auth scheme. For Bearer tokens (RFC 6750), you specify the realm and optional error details. For Basic auth: `WWW-Authenticate: Basic realm="User Visible Realm"`.

> **Trap:** Using 500 for validation errors
>
> A validation error is a client error. The client sent bad data. This is squarely in the 4xx range. When you return 500, you're telling the client "the server broke" — which triggers alerts, confuses monitoring, and makes clients think they should retry. Use 422 (Unprocessable Entity) or 400 (Bad Request).
>
> **Follow-up:** "So when do you actually use 500?"
> Only for truly unexpected server-side failures: unhandled exceptions, database connection failures, disk full, out of memory. A 500 response should mean "something is wrong on our end, we need to fix it, please try again later." Every 500 should trigger an alert in your monitoring system.

> **Trap:** Returning 200 with an error in the body instead of proper 4xx
>
> ```json
> // BAD — 200 with error in body
> HTTP/1.1 200 OK
> { "error": true, "message": "User not found" }
> ```
>
> This breaks HTTP semantics. Intermediaries (proxies, CDNs, API gateways) see 200 and think everything is fine. They cache it. They don't retry. They don't alert. Use proper status codes. That's what they're for.
>
> The only acceptable exception is when the response itself is successful but contains an application-level error (like a payment gateway response where "declined" is a valid outcome). Even then, consider whether 200 is really appropriate vs 402 Payment Required.

---

## 4. Headers — Request & Response

HTTP headers carry metadata about the request and response. They're the invisible part of the API that most engineers don't think about until something breaks.

### Common Request Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Accept` | What media types the client can handle | `Accept: application/json` |
| `Content-Type` | Media type of the request body | `Content-Type: application/json` |
| `Authorization` | Credentials for authentication | `Authorization: Bearer eyJhbGci...` |
| `User-Agent` | Client software identifier | `User-Agent: curl/7.68.0` |
| `Referer` | URL of the referring page | `Referer: https://example.com/checkout` |
| `Cache-Control` | Caching directives | `Cache-Control: no-cache` |
| `If-None-Match` | Conditional GET — ETag value | `If-None-Match: "abc123"` |
| `If-Modified-Since` | Conditional GET — timestamp | `If-Modified-Since: Wed, 21 Oct 2020 07:28:00 GMT` |
| `If-Match` | Conditional request — only if ETag matches | `If-Match: "abc123"` |
| `If-Unmodified-Since` | Conditional request — only if not modified | `If-Unmodified-Since: Wed, 21 Oct 2020 07:28:00 GMT` |
| `X-Request-Id` | Trace ID for request correlation | `X-Request-Id: 550e8400-e29b-41d4-a716-446655440000` |

### Common Response Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Media type of the response body | `Content-Type: application/json; charset=utf-8` |
| `Content-Length` | Size of the response body in bytes | `Content-Length: 348` |
| `Location` | URL of a newly created resource | `Location: /users/42` |
| `ETag` | Entity tag for caching/conditional requests | `ETag: "686897696a7c876b7e"` |
| `Cache-Control` | Caching directives | `Cache-Control: max-age=3600, private` |
| `Set-Cookie` | Set a cookie on the client | `Set-Cookie: session_id=abc123; HttpOnly; Secure` |
| `WWW-Authenticate` | Indicate authentication scheme | `WWW-Authenticate: Bearer` |
| `Retry-After` | When to retry (rate limiting, maintenance) | `Retry-After: 120` |
| `Allow` | Allowed HTTP methods on a resource | `Allow: GET, PUT, DELETE` |
| `Vary` | Which headers affect the representation | `Vary: Accept-Encoding, Accept` |

```http
GET /users/42
Accept: application/json
If-None-Match: "abc123"

---

HTTP/1.1 304 Not Modified
ETag: "abc123"
Cache-Control: public, max-age=3600
```

```http
POST /users
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

{
  "name": "Bob"
}

---

HTTP/1.1 201 Created
Location: /users/43
ETag: "def456"
Content-Type: application/json

{
  "data": { "id": 43, "name": "Bob" }
}
```

### Custom Headers — The X- Prefix Deprecation

Historically, custom headers used an `X-` prefix: `X-Request-Id`, `X-RateLimit-Limit`, `X-Custom-Header`.

RFC 6648 (2012) **deprecated** this practice. The reasoning: when custom headers become standardized, you get naming conflicts (e.g., `X-Forwarded-For` became the standard `Forwarded`). Unprefixed headers are more consistent.

| Deprecated (`X-`) | Modern replacement |
|--------------------|-------------------|
| `X-Request-Id` | `Request-Id` (or `X-Request-Id` still common) |
| `X-RateLimit-Limit` | `RateLimit-Limit` |
| `X-Correlation-Id` | `Correlation-Id` |

> **Trap:** Sending sensitive data in headers
>
> Headers are commonly logged by proxies, load balancers, API gateways, and application frameworks. If you put an API key or auth token in a custom header like `X-API-Key: sk_live_xxxx`, it will likely end up in log files, monitoring systems, and error reports. Use the standard `Authorization` header for credentials — some systems have special handling to redact it.
>
> **Follow-up:** "Where should I put trace IDs?"
> Use `X-Request-Id` (or `Request-Id`) for request tracing. This is one case where the X- prefix is still the de facto standard. Forward it from your API gateway (or generate it) and pass it through all services. Include it in every response so clients can reference it when reporting issues.

> **Trap:** Reflecting Origin header without validation (CORS bypass)
>
> Some APIs blindly echo the `Origin` header in `Access-Control-Allow-Origin`:
>
> ```http
> Access-Control-Allow-Origin: *
> # OR worse:
> Access-Control-Allow-Origin: https://evil.com  # verbatim echo of Origin
> ```
>
> If you reflect `Origin` without whitelisting, any website can make authenticated requests from a user's browser. Always validate the `Origin` against a whitelist. If it doesn't match, don't set `Access-Control-Allow-Origin`.
>
> **Follow-up:** "How do I handle CORS properly?"
> 1. Whitelist specific origins, don't use `*` for credentialed requests.
> 2. For preflight (`OPTIONS`), respond with: `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Max-Age`.
> 3. For actual requests, set: `Access-Control-Allow-Origin` (validated value), `Access-Control-Allow-Credentials: true` (if needed), `Access-Control-Expose-Headers` (custom headers the client needs to read).
> 4. Never send `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true` — browsers will reject it.

> **Trap:** Relying on Referer header for authentication
>
> The `Referer` header is easily spoofed. Browsers may omit it for privacy reasons (HTTPS → HTTP transitions, some privacy extensions). Some corporate proxies strip it. Never use `Referer` as an authentication or authorization mechanism. Token-based auth (Bearer, API key) in the `Authorization` header is the correct approach.

---

## 5. URL Design — Resources & Naming

URL design is where API design meets the real world. Good URL design makes your API intuitive. Bad URL design makes developers hate you.

### Principles

**Resources are nouns, not verbs.**

```
Good:   GET /orders
Bad:    GET /getOrders
Worse:  GET /get_all_orders
Terrible: POST /getOrders (yes, people do this)
```

**Plural nouns for collections.**

```
Good:   GET /users, GET /orders, POST /products
Bad:    GET /user, GET /order, POST /product
```

Consistency matters more than the specific choice. If you use plural, use it everywhere.

**Use nesting for hierarchy, not for convenience.**

```
Good:   GET /organizations/{id}/users/{userId}
Good:   GET /orders/{id}/line-items
Bad:    GET /users/42/organizations/5/orders/12/line-items/8
```

Generally, limit nesting to 2-3 levels. Beyond that, use query parameters or top-level resource IDs.

**Kebab-case is the most common convention, but pick one and be consistent.**

| Convention | Example | Notes |
|------------|---------|-------|
| kebab-case | `/order-history` | Most common for URLs |
| snake_case | `/order_history` | Common in DB-backed APIs |
| camelCase | `/orderHistory` | Common in JS-heavy APIs |
| PascalCase | `/OrderHistory` | Unusual for URLs |

### URL Structure Template

```
GET /{resource}                          # List collection
GET /{resource}/{id}                     # Get single resource
POST /{resource}                         # Create resource
PUT /{resource}/{id}                     # Replace resource
PATCH /{resource}/{id}                   # Partial update
DELETE /{resource}/{id}                  # Delete resource
GET /{resource}/{id}/{sub-resource}      # List sub-resources
POST /{resource}/{id}/{operation}        # Non-CRUD action
```

### Query Parameters for Filtering, Sorting, and Pagination

```
GET /orders?status=active&sort=-created_at&page=2&per_page=25
```

| Purpose | Convention | Example |
|---------|-----------|---------|
| Filter | `?key=value` | `?status=active` |
| Multiple values | `?key=val1,val2` | `?status=active,pending` |
| Range filter | `?key[min]=&key[max]=` | `?price[min]=10&price[max]=100` |
| Sorting | `?sort=field` or `?sort=-field` (desc) | `?sort=-created_at` |
| Pagination | `?page=&per_page=` or `?offset=&limit=` | `?page=2&per_page=25` |
| Field selection | `?fields=id,name,email` | `?fields=id,name` |
| Searching | `?q=` or `?search=` | `?q=alice` |
| Embedding relations | `?include=user,items` | `?include=items` |

### HATEOAS Links

HATEOAS (Hypermedia As The Engine Of Application State) means responses include links that tell the client what they can do next.

```http
GET /orders/42
Accept: application/json

---

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "id": 42,
    "status": "pending",
    "total": 29.99,
    "items": [...]
  },
  "links": {
    "self": "/orders/42",
    "cancel": "/orders/42/cancel",
    "pay": "/orders/42/pay",
    "invoice": "/invoices/order-42"
  }
}
```

The client navigates the API through these links. No hardcoded URLs in client code. In practice, almost no one achieves Level 3 REST, but including a `links` object is a pragmatic middle ground.

> **Trap:** Verbs in URLs (`/createOrder`, `/deleteUser`)
>
> This is the most obvious sign of an RPC-style API pretending to be REST. Verbs belong in HTTP methods (POST, DELETE), not in URLs. If you have `/createOrder`, you should have `POST /orders`. If you have `/deleteUser`, you should have `DELETE /users/{id}`.
>
> The exception is operations that don't map cleanly to CRUD. For those, a verb-like sub-resource is acceptable: `POST /orders/{id}/cancel`, `POST /users/{id}/activate`. But even these can often be modeled as state transitions on a resource (`PATCH /orders/{id} { "status": "cancelled" }`).
>
> **Follow-up:** "What about /api/v1 — is that RESTful?"
> Yes, versioning is fine in URLs. `/api/v1/users` is RESTful. The `/api` prefix is optional but conventional. The `v1` indicates the version. Some prefer header-based versioning (`Accept: application/vnd.myapp.v1+json`) which is cleaner for evolvability. URL versioning is simpler and more discoverable.

> **Trap:** Deeply nested URLs (`/organizations/5/projects/12/sprints/8/tasks/42`)
>
> This creates problems:
> 1. URL length limits (though rare in practice).
> 2. Clients must know the full hierarchy to access a resource.
> 3. If the hierarchy changes, the URL breaks.
>
> Instead, flatten where possible using the resource's own ID:
> ```
> GET /tasks/42
> ```
> With query parameters for context:
> ```
> GET /tasks/42?include=sprint,project,org
> ```
> Only nest when the sub-resource has no meaning outside the parent (e.g., line items on an order — `/orders/42/line-items/5`).

> **Trap:** Inconsistent casing
>
> Nothing confuses API consumers more than mixing conventions:
> ```
> GET /order-history           # kebab-case
> GET /order_history/123       # snake_case
> GET /orderDetails/456        # camelCase
> ```
>
> Choose ONE convention and apply it everywhere. Kebab-case is the most common for URLs because URLs are case-insensitive in the path (per RFC 3986, but the path is case-sensitive — just stay consistent).

> **Trap:** Exposing database IDs in URLs without protection
>
> Sequential numeric IDs (`/users/1`, `/users/2`, `/users/3`) let anyone enumerate your users. This is an information disclosure vulnerability.
>
> Mitigations:
> 1. Use UUIDs instead of auto-increment IDs: `/users/550e8400-e29b-41d4-a716-446655440000`
> 2. Use opaque tokens: `/users/abc123xyz`
> 3. Implement proper authorization (check that the authenticated user has access to the requested resource).
>
> UUIDs are the most common solution. They're globally unique, non-sequential, and don't leak data size. The tradeoff is marginally larger indexes and slightly worse cache locality.

---

## 6. Content Negotiation

Content negotiation is how the client and server agree on the format of the data being exchanged. HTTP provides a built-in mechanism for this.

### How It Works

**Server-driven negotiation:** The client tells the server what it can accept via request headers. The server picks the best representation and tells the client what it got via response headers.

```
Client: "I can accept JSON or XML, but prefer JSON."
Server: "Here's JSON." (or 406 if neither is available)
```

```
Client: "Here's JSON."
Server: "I understand JSON. Processing."
```

### The Headers

| Header | Direction | Purpose | Example |
|--------|-----------|---------|---------|
| `Accept` | Request | Media types the client can process | `Accept: application/json` |
| `Content-Type` | Request | Media type of the request body | `Content-Type: application/json` |
| `Content-Type` | Response | Media type of the response body | `Content-Type: application/json; charset=utf-8` |

### Media Types (MIME Types)

| Media Type | Usage |
|-----------|-------|
| `application/json` | JSON data. The standard for REST APIs. |
| `application/xml` | XML data. Legacy. |
| `text/csv` | CSV for tabular data exports. |
| `text/plain` | Plain text. Occasionally used. |
| `text/html` | HTML. Sometimes used for error pages or docs. |
| `multipart/form-data` | File uploads, mixed data. |
| `application/octet-stream` | Binary data (file downloads). |
| `application/x-www-form-urlencoded` | Form data (legacy). |

### Vendor-Specific Media Types

For versioning and API-specific content type control:

```
Accept: application/vnd.myapp.v1+json
Content-Type: application/vnd.myapp.user+json
```

Format: `application/vnd.{vendor}.{resource}+{format}`

Benefits:
- Clients explicitly request a version.
- Multiple representations of the same resource (full vs summary).
- Backward-compatible API evolution.

### Quality Values (q-factor)

Clients can express preference using quality values from 0 to 1:

```
Accept: text/html;q=0.9, application/json;q=1.0, */*;q=0.1
```

This says: "I prefer JSON (1.0), then HTML (0.9), and I'll take anything (0.1) if those aren't available."

```http
GET /users/42
Accept: application/json;q=1.0, application/xml;q=0.5

---

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Vary: Accept

{
  "data": { "id": 42, "name": "Alice" }
}
```

### Agent-Driven Negotiation

The server returns 300 Multiple Choices or a page with links to available representations, and the client picks. Rarely used in API contexts.

> **Trap:** Not returning 406 when the requested format is unsupported
>
> ```http
> GET /users/42
> Accept: application/xml
> ```
>
> If your API only supports JSON, you should return:
> ```http
> HTTP/1.1 406 Not Acceptable
> Content-Type: application/json
>
> { "error": "This API only supports application/json" }
> ```
>
> What many APIs do instead: return JSON anyway with 200. This breaks the contract. The client explicitly said "I can only handle XML." They might not be able to parse JSON. Respond with 406.
>
> **Follow-up:** "Should I support content negotiation at all?"
> If you're building a public API, supporting both JSON and XML is a nice-to-have but rarely worth the maintenance cost. JSON is the de facto standard. Support only JSON unless you have a specific need (legacy clients, enterprise XML requirements). If you only support JSON, be strict about it — return 406 for any non-JSON `Accept` header.

> **Trap:** Accepting any content type silently
>
> ```http
> POST /users
> Content-Type: text/plain
>
> name=Alice&email=alice@example.com
> ```
>
> If your API accepts any Content-Type and tries to parse it, you'll get confusing errors or silent failure. Always validate `Content-Type` and return 415 Unsupported Media Type if it's not supported. A JSON API should reject anything that's not `application/json`.

---

## 7. REST vs the Alternatives

A senior engineer should understand when to use REST and when to use something else. There is no one-size-fits-all.

### RPC (Remote Procedure Call)

**Protocols:** XML-RPC, JSON-RPC
**Philosophy:** "Call a function on a remote server."

```http
POST /api/rpc
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "createUser",
  "params": { "name": "Alice", "email": "alice@example.com" },
  "id": 1
}

---

HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "result": { "id": 42, "name": "Alice" },
  "id": 1
}
```

**When to use:** Simple internal services, action-oriented operations, when you don't want to model everything as resources.
**When to avoid:** Public APIs, cacheable operations, anything that benefits from HTTP semantics.

### SOAP

**Protocol:** SOAP (Simple Object Access Protocol)
**Philosophy:** "Enterprise-grade messaging with strict contracts."

```http
POST /UserService
Content-Type: text/xml; charset=utf-8
SOAPAction: "createUser"

<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <createUser xmlns="http://example.com/user">
      <name>Alice</name>
      <email>alice@example.com</email>
    </createUser>
  </soap:Body>
</soap:Envelope>
```

**When to use:** Enterprise environments requiring formal contracts, WS-Security, transactional guarantees. Banking, insurance, government.
**When to avoid:** Anything new. Anything web or mobile. Any situation where developer experience matters.

### GraphQL

**Protocol:** GraphQL (query language)
**Philosophy:** "Let the client decide exactly what data they need."

```http
POST /graphql
Content-Type: application/json

{
  "query": "query { user(id: 42) { name email orders { id total } } }"
}

---

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "orders": [
        { "id": 1, "total": 29.99 }
      ]
    }
  }
}
```

**When to use:** Complex data relationships, multiple client types (web + mobile) needing different data shapes, rapid frontend iteration.
**When to avoid:** Simple CRUD APIs, cache-heavy workloads, operations requiring HTTP caching semantics.

### gRPC

**Protocol:** gRPC (HTTP/2, Protocol Buffers)
**Philosophy:** "High-performance, typed, bi-directional streaming."

```
// Proto definition
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc CreateUser (CreateUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}
```

**When to use:** Microservice-to-microservice communication, real-time streaming, polyglot environments, high-performance systems.
**When to avoid:** Public browser APIs, simple operations, teams without protobuf experience.

### Decision Matrix

| Factor | REST | RPC | GraphQL | gRPC |
|--------|------|-----|---------|------|
| Public API | Best | Good | Good | Poor |
| Internal services | Good | Good | OK | Best |
| Complex queries | Poor | Poor | Best | OK |
| Caching | Best | Good | Poor | Poor |
| Streaming | Poor | Poor | OK | Best |
| Developer experience | Best | Good | Best | OK |
| Tooling ecosystem | Best | Good | Good | Good |
| Schema enforcement | None | Loose | Strong | Strong |
| Learning curve | Low | Low | Medium | Medium |

> **Trap:** Using GraphQL for simple CRUD
>
> GraphQL adds complexity: a schema language, resolvers, a query parser, N+1 prevention (DataLoader), cost analysis, rate limiting at the field level — all for a simple CRUD app where REST would work perfectly. If your API is "GET this, POST that, PUT this," use REST. GraphQL shines when clients have diverse data needs. For a blog with one mobile app, it's over-engineering.
>
> **Follow-up:** "What about the N+1 problem in GraphQL?"
> GraphQL is famous for the N+1 problem: a query like `users { orders { items } }` can trigger 1 query for users, N queries for orders, and M queries for items. DataLoader solves this by batching and caching requests within a single request context. If you adopt GraphQL without DataLoader, your database will suffer. REST has the same problem with `?include=orders.items`, but it's explicit rather than automatic.

> **Trap:** REST for streaming — use gRPC
>
> REST doesn't support server push or streaming natively. You can hack it with Server-Sent Events (SSE) or WebSocket upgrades, but HTTP was designed for request-response, not streaming. If you need real-time updates, bi-directional streaming, or continuous data flow, gRPC (built on HTTP/2 with streaming support) is a better fit.
>
> **Follow-up:** "Can you use WebSockets with REST?"
> You can, but it's a different protocol. REST is about stateless request-response. WebSockets are stateful full-duplex connections. If you need both, separate your architecture: REST for CRUD, WebSockets for real-time events, or use SSE for server-to-client streaming.

> **Trap:** RPC over HTTP GET with caching (the "YouTube approach")
>
> Some services (YouTube, Gmail) use RPC-style endpoints over HTTP GET with query parameters, relying on HTTP caching for performance. YouTube's `/youtubei/v1/` endpoints are POST-based RPC. This works for them at scale because they need aggressive caching and have specific caching infrastructure. For most applications, this pattern sacrifices API clarity for caching gains. Don't copy YouTube unless you're YouTube.

---

## 8. Idempotence & Safety — Deep Dive

This is the most common interview topic in the basic tier. You must understand the difference between safe and idempotent, and explain *why* each matters.

### Definitions

**Safe method:** A method that does not modify server state. The client can make the request without fear of side effects.

```
GET /users/42   → Read user, no side effects (SAFE)
HEAD /users/42  → Same as GET, no body (SAFE)
OPTIONS /users  → Returns allowed methods (SAFE)
```

**Idempotent method:** Making N identical requests produces the same server state as making 1 request. The *response* may differ, but the *server state* is the same.

```
PUT /users/42   → Set name to Alice (IDEMPOTENT — doing it twice has same effect)
DELETE /users/42 → Remove user (IDEMPOTENT — first returns 200, second returns 404)
GET /users/42   → No state change (IDEMPOTENT and SAFE)
```

### Method Classification

| Method | Safe | Idempotent | Notes |
|--------|------|------------|-------|
| GET | Yes | Yes | |
| HEAD | Yes | Yes | |
| OPTIONS | Yes | Yes | |
| TRACE | Yes | Yes | |
| PUT | No | Yes | Full replacement |
| DELETE | No | Yes | Resource gone after first |
| PATCH | No | **No** (by default) | Depends on patch format |
| POST | No | No | Always creates new state |
| CONNECT | No | No | Tunnel state |

### Why Idempotence Matters

**1. Retry Logic**

Network failures happen. If a client sends a PUT request and the connection drops before receiving a response, they don't know if the request was processed. With idempotent methods, they can safely retry:

```python
def create_or_update_user(user_id, data):
    for attempt in range(3):
        try:
            response = http.put(f"/users/{user_id}", json=data)
            return response
        except ConnectionError:
            if attempt < 2:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise
```

With PUT, this works. If the first request succeeded but the response was lost, the second request is a no-op. If the first request failed, the second one processes it.

With POST, retrying would create duplicate resources — which is why POST often requires idempotency keys.

**2. Idempotency Keys for POST**

Since POST is not idempotent, many APIs implement idempotency keys:

```http
POST /payments
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "amount": 1000,
  "currency": "USD"
}
```

The server stores the response keyed by the `Idempotency-Key`. If the same key arrives again, the server returns the stored response without processing the payment again. Stripe's API popularized this pattern.

**3. Duplicate Detection**

Idempotency is crucial for financial transactions. Without it, a network retry could charge a customer twice. Every payment API (Stripe, PayPal, Square) implements idempotency keys for this reason.

> **Trap:** Assuming POST can be made safe
>
> POST is never safe. Its semantics are "process this request, which may change server state." Even if your specific POST endpoint doesn't change state, the HTTP method still says "this might." Use GET for read-only operations. If you need a body for complex queries, use a POST endpoint that's explicitly designed for querying (e.g., `POST /users/search`) and document it carefully, but accept that this is a pragmatic concession, not a RESTful pattern.
>
> **Follow-up:** "What about POST /search with a body for complex queries?"
> Some APIs use POST for search when the query is complex (nested filters, large payloads that exceed URL length limits). This is pragmatic but not RESTful. Alternatives:
> 1. Use GET with query parameters for simple searches.
> 2. Use GET with a persisted search resource: `POST /saved-searches` → `GET /saved-searches/{id}/results`.
> 3. Accept the POST /search pattern and document that it's a deviation from REST.

> **Trap:** Not understanding that idempotence is about server state, not responses
>
> ```http
> DELETE /users/42
> # First call: 200 OK
> # Second call: 404 Not Found
> ```
>
> DELETE is still idempotent. The server state after both requests is identical: user 42 does not exist. The *responses* are different — one returns 200, the other 404 — but idempotence is about server state, not the response. Don't confuse them.
>
> Similarly with PUT:
> ```http
> PUT /users/42
> Content-Type: application/json
>
> { "name": "Alice" }
>
> # First call: 200 OK, ETag: "v1"
> # Second call: 200 OK, ETag: "v1" (or 204 No Content)
> ```
>
> Server state is the same after both calls. That's idempotence.

---

## 9. Request/Response Structure

Consistency is the single most important quality of a well-designed API. Every response should look like every other response. The consumer should be able to write one parser that works for every endpoint.

### A Consistent JSON Envelope

Most APIs use a top-level envelope with standard keys:

```json
{
  "data": { ... },
  "error": null,
  "meta": { ... },
  "links": { ... }
}
```

| Key | Required | Contents |
|-----|----------|----------|
| `data` | Always | The primary resource(s). Object for single, array for collection. |
| `error` | On error | Error object with code, message, details. Null on success. |
| `meta` | Optional | Pagination info, request ID, timestamps. |
| `links` | Recommended | HATEOAS-style links for navigation and related resources. |

### Single Resource Response

```http
GET /users/42

---

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "id": 42,
    "name": "Alice",
    "email": "alice@example.com",
    "created_at": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "request_id": "req_abc123"
  },
  "links": {
    "self": "/users/42",
    "orders": "/users/42/orders"
  }
}
```

### Collection Response

```http
GET /users?page=1&per_page=10

---

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [
    {
      "id": 42,
      "name": "Alice",
      "email": "alice@example.com"
    },
    {
      "id": 43,
      "name": "Bob",
      "email": "bob@example.com"
    }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 10,
    "total": 57,
    "last_page": 6,
    "request_id": "req_def456"
  },
  "links": {
    "self": "/users?page=1&per_page=10",
    "first": "/users?page=1&per_page=10",
    "prev": null,
    "next": "/users?page=2&per_page=10",
    "last": "/users?page=6&per_page=10"
  }
}
```

### Error Response

```http
POST /users
Content-Type: application/json

{
  "name": "",
  "email": "not-an-email"
}

---

HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The given data was invalid.",
    "details": {
      "name": ["The name field is required."],
      "email": ["The email must be a valid email address."]
    }
  },
  "meta": {
    "request_id": "req_ghi789"
  }
}
```

### JSON:API Specification

[JSON:API](https://jsonapi.org/) is a formal specification for building JSON APIs. It standardizes:

- Resource objects with `type` and `id`
- Relationship objects
- Inclusion of related resources via `?include=`
- Sparse fieldsets via `?fields[resource]=`
- Error objects with standard structure
- Pagination via `links`

```json
{
  "data": {
    "type": "users",
    "id": "42",
    "attributes": {
      "name": "Alice",
      "email": "alice@example.com"
    },
    "relationships": {
      "orders": {
        "links": {
          "related": "/users/42/orders"
        }
      }
    }
  }
}
```

### Sideloading / Compound Documents

When a client needs related resources, rather than making N+1 requests, include them in the same response:

```http
GET /orders/42?include=user,items

---

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "id": 1,
    "total": 29.99,
    "user_id": 42,
    "items": [1, 2, 3]
  },
  "included": {
    "users": [
      {
        "id": 42,
        "name": "Alice",
        "email": "alice@example.com"
      }
    ],
    "items": [
      { "id": 1, "name": "Widget", "price": 14.99 },
      { "id": 2, "name": "Gadget", "price": 10.00 },
      { "id": 3, "name": "Doodad", "price": 5.00 }
    ]
  }
}
```

### Sparse Fieldsets

Let the client select which fields to include:

```http
GET /users/42?fields=id,name
```

```json
{
  "data": {
    "id": 42,
    "name": "Alice"
  }
}
```

This is especially important for mobile clients where bandwidth matters.

> **Trap:** Inconsistent response shapes across endpoints
>
> ```json
> // GET /users/42
> { "user": { "id": 42, "name": "Alice" } }
>
> // GET /users
> { "data": [{ "id": 42, "name": "Alice" }], "total": 1 }
>
> // POST /users
> { "created": true, "id": 42 }
> ```
>
> Each endpoint has a different shape. The client needs special-case code for every endpoint. This is the #1 API design sin. **Be consistent:**
>
> - Single resource: always wrap in `data`
> - Collection: always use `data` array with `meta` for pagination
> - Errors: always use `error` object with consistent structure
> - Always include `meta` with `request_id` for debugging
>
> **Follow-up:** "Should I use `data` wrapper even for single resources?"
> Yes. Consistency trumps brevity. If single resources return `{ "user": {...} }` and collections return `{ "data": [...], "meta": {...} }`, the client can't write a generic response handler. Always use `data`. The client reads `response.data.id` regardless of whether it's a single resource or a collection. This also makes adding pagination later easier — a paginated single-resource endpoint is just a collection with one item.

---

## 10. Tier 1 Q&A Drill

These are the questions you should be able to answer after mastering this tier. Practice them out loud.

### Q1: What are the six constraints of REST?

The six constraints defined by Roy Fielding are: (1) Client-Server — separation of concerns, (2) Stateless — no client context on the server between requests, (3) Cacheable — responses define their cacheability, (4) Uniform Interface — resources identified through URIs, manipulated through representations, self-descriptive messages, HATEOAS, (5) Layered System — intermediaries can be inserted without client knowledge, and (6) Code on Demand (optional) — servers can transfer executable code to clients.

### Q2: What's the difference between safe and idempotent? Give examples.

Safe means the method causes no side effects on the server — GET, HEAD, OPTIONS are safe. Idempotent means multiple identical requests produce the same server state as a single request — GET, PUT, DELETE, HEAD, OPTIONS are idempotent. POST is neither. PATCH is not idempotent by default. The key distinction: safe is about no state change at all; idempotent is about repeated calls producing the same state.

### Q3: When would you use 401 vs 403?

401 Unauthorized means the client is not authenticated — they haven't provided credentials or their credentials are invalid. The response must include a `WWW-Authenticate` header. 403 Forbidden means the client is authenticated but doesn't have permission to access the resource. Use 401 when the user isn't logged in; use 403 when the user is logged in but can't perform the action.

### Q4: What is the Richardson Maturity Model?

A four-level model for grading REST APIs: Level 0 (The Swamp of POX) — single URI, single method (POST), XML-RPC style. Level 1 (Resources) — multiple URIs but still only POST. Level 2 (HTTP Verbs) — proper use of GET, POST, PUT, DELETE, PATCH with HTTP status codes. Level 3 (HATEOAS) — responses include links for navigation. Most APIs claiming to be REST are Level 2.

### Q5: Explain idempotency keys and why they matter.

Idempotency keys are tokens (typically UUIDs) sent with POST requests to make them idempotent. The server stores the response keyed by the idempotency key. If the same key is received again, the server returns the stored response instead of processing the request again. This is critical for payment APIs (Stripe, PayPal) where network retries must not charge a customer twice.

### Q6: What's wrong with using POST for all operations?

It breaks HTTP semantics. Intermediaries (proxies, CDNs, browsers) can't distinguish safe reads from dangerous writes. POST is not cacheable — every request goes to the server. You lose conditional GETs (ETags), bookmarkable URLs, and the ability to use standard HTTP tooling. It's Level 0 on the Richardson Maturity Model.

### Q7: How do you handle partial updates — PUT vs PATCH?

PUT replaces the entire resource. If you send only the fields you want to change, a PUT would set unspecified fields to their defaults (or null). PATCH applies a partial update — only the fields you specify are modified. PUT is idempotent. PATCH is not necessarily idempotent (depends on the patch format). Use PUT when the client is sending a complete representation. Use PATCH when the client is sending only changes.

### Q8: What headers would you use for caching in a REST API?

`Cache-Control` (max-age, public, private, no-cache, no-store), `ETag` (entity tag for conditional requests), `Last-Modified` (timestamp), `Expires` (deprecated but still used), `Vary` (which request headers affect the response). Conditional requests use `If-None-Match` (ETag comparison) and `If-Modified-Since` (timestamp comparison) to return 304 Not Modified when the resource hasn't changed.

### Q9: What is HATEOAS and why does it matter?

HATEOAS (Hypermedia As The Engine Of Application State) means API responses include links that tell clients what actions are available next. Instead of hardcoding URLs, clients discover them from responses. This decouples client and server — the server can change URL structures as long as the link relations stay the same. It's Level 3 on the Richardson Maturity Model and is almost never implemented in practice.

### Q10: How would you design an API for a banking transfer operation?

This operation doesn't map cleanly to CRUD. Options: (1) Resource approach — `POST /transfers` with `from_account`, `to_account`, `amount`. This creates a transfer resource. (2) Action approach — `POST /accounts/{id}/transfer` with target account and amount. Both are better than `POST /transferFunds`. Include an `Idempotency-Key` header. Use 202 Accepted if the transfer is async. Return 409 Conflict if insufficient funds. Return a `Location` header pointing to the transfer resource for status tracking.
