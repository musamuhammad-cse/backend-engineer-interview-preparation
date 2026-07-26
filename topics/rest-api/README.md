# REST API — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS (Laravel API), 20K+ DAU trading platform (trading API), Chronos (gRPC job scheduler)

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
| [`01-basic.md`](./01-basic.md) | REST principles & constraints, HTTP methods, status codes, headers, content negotiation, URL design, idempotence & safety, differences from RPC/SOAP/GraphQL, request/response structure | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | API versioning strategies, pagination (offset vs cursor), filtering/sorting, error handling patterns, authentication (JWT, API keys, OAuth2), rate limiting, caching (ETag, Cache-Control), CORS, OpenAPI/Swagger documentation | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | HATEOAS & hypermedia, backward compatibility strategies, idempotency patterns (Idempotency-Key), bulk operations & batch APIs, webhooks (delivery guarantees, retry, signing), API gateways, GraphQL vs REST tradeoffs, performance optimization, OWASP API security, contract testing (Pact), API governance & consistency | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 160+ interview questions, code puzzles, system design prompts (URL shortener, rate limiter, payment API, notification service, inventory API), debugging scenarios, STAR stories | Ongoing drill |

---

## Coverage map

### REST fundamentals
- REST architectural constraints: stateless, client-server, cacheable, uniform interface, layered system, code on demand
- HTTP methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS — semantics, idempotence, safety
- HTTP status codes: 1xx, 2xx, 3xx, 4xx, 5xx — when to use each
- Request/response structure: headers (Content-Type, Accept, Authorization, Cache-Control), body formats (JSON, XML, Protobuf)
- Content negotiation: `Accept` header, media types, vendor-specific media types
- URL design: resources (plural nouns), nesting, naming conventions, kebab-case vs snake_case
- Differences from RPC, SOAP, GraphQL — trade-offs for each

### API design patterns
- Versioning: URL path (`/v1/`), header (`Accept: application/vnd.api+json;version=1`), query param
- Pagination: offset/limit vs cursor-based vs keyset — trade-offs
- Filtering, sorting, field selection: query parameter conventions
- Error handling: consistent error response format, problem details (RFC 7807), error codes, validation errors
- Authentication: JWT (structure, signing, expiry), API keys, OAuth 2.0 flows, Bearer tokens
- Rate limiting: fixed window, sliding window, token bucket, leaky bucket — per user/IP/tenant
- Caching: Cache-Control, ETag, Last-Modified, conditional requests (If-None-Match, If-Modified-Since)
- CORS: origins, methods, headers, preflight, credentials

### Advanced REST
- HATEOAS: hypermedia as the engine of application state, discovery, Richardson Maturity Model (Level 3)
- Backward compatibility: Postel's Law, tolerance, additive changes, deprecation policies
- Idempotency: Idempotency-Key header, replay detection, idempotency key scope and TTL
- Bulk operations: batch endpoints, async processing, webhook callbacks
- Webhooks: delivery guarantees (at-least-once), idempotency, retry with backoff, signing (HMAC), replay prevention
- API gateways: rate limiting, auth, transformation, aggregation, circuit breaking
- GraphQL vs REST: when each excels, hybrid approaches
- Performance: response compression, connection keep-alive, HTTP/2, HTTP/3
- Security: OWASP API Top 10, injection, broken auth, excessive data exposure, mass assignment, SSRF

### API operations
- Documentation: OpenAPI/Swagger, API Blueprint, Stoplight, Postman collections
- Contract testing: Pact (consumer-driven contracts), provider verification
- API versioning and deprecation: sunset headers, migration guides, parallel running
- Monitoring: API metrics (latency, error rate, request count), SLIs/SLOs, dashboards
- API governance: naming conventions, consistency checks, linting (spectral)

---

## Study order recommendation

REST API design is a universal backend skill. The Basic tier covers HTTP fundamentals that even senior engineers get wrong under pressure (idempotence vs safety, when to use 201 vs 200 vs 204). Do not skip it.

```
Week 1:  01-basic.md       + Basic Q&A drill
Week 2:  02-intermediate.md + Intermediate Q&A drill
Week 3:  03-senior.md       + Senior Q&A drill
Week 4+: 04-question-bank.md daily drill + design prompts
```

**Next topic in skill order:** Databases (PostgreSQL, MySQL, indexing, transactions, isolation levels).
