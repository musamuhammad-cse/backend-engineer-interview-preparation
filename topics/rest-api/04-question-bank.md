# REST API — Question Bank & Drill Material

A comprehensive collection of REST API questions, puzzles, and design prompts tailored for a Senior Backend Engineer (8+ years) with expertise in PHP/Laravel, Go, and JavaScript/TypeScript. Use this to drill fundamentals, practice system design, and prepare behavioural stories for the interview loop.

> **How to use this file:** Each section is self-contained. Rapid-Fire sections use `<details>` blocks so you can quiz yourself. Code puzzles include reasoning walkthroughs. System design prompts follow a structured template. Star stories are frameworks — fill in your real experience.

---

## Table of Contents

1. [Rapid-Fire: REST Fundamentals & HTTP](#1-rapid-fire-rest-fundamentals--http-40-questions)
2. [Rapid-Fire: API Design Patterns](#2-rapid-fire-api-design-patterns-40-questions)
3. [Rapid-Fire: Advanced REST & Operations](#3-rapid-fire-advanced-rest--operations-30-questions)
4. [Code Puzzles](#4-code-puzzles)
5. [System Design Prompts](#5-system-design-prompts)
6. [Debugging Scenarios](#6-debugging-scenarios)
7. [STAR Stories](#7-star-stories)
8. [Questions to Ask the Interviewer](#8-questions-to-ask-the-interviewer)
9. [Red Flags to Avoid](#9-red-flags-to-avoid)

---

## 1. Rapid-Fire: REST Fundamentals & HTTP (40 questions)

**Q1:** What are the six constraints of the REST architectural style?

<details><summary>Answer</summary>

1. **Client-Server** — separation of concerns
2. **Stateless** — each request from client contains all necessary information
3. **Cacheable** — responses must implicitly or explicitly define themselves as cacheable
4. **Uniform Interface** — identification of resources; manipulation through representations; self-descriptive messages; HATEOAS
5. **Layered System** — intermediary servers (proxies, gateways) can be inserted transparently
6. **Code on Demand (optional)** — servers can temporarily extend client functionality via scripts

</details>

**Q2:** Which HTTP methods are **idempotent**? Which are **safe**?

<details><summary>Answer</summary>

- **Safe:** GET, HEAD, OPTIONS, TRACE — no side effects
- **Idempotent:** GET, HEAD, PUT, DELETE, OPTIONS, TRACE — repeating the request yields the same server state (PUT replaces the resource each time; DELETE returns 404 on subsequent calls but remains idempotent in effect)
- **Not idempotent:** POST, PATCH (unless using a conditional/atomic patch)

</details>

**Q3:** What is the difference between `PUT` and `PATCH`?

<details><summary>Answer</summary>

PUT replaces the **entire** resource. PATCH applies a **partial** modification.

```http
PUT /users/42
Content-Type: application/json

{"name": "Alice", "email": "alice@example.com"}
// Replaces every field — missing fields may be set to null

PATCH /users/42
Content-Type: application/json

{"name": "Alice"}
// Only updates the name field; other fields unchanged
```

</details>

**Q4:** What does `PUT /items` (no ID) mean semantically? Is it valid REST?

<details><summary>Answer</summary>

PUT on a collection is unusual. It implies **replace the entire collection** — idempotent but dangerous. Most APIs use POST for collection creation. A PUT on a collection endpoint could be valid for bulk replacement but is rarely seen. Prefer POST for creation; PUT for a specific resource identified by the client (e.g. `PUT /items/client-generated-id`).

</details>

**Q5:** When should you return `201 Created` vs `200 OK` vs `204 No Content`?

<details><summary>Answer</summary>

- **201 Created** — resource was created as a result (POST, sometimes PUT)
- **200 OK** — success with a response body (GET, PUT, PATCH)
- **204 No Content** — success with **no** response body (DELETE, sometimes PUT/PATCH if no body returned)
- **202 Accepted** — request accepted but not yet processed (async operations)

</details>

**Q6:** What is `303 See Other` used for?

<details><summary>Answer</summary>

Used after a POST to redirect the client to a different resource (often a status page or the created resource). Unlike `302`, the client MUST use GET on the redirect target regardless of the original method.

```http
HTTP/1.1 303 See Other
Location: /orders/42/status
```

</details>

**Q7:** Explain content negotiation. How does a server determine the response format?

<details><summary>Answer</summary>

The client declares preferences via request headers; the server chooses the best match:

- `Accept` — media type (`application/json`, `application/xml`)
- `Accept-Language` — language (`en-US`, `fr-CA`)
- `Accept-Encoding` — compression (`gzip`, `br`)
- `Accept-Charset` — character set (`utf-8`)

The server responds with `Content-Type` and optionally `Content-Encoding`. If no acceptable representation exists, return `406 Not Acceptable`.

</details>

**Q8:** What does `Vary: Accept-Encoding` do?

<details><summary>Answer</summary>

Tells caches that the response varies based on the `Accept-Encoding` request header. Two clients with different Accept-Encoding values get different cached representations. Prevents serving a gzipped response to a client that doesn't support it (or vice versa).

</details>

**Q9:** What is the Richardson Maturity Model? Name the levels.

<details><summary>Answer</summary>

A maturity model for REST APIs from Leonard Richardson:

- **Level 0 (The Swamp of POX):** Single URI, single HTTP method (usually POST), all actions encoded in the body — essentially RPC over HTTP.
- **Level 1 (Resources):** Multiple URIs for different resources but still one HTTP method.
- **Level 2 (HTTP Verbs):** Proper use of GET, POST, PUT, DELETE, status codes.
- **Level 3 (Hypermedia Controls / HATEOAS):** Responses include links that guide the client to discover available actions.

Most real-world APIs are Level 2. Level 3 is the ideal but rarely fully implemented.

</details>

**Q10:** What status code does an _unauthenticated_ request get vs an _unauthorized_ request?

<details><summary>Answer</summary>

- **401 Unauthorized** — missing or invalid authentication (the client must identify itself)
- **403 Forbidden** — the client is authenticated but lacks permission for the resource

The naming is confusing — 401 is really about **authentication**, 403 about **authorization**.

</details>

**Q11:** When would you use `429 Too Many Requests`?

<details><summary>Answer</summary>

Rate limiting — the client has exceeded the allowed number of requests. Should include a `Retry-After` header (either seconds as integer or HTTP-date).

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 120
Content-Type: application/json

{"error": "rate_limit_exceeded", "retry_after_seconds": 120}
```

</details>

**Q12:** What is the `Prefer` header used for?

<details><summary>Answer</summary>

Part of RFC 7240 — allows the client to request processing preferences:

- `Prefer: return=minimal` — omit response body
- `Prefer: return=representation` — include full representation
- `Prefer: respond-async` — server should process asynchronously, return 202
- `Prefer: wait=<seconds>` — maximum time the client is willing to wait

Server indicates applied preferences with `Preference-Applied` header.

</details>

**Q13:** Explain `POST /books` vs `PUT /books/978-3-16-148410-0`. Which is idempotent?

<details><summary>Answer</summary>

- POST to a collection creates a new resource; the server assigns the ID. Not idempotent — multiple POSTs create multiple resources.
- PUT with a client-specified ID. Idempotent — the same PUT executed twice results in the same resource state. If the resource doesn't exist, the server may create it (PUT-as-create).

Use POST when the server owns ID generation. Use PUT when the client controls the identifier.

</details>

**Q14:** Can `GET` have a request body?

<details><summary>Answer</summary>

Technically yes — HTTP doesn't forbid it. But practically no: many proxies, browsers, and servers strip or reject GET bodies. It violates the convention. Use query parameters for GET requests. If you need a body, use POST or a query parameter that encodes the payload.

</details>

**Q15:** What is the difference between `302 Found`, `307 Temporary Redirect`, and `303 See Other`?

<details><summary>Answer</summary>

- **302 Found:** The original HTTP/1.0 redirect. Most clients change POST to GET despite the spec saying otherwise.
- **303 See Other:** Always redirect with GET, regardless of original method. Used after form submissions (PRG pattern).
- **307 Temporary Redirect:** Maintains the original HTTP method. POST stays POST. Guarantees method preservation.

Use 303 for POST-redirect-GET. Use 307 when the original method must be preserved.

</details>

**Q16:** What does `Cache-Control: no-transform` mean?

<details><summary>Answer</summary>

Intermediaries (proxies, CDNs) must not modify the response body. Important for APIs serving encrypted or authenticated content where compression or image re-encoding would break the contract.

</details>

**Q17:** How does `ETag` support optimistic concurrency?

<details><summary>Answer</summary>

The server returns an `ETag` (hash/version of the resource). The client sends it back via `If-Match`:

```http
PUT /users/42
If-Match: "abc123"

{"name": "Alice"}
```

Server checks the current ETag matches `"abc123"`. If another client modified it in the meantime, ETag differs → `412 Precondition Failed`. The client must re-fetch and retry.

</details>

**Q18:** What is the difference between `Expires` and `Cache-Control: max-age`?

<details><summary>Answer</summary>

- `Expires` is HTTP/1.0 — an absolute date (`Expires: Wed, 21 Oct 2026 07:28:00 GMT`)
- `Cache-Control: max-age=3600` is HTTP/1.1 — relative seconds from the time of response
- When both are present, `max-age` takes precedence
- `max-age` is preferred — avoids clock sync issues

</details>

**Q19:** What is `HEAD` used for in REST APIs?

<details><summary>Answer</summary>

Identical to GET but returns no body. Used to:
- Check if a resource exists (200 vs 404)
- Read metadata via headers (Content-Length, ETag, Last-Modified)
- Test cache freshness
- Check CORS headers without downloading the payload

</details>

**Q20:** What does `OPTIONS *` do?

<details><summary>Answer</summary>

`OPTIONS *` (wildcard URI) queries the server's capabilities rather than a specific resource. The server responds with `Allow` and optionally `Public` headers listing supported methods and features. Useful for health checks and capability discovery.

</details>

**Q21:** Why is `POST` neither safe nor idempotent?

<details><summary>Answer</summary>

POST is designed for non-idempotent creation or processing:
- **Not safe** because it mutates server state
- **Not idempotent** because multiple POSTs create multiple resources or trigger multiple side effects
- However, a server can choose to make POST idempotent via idempotency keys (e.g., `Idempotency-Key` header)

</details>

**Q22:** What is the difference between `application/json` and `application/vnd.api+json`?

<details><summary>Answer</summary>

`application/vnd.api+json` is the media type for **JSON:API** — a specification that standardises resource documents, pagination, errors, includes, and sparse fieldsets. `application/json` is generic. Using a vendor media type enables content negotiation for different API versions or serialisation formats.

</details>

**Q23:** What is `415 Unsupported Media Type` vs `406 Not Acceptable`?

<details><summary>Answer</summary>

- **415** — the request body format isn't supported (e.g., server only accepts JSON but client sends XML). Client error in `Content-Type`.
- **406** — the server cannot produce a response in the format the client requested via `Accept`. Server error (configuration gap) or client error for unsupported preferences.

</details>

**Q24:** Can you overload POST to perform reads? Is this a REST anti-pattern?

<details><summary>Answer</summary>

Yes, it's common when query parameters are too complex or long for a GET URL (search endpoints, GraphQL-style queries). It's technically an anti-pattern because POST is semantically for creation/submission, but pragmatically acceptable for complex queries. The key is consistency — document clearly that this particular POST is functionally read-only.

```http
POST /search
Content-Type: application/json

{"filters": {"status": "active"}, "sort": "created_at", "limit": 50}
```

</details>

**Q25:** What is the `Link` header used for in REST?

<details><summary>Answer</summary>

Carries hypermedia links in the response header rather than the body. Used for pagination (`rel="next"`, `rel="prev"`), discovery (`rel="self"`), and HATEOAS. RFC 5988 / RFC 8288.

```http
Link: </users?page=2>; rel="next", </users?page=1>; rel="prev"
```

</details>

**Q26:** How does `Transfer-Encoding: chunked` work?

<details><summary>Answer</summary>

The server streams the response without knowing the total size upfront. Each chunk is prefixed with its byte size in hex, ending with a zero-length chunk. Used for streaming large responses or server-sent events. Prevents the server from buffering the entire response.

</details>

**Q27:** What is `Server-Sent Events` and how does it differ from WebSocket?

<details><summary>Answer</summary>

SSE is a unidirectional (server-to-client) text-based protocol over HTTP. The server sets `Content-Type: text/event-stream` and keeps the connection open. Uses standard HTTP — works through proxies, simpler than WebSocket. Use SSE for push notifications, live feeds. WebSocket is bidirectional, binary, and requires upgrading the connection.

```http
GET /events
Accept: text/event-stream

data: {"event": "order_placed", "order_id": 42}

```

</details>

**Q28:** What status code indicates a resource was deleted and is no longer available?

<details><summary>Answer</summary>

- `204 No Content` — typical response for DELETE
- `410 Gone` — the resource existed but was explicitly deleted and is gone permanently (vs 404 which is ambiguous). Helpful for clients to distinguish "never existed" from "was deleted".
- `404 Not Found` — also acceptable for deleted resources if you don't want to leak information

</details>

**Q29:** Should `DELETE` return a body?

<details><summary>Answer</summary>

Preferably `204 No Content` with no body. Some APIs return the deleted resource (200) or a confirmation message. Returning a body is not wrong but 204 is the most semantically correct and avoids wasting bandwidth.

</details>

**Q30:** What does `502 Bad Gateway` vs `504 Gateway Timeout` mean?

<details><summary>Answer</summary>

- **502** — an upstream server returned an invalid response (e.g., the upstream is down, returns garbage, or protocol mismatch)
- **504** — an upstream server didn't respond within the gateway's timeout window

Both are common with API gateways, load balancers, and reverse proxies.

</details>

**Q31:** What is `413 Content Too Large` vs `414 URI Too Long`?

<details><summary>Answer</summary>

- **413** — the request body exceeds the server's maximum allowed size
- **414** — the request URI (including query string) exceeds the server's maximum allowed length

</details>

**Q32:** Name three uses of the `Location` header.

<details><summary>Answer</summary>

1. **201 Created** — point to the newly created resource
2. **3xx redirect** — specify the redirect target
3. **202 Accepted** — point to a status monitoring endpoint for async operations

</details>

**Q33:** What is `501 Not Implemented` vs `503 Service Unavailable`?

<details><summary>Answer</summary>

- **501** — the server does not support the functionality needed to fulfil the request (e.g., a method not recognised)
- **503** — the server is temporarily unable (overloaded, down for maintenance). Include `Retry-After` if possible.

</details>

**Q34:** Can `POST` be used to update a resource?

<details><summary>Answer</summary>

Yes, but it's unconventional. Some APIs use POST for partial updates when PATCH isn't well-supported by clients. Others use POST for "actions" (e.g., `POST /orders/42/cancel`). The RESTful approach is: PUT for full replacement, PATCH for partial update, POST for creation or action.

</details>

**Q35:** Explain the `Range` header in HTTP.

<details><summary>Answer</summary>

A client can request a byte range of a resource:

```http
GET /large-file.zip
Range: bytes=0-1023
```

Server responds with `206 Partial Content` and the requested range. Useful for resumable downloads, streaming, and large payloads. The server advertises support via `Accept-Ranges: bytes`.

</details>

**Q36:** What is `If-None-Match` used for?

<details><summary>Answer</summary>

Conditional GET — the client sends ETags it has cached. If none match (resource unchanged), the server returns `304 Not Modified` with no body. If the resource changed, the server returns 200 with the new body and ETag. Saves bandwidth by avoiding re-downloading unchanged resources.

```http
GET /users/42
If-None-Match: "abc123"
// → 304 Not Modified (use cache)
// → 200 OK with new body & ETag "def456"
```

</details>

**Q37:** What is the `Allow` header?

<details><summary>Answer</summary>

Lists the HTTP methods supported by a resource. Returned in response to OPTIONS or in a `405 Method Not Allowed` response.

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, PUT, DELETE
```

</details>

**Q38:** Compare RPC-style APIs with REST.

<details><summary>Answer</summary>

| Aspect | RPC | REST |
|--------|-----|------|
| Endpoint structure | `POST /rpc/createUser` or `/api?action=createUser` | `POST /users` |
| State | Operations (verbs in URLs) | Resources (nouns in URLs) |
| Cacheability | Poor — same URL different bodies | Good — URLs identify resources |
| Tooling | Generic HTTP client often enough | Standard HTTP semantics leverage caches, proxies |
| Contract | Usually requires shared library/stub | Self-descriptive via media types and links |

Real systems often blend both. The key is consistency within a single API surface.

</details>

**Q39:** What's the main difference between SOAP and REST?

<details><summary>Answer</summary>

SOAP is a **protocol** (strict contract, XML envelope, WS-* standards, stateful possible). REST is an **architectural style** (stateless, uniform interface, cacheable, leverages HTTP fully). SOAP is heavier, enterprise-oriented, with built-in security/transaction support. REST is lighter, scales better, and is the default for web/mobile APIs.

</details>

**Q40:** How does GraphQL differ from REST in data fetching?

<details><summary>Answer</summary>

GraphQL: client specifies exactly the fields it needs in a single query, avoids over-fetching and under-fetching, one endpoint (`POST /graphql`). REST: multiple endpoints return fixed structures; over-fetching is common. GraphQL shifts complexity to the server (resolver performance, N+1 problem, query cost analysis). REST is simpler to cache at the HTTP level.

---

## 2. Rapid-Fire: API Design Patterns (40 questions)

**Q41:** What are the major API versioning strategies? Pros and cons?

<details><summary>Answer</summary>

1. **URI path**: `GET /v1/users` — simple, explicit, but clutters URLs
2. **Query parameter**: `GET /users?version=1` — easy to implement, but can be ignored/missed by caches
3. **Custom header**: `GET /users` with `Accept: application/vnd.company.v1+json` — clean URL, proper content negotiation, but harder to test via browser
4. **Content-Type versioning**: `GET /users` with `Content-Type: application/vnd.company.v1+json` — similar to header approach

**Recommendation:** URI path for public APIs (explicit, cache-friendly). Header versioning for internal APIs.

</details>

**Q42:** Offset pagination vs cursor pagination — when to use which?

<details><summary>Answer</summary>

| Aspect | Offset (`?page=2&limit=20`) | Cursor (`?cursor=eyJpZCI6NDJ9`) |
|--------|------|--------|
| Consistency | Skips/duplicates if data changes between pages | Stable — cursor points to a specific item |
| DB performance | `OFFSET` scans discarded rows | Efficient index-based lookup (`WHERE id > ?`) |
| Random access | Yes — jump to any page | No — only next/prev |
| UX | Good for numbered pagination | Good for infinite scroll / feeds |

Use cursor for real-time data (feeds, logs, search results). Use offset for stable, small datasets where random page access matters.

</details>

**Q43:** How do you handle filtering, sorting, and searching in a REST API?

<details><summary>Answer</summary>

```http
# Filtering
GET /products?category=electronics&price[gte]=10&price[lte]=100

# Sorting
GET /products?sort=-created_at,name

# Searching
GET /products?q=wireless+headphones

# Sparse fields
GET /products?fields=id,name,price

# Full-text search
POST /products/search
Content-Type: application/json

{"query": "wireless headphones", "filters": {"category": "electronics"}}
```

Key principles: use query parameters, consistent syntax for operators (`eq`, `gte`, `lte`, `in`), document supported operators, validate filter fields to prevent SQL injection.

</details>

**Q44:** Design a standard error response envelope.

<details><summary>Answer</summary>

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body contains invalid fields.",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address",
        "value": "not-an-email"
      }
    ],
    "request_id": "req_abc123",
    "docs_url": "https://docs.api.com/errors#VALIDATION_ERROR"
  }
}
```

Always return `request_id` (correlation ID), machine-readable `code`, human-readable `message`, and structured `details`. Follow RFC 7807 (Problem Details) for standardisation.

</details>

**Q45:** What's the difference between JWT and opaque tokens for API auth?

<details><summary>Answer</summary>

- **JWT (Bearer):** Self-contained — contains claims, no database lookup per request. Can verify offline. Downside: hard to revoke (you must wait for expiry or maintain a blocklist). Can leak info if not encrypted.
- **Opaque token:** Random string stored server-side. Requires a DB/cache lookup per request. Easy to revoke (delete the token). No information leakage.

Use JWTs for distributed systems where validation must happen without central auth. Use opaque tokens for simple, revocable auth.

</details>

**Q46:** How does OAuth2 Authorization Code flow work?

<details><summary>Answer</summary>

1. Client redirects user to authorization server
2. User authenticates and authorizes the client
3. Auth server redirects back with an authorization `code`
4. Client exchanges `code` (+ `client_secret`) for an `access_token` and optionally a `refresh_token`
5. Client uses `access_token` in API requests (`Authorization: Bearer <token>`)
6. When access token expires, client uses `refresh_token` to get a new one

Always use PKCE (Proof Key for Code Exchange) for public clients (SPAs, mobile apps).

</details>

**Q47:** How do you implement rate limiting? What headers should you return?

<details><summary>Answer</summary>

Common algorithms: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window Log, Sliding Window Counter.

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 87
RateLimit-Reset: 1680000000
```

Standard headers per `RateLimit` draft (RFC草案) or use `X-RateLimit-*` for wider adoption.

For distributed rate limiting, use Redis (sorted sets for sliding window, or Lua scripts for atomicity). At high scale (100K+ req/s), consider local counters with periodic sync.

</details>

**Q48:** What is the `X-Request-Id` header used for?

<details><summary>Answer</summary>

Correlation ID — a unique identifier for every request. Generated by the client or the first server in the chain (gateway). Passed through all microservices. Critical for debugging, tracing, and logging. Should be included in error responses and logs.

Clients can generate UUIDs and send `X-Request-Id: uuid`. Servers should propagate and return it.

</details>

**Q49:** How do you implement conditional requests with ETags?

<details><summary>Answer</summary>

Server generates an ETag (usually MD5 hash of the response body or a version number):

```http
GET /users/42
→ 200 OK
ETag: "d5f4e1a2b3c4"
```

Client caches and later sends:

```http
GET /users/42
If-None-Match: "d5f4e1a2b3c4"
→ 304 Not Modified
```

For updates:

```http
PUT /users/42
If-Match: "d5f4e1a2b3c4"
→ 412 Precondition Failed (if concurrent edit occurred)
```

</details>

**Q50:** What CORS headers are required for a simple cross-origin request?

<details><summary>Answer</summary>

The server must include:

```http
Access-Control-Allow-Origin: https://trusted-domain.com
```

For requests with custom headers, non-simple content types, or credentials, a **preflight** `OPTIONS` request is sent, and the server must respond with:

```http
Access-Control-Allow-Origin: https://trusted-domain.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

If credentials (cookies, auth headers) are included:

```http
Access-Control-Allow-Credentials: true
```

Note: with credentials, `Access-Control-Allow-Origin` cannot be `*`.

</details>

**Q51:** What is `OPTIONS` preflight caching via `Access-Control-Max-Age`?

<details><summary>Answer</summary>

The browser caches the preflight response for the duration specified in `Access-Control-Max-Age` (seconds). During that window, it skips the OPTIONS request and sends the actual request directly. Reduces latency for repeated cross-origin calls. Default is 5 seconds if not specified. Setting it too high (e.g., 86400) can cause issues if CORS config changes.

</details>

**Q52:** What is OpenAPI and why is it important?

<details><summary>Answer</summary>

OpenAPI is a specification for describing REST APIs in a machine-readable format (YAML/JSON). It defines endpoints, request/response schemas, authentication, parameters, and error codes.

**Importance:**
- Automated documentation generation (Swagger UI, Redoc)
- Client SDK generation (OpenAPI Generator)
- Contract testing and validation
- API governance and linting (Spectral)
- Mock server generation

</details>

**Q53:** How do you handle bulk operations in a REST API?

<details><summary>Answer</summary>

Three approaches:

1. **JSON batch array:**
```http
POST /batch
Content-Type: application/json

[
  {"method": "PUT", "path": "/users/1", "body": {"name": "Alice"}},
  {"method": "DELETE", "path": "/users/2"},
  {"method": "POST", "path": "/users", "body": {"name": "Bob"}}
]
```
Response maps each operation to its status.

2. **Bulk resource endpoint:**
```http
PATCH /users
Content-Type: application/json

{"updates": [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]}
```

3. **Async job:** POST a bulk file, get a job ID, poll for completion.

Trade-off: batch operations break REST semantics (single action per request). Use them sparingly and document idempotency guarantees.

</details>

**Q54:** What is `Accept-Post` and `Accept-Patch`?

<details><summary>Answer</summary>

Response headers from `OPTIONS` that advertise which media types are accepted for POST and PATCH requests to a resource. Allows clients to discover supported request formats at runtime.

```http
OPTIONS /users
→ Allow: GET, POST, PATCH, DELETE
Accept-Post: application/json, application/vnd.api+json
Accept-Patch: application/json-patch+json
```

</details>

**Q55:** How do you handle partial success in batch operations?

<details><summary>Answer</summary>

Return `207 Multi-Status` (WebDAV extension, RFC 4918). Each operation in the batch gets its own status code and response.

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json

[
  {"path": "/users/1", "status": 200, "body": {"id": 1, "name": "Alice"}},
  {"path": "/users/2", "status": 404, "body": {"error": "User not found"}},
  {"path": "/users", "status": 201, "body": {"id": 100, "name": "Bob"}}
]
```

</details>

**Q56:** What is the difference between rate limiting and throttling?

<details><summary>Answer</summary>

- **Rate limiting:** Restricts the total number of requests in a time window. Usually returns `429 Too Many Requests` when exceeded.
- **Throttling:** Controls the rate at which requests are processed (may queue or slow down requests rather than rejecting them). Can delay rather than drop.

Throttling is often implemented server-side to smooth traffic. Rate limiting is enforced at the edge (API gateway). A system typically does both: rate limit at the perimeter, throttle internally.

</details>

**Q57:** How would you design a time-based cache invalidation header?

<details><summary>Answer</summary>

```http
HTTP/1.1 200 OK
Cache-Control: private, max-age=300
Expires: Wed, 21 Oct 2026 07:28:00 GMT
Last-Modified: Wed, 21 Oct 2026 07:23:00 GMT
```

- `private` — response is specific to the user (don't cache in shared caches)
- `max-age=300` — cache for 5 minutes
- `Last-Modified` — enables conditional `If-Modified-Since` requests
- Optionally `s-maxage=60` for CDN cache duration (shorter than client cache)

For API responses that should never be cached: `Cache-Control: no-store, no-cache, must-revalidate`.

</details>

**Q58:** How do you implement fuzzy search on a collection endpoint?

<details><summary>Answer</summary>

Dedicated search endpoint or query parameter:

```http
GET /products?q=wirelss headphnes&fuzzy=true
```

Server-side options:
- **PostgreSQL:** `trgm` extension with `similarity()` or `ILIKE`
- **Elasticsearch/MeiliSearch:** `fuzziness` parameter with Levenshtein distance
- **Go:** `bleve` library
- **PHP:** `Laravel Scout` with MeiliSearch or Algolia

Set a minimum similarity threshold. Return results sorted by relevance. Consider using a dedicated search service for production.

</details>

**Q59:** What are HTTP Strict Transport Security (HSTS) headers?

<details><summary>Answer</summary>

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

Tells browsers to always connect via HTTPS for the specified domain. Prevents SSL stripping attacks. `preload` allows submission to browser hardcoded HSTS lists. `includeSubDomains` covers all subdomains.

**Trap:** Once set, HSTS cannot be undone until `max-age` expires. Be absolutely sure your HTTPS config is solid.

</details>

**Q60:** How would you handle API deprecation?

<details><summary>Answer</summary>

1. Announce deprecation via changelog and direct communication (email, dashboard)
2. Add deprecation headers to responses:
```http
Sunset: Sat, 01 Jan 2028 00:00:00 GMT
Deprecation: true
Link: </v2/users>; rel="successor-version"
```
3. Maintain backward compatibility for the announced period (6-12 months)
4. Monitor usage of deprecated endpoints
5. After sunset date, return `410 Gone` with migration instructions

Never remove an endpoint without a sunset header period.

</details>

**Q61:** What is the proper way to return errors for validation?

<details><summary>Answer</summary>

Return `422 Unprocessable Entity` (preferred over 400 for semantic correctness, though 400 is also common). Include structured details:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {"field": "email", "code": "REQUIRED", "message": "Email is required"},
      {"field": "age", "code": "MINIMUM", "message": "Must be at least 18", "meta": {"min": 18}}
    ]
  }
}
```

Use consistent field paths (dot notation for nested: `address.zip_code`), machine-readable codes, and allow clients to map errors to form fields.

</details>

**Q62:** Should an API return `400` or `422` for invalid request body?

<details><summary>Answer</summary>

- **400 Bad Request** — generic client error (malformed JSON, missing required header, invalid URI)
- **422 Unprocessable Entity** — syntactically correct JSON but semantically invalid (validation failure)
- **400** is simpler for clients to handle (one error type)
- **422** is more precise but requires client awareness

Either is fine — consistency matters more. If you use 422, treat all validation errors the same way.

</details>

**Q63:** How do you handle concurrent updates without data loss?

<details><summary>Answer</summary>

Three strategies:

1. **Optimistic locking (ETag/If-Match):** Client sends ETag from read, server rejects if stale (412).
2. **Pessimistic locking:** `POST /documents/42/lock` — acquire a lock before editing. Risk of abandoned locks.
3. **Last-write-wins (LWW):** Accept the latest timestamp/version. Simple but lossy. Only safe for idempotent/mergeable writes.

**Recommendation:** Start with optimistic locking. It scales well and prevents silent overwrites.

</details>

**Q64:** What is the `Prefer: respond-async` pattern?

<details><summary>Answer</summary>

Client signals it's willing to accept asynchronous processing:

```http
POST /reports
Prefer: respond-async

→ 202 Accepted
Location: /operations/abc-123
```

The server processes the request offline. The client polls `Location` to check status. When complete, the status endpoint returns `303 See Other` pointing to the result.

Common for report generation, bulk imports, video transcoding, etc.

</details>

**Q65:** What is a webhook? How is it different from polling?

<details><summary>Answer</summary>

- **Webhook:** The server sends an HTTP request to a client-registered URL when an event occurs. Real-time, efficient, but requires the client to expose an endpoint.
- **Polling:** The client repeatedly calls the server to check for new data. Simpler to implement client-side but wastes resources and introduces latency.

Webhooks are push-based. Polling is pull-based. Use webhooks for event notifications; use polling for data that changes infrequently or when the client can't expose a webhook endpoint.

</details>

**Q66:** How do you secure webhook delivery?

<details><summary>Answer</summary>

1. **Signing:** Sign the payload with HMAC-SHA256 and include the signature in a header (`X-Signature-256`). The client verifies using a shared secret.
2. **HTTPS only:** Always deliver over TLS.
3. **IP allowlisting:** Client can allowlist the webhook provider's IPs.
4. **Idempotency key:** Include `X-Idempotency-Key` or `X-Event-Id` so clients can deduplicate.
5. **Retry with backoff:** Exponential backoff with a max retry window (e.g., 3 days).
6. **Webhook secret rotation:** Allow clients to rotate secrets without downtime.

</details>

**Q67:** How do you implement cursor-based pagination in SQL?

<details><summary>Answer</summary>

```sql
-- Cursor: last item's ID from previous page
SELECT id, name, created_at
FROM users
WHERE id > :cursor
ORDER BY id ASC
LIMIT :limit
```

For composite cursors (sort by `created_at`):

```sql
WHERE (created_at > :cursor_created_at)
   OR (created_at = :cursor_created_at AND id > :cursor_id)
ORDER BY created_at ASC, id ASC
LIMIT :limit
```

Encode the cursor as Base64 JSON:

```json
{"id": 42, "created_at": "2026-07-26T00:00:00Z"}
```

```http
Link: </users?cursor=eyJpZCI6NDJ9&limit=20>; rel="next"
```

</details>

**Q68:** What is `Sparse Fieldsets` and how do you implement them?

<details><summary>Answer</summary>

Allows the client to request only specific fields:

```http
GET /users/42?fields=id,name,email
```

Implementation: parse the `fields` parameter, whitelist against allowed fields, project only those fields in the database query (SELECT) and serialisation. Always include the `id` field implicitly.

Benefits: reduces response size, speeds up serialisation, improves client performance.

</details>

**Q69:** How do you design a webhook retry mechanism?

<details><summary>Answer</summary>

```text
Attempt 1: immediate
Attempt 2: +10 seconds
Attempt 3: +1 minute
Attempt 4: +10 minutes
Attempt 5: +1 hour
Attempt 6: +6 hours
Attempt 7: +24 hours
```

Use exponential backoff with jitter:

```
delay = min(cap, base * 2^attempt) * random(0.5, 1.5)
```

Track delivery state per webhook: `pending → delivered | failed`. After max retries (e.g., 72 hours), mark as `dead` and alert the customer. Provide a dashboard for manual retry.

</details>

**Q70:** What is the difference between `Content-Disposition: inline` and `attachment`?

<details><summary>Answer</summary>

- **inline** — browser should display the file within the page (PDF viewer, image)
- **attachment** — browser should download the file as a file download prompt

```http
Content-Disposition: attachment; filename="report.pdf"
```

For APIs that serve file downloads, use `attachment` with a meaningful `filename` parameter.

</details>

**Q71:** How do you handle file uploads in a REST API?

<details><summary>Answer</summary>

```http
POST /uploads
Content-Type: multipart/form-data; boundary=----Boundary

------Boundary
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<binary data>
------Boundary
Content-Disposition: form-data; name="description"

Profile photo
------Boundary--
```

Or use a two-step approach:
1. Client requests a presigned upload URL
2. Client uploads directly to S3/GCS
3. Client calls API with the file reference

Prefer presigned URLs for large files (reduces server load, better throughput).

</details>

**Q72:** What are the trade-offs of using `POST` vs `PUT` for resource creation?

<details><summary>Answer</summary>

| Aspect | POST | PUT |
|--------|------|-----|
| ID generation | Server-generated | Client-provided |
| Idempotency | Not idempotent | Idempotent |
| Endpoint | `POST /users` | `PUT /users/{id}` |
| Use case | Creation when server owns IDs | Creation/upsert when client owns IDs |

Use POST for typical creation. Use PUT when the client can generate unique IDs (e.g., UUID v4, natural keys like ISBN).

</details>

**Q73:** What is the `WWW-Authenticate` header?

<details><summary>Answer</summary>

Sent with `401 Unauthorized` to inform the client which authentication scheme to use:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api", error="invalid_token", error_description="Token expired"
```

**Trap:** Many APIs omit this header, making automated authentication discovery impossible. Always include it.

</details>

**Q74:** How would you design an API to support Excel/CSV export?

<details><summary>Answer</summary>

Option 1 — Content negotiation:
```http
GET /reports/sales?format=csv
Accept: text/csv
```

Option 2 — Dedicated export endpoint:
```http
POST /reports/sales/export
Content-Type: application/json

{"format": "csv", "filters": {"date_from": "2026-01-01"}}
→ 202 Accepted
Location: /exports/abc-123
```

Option 3 — Async with download URL:
```http
GET /exports/abc-123
→ 200 OK
Content-Type: text/csv

<CSV content>
```

For large exports (>100MB), use async processing with streaming and presigned S3 URLs.

</details>

**Q75:** What is HATEOAS and why do few APIs implement it fully?

<details><summary>Answer</summary>

HATEOAS (Hypermedia as the Engine of Application State) means responses contain links that guide the client on possible next actions:

```json
{
  "id": 42,
  "name": "Alice",
  "_links": {
    "self": {"href": "/users/42"},
    "orders": {"href": "/users/42/orders"},
    "update": {"href": "/users/42", "method": "PUT"}
  }
}
```

Few APIs implement it because:
- Increases response size
- Additional server-side logic to determine available links per user/state
- Most clients hardcode URLs anyway
- Tooling and library support is weak
- Real benefit is limited for mobile/SPA clients that embed routing logic

</details>

**Q76:** How do you handle API request tracing across microservices?

<details><summary>Answer</summary>

Use a distributed tracing header (e.g., `X-Request-Id`, `X-Trace-Id`, or W3C `traceparent`):

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

- Root `X-Request-Id` generated at the ingress (gateway)
- Each service propagates it and adds its own span ID
- All logs include the request ID
- Aggregated tracing (Jaeger, Zipkin, OpenTelemetry) reconstructs the full flow
- Always return the request ID to the client in responses and error payloads

</details>

**Q77:** What is content negotiation by `Accept-Encoding` used for?

<details><summary>Answer</summary>

The client tells the server which compression algorithms it supports:

```http
Accept-Encoding: gzip, br
```

The server chooses (e.g., Brotli for browsers, gzip for API clients) and responds with:

```http
Content-Encoding: gzip
```

**Trap:** Don't compress already-encrypted payloads (redundant). Don't compress tiny responses (< 1KB) — overhead outweighs benefit. Be careful with BREACH attack — consider disabling compression for responses containing secrets.

</details>

**Q78:** What is the `Expect: 100-continue` mechanism?

<details><summary>Answer</summary>

The client sends headers but waits for the server's `100 Continue` before sending the body. The server can reject the request (e.g., auth failure, payload too large) before the body is sent — saves bandwidth.

The server responds with `100 Continue` to signal the client to proceed, or `417 Expectation Failed` to reject.

Useful for large uploads where the server can quickly validate headers before accepting the full body.

</details>

**Q79:** What is a mutation endpoint design pattern in REST?

<details><summary>Answer</summary>

Some APIs expose explicit action endpoints using POST:

```http
POST /orders/42/cancel
POST /users/42/activate
POST /accounts/42/deactivate
```

This is useful when the action is a clear business operation that doesn't map cleanly to CRUD (e.g., cancelling an order ≠ deleting it, and PATCHing status is too weak semantically). It's a pragmatic deviation from strict REST but widely accepted.

</details>

**Q80:** How do you validate incoming JSON payloads without coupling to DB schema?

<details><summary>Answer</summary>

Use a **schema-first** approach:

1. Define your API contract in OpenAPI / JSON Schema
2. Validate requests against the schema **before** hitting business logic
3. Decouple API validation from DB validation (they may have different rules)

```go
// Go example with go-playground/validator
type CreateUserRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Name     string `json:"name" validate:"required,min=2,max=100"`
    Age      int    `json:"age" validate:"gte=18,lte=120"`
}
```

```php
// Laravel example with FormRequest
class StoreUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => 'required|email|unique:users,email',
            'name' => 'required|string|min:2|max:100',
            'age' => 'required|integer|min:18|max:120',
        ];
    }
}
```

---

## 3. Rapid-Fire: Advanced REST & Operations (30 questions)

**Q81:** How do you maintain backward compatibility when adding fields to a response?

<details><summary>Answer</summary>

**Safe additions:** Adding new fields to a JSON response body is generally backward-compatible — old clients ignore unknown fields. Follow these rules:
- Don't change existing field types or semantics
- Don't remove fields
- New fields are optional; don't assume they exist
- New enum values should be handled gracefully by clients
- New fields default to null or a sensible default

Use the `extensions` pattern for additive changes:
```json
{"name": "Alice", "_extensions": {"membership_tier": "gold"}}
```

</details>

**Q82:** How do you design an idempotency key system for payments?

<details><summary>Answer</summary>

```http
POST /payments
Idempotency-Key: 7a1b2c3d-...

→ 201 Created
→ 200 OK (if same key was already used, returns same result)
```

Implementation:
1. Client generates a UUID and sends it as `Idempotency-Key`
2. Server checks if key was already processed — if yes, returns cached response (200/201/4xx exactly as first response)
3. If new, processes payment, stores `{key → response, status_code}` with TTL (24h minimum)
4. If processing fails (500), the client can retry with the same key — server should NOT cache failure unless it's a definitive failure

**Critical:** Idempotency must survive server restarts. Use Redis with persistence or DB. Lock on the key to prevent race conditions.

</details>

**Q83:** What is the Idempotency-Key header scope? Should it be global or per-endpoint?

<details><summary>Answer</summary>

**Per-endpoint (method + path):** The same key can be reused for different operations. Stripe uses this approach — a key is unique to the operation.

**Global:** One key for the entire API — simpler but forces clients to track per-operation.

Recommendation: scope to `{method, path, key}`. Store the hash of the full request to detect replay attempts.

**Trap:** A key used for a POST payment and then reused for a POST refund would be dangerous if scoped globally. Always scope per operation.

</details>

**Q84:** What are webhook event ordering guarantees?

<details><summary>Answer</summary>

Most webhook systems guarantee **at-least-once** delivery. To handle ordering:

1. Include a `sequence` number or `timestamp` in each event payload — the client can reconstruct order
2. Use a cursor-based approach for ordered events (like Kafka offsets)
3. Provide a "catch-up" API endpoint that returns events in order
4. Clients should be **idempotent** against webhook events (deduplicate by event ID)
5. If strict ordering is required, consider splitting into sequential event streams per resource

</details>

**Q85:** How do you implement an API gateway? When is one necessary?

<details><summary>Answer</summary>

An API gateway is a reverse proxy that sits between clients and backend services. It handles:
- Authentication/authorization
- Rate limiting
- Request routing to microservices
- Response aggregation (fan-out)
- Protocol translation (HTTP → gRPC)
- Caching
- Logging and monitoring
- TLS termination
- IP allowlisting/blocklisting

**When to use:** Multiple microservices > 3-5, need for centralized auth, different protocols across services, or public API with multiple consumers.

**When NOT to use:** Monolith with 1-2 services (adds unnecessary latency/complexity).

Popular solutions: Kong, Traefik, AWS API Gateway, Envoy, NGINX.

</details>

**Q86:** What is contract testing and how does it apply to REST APIs?

<details><summary>Answer</summary>

Contract testing verifies that two systems (e.g., API provider and consumer) agree on the API contract. Tools like **Pact** let consumers define expected interactions (request → response); the provider runs these as tests to ensure it doesn't break contracts.

**Benefits:**
- Catch breaking changes during CI (before deploy)
- No need for end-to-end test environments
- Documents real consumer expectations
- Encourages API-first design

**Pact flow:**
1. Consumer writes a Pact test (generates a contract file)
2. Contract is shared (Pact Broker)
3. Provider verifies its API matches the contract
4. CI blocks deployment if verification fails

</details>

**Q87:** What is API governance? What would you enforce?

<details><summary>Answer</summary>

API governance is the set of policies, standards, and processes that ensure APIs are consistent, secure, and maintainable. Key enforcement areas:

1. **Naming conventions** — `snake_case` vs `camelCase`, plural collection names (`/users` not `/user`)
2. **Error format** — consistent envelope, codes, field paths
3. **Versioning strategy** — URI path vs header, deprecation policy
4. **Security** — authentication required, no secrets in URLs, rate limits enforced
5. **Pagination** — consistent cursor format, limits enforced
6. **Documentation** — OpenAPI spec required, examples included
7. **Monitoring** — SLOs for latency/availability, error budget policy

Tooling: Spectral (OpenAPI linting), API style guides, review checklists, CI gates.

</details>

**Q88:** What is the OWASP API Top 10? Name the top 5 risks.

<details><summary>Answer</summary>

1. **API1:2023 — Broken Object Level Authorization** — users can access objects they shouldn't
2. **API2:2023 — Broken Authentication** — weak auth, leaked tokens, no rate limiting on login
3. **API3:2023 — Broken Object Property Level Authorization** — users can read/modify fields they shouldn't
4. **API4:2023 — Unrestricted Resource Consumption** — no rate limiting, no pagination limits, DoS
5. **API5:2023 — Broken Function Level Authorization** — users can access admin endpoints

Remaining: API6 (Unrestricted Access to Sensitive Business Flows), API7 (Server-Side Request Forgery), API8 (Security Misconfiguration), API9 (Improper Inventory Management), API10 (Unsafe Consumption of APIs).

</details>

**Q89:** How do you prevent mass assignment in REST APIs?

<details><summary>Answer</summary>

- **Whitelist approach:** define which fields can be written via the API. Don't pass request payload directly to DB models.
- **DTOs/Form Requests:** use dedicated request objects that only contain writable fields
- **Write-only views:** define a "write" schema in OpenAPI that differs from the "read" schema
- **Object-level permissions:** verify the user can write each specific field

```php
// Laravel — guarded/fillable attributes
class User extends Model
{
    protected $fillable = ['name', 'email'];  // only these can be mass-assigned
}
```

```go
// Go — explicit binding
type UpdateUserRequest struct {
    Name  *string `json:"name"`
    Email *string `json:"email"`
    // Role is NOT in the struct — cannot be set via API
}
```

</details>

**Q90:** What is the difference between `Authorization` and `X-Api-Key` authentication?

<details><summary>Answer</summary>

- **Authorization: Bearer \<token\>** — standard HTTP auth header, used for OAuth2/JWT. Follows RFC 6750. Interoperable, well-understood by proxies and tools.
- **X-Api-Key: \<key\>** — custom header for API key auth. Non-standard. Some proxies strip custom headers. Can be logged more easily (security concern).

Prefer `Authorization: Bearer` for tokens. Use `X-Api-Key` for simple server-to-server API keys, but be aware of the trade-offs.

</details>

**Q91:** What is a "retry storm" and how do you prevent it?

<details><summary>Answer</summary>

A retry storm occurs when many clients simultaneously retry after an outage/server restart, overwhelming the server. Prevent with:

1. **Exponential backoff with jitter** — clients wait progressively longer with randomness
2. **Client-side circuit breaker** — stop retrying after N failures
3. **Server-side shedding** — return `503` with `Retry-After` during recovery
4. **Rate limit retries** — apply stricter rate limits to retry traffic
5. **Random initial delay** — stagger the first retry across clients

In AWS, the "Retry-After" header with a non-zero value is legally binding for some services.

</details>

**Q92:** How do you handle API version sunset?

<details><summary>Answer</summary>

1. Announce sunset date 6+ months in advance
2. Add `Sunset` and `Deprecation` headers to existing responses
3. Provide migration guide and changelog
4. Monitor remaining traffic — contact high-volume consumers
5. After sunset date, return `410 Gone` with a link to the new version
6. Keep a redirect for at least 30 days after sunset

**Trap:** Don't block client IPs or return 404 — these cause silent failures. Always explain what happened and where to go.

</details>

**Q93:** What is the `Prefer: return=minimal` header?

<details><summary>Answer</summary>

Indicates the client only wants a status confirmation, not the full resource representation:

```http
POST /orders
Prefer: return=minimal
Content-Type: application/json

{"product_id": 42, "quantity": 1}

→ 201 Created
Location: /orders/123
  (no body, or minimal body)
```

Reduces response size for high-throughput writing clients.

</details>

**Q94:** How do you support GraphQL-style nested resource inclusion in REST?

<details><summary>Answer</summary>

Using `include` or `expand` query parameters:

```http
GET /orders/42?include=items,items.product,customer
```

Server uses eager loading to fetch related resources and includes them in the response:

```json
{
  "id": 42,
  "total": 99.99,
  "customer": {"id": 1, "name": "Alice"},
  "items": [
    {
      "id": 1,
      "product": {"id": 10, "name": "Widget"},
      "quantity": 2
    }
  ]
}
```

**Security:** Validate `include` against a whitelist to prevent N+1 attacks.

</details>

**Q95:** What is "over-fetching" and "under-fetching" in REST?

<details><summary>Answer</summary>

- **Over-fetching:** The API returns more data than the client needs (e.g., returning all user fields when only name is needed). Solved by `fields` query parameter or GraphQL.
- **Under-fetching:** The client needs multiple API calls to assemble the required data (e.g., call `/users`, then for each user call `/orders`). Solved by `include` parameter, GraphQL, or BFF (Backend for Frontend).

REST typically over-fetches (fixed response shape). GraphQL solves both but introduces complexity.

</details>

**Q96:** How do you handle conflicting webhook deliveries (same event delivered twice)?

<details><summary>Answer</summary>

Webhooks are **at-least-once**. Include a unique event ID in each delivery:

```json
{
  "event_id": "evt_abc123",
  "type": "order.created",
  "data": {"order_id": 42},
  "timestamp": "2026-07-26T12:00:00Z"
}
```

Clients must deduplicate by `event_id`. Store processed event IDs and skip duplicates. Event IDs should be globally unique and monotonically increasing for ordering.

</details>

**Q97:** What is the problem with `X-RateLimit-*` vs standard `RateLimit-*` headers?

<details><summary>Answer</summary>

`X-*` headers are deprecated per RFC 6648 — they were the convention for "unstandardised" headers. The industry is moving to `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` per the RateLimit header fields for HTTP draft. However, `X-RateLimit-*` still has wider implementation (GitHub, Twitter). Use both during transition.

</details>

**Q98:** How do you design a pagination response with metadata?

<details><summary>Answer</summary>

```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6NDJ9",
    "has_more": true,
    "total": 1000,
    "limit": 20
  }
}
```

With Link header for discoverability:

```http
Link: </items?cursor=eyJpZCI6NDJ9&limit=20>; rel="next",
      </items?cursor=eyJpZCI6MH0=&limit=20>; rel="prev"
```

Avoid exposing `total` for cursor-based pagination if counting is expensive (large datasets). Use `has_more` only.

</details>

**Q99:** How do you handle schema migration in a versioned API?

<details><summary>Answer</summary>

1. **New version, new endpoint** — `POST /v2/users` is completely separate from `POST /v1/users`. Backing services can change freely.
2. **Shared backend, versioned serializers** — same DB, different serialisation. The `v1` UserSerializer and `v2` UserSerializer map to the same DB model but output different fields.
3. **DB-level versioning** — store version-specific data (table per version, or polymorphic columns). Adds significant complexity.

Recommendation: Shared backend with versioned serializers for incremental changes. Cut a new version only for breaking changes.

</details>

**Q100:** What is OpenAPI `discriminator` for?

<details><summary>Answer</summary>

Used for polymorphic schemas — when a property value determines the object type:

```yaml
Pet:
  oneOf:
    - $ref: '#/components/schemas/Cat'
    - $ref: '#/components/schemas/Dog'
  discriminator:
    propertyName: pet_type
```

When a response includes `{"pet_type": "Cat", "meows": true}`, the discriminator maps it to the Cat schema. Essential for describing APIs with inheritance.

</details>

**Q101:** Explain connection pooling for HTTP services and common pitfalls.

<details><summary>Answer</summary>

Connection pooling reuses TCP connections for multiple HTTP requests — avoids the overhead of TLS handshakes per request.

**Pitfalls:**
1. **Exhaustion under load** — too few connections cause queuing; too many exhaust file descriptors
2. **Stale connections** — idle connections closed by firewalls/LB without notification lead to "connection reset" errors
3. **No TLS session reuse** — pooling without TLS session caching still pays handshake cost
4. **Per-host limits** — shared pools across many hosts may starve individual hosts
5. **Memory leaks** — uncleaned connections over time

In Go, use `http.Transport` with sensible `MaxIdleConnsPerHost`. In PHP, use persistent cURL handles or Guzzle pool with connection limits.

</details>

**Q102:** What is the `Idempotency-Key` TTL and why does it matter?

<details><summary>Answer</summary>

The TTL determines how long the server remembers a processed idempotency key. Stripe uses 24 hours. The TTL must cover:
- The longest possible client retry interval
- Network timeouts
- Backend processing time

**Short TTL:** Client may retry after key expiry, causing duplicate processing.
**Long TTL:** More storage, key exhaustion.

Set TTL to 24 hours minimum for payment APIs. Store keys in Redis with automatic expiry. On key expiry, the server will process the request again — the client must handle this.

</details>

**Q103:** How do you detect and prevent API abuse?

<details><summary>Answer</summary>

1. **Rate limiting** — per user, per IP, per endpoint
2. **Anomaly detection** — sudden spikes from a single user, unusual patterns (e.g., scanning endpoints)
3. **Honeytokens** — fake API keys that alert when used
4. **IP reputation** — block known bad actors
5. **User-agent analysis** — detect non-browser or spoofed agents
6. **Behavioural limits** — max N accounts per IP, max N password reset attempts
7. **Captchas** — for sensitive endpoints (login, registration)
8. **API key rotation** — force periodic rotation, revoke compromised keys

At scale, use a Web Application Firewall (WAF) + edge rate limiting (Cloudflare, AWS WAF).

</details>

**Q104:** How do GraphQL and REST differ in versioning?

<details><summary>Answer</summary>

- **REST:** URLs or headers versioned (`/v1/users`). Breaking changes require a new version. Multiple versions coexist.
- **GraphQL:** No versioning by default — deprecate fields via the `@deprecated` directive. Clients only request what they need, so adding new fields is non-breaking. Breaking changes (removing a field) are communicated via deprecation.

GraphQL's approach works well when API consumers control their queries. For third-party APIs, field removal still breaks clients that don't check deprecation.

</details>

**Q105:** What are common pitfalls with PATCH semantics?

<details><summary>Answer</summary>

1. **JSON Merge Patch (RFC 7396):** Sending `{"name": null}` sets name to null — can't distinguish "set to null" from "omit field"
2. **JSON Patch (RFC 6902):** Explicit operations like `{"op": "replace", "path": "/name", "value": "Alice"}` — more verbose but unambiguous
3. **Null vs absent:** Clients often send `PATCH {"name": ""}` to clear, but the server interprets it as "set to empty string", not "clear the field"
4. **Partial update with validation:** PATCH should validate the final state, not just the patched fields
5. **Idempotency:** PATCH is not inherently idempotent (e.g., `{"counter": "counter + 1"}`). Use if-match/etag for safe partial updates

</details>

**Q106:** How do you implement search-as-a-service integration in a REST API?

<details><summary>Answer</summary>

1. **Indexing:** Use DB triggers or CDC (Change Data Capture) to feed Elasticsearch/MeiliSearch
2. **Search endpoint:**
```http
GET /products/search?q=wireless+headphones&sort=_score&filters=price:10-100
```
3. **Full-text search with ranking:**
```http
POST /products/search
Content-Type: application/json

{"query": "wireless headphones", "filters": {"category": "electronics"}, "sort": "price:asc"}
```
4. **Federated search** across multiple indices:
```http
POST /search
Content-Type: application/json

{"queries": {"products": {...}, "users": {...}}}
```

Ensure search index is near real-time (NRT) — tolerate seconds of delay. Use webhook or polling to keep index in sync.

</details>

**Q107:** What does the `Sec-WebSocket-Key` header do?

<details><summary>Answer</summary>

Used during WebSocket handshake to prevent caching proxies from serving stale WebSocket connections. The client sends a random key, the server concatenates it with a GUID, hashes with SHA-1, and returns the hash in `Sec-WebSocket-Accept`. This proves the server understands the WebSocket protocol.

</details>

**Q108:** What is the `Early-Data` header in HTTP?

<details><summary>Answer</summary>

Used with TLS 1.3 0-RTT (zero round trip time resumption). The client can send data immediately after the TLS `ClientHello`, before the handshake completes. The `Early-Data` header informs the server that the request body was sent as early data.

**Risk:** 0-RTT is vulnerable to replay attacks. The server must be idempotent for early data requests.

</details>

**Q109:** What is a "slow loris" attack and how do you mitigate it?

<details><summary>Answer</summary>

A DoS attack where the attacker sends HTTP headers slowly, one byte at a time, keeping connections open and exhausting server connection pools.

**Mitigations:**
- Set `request_timeout` on the server (e.g., NGINX `client_header_timeout`)
- Limit concurrent connections per IP
- Use a reverse proxy with connection buffering
- Set `client_body_timeout` for request body

</details>

**Q110:** Explain the difference between `Content-Encoding` and `Transfer-Encoding`.

<details><summary>Answer</summary>

- **Content-Encoding:** The body encoding is part of the resource (e.g., `gzip`). The resource IS compressed. Caches can store the compressed version independently.
- **Transfer-Encoding:** The encoding is applied between hops only (e.g., `chunked`). A proxy may decompress and re-compress. Not part of the resource representation.

`Content-Encoding: gzip` means the resource itself is gzipped. `Transfer-Encoding: chunked` means the data is streamed in chunks; it's a wire format, not a resource property.

---

## 4. Code Puzzles

### Puzzle 1: Idempotency & Status Code Reasoning

Given the following scenarios, determine which HTTP methods are idempotent and what status code the server should return:

**Scenario A:** A client sends `DELETE /items/42` three times in a row. The first returns `200 OK`. What should the second and third return?

**Scenario B:** A client sends `PUT /users/42` with `{"name": "Alice"}`. The resource doesn't exist. What status code? If they send the same request again, what status code?

**Scenario C:** A client sends `POST /orders` without an idempotency key. The server creates the order (201), but the TCP connection drops before the client receives the response. The client retries. What happens?

<details><summary>Answer</summary>

**Scenario A:**

```http
DELETE /items/42
→ 200 OK (resource deleted)

DELETE /items/42
→ 404 Not Found (resource doesn't exist; operation is still idempotent — server state unchanged)

DELETE /items/42
→ 404 Not Found
```

DELETE is idempotent: the effect on server state is the same (resource is gone), even if the response code changes.

Some APIs return `204 No Content` for DELETE. Subsequent deletes returning 404 is acceptable. The idempotency guarantee is about **server state**, not the response code.

**Scenario B:**

```http
PUT /users/42
→ 201 Created (resource didn't exist, now it does)

PUT /users/42
→ 200 OK (resource exists, replaced with same state)
```

PUT is idempotent. The first request creates the resource; the second replaces it with identical content. Both result in the same server state. Returning 201 for creation, 200 for subsequent updates is a common convention.

**Scenario C:**

This is the classic **lost response** problem. Without an idempotency key:

1. First POST: Server creates order #42, stores it, returns 201
2. TCP drops, client gets nothing
3. Client retries with the same POST (no idempotency key)
4. Server creates **order #43** — duplicate order

With an idempotency key:

```http
POST /orders
Idempotency-Key: 7a1b2c3d-...

→ 201 Created

Retry with same key:
POST /orders
Idempotency-Key: 7a1b2c3d-...

→ 200 OK (same order #42 returned, no duplicate)
```

The server checks the key, finds it was already processed, and returns the cached response.

</details>

### Puzzle 2: CORS Issue Identification

Given the following request/response pairs, identify whether there's a CORS issue and what the browser will do:

**Case A:**
```
Preflight OPTIONS request:
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, X-Custom-Header

Preflight response:
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Authorization
```

**Case B:**
```
Simple cross-origin GET request:
Origin: https://app.example.com

Response:
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Set-Cookie: session=abc123
```

**Case C:**
```
Request with credentials:
GET /api/user
Origin: https://app.example.com
Credentials: include

Response:
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

<details><summary>Answer</summary>

**Case A — CORS FAILURE** 🚫

The preflight response allows `POST` but the client requests `POST` — that's fine. However, the client requested `Authorization, X-Custom-Header` but the server only allows `Authorization`. Since `X-Custom-Header` is not in the allowed list, the browser will **block the actual request**.

**Fix:** Add `X-Custom-Header` to `Access-Control-Allow-Headers`.

**Case B — CORS SUCCESS** ✅

The GET has no preflight (simple request — GET, no custom headers). The response allows the origin with credentials. The cookie will be set.

One subtle point: with `Access-Control-Allow-Credentials: true`, the browser requires `Access-Control-Allow-Origin` to be explicit (not `*`). Here it's explicit, so it's fine.

**Case C — CORS FAILURE** 🚫

When `Access-Control-Allow-Credentials: true`, the `Access-Control-Allow-Origin` header **cannot** be `*`. It must be an explicit origin. The browser will reject this.

**Fix:** Set `Access-Control-Allow-Origin: https://app.example.com` explicitly. For multiple origins, check the request's `Origin` header dynamically.

</details>

### Puzzle 3: Cursor Pagination Encoding/Decoding

Given this cursor-based pagination implementation, trace the flow:

```go
type Cursor struct {
    ID        int64     `json:"id"`
    CreatedAt time.Time `json:"created_at"`
}

func EncodeCursor(id int64, createdAt time.Time) string {
    c := Cursor{ID: id, CreatedAt: createdAt}
    data, _ := json.Marshal(c)
    return base64.URLEncoding.EncodeToString(data)
}

func DecodeCursor(encoded string) (*Cursor, error) {
    data, err := base64.URLEncoding.DecodeString(encoded)
    if err != nil {
        return nil, err
    }
    var c Cursor
    if err := json.Unmarshal(data, &c); err != nil {
        return nil, err
    }
    return &c, nil
}
```

**Questions:**

1. What is the cursor for the last user on page 1 (user ID=50, created_at="2026-07-26T10:00:00Z")?
2. Write the SQL query for the next page using this cursor.
3. What happens if a malicious client modifies the cursor base64 string?
4. How would you add cursor expiry?

<details><summary>Answer</summary>

**1. Encoded cursor:**

```json
{"id":50,"created_at":"2026-07-26T10:00:00Z"}
```

Base64 URL-encoded:
`eyJpZCI6NTAsImNyZWF0ZWRfYXQiOiIyMDI2LTA3LTI2VDEwOjAwOjAwWiJ9`

**2. SQL query for next page:**

```sql
SELECT id, name, email, created_at
FROM users
WHERE (created_at > '2026-07-26T10:00:00Z')
   OR (created_at = '2026-07-26T10:00:00Z' AND id > 50)
ORDER BY created_at ASC, id ASC
LIMIT 20
```

The composite cursor handles the case where multiple users have the same `created_at` timestamp. Using `id` as tiebreaker ensures stable ordering.

**3. Malicious cursor modification:**

A client modifying the base64 string will produce either:
- Invalid base64 → decode error → return `400 Bad Request` with `INVALID_CURSOR`
- Valid base64 but invalid JSON → decode error → `400 Bad Request`
- Valid JSON but wrong type → validation error (e.g., `id` not int64) → `400 Bad Request`
- Valid cursor but pointing to non-existent ID → query returns empty results (graceful)

Always validate and sanitise cursor input. Never trust client-supplied cursors for security decisions (they only control pagination position, not data access — authorisation is enforced separately).

**4. Cursor expiry:**

Add an expiry timestamp to the cursor:

```go
type Cursor struct {
    ID        int64     `json:"id"`
    CreatedAt time.Time `json:"created_at"`
    ExpiresAt time.Time `json:"exp"`
}
```

Encode `time.Now().Add(1 * time.Hour)` as the expiry. On decode:

```go
if time.Now().After(c.ExpiresAt) {
    return nil, errors.New("cursor expired")
}
```

This prevents replay attacks and forces clients to refresh outdated pagination states. Alternatively, sign the cursor with HMAC:

```go
func SignCursor(cursor string, secret []byte) string {
    mac := hmac.New(sha256.New, secret)
    mac.Write([]byte(cursor))
    return cursor + "." + base64.URLEncoding.EncodeToString(mac.Sum(nil))
}
```

The client cannot forge a valid cursor without the secret.

</details>

### Puzzle 4: ETag Conditional Request Flow

Trace the following interaction:

Client has cached: `{"user": {"id": 42, "name": "Alice"}}` with `ETag: "a1b2c3"`.

**Request 1:**
```http
GET /users/42
If-None-Match: "a1b2c3"
```

Server response:
```http
304 Not Modified
ETag: "a1b2c3"
```

**Request 2 (concurrent update by another client):**
```http
PUT /users/42
If-Match: "a1b2c3"
Content-Type: application/json

{"name": "Bob"}
```

Server processes, returns:
```http
200 OK
ETag: "d4e5f6"
```

**Request 3 (same as Request 1, with stale ETag):**
```http
GET /users/42
If-None-Match: "a1b2c3"
```

What does the server return?

**Request 4 (idempotent retry of Request 2):**
```http
PUT /users/42
If-Match: "a1b2c3"
Content-Type: application/json

{"name": "Bob"}
```

What does the server return?

<details><summary>Answer</summary>

**Request 3 — Server returns:**

```http
200 OK
ETag: "d4e5f6"
Content-Type: application/json

{"id": 42, "name": "Bob"}
```

The client's `If-None-Match: "a1b2c3"` no longer matches the current ETag (`d4e5f6`), so the server returns the full response. The client updates its cache with the new ETag and body.

**Request 4 — Server returns:**

```http
412 Precondition Failed
```

The `If-Match: "a1b2c3"` precondition fails because the current ETag is `d4e5f6`, not `a1b2c3`. The PUT is **not** idempotent in the sense of "same request always succeeds" — the idempotency guarantee is "same request produces same server state if accepted". Here the precondition prevents re-application.

The client must re-GET to get the new ETag, then re-apply the PUT with the updated `If-Match`.

**Key insight:** ETag-based optimistic concurrency means concurrent writes can fail with 412. The client must handle this by refreshing and retrying.

</details>

### Puzzle 5: Webhook Signature Verification

Given a webhook payload and signature:

```
Payload:  {"event":"order.created","order_id":42,"timestamp":1721980800}
Secret:   whsec_abc123def456
Received HMAC-SHA256 hex signature:  7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a

Signed payload format:  timestamp + "." + body
```

**Questions:**

1. Write the verification logic (pseudocode or Go/PHP/JS).
2. What is the purpose of including the `timestamp` in the signed payload?
3. How do you prevent replay attacks?
4. What if the signature uses a different algorithm than expected?

<details><summary>Answer</summary>

**1. Verification logic (Go):**

```go
func VerifyWebhook(secret, body []byte, timestamp int64, receivedSig string) error {
    mac := hmac.New(sha256.New, secret)
    _, _ = fmt.Fprintf(mac, "%d.%s", timestamp, body)
    expected := hex.EncodeToString(mac.Sum(nil))

    if !hmac.Equal([]byte(expected), []byte(receivedSig)) {
        return errors.New("invalid signature")
    }
    return nil
}
```

**PHP:**

```php
function verifyWebhook(string $secret, string $body, int $timestamp, string $sig): bool {
    $expected = hash_hmac('sha256', "$timestamp.$body", $secret);
    return hash_equals($expected, $sig);  // timing-safe comparison
}
```

**JS:**

```js
const crypto = require('crypto');

function verifyWebhook(secret, body, timestamp, sig) {
    const expected = crypto
        .createHmac('sha256', secret)
        .update(`${timestamp}.${body}`)
        .digest('hex');
    return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(sig));
}
```

Always use **constant-time comparison** (`hash_equals` in PHP, `crypto.timingSafeEqual` in Node, `hmac.Equal` in Go) to prevent timing attacks.

**2. Why include the timestamp:**

Prevents **replay attacks** — an attacker who captures a valid signature can replay it later. By including the timestamp, the receiver can reject signatures older than a tolerance (e.g., 5 minutes):

```go
if time.Now().Unix() - timestamp > 300 {
    return errors.New("signature too old")
}
```

**3. Prevent replay attacks:**

- Store processed `(event_id, timestamp)` pairs with TTL
- Reject signatures with timestamps outside a 5-minute window
- Use a unique `event_id` in the payload and reject duplicates

Combined approach: verify timestamp freshness, then deduplicate by event ID.

**4. Different algorithm:**

If the signature uses SHA-512 but your code only checks SHA-256, the comparison will fail. But **never fall back to a less secure algorithm** if the signature is invalid. Log the algorithm from the header, validate it against expected algorithms:

```go
func VerifyWebhook(secret []byte, body []byte, timestamp int64, sig string, alg string) error {
    var mac hash.Hash
    switch alg {
    case "sha256": mac = hmac.New(sha256.New, secret)
    case "sha512": mac = hmac.New(sha512.New, secret)
    default: return errors.New("unsupported algorithm: " + alg)
    }
    // ... continue with MAC computation
}
```

</details>

### Puzzle 6: Rate Limit Header Interpretation

A client receives these responses:

**Response A:**
```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 42
RateLimit-Reset: 1721981400
```

**Response B:**
```http
HTTP/1.1 429 Too Many Requests
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1721981400
Retry-After: 120
```

**Questions:**

1. A client sends 60 requests and receives Response A each time. After request 60, RateLimit-Remaining = 42. How many total requests does the client think it can make? What's wrong?
2. Interpret the `RateLimit-Reset` value in Response B. When can the client retry?
3. What is a "rolling window" rate limit vs "fixed window"?
4. Design a client-side algorithm that respects rate limits using these headers.

<details><summary>Answer</summary>

**1. The client-side count discrepancy:**

If after 60 requests the client still sees `RateLimit-Remaining: 42`, that means:

- The counter counts to 60, but the server says only 58 were counted (100 - 42 = 58)
- Possible reasons:
  - Multiple clients sharing the same API key
  - The server's window is different from the client's perception
  - Load balancers distribute requests across rate limiter instances with eventual consistency

**Key insight:** The `Remaining` count is the **server's authoritative** count. The client should trust it, not its own tally. Never use local counting to determine when to stop — always trust the server's headers.

**2. Retry interpretation:**

`RateLimit-Reset: 1721981400` is a **Unix timestamp** (seconds since epoch). The window resets at that time.

`Retry-After: 120` says the client should wait 120 seconds before retrying.

The client should use `Retry-After` for immediate retry scheduling, and `RateLimit-Reset` for long-term throttling.

If `Retry-After` and `RateLimit-Reset` disagree, prefer `Retry-After` (more specific to the client's situation).

**3. Fixed Window vs Rolling Window:**

**Fixed Window:**
```
[---- Window 1 ----] [---- Window 2 ----]
12:00:00             12:01:00             12:02:00
```
Counter resets at fixed boundaries. Problem: a burst at the end of one window and start of next can double the effective limit.

**Rolling Window (Sliding Window):**
```
[------ Rolling 60s ------]
```
Each request's timestamp is tracked. The window slides continuously. More accurate but more memory-intensive.

Hybrid: **Sliding Window Counter** — use the previous window's count + current window's count * overlap ratio. Redis sorted sets with `ZREMRANGEBYSCORE` for sliding windows.

**4. Client-side rate limit algorithm:**

```go
type RateLimitState struct {
    Limit     int
    Remaining int
    ResetAt   time.Time
}

func RespectRateLimit(state *RateLimitState) {
    if state.Remaining <= 0 {
        // Wait until reset
        wait := time.Until(state.ResetAt) + time.Second
        time.Sleep(wait)
        return
    }

    // Calculate delay to avoid hammering
    // Spread requests evenly across the remaining window
    windowDuration := time.Until(state.ResetAt)
    delay := windowDuration / time.Duration(state.Remaining + 1)
    time.Sleep(delay)
}
```

Key principle: **conservative** — always assume the remaining count is accurate and spread requests evenly.

</details>

### Puzzle 7: HTTP Method Semantics in a Banking API

Consider a banking API with accounts and transactions. For each of these operations, determine the appropriate HTTP method, idempotency guarantee, and status codes:

1. **Transfer funds** between two accounts
2. **Check account balance**
3. **Freeze an account** (temporary hold)
4. **Generate a monthly statement** (takes 30 seconds)
5. **Update account nickname**

<details><summary>Answer</summary>

**1. Transfer funds:**

```http
POST /transfers
Idempotency-Key: uuid-here
Content-Type: application/json

{"from": "acc_123", "to": "acc_456", "amount": 100.00, "currency": "USD"}

→ 201 Created (transfer initiated)
→ 200 OK (same idempotency key — returns same transfer)
→ 409 Conflict (insufficient funds)
```

POST with idempotency key. NOT idempotent without the key. Double-entry bookkeeping ensures atomicity. Use **transactional outbox** pattern to guarantee exactly-once processing.

**2. Check balance:**

```http
GET /accounts/acc_123/balance
→ 200 OK
→ 304 Not Modified (with If-None-Match)
```

GET — safe and idempotent. Cacheable with ETag support.

**3. Freeze account:**

```http
POST /accounts/acc_123/freeze

→ 200 OK (account frozen)
→ 409 Conflict (already frozen)
```

POST action endpoint (or PATCH with status). POST is not idempotent here — second freeze returns 409. Alternatively:

```http
PATCH /accounts/acc_123
Content-Type: application/json

{"status": "frozen"}

→ 200 OK (with ETag)
→ 412 Precondition Failed (concurrent modification)
```

**4. Generate monthly statement:**

```http
POST /statements
Prefer: respond-async
Content-Type: application/json

{"account": "acc_123", "month": "2026-07"}

→ 202 Accepted
Location: /operations/op_456

GET /operations/op_456
→ 200 OK
→ 303 See Other (redirect to completed statement)
```

POST (not idempotent) + async processing. The operation endpoint is read-only.

**5. Update account nickname:**

```http
PATCH /accounts/acc_123
Content-Type: application/json

{"nickname": "Main Checking"}

→ 200 OK
→ 412 Precondition Failed (with If-Match)
```

PATCH — idempotent only if the patch is a full replacement of the nickname field. Safe with conditional headers.

</details>

### Puzzle 8: API Gateway Circuit Breaker Trace

An API gateway uses a circuit breaker with these settings:

- **Closed** → normal operation, requests pass through
- **Open** → requests fail immediately (503) for 30 seconds
- **Half-Open** → allows 5 test requests; if 80% succeed, close; else re-open

Trace the state for this scenario:

```
T=0s:  100 normal requests → all succeed
T=10s: 100 requests → 60 fail (upstream timeout)
T=15s: 50 requests → 50 fail
T=20s: state changes
T=20s: 1 request arrives
T=25s: 5 test requests arrive
T=35s: 50 requests arrive
```

<details><summary>Answer</summary>

```
T=0s:  State = CLOSED. 100/100 succeed. Error rate = 0%.
T=10s: State = CLOSED. 60 failures in this batch. Rolling error rate exceeds threshold (e.g., 50% in 60s window).
T=15s: State = CLOSED → OPEN. The circuit breaker trips after detecting 60 failures in the last 10 seconds.
        All 50 requests fail immediately with 503 Service Unavailable.
T=20s: State = OPEN (has been open for 0s). The 30-second open timer starts at T=15s, so it's still open.
        1 request arrives → 503 Service Unavailable immediately.
T=25s: State still OPEN (15s elapsed, need 30s).
T=45s: State transitions OPEN → HALF-OPEN (30s elapsed since T=15s).
T=45s+: 5 test requests arrive:
  - 4 succeed, 1 fail → 80% success → state → CLOSED
  - (If 3 succeed, 2 fail → 60% success → state → OPEN again)
T=45s+ (assume 4/5 succeed): State = CLOSED.
T=50s+: 50 requests → all succeed (upstream recovered).
```

**Key considerations:**
- The error rate calculation must be **rolling** (e.g., failures in the last 60 seconds), not per-batch
- The half-open probe count must be small enough to limit damage but large enough for statistical significance
- Circuit breakers should distinguish between client errors (4xx) and server errors (5xx) — only 5xx should trip
- Implement a **gradual recovery**: after closing, reduce concurrency temporarily to avoid overwhelming the upstream

</details>

---

## 5. System Design Prompts

### Design 1: URL Shortener API (bit.ly clone)

**Requirements:**
- Shorten a long URL → 7-character base62 key
- Redirect: `GET /{key}` → 301 to original URL
- 100M total URLs stored, 10K writes/day, 1M redirects/day
- Analytics: count clicks per URL per day
- Expire unused URLs after 1 year

**Estimation:**
- Storage: 100M × ~500 bytes = 50GB
- Write: 10K/day = ~0.1 writes/sec (trivial)
- Read: 1M/day = ~12 reads/sec (peak 10x = 120 QPS)
- Key space: base62(7) ≈ 3.5 trillion — enough

**Data Model:**

```json
{
  "key": "abc1234",          // 7-char base62, unique
  "original_url": "https://...",
  "created_at": "2026-07-26T...",
  "expires_at": "2027-07-26T...",
  "click_count": 42,
  "user_id": "user_xxx"
}
```

**Key Decisions:**
1. **Key generation:** Use distributed ID generator (Snowflake) → encode to base62. Avoid DB auto-increment (predictable).
2. **Redirect:** HTTP 301 (permanent) for browser caching. 302 (temporary) if analytics per redirect matter.
3. **Storage:** PostgreSQL (primary), Redis cache for hot URLs (LRU, TTL 24h).
4. **Cache strategy:** Cache-on-read: miss → query DB → populate cache. Write-through for click counts.
5. **Analytics:** Async increment via Redis + batch flush to DB every 5 minutes.
6. **Expiry:** Background cron job scans for `expires_at < NOW()`, soft deletes from DB, invalidates cache.

**API Surface:**
```http
POST /shorten
Content-Type: application/json
Authorization: Bearer <token>

{"url": "https://example.com/very/long/path?with=params"}

→ 201 Created
Location: /abc1234
{"key": "abc1234", "short_url": "https://short.url/abc1234"}
```

```http
GET /abc1234
→ 301 Moved Permanently
Location: https://example.com/very/long/path?with=params
```

```http
GET /abc1234/stats
→ 200 OK
{"key": "abc1234", "total_clicks": 42, "clicks_today": 5}
```

**Trade-offs:**
- Base62 vs UUID: Base62 is shorter (7 chars), UUIDs allow offline generation
- 301 vs 302: 301 is cached by browsers (faster subsequent redirects), 302 lets you track every click
- SQL vs NoSQL: SQL for relational data (users, analytics), Redis for hot cache

---

### Design 2: Rate Limiter for a Public API

**Requirements:**
- 100 requests/minute per API key
- 1,000 requests/hour per API key
- Burst up to 20 requests
- Distributed across 10 servers
- Low latency (< 5ms overhead)
- Configurable per-endpoint limits

**Estimation:**
- 10K API keys, 1M requests/day peak = ~1,000 QPS
- 10 server nodes → 100 QPS per node local counts

**Algorithm: Sliding Window Counter (Redis)**

```
Current window:  [timestamp // 60 * 60]  — fixed 1-min window
Previous window: current - 60

Rate = prev_count * (1 - overlap_ratio) + current_count
```

**Data Model (Redis):**
```
Key: ratelimit:{key}:{endpoint}:{window}
Value: counter (INCR)
TTL: 120 seconds

Key: ratelimit:{key}:{endpoint}:prev_window
Value: counter
TTL: 120 seconds
```

**Key Decisions:**
1. **Local + central hybrid:** Each node maintains a local token bucket for burst. Periodically syncs with Redis for global limits. This avoids Redis bottleneck at high QPS.
2. **Token bucket for bursts:** 20 bucket tokens refilled at rate 100/60 per second. Local, no Redis call for token bucket.
3. **Redis for hourly limit:** Use sorted sets (`ZADD` + `ZREMRANGEBYSCORE` + `ZCARD`) for sliding window. Acceptable for 1K QPS.
4. **Headers:** `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, `Retry-After` on 429.

**API Surface:**
```http
POST /api/endpoint
X-Api-Key: abc123

→ 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 87
RateLimit-Reset: 1721981400

→ 429 Too Many Requests
RateLimit-Limit: 100
RateLimit-Remaining: 0
Retry-After: 45
```

**Trade-offs:**
- Redis sorted sets vs INCR counters: Sorted sets are accurate but more memory; INCR counters are efficient but have window boundary issues
- Centralised vs distributed: Centralised is simpler but adds latency; distributed is faster but has eventual consistency (bursts can exceed limits)
- Token bucket vs sliding window: Token bucket handles bursts naturally; sliding window is more precise for long-term limits

---

### Design 3: Payment Processing API

**Requirements:**
- Charge a customer (create payment intent)
- Refund a payment
- Webhook notifications for payment status changes
- Idempotency support for retries
- Idempotency key expiry after 24 hours
- 100K payments/day, 99.99% uptime requirement

**Estimation:**
- 100K payments/day = ~1.2 writes/sec (peak 20/sec)
- Refund rate: 5%
- Webhook delivery: 3 retries, 72-hour window
- Idempotency cache: 24h TTL, ~100K entries

**Data Model:**

```json
// Payment
{
  "id": "pi_abc123",
  "amount": 2999,  // cents
  "currency": "usd",
  "status": "succeeded",  // pending | processing | succeeded | failed | refunded
  "customer_id": "cus_xyz",
  "idempotency_key": "key_uuid",
  "created_at": "...",
  "metadata": {}
}

// Refund
{
  "id": "ref_abc123",
  "payment_id": "pi_abc123",
  "amount": 2999,
  "status": "succeeded",
  "reason": "customer_request"
}
```

**Key Decisions:**
1. **Idempotency store:** Redis with 24h TTL. Store `{key → {response, status_code, created_at}}`. On retry, return cached response.
2. **Duplicate detection:** Use a unique constraint on `idempotency_key` in the DB. Double-write: Redis + DB.
3. **Payment state machine:** `pending → processing → succeeded/failed`. Use optimistic locking on status transitions.
4. **Webhook system:** Event table + outbox pattern. Worker polls event table and delivers to registered webhook URLs.
5. **At-least-once delivery:** Each event has a unique `event_id`. Clients deduplicate.
6. **Refund idempotency:** Separate idempotency key space per operation (payment vs refund).

**API Surface:**
```http
POST /v1/payments
Idempotency-Key: 7a1b2c3d-...
Content-Type: application/json

{
  "amount": 2999,
  "currency": "usd",
  "customer_id": "cus_xyz",
  "description": "Pro plan monthly"
}

→ 201 Created
{"id": "pi_abc123", "status": "processing"}
```

```http
GET /v1/payments/pi_abc123
→ 200 OK
{"id": "pi_abc123", "status": "succeeded", "amount": 2999}
```

```http
POST /v1/payments/pi_abc123/refund
Idempotency-Key: 8b2c3d4e-...
→ 200 OK
{"id": "ref_abc123", "payment_id": "pi_abc123", "status": "succeeded"}
```

**Webhook payload:**
```json
{
  "event_id": "evt_abc123",
  "type": "payment.succeeded",
  "data": {"payment_id": "pi_abc123", "amount": 2999},
  "timestamp": 1721980800
}
// Headers: X-Signature-256: <HMAC-SHA256>
```

**Trade-offs:**
- 2PC vs transactional outbox: 2PC is too complex; outbox with CDC (e.g., Debezium) ensures DB + event queue consistency
- Blockchain vs traditional: Blockchain adds latency and cost; traditional DB + audit log is sufficient for most use cases
- Synchronous vs async: Synchronous payment processing for immediate confirmation; async for slow payment methods (ACH, wire)

---

### Design 4: Multi-Tenant Inventory API

**Requirements:**
- Row-level tenancy (all tenants share the same DB tables)
- Tenant-scoped endpoints: `GET /{tenant_id}/items`
- Composite keys: `(tenant_id, sku)` as primary key
- Each tenant has different stock counts, prices, and categories
- 10K tenants, 1M SKUs total, 1K requests/sec
- Admins can query across tenants (audit, reporting)

**Estimation:**
- 10K tenants × 100 SKUs avg = 1M SKUs
- Write: 100/sec (stock updates, new SKUs)
- Read: 900/sec (catalogue browsing, stock checks)
- Storage: ~1GB (lean model)

**Data Model:**

```sql
CREATE TABLE inventory_items (
    tenant_id    BIGINT NOT NULL,
    sku          VARCHAR(64) NOT NULL,
    name         VARCHAR(255) NOT NULL,
    description  TEXT,
    price        BIGINT NOT NULL,  -- cents
    stock        INT NOT NULL DEFAULT 0,
    category     VARCHAR(128),
    version      INT NOT NULL DEFAULT 1,
    created_at   TIMESTAMP NOT NULL,
    updated_at   TIMESTAMP NOT NULL,
    PRIMARY KEY (tenant_id, sku)
);

-- Index for category queries within a tenant
CREATE INDEX idx_inventory_tenant_category
    ON inventory_items (tenant_id, category);

-- Index for admin cross-tenant queries
CREATE INDEX idx_inventory_updated
    ON inventory_items (updated_at);
```

**Key Decisions:**
1. **Row-level tenancy:** `tenant_id` as partition key in all queries. Every SQL query includes `WHERE tenant_id = ?` — non-negotiable.
2. **Composite PK:** `(tenant_id, sku)` — SKUs are unique per tenant, not globally.
3. **Connection pooling per tenant?** No — too many connections. Shared pool with parameterised queries.
4. **Row-level security (RLS):** PostgreSQL RLS policies enforce tenant isolation at the DB level — defense in depth.
5. **Caching:** Redis cache keyed by `{tenant_id}:{sku}` for hot items. Global cache for catalogue data (same across tenants).
6. **Version column:** Optimistic concurrency for stock updates — `UPDATE ... WHERE tenant_id = ? AND sku = ? AND version = ?`.

**API Surface:**

```http
GET /v1/{tenant_id}/items?category=electronics&page=1&limit=20
Authorization: Bearer <token>

→ 200 OK
{
  "data": [
    {"sku": "WID-001", "name": "Widget", "price": 999, "stock": 42}
  ],
  "pagination": {"cursor": "...", "has_more": false}
}
```

```http
PATCH /v1/{tenant_id}/items/WID-001
If-Match: <version>
Content-Type: application/json

{"stock": 50}

→ 200 OK
{"sku": "WID-001", "stock": 50, "version": 2}
```

```http
POST /v1/{tenant_id}/items
Content-Type: application/json

{"sku": "WID-002", "name": "Super Widget", "price": 1999, "stock": 10}

→ 201 Created
```

**Admin cross-tenant:**
```http
GET /v1/admin/items?tenant_ids=1,2,3&stock_lt=10
Authorization: Bearer <admin_token>

→ 200 OK
```

**Trade-offs:**
- Row-level vs schema-per-tenant: Row-level is easier to manage (fewer tables, simpler migrations) but weaker isolation. Schema-per-tenant is harder to manage (N schemas) but stronger isolation.
- Composite key vs UUID: Composite key makes queries more explicit but requires all queries to include tenant_id. UUIDs with tenant_id column add index overhead.
- RLS vs application-level filtering: RLS is defense in depth, but adds query overhead. Application-level filtering is faster but riskier (a missing `WHERE tenant_id = ?` leaks data).

---

### Design 5: Notification Service API

**Requirements:**
- Send notifications via email, SMS, push (FCM/APNs)
- Webhook delivery for notification status
- Template-based message rendering
- Rate limiting per channel (e.g., max 10 SMS/min per user)
- 1M notifications/day, 50% email, 30% push, 20% SMS
- Delivery tracking and analytics

**Estimation:**
- 1M/day → ~12/sec (peak 100/sec)
- Email: 500K/day (SendGrid, SES, Mailgun)
- SMS: 200K/day (Twilio, Vonage)
- Push: 300K/day (FCM)
- Webhook retries: 3 attempts, exponential backoff

**Data Model:**

```json
// Notification
{
  "id": "notif_abc123",
  "user_id": "usr_xyz",
  "channel": "email",
  "template_id": "tpl_welcome",
  "template_data": {"name": "Alice"},
  "status": "pending",  // pending | sent | delivered | failed | bounced
  "channel_message_id": "sendgrid_msg_id",
  "created_at": "...",
  "sent_at": "..."
}

// Template
{
  "id": "tpl_welcome",
  "channels": {
    "email": {"subject": "Welcome {{name}}!", "body": "Hi {{name}},..."},
    "sms": {"body": "Welcome {{name}}!"},
    "push": {"title": "Welcome {{name}}", "body": "Glad to have you!"}
  },
  "version": 3
}
```

**Key Decisions:**
1. **Queue-based processing:** Notifications go into a SQS/RabbitMQ/Redis queue per channel. Workers consume and send.
2. **Template rendering server-side:** Store templates in DB/cache, render with Go templates or Laravel Blade.
3. **Channel-specific rate limits:** Track sent count per `{user_id, channel, minute}` in Redis. Use token bucket for SMS (expensive, rate-limited by provider).
4. **Webhook delivery:** When notification status changes, push to registered webhook URL. Use transactional outbox: write event to `webhook_events` table in the same transaction as notification status update.
5. **Retry with dead letter:** Failed sends go to a retry queue (up to 3 retries), then dead letter queue for manual inspection.
6. **Bounce handling:** Parse provider callbacks (SES bounce notification, Twilio delivery receipts) and update status.

**API Surface:**

```http
POST /v1/notifications/send
Content-Type: application/json
Authorization: Bearer <token>

{
  "user_id": "usr_xyz",
  "channels": ["email", "push"],
  "template_id": "tpl_welcome",
  "template_data": {"name": "Alice"},
  "metadata": {"source": "signup_flow"}
}

→ 202 Accepted
{
  "notification_id": "notif_abc123",
  "status": "pending"
}
```

```http
GET /v1/notifications/notif_abc123
→ 200 OK
{"id": "notif_abc123", "status": "delivered", "channel": "email", "sent_at": "..."}
```

```http
POST /v1/users/usr_xyz/preferences
Content-Type: application/json

{"channels": {"email": true, "sms": false, "push": true}}
```

**Trade-offs:**
- Queue per channel vs single queue: Per-channel isolates failures (SMS provider down doesn't block email) but requires more infrastructure. Single queue is simpler.
- Template in app code vs DB: DB templates allow non-deploy updates but add latency. Version templates for backward compatibility.
- Sync vs async: Always async for notifications — users shouldn't wait for SMS delivery to get an API response.

---

### Design 6: Trading Platform API

**Requirements:**
- Place limit/market orders
- Order matching engine (price-time priority)
- Real-time order book updates via WebSocket
- Idempotency support (client-generated order IDs)
- High concurrency: 20K DAU, 500 orders/sec peak
- Concurrency control to prevent overselling

**Estimation:**
- 20K DAU, active during market hours
- 500 orders/sec peak, 100K orders/day
- Order book: 100-500 price levels per symbol
- WebSocket: 10K concurrent connections

**Data Model:**

```json
// Order
{
  "order_id": "ord_abc123",       // client-generated (idempotency key)
  "user_id": "usr_xyz",
  "symbol": "AAPL",
  "side": "buy",                  // buy | sell
  "type": "limit",                // market | limit
  "price": 15000,                 // cents ($150.00)
  "quantity": 100,
  "filled_quantity": 0,
  "status": "open",               // open | partially_filled | filled | cancelled | rejected
  "created_at": "...",
  "version": 1
}

// Trade (result of matching)
{
  "trade_id": "trd_abc123",
  "buy_order_id": "ord_buy_123",
  "sell_order_id": "ord_sell_456",
  "symbol": "AAPL",
  "price": 15000,
  "quantity": 100,
  "executed_at": "..."
}
```

**Key Decisions:**
1. **Idempotency via client-generated IDs:** The `order_id` is a UUID generated by the client. On duplicate, return the existing order (idempotent). Use `INSERT ... ON CONFLICT DO NOTHING` or unique constraint.
2. **Order matching:** In-memory order book per symbol (sorted maps: bids DESC, asks ASC). Execute matching synchronously within a mutex per symbol. Write trades to DB asynchronously.
3. **Concurrency control:** Per-symbol mutex/lock. No two matching operations run simultaneously for the same symbol. Use Redis Redlock or a sharded mutex across instances.
4. **WebSocket push:** After order match, push updated order book and trade confirmation to relevant users via WebSocket. Fan-out per user connection.
5. **Balance reservation:** Before placing an order, reserve funds (buyer) or lock shares (seller). Use optimistic locking on user balances.
6. **Order types:** Limit orders go to the order book. Market orders match immediately against available liquidity.

**API Surface:**

```http
POST /v1/orders
Idempotency-Key: "<client-generated-order-id>"
Content-Type: application/json
Authorization: Bearer <token>

{
  "symbol": "AAPL",
  "side": "buy",
  "type": "limit",
  "price": 15000,
  "quantity": 100
}

→ 201 Created
{
  "order_id": "ord_abc123",
  "symbol": "AAPL",
  "status": "open",
  "filled_quantity": 0
}
```

```http
DELETE /v1/orders/ord_abc123
→ 204 No Content
```

```http
GET /v1/orders?symbol=AAPL&status=open
→ 200 OK
{"data": [...], "pagination": {"cursor": "...", "has_more": false}}
```

**WebSocket events:**
```json
// Trade execution
{"type": "trade", "data": {"order_id": "ord_abc123", "price": 15000, "quantity": 50}}

// Order book update
{"type": "orderbook", "symbol": "AAPL", "bids": [[15000, 200], [14990, 500]], "asks": [[15010, 300]]}

// Balance update
{"type": "balance", "data": {"available": 500000, "reserved": 100000}}
```

**Trade-offs:**
- In-memory matching vs DB matching: In-memory is 1000x faster but requires recovery on restart (WAL or snapshot). DB matching is durable but slower.
- Per-symbol mutex vs distributed lock: Per-symbol is simple and fast for a single instance. Distributed lock (Redis Redlock) needed for multi-instance deployment.
- REST + WebSocket vs WebSocket-only: REST for CRUD operations (order history, balance), WebSocket for real-time events. Separation of concerns.
- Optimistic vs pessimistic concurrency: Optimistic locking on balance (version column) works for low contention. Pessimistic for high contention symbols.

---

## 6. Debugging Scenarios

### Scenario 1: API Returning 500 Under Load

**Report:** "Our API returns 500 errors randomly during peak hours (2 PM - 3 PM). When testing alone with the same payload, it works fine. The 500 responses have no error body, just a generic HTML page."

**Possible causes:**

1. **Rate limit exhaustion at the upstream DB connection pool:**
   - Connection pool max_connections hit under load
   - New requests wait for a connection, timeout, throw a connection error
   - The web server (PHP-FPM, Go HTTP server) returns 500 when the DB connection fails
   - **Fix:** Increase pool size, add connection queue with timeout, monitor `pg_stat_activity` or `SHOW PROCESSLIST`

2. **Thread/goroutine exhaustion:**
   - PHP-FPM max_children too low → requests queue, timeout, 500
   - Go: all goroutines blocked on IO, no workers available
   - **Fix:** Increase workers, add request queuing with bounded wait

3. **Upstream service timeout:**
   - A downstream service (cache, search, auth) becomes slow under load
   - The API's HTTP client times out → 500
   - **Fix:** Add circuit breaker, reduce timeouts, increase upstream capacity

4. **Memory exhaustion:**
   - Large payloads or inefficient queries consume all PHP memory (memory_limit)
   - Go: heap grows unbounded, GC pressure, OOM killed
   - **Fix:** Profile memory, add limits per request, enable swap/monitoring

**Diagnosis steps:**
- Check `php_error_log` or Go pprof
- Monitor connection pool metrics during peak
- Enable request logging with timing breakdown
- Test with `ab` or `wrk` to reproduce under load
- Look at nginx error log for upstream timeouts

**Production fix:** Connection pool with proper sizing, request timeouts, circuit breaker for downstream services, and autoscaling.

---

### Scenario 2: Intermittent 401 Errors

**Report:** "Clients report occasional 401 Unauthorized errors. The token looks valid. The error comes and goes. It's more frequent around the top of the hour."

**Root cause analysis:**

1. **JWT clock skew:**
   - The server and client/system clocks are out of sync
   - JWT `iat` (issued at) or `exp` (expiry) validation fails if clock skew exceeds tolerance
   - **Fix:** Accept a clock skew window (e.g., `$leeway = 60` in PHP Firebase JWT). Use NTP on all servers.

2. **Token refresh race condition:**
   - Two concurrent requests: one with a just-refreshed token, one with an about-to-expire token
   - The refresh response arrives slightly after the expired token request
   - **Fix:** Implement a retry mechanism on 401: if token expired, refresh and retry. Use a token refresh lock to prevent concurrent refreshes.

3. **Load balancer hashing:**
   - Authentication state stored in local memory on one instance
   - Request routed to a different instance where the token isn't cached
   - **Fix:** Use a shared Redis session cache or validate JWT offline (no server-side state needed).

4. **Top-of-hour cron jobs:**
   - Scheduled tasks (cache clear, token revocation list refresh) run at :00
   - During the refresh window, some valid tokens are rejected
   - **Fix:** Stagger cron jobs, use atomic cache updates, add monitoring during transitions.

**Client-side mitigation:**
```javascript
async function fetchWithRetry(url, options) {
    const res = await fetch(url, options);
    if (res.status === 401 && options.retry !== false) {
        await refreshToken();
        return fetchWithRetry(url, {...options, retry: false});
    }
    return res;
}
```

---

### Scenario 3: Webhook Deliveries Failing for One Customer

**Report:** "Webhook deliveries to customer X fail consistently. Other customers receive webhooks fine. Customer X reports no changes to their endpoint."

**Checklist:**

1. **Payload too large:**
   - Customer's endpoint has a request body size limit (e.g., 1MB)
   - Your webhook payload exceeds it (e.g., large order with many line items)
   - **Fix:** Trim payload (don't send full order details — send summary + link). Check HTTP status (413).

2. **Timeout:**
   - Customer's endpoint is slow (> 5 seconds)
   - Your webhook client times out and marks delivery as failed
   - **Fix:** Log the exact error (timeout vs connection refused). Consider async processing with longer timeout for this customer.

3. **IP allowlist change:**
   - Customer added IP allowlisting but didn't include your new webhook source IPs
   - Your webhook service migrated to new IPs without notice
   - **Fix:** Check connection refused vs connection timeout. Provide customer with current IP ranges. Add to documentation.

4. **TLS/certificate issue:**
   - Customer's TLS certificate expired or is self-signed
   - Your client enforces strict TLS verification
   - **Fix:** Check TLS error in logs. Ask customer to renew or update certificate.

5. **Rate limiting by customer:**
   - Customer's endpoint rate-limits incoming requests
   - Your webhook batch creates a burst that exceeds their limit
   - **Fix:** Add jitter/delay between webhooks. Check customer endpoint's rate limit headers.

**Diagnosis script idea:**
```bash
# Test connectivity, TLS, and body size
curl -v -X POST "https://customer.example.com/webhook" \
  -H "Content-Type: application/json" \
  -d "$(cat sample_payload.json)" \
  --connect-timeout 5 --max-time 10
```

---

### Scenario 4: Pagination Returning Duplicate/Skipped Items

**Report:** "Our offset-based pagination (`?page=2&limit=20`) sometimes shows items that were on page 1, and sometimes skips items entirely. This happens when users are actively adding/deleting items."

**Root cause:** Offset pagination is unstable under write load.

**Example:**
```
Page 1: SELECT * FROM items ORDER BY id LIMIT 20 OFFSET 0
  → items 1, 2, 3, ..., 20

User inserts item at position 5 (id=100)

Page 2: SELECT * FROM items ORDER BY id LIMIT 20 OFFSET 20
  → items 21-40, BUT item 100 is now the 21st item in the total ordering
  → The old item 20 shifted to position 21
  → Result: item 20 appears on page 2 (DUPLICATE) and the last item (40) falls off (SKIP)
```

**Solutions:**

1. **Switch to cursor-based pagination:**
```sql
SELECT * FROM items WHERE id > :last_id ORDER BY id LIMIT 20
```
Stable — no duplicates or skips. Use this for real-time data.

2. **Use keyset pagination with a stable sort:**
```sql
SELECT * FROM items
WHERE (created_at, id) > (:last_created_at, :last_id)
ORDER BY created_at, id
LIMIT 20
```

3. **Snapshot/consistent view:**
   - Take a snapshot of IDs before paginating: `SELECT id FROM items ORDER BY id` (cache for 30 seconds)
   - Paginate using the snapshot
   - Trade-off: stale data, memory overhead

4. **If offset is required** (numbered pages for UX):
   - Freeze the total count and ordering
   - Accept that pagination may be slightly stale during writes
   - Add a note: "Results may shift during active editing"

---

### Scenario 5: CORS Errors After API Version Deployment

**Report:** "After deploying v2 of our API, our SPA users started seeing CORS errors. v1 still works fine. v2 is deployed on a new subdomain (`api-v2.example.com`)."

**Diagnosis:**

1. **Different origin:**
   - v1: `https://api.example.com`
   - v2: `https://api-v2.example.com`
   - The browser sees a cross-origin request from the SPA origin to `api-v2.example.com`
   - If CORS headers aren't set on v2, the browser blocks the request

2. **Preflight changes:**
   - v2 added a new custom header (`X-API-Version`) or changed content type
   - The browser now sends a preflight OPTIONS request (because non-simple request)
   - v2's OPTIONS handler isn't configured to respond with proper CORS headers

3. **Missing `Access-Control-Allow-Origin`:**
   - v2 server doesn't include the CORS header in responses
   - v1 had a middleware that added CORS headers; v2 forgot to include it

4. **Wildcard origin with credentials:**
   - v1: `Access-Control-Allow-Origin: *`
   - v2 needs `credentials: include` (cookies/auth), but `*` doesn't work with credentials
   - Browser rejects the response

**Fix:**
- Ensure all CORS middleware is ported to v2
- Add explicit origin matching (not `*`) if credentials are used
- Handle OPTIONS preflight for any new custom headers
- Test with `curl -H "Origin: https://app.example.com" -H "Access-Control-Request-Method: POST" -X OPTIONS https://api-v2.example.com/endpoint`
- Use a standardised CORS middleware (Laravel: `fruitcake/laravel-cors`, Go: `rs/cors`)

---

## 7. STAR Stories

### Story 1: API Redesign for Multi-Tenant SaaS

**Situation:** Our SaaS product had grown organically over 4 years. Each team built APIs differently — inconsistent error formats, no versioning, offset pagination mixed with cursor, some endpoints used snake_case, others camelCase. As we grew to 500+ tenants, integration partners complained about the lack of consistency.

**Task:** Lead the API redesign to create a unified, versioned, consistently designed REST API — without breaking existing integrations.

**Action:**
- Audited all 40+ existing endpoints and documented inconsistencies
- Published an API Style Guide (error format, naming, pagination, versioning)
- Chose URI versioning (`/v1/`, `/v2/`) — explicit and cache-friendly
- Implemented a standard error envelope with machine-readable codes (RFC 7807-inspired)
- Standardised cursor-based pagination for all collection endpoints (consistent `Link` header + response envelope)
- Created an OpenAPI spec for the unified API — used Spectral for linting in CI
- Built a compatibility layer: v1 endpoints internally called v2 handlers with response transformation
- Ran both versions in parallel for 6 months with a deprecation sunset header on v1
- Monitored adoption via analytics dashboard; 90% migration in 3 months

**Result:**
- Integration time for new partners dropped from 2 weeks to 3 days
- Error rates decreased by 40% (consistent error codes meant fewer client-side crashes)
- Support tickets related to API issues dropped by 60%
- API governance became part of the review process — every new endpoint must pass OpenAPI linting

**Key takeaway:** A clear style guide + tooling (OpenAPI, Spectral) + a migration path (not a big bang) was critical to adoption.

---

### Story 2: Idempotency Implementation for Trading Platform

**Situation:** Our trading platform processed 20K DAU with 500 orders/sec. Occasionally, network timeouts caused the mobile app to retry orders. Without idempotency, some users were charged twice for the same order — a critical financial error. Each double-charge cost ~$50 in support + goodwill.

**Task:** Design and implement an idempotency system that guarantees exactly-once order placement at high concurrency without adding latency to the critical path.

**Action:**
- Client-side: Mobile/web SDKs generate a UUID `client_order_id` and include it as a header (`Idempotency-Key`)
- Server-side:
  - PostgreSQL unique constraint on `client_order_id` in the orders table — the primary idempotency guarantee
  - Redis cache with 24h TTL: `{client_order_id → order_response}` — serves as fast path (avoid DB query on retry)
  - Write path: try INSERT → ON CONFLICT client_order_id DO NOTHING → return existing order
  - Race condition guard: `pg_advisory_xact_lock(hash(client_order_id))` within the critical section
- For balance operations: use optimistic locking (version column on user balance) within the same transaction
- For order matching: per-symbol mutex ensures atomic matching

**Result:**
- Zero double-charge incidents since deployment (6+ months)
- Idempotency key lookup: ~500µs (Redis) + ~2ms (DB fallback)
- 99.99% of retries resolved via Redis, never hitting the DB duplicate path
- Became the pattern for all write operations in the platform

**Key takeaway:** Dual-write (DB constraint + Redis cache) with transactional locking prevents races. Idempotency isn't just about not creating duplicates — it's about returning the **same response** on replay so clients can safely retry.

---

### Story 3: Webhook System for Inventory Sync Integrations

**Situation:** Our multi-tenant inventory SaaS needed to sync stock changes to partner systems (Shopify, Amazon, ERPs). Partners used webhooks, but we lacked a reliable delivery system — 15% of webhook deliveries failed, causing inventory discrepancies worth thousands of dollars per week.

**Task:** Build a reliable webhook delivery system with 99.9% delivery success, retries with backoff, signature verification, and a dashboard for monitoring.

**Action:**
- **Event sourcing:** Inventory changes were written to an `inventory_events` table (event type, entity ID, payload, timestamp). This became the source of truth.
- **Transactional outbox:** Writing inventory change + writing webhook event were in the same DB transaction. A CDC stream (Debezium → Kafka) picked up events for delivery.
- **Delivery worker:** Go service with configurable concurrency. Each webhook URL had its own queue to prevent a slow partner from blocking others.
- **Retry logic:** Exponential backoff: 10s → 1m → 10m → 1h → 6h → 24h. Max 5 retries in 72 hours. Dead letter after.
- **Signature:** HMAC-SHA256 with shared secret. Included `timestamp` in signed payload to prevent replay.
- **Dashboard:** Delivery success rate, retry count, last delivery timestamp per partner. Manual retry button.
- **Idempotency:** Each event had a unique `event_id`. Partners were asked to deduplicate by event_id.

**Result:**
- Webhook delivery success rate: 99.97% (from 85%)
- Inventory discrepancies reduced by 95%
- Average delivery latency: 200ms (p95 under 2s)
- Partners could self-serve via the dashboard, reducing support tickets by 200/month

**Key takeaway:** A webhook system is only as reliable as its weakest link — queue design, retry logic, and monitoring all matter. The transactional outbox pattern was the key to not losing events during crashes.

---

## 8. Questions to Ask the Interviewer

1. **"How does the team approach API versioning and deprecation? Is there a formal sunset policy?"**
   — Reveals their maturity level around backward compatibility.

2. **"What API governance do you have? Is there a style guide, OpenAPI linting in CI, or an API review process?"**
   — Shows if they've formalised quality control or if it's organic.

3. **"How do you handle idempotency for payment or order operations?"**
   — Critical for any fintech/e-commerce company. Reveals their reliability mindset.

4. **"What's the most challenging API production incident you've dealt with in the past year?"**
   — Learn about their operational maturity and blameless postmortem culture.

5. **"How do you ensure backward compatibility when evolving the API?"**
   — Understand their API-first vs code-first approach.

6. **"What's your approach to API testing — contract testing, integration tests, or end-to-end?"**
   — Reveals testing maturity. Look for mention of Pact (contract testing) or at least comprehensive integration tests.

7. **"How are webhooks delivered — what guarantees do you provide (at-least-once, exactly-once, ordering)?"**
   — Important if the role involves integrations.

8. **"What does the API gateways landscape look like here? How do you handle auth, rate limiting, and routing?"**
   — Understand their infrastructure. Kong? AWS API Gateway? Custom?

9. **"How do you handle API deprecation when a breaking change is unavoidable?"**
   — Their answer will tell you if they've actually done this or if they're winging it.

10. **"What's the ratio of API designers to API consumers? How do you gather feedback from API users?"**
    — Shows if they treat APIs as products or as afterthoughts.

---

## 9. Red Flags to Avoid

1. **"We don't version our APIs — just add fields and no one complains."**
   - Naive. Field additions are fine, but not all changes can be additive. No versioning = eventual pain.

2. **"We use 200 OK for everything, even errors. The status is in the response body."**
   - HTTP status codes exist for a reason. This breaks tooling, caches, and proxies.

3. **"Authentication is via a shared API key in the URL query string."**
   - Secrets in URLs are logged by proxies, leak in referrer headers, and appear in browser history. Use `Authorization` header.

4. **"Pagination? We just return everything. The clients can filter on their end."**
   - Unbounded responses will crash clients and your server. Any scale > 100 items needs pagination.

5. **"We don't have rate limits — our users are trusted."**
   - Famous last words. Every public API needs rate limiting. Even internal ones.

6. **"Webhooks are fire-and-forget. If the delivery fails, we log it and move on."**
   - Unreliable webhooks destroy trust. At minimum, retry with backoff and alert on failure.

7. **"Our API downtime is announced on our Twitter/X account."**
   - API status should have a dedicated status page, not a social media account. Expect SLAs with real uptime guarantees.

8. **"We have one monolith API. Microservices add complexity."**
   - Valid point, but if the monolith can't be independently deployed or scaled, it's a red flag. Look for at least modular monolith architecture.

9. **"We use REST but our endpoints are `POST /getUser` and `POST /createUser`."**
   - RPC in disguise. They don't understand REST fundamentals. Nit-picky but telling for a senior role.

10. **"The API documentation is the code. Just read the source."**
    - No. OpenAPI documentation should exist. "Code as documentation" does not work for external consumers.

---

*End of REST API Question Bank*
