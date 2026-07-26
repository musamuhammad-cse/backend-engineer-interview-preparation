# REST API — Tier 2: Intermediate (API Design Patterns & Practices)

> **Target:** Senior Backend Engineer — 8+ years, PHP/Laravel, Go, JS  
> **Prerequisite:** Tier 1 (Basic) — HTTP methods, status codes, REST constraints, URL design  
> **Study mode:** Active recall. Read → Close → Explain aloud → Type examples from memory → Answer Q&A

This tier covers the real-world API design decisions you'll debate in system design interviews and implement daily: how to version without angering clients, paginate without melting your database, authenticate without introducing vulnerabilities, and document without lying. Every section identifies the traps that separate engineers who've read about APIs from engineers who've built them at scale.

---

## Table of Contents

1. [API Versioning](#1-api-versioning)
2. [Pagination](#2-pagination)
3. [Filtering, Sorting & Field Selection](#3-filtering-sorting--field-selection)
4. [Error Handling](#4-error-handling)
5. [Authentication](#5-authentication)
6. [Rate Limiting](#6-rate-limiting)
7. [Caching](#7-caching)
8. [CORS](#8-cors)
9. [OpenAPI / Swagger](#9-openapi--swagger)
10. [Tier 2 Q&A Drill](#10-tier-2-qa-drill)

---

## 1. API Versioning

### Why version

APIs evolve. Clients depend on contract stability. A breaking change — renaming a field, changing a response type, removing an endpoint — must not silently break integrations that ran fine yesterday.

**Breaking changes that require a version bump:**

- Removing or renaming a field/property
- Changing a field's type (string → object)
- Changing the semantics of an existing endpoint (e.g., `POST /orders` now also sends email)
- Removing an endpoint
- Changing error response structure
- Adding a new required request field

**Non-breaking changes (version bump NOT required):**

- Adding a new field to a response (clients ignore unknown fields)
- Adding a new endpoint
- Extending an enum with new values
- Making response times faster
- Adding an optional request field

### Strategies

#### 1. URL Path Versioning (`/v1/`, `/v2/`)

```
GET /v1/users/123
GET /v2/users/123
```

**Pros:**
- Explicit and visible — impossible to miss which version is used
- Easy to route at the gateway/load balancer level
- Simple to test and debug (copy-paste a URL)
- Works with any HTTP client, no special header handling

**Cons:**
- URL pollution — the resource identity changes across versions (REST purists argue `/v1/users/123` and `/v2/users/123` are different resources, not the same resource at different versions)
- Can encourage lazy version bumps for minor changes
- Caching: `/v1/users/123` and `/v2/users/123` are cached separately (pro or con depending on perspective)
- Default-version problem: what does `GET /users/123` return?

```http
GET /v1/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com"
}
```

#### 2. Header Versioning (Accept header / Custom header)

```
GET /users/123
Accept: application/vnd.myapp+json;version=1
```

Or a custom header:
```
GET /users/123
X-API-Version: 1
```

**Pros:**
- URL stays clean — resource identity is preserved
- Content negotiation is the HTTP-correct way (media type parameter)
- Gateway can inspect headers and route

**Cons:**
- Invisible in browser dev tools unless headers are explicitly shown
- Harder to test and debug (curl needs `-H` flag every time)
- Clients must implement header logic
- Default-version logic still needed when no version header is sent
- Caching: without careful Vary header setup, caches may serve wrong version

```http
GET /users/123 HTTP/1.1
Host: api.example.com
Accept: application/vnd.myapp+json;version=1

HTTP/1.1 200 OK
Content-Type: application/vnd.myapp+json;version=1

{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com"
}
```

#### 3. Query Parameter Versioning

```
GET /users/123?version=1
```

**Pros:**
- Trivial to implement on the server side
- Easy for clients to use (just add a query param)
- Visible in logs/analytics by default

**Cons:**
- Query params are ignored by some caches
- Version can be bookmarked, shared, and accidentally propagated
- Pollutes the URL for every request
- Poor HTTP semantics — version is part of resource identification, not a request modifier
- Conflicts with other query parameters in complex URLs

### Comparison Table

| Criterion | URL Path | Header | Query Param |
|---|---|---|---|
| Visibility | Excellent | Poor | Good |
| REST purity | Low | High | Low |
| Cache-friendliness | Good | Needs Vary | Poor |
| Client complexity | Low | Medium | Low |
| Gateway routing | Trivial | Requires inspection | Trivial |
| Browser testing | Easy | Cumbersome | Easy |
| Default version needed | Yes | Yes | Yes |

### Deprecation Policy

Versioning without a deprecation policy is not versioning — it's chaos.

**Sunset header:**

```http
HTTP/1.1 200 OK
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Deprecation: true
```

- `Sunset`: the RFC 8594 header — tells clients when the version will be removed
- `Deprecation`: signals this version is deprecated but still functional

**Migration workflow:**

```
T0:  Announce v2 beta, v1 still stable
T1:  Ship v2 GA, v1 fully supported
T2:  Mark v1 deprecated (add Deprecation header), announce sunset date
T3:  Begin parallel run — both versions operational, monitor v2 adoption
T4:  404 all v1 requests, return 410 Gone with link to migration guide
```

**Parallel run period:** Run both versions simultaneously for a minimum of 3–6 months. Monitor v2 adoption before cutting v1.

```json
{
  "error": {
    "code": "VERSION_DEPRECATED",
    "message": "API version 1 is deprecated. Please migrate to v2.",
    "migration_url": "https://docs.example.com/migration-v1-to-v2",
    "sunset_date": "2025-12-31"
  }
}
```

### When to break backward compatibility

- Security vulnerability that requires a contract change
- Legal/compliance requirement (GDPR, data retention)
- The cost of maintaining the old contract exceeds the cost of breaking every client
- The old contract has zero active users (measure first — don't guess)

> **Trap:** Not versioning from day one. You ship v1 without a version prefix, then need to make a breaking change. Now every client breaks, or you're stuck with a monstrosity like `GET /users/123?include_profile=true` to work around your own API.
>
> **Trap:** Removing fields without a deprecation notice. You delete `display_name` from the response on Tuesday. By Wednesday, three mobile apps have crashed in production. A field removal must go through: DEPRECATE → warn → LOGGING → measure zero usage → REMOVE.
>
> **Trap:** Keeping N versions running forever. You now support v1, v2, v3, v4, and v5. Your test matrix explodes. Each endpoint change requires backporting or conditional logic. Set a firm sunset policy and enforce it. Three versions max. If clients won't migrate, offer a paid extended-support tier.

> **Follow-up:** "How would you handle a client who refuses to migrate off v1?" — Offer a paid extended-support window with SLA penalties. If they're paying enough, maintain a thin compatibility shim that translates v1 calls to v2 internally. Log their usage and put the migration cost on them.
>
> **Follow-up:** "What if you need to change a field type from string to array?" — Add a new field with the correct type, deprecate the old one. Run both until old field has zero reads. Example: `phone` (deprecated) and `phones` (array). A single-integer phone is a data modeling problem, not an API versioning problem.

---

## 2. Pagination

### Offset/Limit Pagination

```
GET /items?offset=0&limit=25
```

```http
GET /api/items?offset=0&limit=25 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [
    { "id": 1, "name": "Item 1" },
    { "id": 2, "name": "Item 2" }
  ],
  "pagination": {
    "offset": 0,
    "limit": 25,
    "total": 1542,
    "next_offset": 25,
    "prev_offset": null
  }
}
```

**Pros:**
- Intuitive — page numbers map directly: `page = offset / limit + 1`
- User can jump to any page: `?offset=500&limit=25`
- Total count gives users progress information
- Easy to implement with `OFFSET` / `LIMIT` in SQL

**Cons:**
- Performance degrades at depth: `OFFSET 100000 LIMIT 25` still reads 100025 rows
- Unstable under concurrent writes: inserting a row on page 1 shifts everything on page 2
- `SELECT count(*)` on large tables is expensive
- No guarantee of consistency across pages

### Cursor-Based Pagination

```
GET /items?cursor=eyJpZCI6MTAwfQ==&limit=25
```

```http
GET /api/items?cursor=eyJpZCI6MTAwfQ==&limit=25 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [
    { "id": 101, "name": "Item 101" }
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6MTI1fQ==",
    "prev_cursor": "eyJpZCI6OTl9",
    "limit": 25
  }
}
```

**Pros:**
- O(1) performance regardless of page depth — `WHERE id > :last ORDER BY id LIMIT 25` is index-only
- Stable under concurrent writes — new records don't shift existing pages
- No expensive `count(*)` needed
- Handles real-time data well (feeds, activity logs, messages)

**Cons:**
- No total count by default (can be provided as estimate)
- No arbitrary page jumps — user cannot go to page 50
- Forward-only unless bidirectional cursors are implemented
- Requires a unique, sortable column (usually the primary key or a timestamp + ID composite)
- Cursor encoding adds slight complexity

**Keyset pagination (the underlying mechanism):**

```sql
-- First page
SELECT * FROM items ORDER BY id ASC LIMIT 25;

-- Next page: WHERE id > last_id_of_previous_page
SELECT * FROM items ORDER BY id ASC
  WHERE id > :last_id
  LIMIT 25;

-- Previous page (reverse order + flip)
SELECT * FROM items ORDER BY id DESC
  WHERE id < :first_id_of_current_page
  LIMIT 25;
```

**Cursor encoding:**

```python
import base64, json

def encode_cursor(id, created_at):
    payload = json.dumps({"id": id, "created_at": created_at})
    return base64.urlsafe_b64encode(payload.encode()).decode()

def decode_cursor(cursor):
    payload = json.loads(base64.urlsafe_b64decode(cursor))
    return payload["id"], payload["created_at"]
```

### Comparison Table

| Property | Offset/Limit | Cursor-based |
|---|---|---|
| Performance at depth | Degrades linearly | O(1), constant |
| Stable under writes | No | Yes |
| Random page access | Yes | No (forward/backward only) |
| Total count | Available (expensive) | Not available (estimate or omit) |
| Implementation complexity | Trivial | Medium |
| Real-time data suitability | Poor | Excellent |
| Database index requirement | None | Required (ordered column) |
| REST pagination metadata | offset, limit, total | next_cursor, prev_cursor |

### Response Format Patterns

**Cursor-based with optional total:**

```json
{
  "data": [ ... ],
  "meta": {
    "total": 1542,
    "total_estimated": true
  },
  "links": {
    "next": "https://api.example.com/items?cursor=eyJpZCI6MTI1fQ==&limit=25",
    "prev": "https://api.example.com/items?cursor=eyJpZCI6OTl9&limit=25"
  }
}
```

**Offset-based with page info:**

```json
{
  "data": [ ... ],
  "meta": {
    "current_page": 1,
    "per_page": 25,
    "total_pages": 62,
    "total": 1542
  },
  "links": {
    "next": "https://api.example.com/items?page=2&per_page=25",
    "prev": null,
    "first": "https://api.example.com/items?page=1&per_page=25",
    "last": "https://api.example.com/items?page=62&per_page=25"
  }
}
```

> **Trap:** Using offset pagination for real-time feeds. A user scrolls a feed of notifications. Between page 1 and page 2, 10 new notifications arrive. The user sees the same notification on both pages and misses one entirely. Cursor pagination fixes this — the cursor anchors to the last item seen, not a row count.
>
> **Trap:** Exposing raw database IDs in cursors. `?cursor=124` lets users guess IDs and probe the API. Encode cursors with base64url (not base64 — URL-safe) and optionally include an HMAC signature to prevent tampering.
>
> **Trap:** Not handling empty results. The last page has 0 items and no cursor. Return `"next_cursor": null` (not empty string) so the client knows to stop. Missing this causes clients to loop forever requesting an empty cursor.

> **Follow-up:** "How do you handle pagination when the sort column has duplicate values?" — Use a composite cursor: `(timestamp, id)`. Even if two items have the same timestamp, the unique ID breaks the tie. The WHERE clause becomes: `WHERE (created_at, id) > (:last_ts, :last_id)`.
>
> **Follow-up:** "How would you provide total count for cursor pagination without a full table scan?" — Use an approximate count: `EXPLAIN SELECT count(*) ...` for an estimate, or maintain a separate counter in Redis that increments/decrements on inserts/deletes. Accept that this is eventually consistent.
>
> **Follow-up:** "What about backward pagination with cursors?" — Store both `next_cursor` and `prev_cursor` in the response. The `prev_cursor` encodes the first item on the current page. Query in reverse: `WHERE id < :first_id ORDER BY id DESC LIMIT 25`, then reverse the result order before returning.

---

## 3. Filtering, Sorting & Field Selection

### Filtering Conventions

**Option A: Simple query parameters**

```
GET /users?status=active&role=admin&created_after=2024-01-01
```

```http
GET /api/users?status=active&role=admin&created_after=2024-01-01&created_before=2024-12-31 HTTP/1.1
```

```sql
SELECT * FROM users
WHERE status = 'active'
  AND role = 'admin'
  AND created_at >= '2024-01-01'
  AND created_at <= '2024-12-31';
```

**Option B: Filter expression syntax**

```
GET /users?filter=status:eq:active,created_at:gte:2024-01-01
```

**Option C: JSON-encoded filter**

```
GET /users?filter={"status":"active","created_at":{"$gte":"2024-01-01"}}
```

**Option D: Nested query parameters**

```
GET /users?filter[status]=active&filter[created_at][gte]=2024-01-01
```

### Sorting

```
GET /users?sort=created_at    (ascending, default)
GET /users?sort=-created_at   (descending, prefix with minus)
GET /users?sort=-created_at,name  (multiple: desc by created_at, asc by name)
```

```http
GET /api/users?sort=-created_at,last_name HTTP/1.1
```

```sql
SELECT * FROM users
ORDER BY created_at DESC, last_name ASC;
```

### Field Selection (Sparse Fieldsets)

```
GET /users?fields=id,name,email
GET /users?fields=id,name,email&fields[profile]=avatar,bio
```

```http
GET /api/users/123?fields=id,name,email HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "name": "Alice Smith",
  "email": "alice@example.com"
}
```

Per-resource-type fieldsets (JSON:API style):

```
GET /api/articles?include=author&fields[articles]=title,body&fields[people]=name
```

### Range Filters

```
GET /orders?amount_min=100&amount_max=500
GET /orders?amount=100..500
GET /orders?date=2024-01-01..2024-12-31
```

### Text Search

```
GET /products?q=wireless+charger
GET /products?search=wireless+charger&search_fields=name,description
```

### Multi-Value Filters

```
GET /users?status=active,pending
GET /users?status[]=active&status[]=pending
GET /users?filter=status:in:(active,pending)
```

```sql
SELECT * FROM users
WHERE status IN ('active', 'pending');
```

### Standardizing Query Parameter Names

| Operation | Parameter | Example |
|---|---|---|
| Exact match | `field=value` | `status=active` |
| Range (gte) | `field_from` or `field_min` | `price_from=10` |
| Range (lte) | `field_to` or `field_max` | `price_to=100` |
| Greater than | `field_gt` | `age_gt=18` |
| Less than | `field_lt` | `age_lt=65` |
| Text search | `q` or `search` | `q=laptop` |
| Sort | `sort` | `sort=-created_at` |
| Fields | `fields` | `fields=id,name` |
| Include relations | `include` or `with` | `include=author` |
| Page number | `page` | `page=2` |
| Page size | `per_page` or `limit` | `limit=25` |
| Cursor | `cursor` | `cursor=eyJpZCI6MTB9` |

### Implementation Pseudocode

```python
def apply_filters(query, filters):
    allowed_filters = ["status", "role", "created_at"]
    for field, value in filters.items():
        if field not in allowed_filters:
            raise ValidationError(f"Filter '{field}' not allowed")
        # Validate value type matches schema
        validate_filter_value(field, value)
        query = query.filter(field, value)
    return query

def apply_sort(query, sort_param, allowed_columns):
    sorts = sort_param.split(",")
    for s in sorts:
        direction = "DESC" if s.startswith("-") else "ASC"
        column = s.lstrip("-")
        if column not in allowed_columns:
            return error(400, f"Cannot sort by '{column}'")
        query = query.order_by(column, direction)
    return query
```

> **Trap:** Not validating filter parameters — SQL injection through raw filters. Never concatenate filter values into SQL. Always use parameterized queries. Even with ORMs, watch out for raw `WHERE` fragments.
>
> **Trap:** Allowing sort on unindexed columns. A user sorts 10M records by `last_login` which has no index. Your database does a filesort and times out. Whitelist sortable columns. Add a database index for every allowed sort column.
>
> **Trap:** Field selection that returns sensitive data. A developer adds `fields=id,name,email,password_hash` to an admin endpoint. A mobile client requests `fields=id,name` but the server blindly includes all requested fields. Always enforce a field allowlist per endpoint — never pass user-requested field names directly to your serialization layer.

> **Follow-up:** "How do you handle filters on nested resources?" — JSON:API-style dotted paths: `?filter[author.name]=John` which maps to `JOIN authors ON ... WHERE authors.name = 'John'`. Whitelist which relationships can be filtered across to prevent expensive joins.
>
> **Follow-up:** "What if a client requests 200 fields?" — Enforce a maximum (e.g., 20 fields). Return 400 Bad Request or silently truncate. A wildcard request `?fields=*` should return only the default public fields, not every column in the database.
>
> **Follow-up:** "How do you handle OR filters?" — `?filter=(status:eq:active|status:eq:pending)` or use a comma-in format: `?status=active,pending`. Document the exact syntax clearly since it's less common.

---

## 4. Error Handling

### Consistent Error Response Format

**JSON:API Error Object:**

```json
{
  "errors": [
    {
      "id": "abc123-def456",
      "status": "422",
      "code": "VALIDATION_ERROR",
      "title": "Validation Failed",
      "detail": "The email field must be a valid email address.",
      "source": {
        "pointer": "/data/attributes/email"
      },
      "meta": {
        "rule": "email_format",
        "attempted_value": "not-an-email"
      }
    }
  ]
}
```

**RFC 7807 Problem Details:**

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "Too many requests. Please try again in 30 seconds.",
  "instance": "/api/users",
  "retry_after_seconds": 30
}
```

### Error Codes vs HTTP Status Codes

| Situation | HTTP Status | Error Code |
|---|---|---|
| Resource not found | 404 | `RESOURCE_NOT_FOUND` |
| Validation failed | 422 | `VALIDATION_ERROR` |
| Authentication required | 401 | `UNAUTHENTICATED` |
| Permission denied | 403 | `FORBIDDEN` |
| Rate limited | 429 | `RATE_LIMIT_EXCEEDED` |
| Internal server error | 500 | `INTERNAL_ERROR` |
| Service unavailable | 503 | `SERVICE_UNAVAILABLE` |
| Conflict (duplicate) | 409 | `CONFLICT` |
| Deprecated endpoint | 410 | `GONE` |

**Rule:** HTTP status codes cover the transport layer (what happened). Error codes cover the application layer (why it happened and what to do about it).

### Validation Errors

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body contains invalid fields.",
    "errors": [
      {
        "field": "email",
        "message": "must be a valid email address",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "age",
        "message": "must be at least 18",
        "code": "MIN_VALUE_EXCEEDED",
        "meta": { "min": 18 }
      },
      {
        "field": "tags",
        "message": "must contain at least 1 tag",
        "code": "REQUIRED"
      }
    ]
  }
}
```

### Error ID for Correlation

```json
{
  "error": {
    "id": "err_abc123xyz",
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred."
  }
}
```

Server-side log correlation:

```json
// Structured log entry
{
  "level": "ERROR",
  "error_id": "err_abc123xyz",
  "message": "Payment gateway timeout",
  "request_id": "req_xyz789",
  "user_id": "u_456",
  "stack_trace": "...",
  "duration_ms": 30123
}
```

### Development vs Production Error Detail

**Development (internal/staging):**

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "Database connection failed",
    "detail": "SQLSTATE[HY000] [2002] Connection refused",
    "trace": [
      "#0 /var/www/src/Repositories/UserRepository.php:42",
      "#1 /var/www/src/Controllers/UserController.php:88"
    ]
  }
}
```

**Production:**

```json
{
  "error": {
    "id": "err_abc123xyz",
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred. Please try again later."
  }
}
```

> **Trap:** Returning stack traces to clients in production. This leaks file paths, internal architecture, library versions, and sometimes credentials. Use an environment flag: `APP_DEBUG=true` for dev, never for production. Pro tip: some frameworks automatically hide details in production, but verify this — don't assume.
>
> **Trap:** Leaking information in error messages. "User not found" vs "Password incorrect" tells an attacker whether a username exists. Use a generic message: "Invalid credentials." Similarly, don't reveal why a payment failed: "Transaction declined" (not "Insufficient funds in account ending in 1234").
>
> **Trap:** Inconsistent error structure across endpoints. One endpoint returns `{"error": "not found"}`, another returns `{"errors": [{"code": 404}]}`, a third returns a bare HTML page. Define a single error schema. Enforce it with your API framework or a middleware. Test it with contract tests.

> **Follow-up:** "How would you handle errors from downstream services?" — Never propagate the downstream error directly. Wrap it in your own error format. Log the downstream error with a correlation ID. Return a generic 502/503 to the client with your own error code like `UPSTREAM_ERROR`.
>
> **Follow-up:** "Should validation errors use 400 or 422?" — 400 is generic "bad request." 422 (Unprocessable Entity) specifically means the request body is syntactically correct but semantically invalid. Use 422 for validation. Use 400 for malformed JSON or missing required headers.
>
> **Follow-up:** "How do you version error responses?" — Error responses are part of your contract. If you change the error format (e.g., from flat to nested), that's a breaking change. Put error schemas in your OpenAPI spec. Version them alongside your data responses.

---

## 5. Authentication

### JWT (JSON Web Token)

**Structure:**

```
header.payload.signature

header:  {"alg": "HS256", "typ": "JWT"}
payload: {"sub": "u_123", "iat": 1700000000, "exp": 1700003600, "iss": "api.example.com", "aud": "myapp"}
```

```json
// Header (decoded)
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "key-v2"
}

// Payload (decoded) — Registered claims
{
  "sub": "user_abc123",
  "iat": 1700000000,
  "exp": 1700086400,
  "iss": "https://auth.example.com",
  "aud": "inventory-api",
  "jti": "token_uuid_1234",
  "scope": "read:orders write:orders"
}
```

**Signing Algorithms:**

| Algorithm | Type | Key | Performance | Use Case |
|---|---|---|---|---|
| HS256 | Symmetric | Shared secret | Fast | Single-service, internal APIs |
| RS256 | Asymmetric | Private/public key pair | Slower | Multi-service, third-party verification |
| ES256 | Asymmetric (ECDSA) | Private/public key pair | Fast | Mobile, IoT (smaller token size) |

**Best Practices:**

```python
import jwt
import time

def create_access_token(user_id, secret, expiry_seconds=3600):
    now = int(time.time())
    payload = {
        "sub": user_id,
        "iat": now,
        "exp": now + expiry_seconds,
        "iss": "api.example.com",
        "aud": "inventory-api",
        "jti": str(uuid.uuid4()),
        "scope": "read:products write:products"
    }
    return jwt.encode(payload, secret, algorithm="HS256")

def verify_token(token, secret, expected_audience):
    try:
        payload = jwt.decode(
            token,
            secret,
            algorithms=["HS256"],
            audience=expected_audience,
            options={"require": ["exp", "iat", "sub", "aud"]}
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise AuthenticationError("Token expired")
    except jwt.InvalidAudienceError:
        raise AuthenticationError("Token audience mismatch")
    except jwt.InvalidTokenError:
        raise AuthenticationError("Invalid token")
```

**Short expiry + refresh tokens:**

```
Access token:  15 minutes (short-lived, limits damage if stolen)
Refresh token: 7 days (long-lived, used only to get new access tokens)
Refresh token rotation: each refresh issues a new refresh token, invalidates old one
```

```http
POST /oauth/token HTTP/1.1
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "r_abc123...",
  "client_id": "myapp"
}

HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "r_def456...",
  "scope": "read:products"
}
```

### API Keys

```http
GET /api/products HTTP/1.1
X-API-Key: sk_live_abc123def456

# Or header-based
Authorization: Bearer sk_live_abc123def456
```

**Pros:**
- Simple to implement and use
- No complex token exchange flow
- Easy to revoke (delete from database)

**Cons:**
- API key alone identifies the client but not the user
- Long-lived — stolen keys are dangerous
- No granular scoping without additional infrastructure
- No expiry by default — requires manual rotation

### OAuth 2.0 Flows

**Authorization Code + PKCE (third-party apps):**

```
Mobile App                          Auth Server                        API Server
    |                                    |                                |
    |--- Authorization Request --------->|                                |
    |<--- Authorization Code (redirect) -|                                |
    |--- Code + Code Verifier + PKCE --->|                                |
    |<--- Access + Refresh Token ---------|                                |
    |--- API Request (Bearer token) ------------------------------------>|
    |<--- Response -------------------------------------------------------|
```

```
// Authorization request
GET https://auth.example.com/authorize?
  response_type=code&
  client_id=myapp&
  redirect_uri=myapp://callback&
  code_challenge=E9Melhoa2Ow...&
  code_challenge_method=S256&
  scope=read:orders+write:orders&
  state=xyz789
```

```
// Token exchange
POST https://auth.example.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=auth_code_here
&redirect_uri=myapp://callback
&client_id=myapp
&code_verifier=verifier_from_step_1
```

**Client Credentials (machine-to-machine):**

```
POST https://auth.example.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=service_a
&client_secret=secret_here
&scope=read:inventory
```

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read:inventory"
}
```

**Device Code (CLI/headless):**

```
POST https://auth.example.com/oauth/device/code
Content-Type: application/x-www-form-urlencoded

client_id=cli-tool
&scope=read:profile
```

```json
{
  "device_code": "d_abc123",
  "user_code": "ABCD-1234",
  "verification_uri": "https://example.com/device",
  "interval": 5,
  "expires_in": 600
}
```

### Bearer Token Usage

```http
GET /api/orders HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Always over HTTPS.** A Bearer token in plain HTTP is game over.

> **Trap:** JWT in URL query parameters. They end up in server access logs, browser history, referrer headers, analytics tools, and CDN logs. Any of these surfaces leaks the token. Use the `Authorization: Bearer` header — it's what it's for.
>
> **Trap:** Not validating the `aud` (audience) claim. Service A issues a token meant for Service A's API. Service B blindly accepts it because it only checks `exp` and signature. Now the token is valid everywhere. Always validate that `aud` matches your service's identifier.
>
> **Trap:** Storing JWTs in localStorage. XSS vulnerability on any page of your domain gives an attacker access to every JWT stored in localStorage. Use `httpOnly` cookies for refresh tokens and keep access tokens in memory (SPA) or short-lived cookies.
>
> **Trap:** Long-lived tokens without revocation mechanism. A 30-day JWT cannot be revoked (unless you maintain a blocklist, defeating the purpose of stateless tokens). Use short expiry (15 min) + refresh tokens stored server-side so they can be revoked.

> **Follow-up:** "Why not always use RS256 over HS256?" — RS256 is slower to verify (asymmetric crypto). For high-throughput internal services where both parties share a secret, HS256 is faster and simpler. RS256 becomes critical when third parties need to verify tokens without sharing a secret.
>
> **Follow-up:** "How do you handle token revocation with JWTs?" — You can't revoke a JWT without a blocklist or short expiry. Real solution: short-lived access tokens (15 min), revocable refresh tokens stored server-side (Redis with TTL). Check refresh token validity on each refresh. On logout, delete the refresh token from Redis.
>
> **Follow-up:** "What's the difference between OAuth 2.0 and OpenID Connect?" — OAuth 2.0 is an authorization framework (delegates access). OpenID Connect is an authentication layer on top of OAuth 2.0 (verifies identity). OIDC adds the `id_token` (a JWT containing user identity claims) alongside the access token.

---

## 6. Rate Limiting

### Why Rate Limit

1. **Fair usage** — one noisy tenant shouldn't starve others
2. **Cost control** — each API call costs compute, database queries, third-party API calls
3. **Abuse prevention** — brute force attacks, DDoS, scraper bots
4. **Infrastructure protection** — prevent cascading failures during traffic spikes

### Algorithms

#### Fixed Window

```
Window: 60 seconds
Limit: 100 requests

Count keys in Redis:  api:rate:user_123:1700000000

00:00 - 00:59:  100 requests (allowed)
00:59 - 01:00:  0 requests (blocked)
01:00 - 01:59:  100 requests (new window)
```

```python
import time

def check_fixed_window(client_id, limit=100, window=60):
    window_key = int(time.time() / window) * window
    key = f"rate_limit:{client_id}:{window_key}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window)
    return count <= limit
```

**Pros:** Simple, low memory.  
**Cons:** Burst at window boundary — 100 requests at 00:59, 100 more at 01:00 (200 in 2 seconds).

#### Sliding Window (Log)

```
Track timestamps of each request within a rolling window.

At T+60s: discard timestamps older than T.
Count = requests with timestamp > (now - window).
```

```python
def check_sliding_window(client_id, limit=100, window=60):
    key = f"rate_limit:{client_id}:sliding"
    now = time.time()
    cutoff = now - window

    # Remove old entries
    redis.zremrangebyscore(key, 0, cutoff)

    # Count current entries
    count = redis.zcard(key)

    if count >= limit:
        return False

    # Add current request
    redis.zadd(key, {str(now): now})
    redis.expire(key, window)
    return True
```

**Pros:** Smooth, no boundary bursts.  
**Cons:** Uses more memory (stores timestamps), slightly slower.

#### Token Bucket

```
Capacity:  100 tokens
Fill rate: 10 tokens/second

Each request consumes 1 token.
If no tokens left, request is rejected.
Tokens accumulate up to capacity.
```

```python
def check_token_bucket(client_id, capacity=100, refill_rate=10):
    key = f"rate_limit:{client_id}:bucket"
    bucket = redis.hgetall(key)

    if not bucket:
        bucket = {"tokens": capacity, "last_refill": time.time()}
    else:
        tokens = float(bucket["tokens"])
        last_refill = float(bucket["last_refill"])
        elapsed = time.time() - last_refill
        new_tokens = min(capacity, tokens + elapsed * refill_rate)
        bucket = {"tokens": new_tokens, "last_refill": time.time()}

    if bucket["tokens"] >= 1:
        bucket["tokens"] -= 1
        redis.hset(key, mapping=bucket)
        return True

    return False
```

**Pros:** Handles bursts gracefully (up to capacity), smooth rate enforcement.  
**Cons:** More complex, clock-dependent.

#### Leaky Bucket

```
Requests enter a queue (bucket). They are processed at a fixed rate.
If the queue is full, requests are rejected (leaked out).
```

**Pros:** Smooth output rate, no bursting.  
**Cons:** Adds latency (requests are queued), more complex.

### Identifying Clients

| Identifier | Pros | Cons |
|---|---|---|
| IP address | Always available | Proxy/NAT hides real client; shared IPs (offices, colleges) |
| User ID/API key | Accurately identifies client | User may not be authenticated yet (login endpoint) |
| JWT `sub` claim | Tied to authenticated user | Requires token decode on every request |
| Tenant ID | Fair per-tenant distribution | Multi-tenant SaaS; one noisy user can exhaust tenant limit |

**Layered approach:**

```
Global rate limit:  10,000 req/min  (protect infrastructure)
Per-user rate limit: 100 req/min    (fair usage)
Per-endpoint rate limit: 10 req/min for /api/login (anti-brute-force)
```

### Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1700003600
```

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1700003600
Retry-After: 45

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retry_after_seconds": 45
  }
}
```

### Distributed Rate Limiting with Redis

```python
class DistributedRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def check(self, key_prefix, client_id, max_requests, window_seconds):
        key = f"{key_prefix}:{client_id}:{int(time.time() / window_seconds)}"
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, window_seconds)
        result = pipe.execute()
        return result[0] <= max_requests, result[0], max_requests

    def get_remaining(self, key_prefix, client_id, max_requests, window_seconds):
        key = f"{key_prefix}:{client_id}:{int(time.time() / window_seconds)}"
        count = int(self.redis.get(key) or 0)
        return max(0, max_requests - count)
```

### Lua Script for Atomicity

```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local current = redis.call("GET", key)
if current and tonumber(current) >= limit then
    return 0  -- rate limited
end

current = redis.call("INCR", key)
if current == 1 then
    redis.call("EXPIRE", key, window)
end

return limit - current  -- remaining
```

> **Trap:** Rate limiting by IP when behind a proxy. Your server sees `127.0.0.1` or the load balancer's IP, not the real client. Always read the `X-Forwarded-For` header (first IP in the chain) or `X-Real-IP`. Validate it — don't blindly trust user-supplied headers.
>
> **Trap:** Not returning `Retry-After`. A rate-limited client without knowing when to retry will retry immediately (wasting resources) or give up permanently. Always return `Retry-After` as seconds or HTTP-date.
>
> **Trap:** Rate limiting health check endpoints. `/health` and `/ready` endpoints are called by load balancers and orchestrators every few seconds. If rate-limited, the instance gets removed from rotation, causing a cascade failure.
>
> **Trap:** Per-API-key rate limits that break shared integrations. A company has 1 API key shared across 5 internal services. Each service thinks it has 100 req/min, but together they hit the limit after 20 req/min each. Solution: rate limit by user (across all keys) and educate clients about separate keys per service. Alternatively, offer team-level rate limits rather than per-key.

> **Follow-up:** "How do you handle rate limiting for an unauthenticated login endpoint?" — Use IP-based rate limiting with very low limits (5 req/min per IP). Add CAPTCHA after 3 failed attempts. Consider behavioral analysis (unusual geolocation, browser fingerprint) for additional signals.
>
> **Follow-up:** "How do you test rate limiting in development?" — Use environment-specific limits (dev: 1000 req/min, prod: 100 req/min). Add a test header `X-ByPass-RateLimit: true` that's stripped by the production gateway. Unit-test the algorithm itself with a mock Redis.
>
> **Follow-up:** "What happens when Redis goes down?" — Fall back to an in-memory approximate rate limiter (best-effort) or allow the request through with a warning log. Your rate limiter should fail open (allow traffic) rather than fail closed (block all traffic) during infrastructure failures. Monitor Redis health and alert immediately.

---

## 7. Caching

### Cache-Control Directives

```http
# Public cacheable resource (avatar image, product listing)
Cache-Control: public, max-age=3600, s-maxage=86400

# User-specific response (profile, orders)
Cache-Control: private, max-age=60

# No caching at all (auth status, live data)
Cache-Control: no-cache, no-store, must-revalidate

# Stale-while-revalidate pattern (serve stale, refresh async)
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

| Directive | Meaning |
|---|---|
| `public` | Any cache (CDN, proxy, browser) may cache |
| `private` | Only the browser may cache (no CDN/proxy) |
| `no-cache` | Must revalidate with origin before serving cached copy |
| `no-store` | Must not cache at all (privacy-sensitive) |
| `max-age=N` | Response is fresh for N seconds from origin time |
| `s-maxage=N` | Overrides max-age for shared caches (CDN) |
| `must-revalidate` | Once stale, must revalidate before serving |
| `stale-while-revalidate=N` | Serve stale content for N seconds while revalidating async |

### ETags for Conditional Requests

```http
# Request
GET /api/products/123 HTTP/1.1
If-None-Match: "abc123hash"

# Response if unchanged
HTTP/1.1 304 Not Modified
ETag: "abc123hash"

# Response if changed
HTTP/1.1 200 OK
ETag: "def456hash"
Content-Type: application/json

{ "id": 123, "name": "Product 123", "updated_at": "2024-06-15T10:00:00Z" }
```

**Weak vs Strong Validators:**

```
Strong ETag:  "abc123"  — byte-for-byte identical content
Weak ETag:    W/"abc123" — semantically equivalent but possibly byte-different (e.g., whitespace changes)

Weak ETags are useful when the representation may differ slightly (different JSON key ordering, whitespace)
but the data is semantically the same.
```

**Generating ETags:**

```python
import hashlib, json

def generate_etag(data, weak=False):
    # Deterministic JSON serialization (sorted keys, no whitespace)
    content = json.dumps(data, sort_keys=True, separators=(",", ":"))
    hash_val = hashlib.md5(content.encode()).hexdigest()
    return f'W/"{hash_val}"' if weak else f'"{hash_val}"'

# For large responses, hash a subset or use last_updated timestamp
def generate_etag_from_timestamp(updated_at):
    return f'"{int(updated_at.timestamp())}"'
```

### Last-Modified / If-Modified-Since

```http
GET /api/products/123 HTTP/1.1
If-Modified-Since: Tue, 15 Jun 2024 10:00:00 GMT

HTTP/1.1 304 Not Modified
```

```http
GET /api/products/123 HTTP/1.1
If-Modified-Since: Tue, 15 Jun 2024 10:00:00 GMT

HTTP/1.1 200 OK
Last-Modified: Wed, 16 Jun 2024 14:30:00 GMT
```

### Cache Invalidation Strategies

| Strategy | Mechanism | Latency | Complexity |
|---|---|---|---|
| TTL-based (time-to-live) | Set max-age, cache expires naturally | Up to max-age delay | None |
| Write-through | On update, purge cache keys | Immediate | Medium — must know all cache keys |
| Cache tags / surrogate keys | Tag responses, purge by tag | Immediate | Higher — requires cache that supports tags |
| Event-driven | Publish invalidation event, cache subscriber purges | Near-immediate | High — needs message broker + event system |

**Tag-based invalidation (e.g., Fastly, CloudFront with Lambda@Edge):**

```http
HTTP/1.1 200 OK
Cache-Tag: product:123, category:electronics, region:us

# Invalidate all responses tagged with product:123
PURGE /api/products/123
```

### CDN Caching (CloudFront, CloudFlare)

```
Client -> CDN Edge -> Origin Server
             |
        (cache hit returns 304 or cached response)
```

**Best practices:**
- Cache static assets aggressively (`max-age=31536000` with content hash in URL)
- Use `s-maxage` to set different TTLs for CDN vs browser
- Enable CDN-level compression (Brotli > Gzip)
- Use origin shield to reduce load on origin for cache misses
- Implement cache invalidation via API (CloudFront create-invalidation, CloudFlare purge)

> **Trap:** Caching authenticated responses as `public`. If you return `Cache-Control: public, max-age=3600` for an endpoint like `GET /api/profile`, User A's profile gets cached and served to User B. Authenticated responses must have `Cache-Control: private` or `no-store` (if user-specific).
>
> **Trap:** ETag comparison for large responses by comparing the full response body. Generating an ETag by hashing 10MB of JSON on every request defeats the purpose of ETags (you've already done the work). Instead, base the ETag on a row `updated_at` timestamp, or a hash of a subset of fields (like hash of `id` + `updated_at`).
>
> **Trap:** Caching before auth checks. A CDN caches `GET /api/admin/users` without checking authorization. The next user receives the first user's cached admin response. Solution: authenticated responses should have `Cache-Control: private`, and CDN should be configured to never cache responses with `Set-Cookie` or `Authorization` headers.

> **Follow-up:** "How do you handle cache invalidation for nested resources?" — When a parent resource changes, all child-dependent cached pages must also be invalidated. Use cache tags: tag the parent response with `user:123`, tag all child responses with the same tag. When user 123 updates their profile, purge by tag `user:123` across all caches.
>
> **Follow-up:** "What is `stale-while-revalidate` and when would you use it?" — It serves stale content to users while asynchronously fetching a fresh version. Use it for news feeds, product listings, or any read-heavy data where slightly stale data is acceptable (1-5 minutes old). Improves perceived latency significantly.
>
> **Follow-up:** "How do you handle caching with API versioning?" — Each version should have its own cache key. URL path versioning handles this naturally (different URLs = different cache keys). Header versioning needs the `Vary: Accept` header so the cache knows to store different versions of the same URL separately.

---

## 8. CORS

### What is CORS

Cross-Origin Resource Sharing is a browser security mechanism that controls which origins (domains) can access resources from your API. Without CORS, a SPA at `https://app.example.com` cannot make `fetch()` requests to `https://api.example.com`.

### Simple Requests vs Preflight

**Simple request** (all conditions must be met):
- Method: GET, HEAD, or POST
- Content-Type: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`
- No custom headers

**Preflight request** (everything else):
- Browser sends OPTIONS request before the actual request
- Server must respond with allowed origins, methods, and headers
- Browser then sends the real request

```
Preflight:

OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type

Response:

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: true

Actual request:

POST /api/data HTTP/1.1
Origin: https://app.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

### Response Headers

| Header | Purpose |
|---|---|
| `Access-Control-Allow-Origin` | Which origin(s) are allowed (`*` or specific) |
| `Access-Control-Allow-Methods` | Which HTTP methods are allowed |
| `Access-Control-Allow-Headers` | Which custom headers are allowed |
| `Access-Control-Max-Age` | How long preflight result can be cached (seconds) |
| `Access-Control-Allow-Credentials` | Whether cookies/auth headers can be sent |
| `Access-Control-Expose-Headers` | Which response headers the client can read |

### Handling OPTIONS Preflight Efficiently

```python
# Middleware for preflight handling
def cors_middleware(request):
    if request.method == "OPTIONS":
        response = Response(status=204)
    else:
        response = handle_request(request)

    allowed_origin = get_allowed_origin(request.headers.get("Origin", ""))
    if allowed_origin:
        response.headers["Access-Control-Allow-Origin"] = allowed_origin
        response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, PATCH"
        response.headers["Access-Control-Allow-Headers"] = "Authorization, Content-Type, X-Requested-With"
        response.headers["Access-Control-Max-Age"] = "86400"

    return response
```

**Performance tip:** Handle OPTIONS at the gateway/load balancer level (Nginx, Envoy) before the request reaches your application servers. Preflight requests carry no body and should be extremely cheap.

```nginx
# Nginx CORS configuration
location /api/ {
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' 'https://app.example.com';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type';
        add_header 'Access-Control-Max-Age' 86400;
        add_header 'Content-Length' 0;
        return 204;
    }
}
```

### Wildcard vs Specific Origins

```
// Wildcard — allows ANY origin
Access-Control-Allow-Origin: *

// Specific origin — allows only this domain
Access-Control-Allow-Origin: https://app.example.com
// Vary: Origin is needed for caching
Vary: Origin
```

**Wildcard caveats:**
- Cannot be used with `Access-Control-Allow-Credentials: true`
- May be too permissive for production APIs

**Dynamic origin:**

```python
ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://staging.example.com",
    "http://localhost:3000"
]

def get_allowed_origin(request_origin):
    if not request_origin:
        return None
    for allowed in ALLOWED_ORIGINS:
        if matches_origin(request_origin, allowed):
            return allowed
    return None
```

### Credentials Mode

```javascript
// Client-side — must be set explicitly
fetch("https://api.example.com/data", {
  credentials: "include",  // sends cookies and auth headers
})
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

**When `credentials: "include"` is set:**
- `Access-Control-Allow-Origin` cannot be `*` — must be an explicit origin
- `Access-Control-Allow-Credentials` must be `true`

### Caching CORS Preflight

```http
HTTP/1.1 204 No Content
Access-Control-Max-Age: 86400  # Cache for 24 hours
```

Without `Access-Control-Max-Age`, browsers send a preflight OPTIONS request before every cross-origin request, doubling latency. Set it to 24 hours (86400 seconds) for production.

> **Trap:** Reflecting the `Origin` header without validation — `Access-Control-Allow-Origin: <reflected-origin>`. If your server returns `Access-Control-Allow-Origin: https://evil.com` because the request had `Origin: https://evil.com`, any website can read your API responses. Always validate against an allowlist.
>
> **Trap:** Not caching preflight responses. Every cross-origin request triggers an OPTIONS preflight, doubling the request count and latency. Set `Access-Control-Max-Age` to a reasonable value (at least 600 seconds, ideally 86400).
>
> **Trap:** Allowing credentials with a wildcard origin. The browser will reject the response: `Access-Control-Allow-Origin: *` cannot be combined with `Access-Control-Allow-Credentials: true`. Use an explicit origin instead.

> **Follow-up:** "How does CORS handle multiple allowed origins?" — HTTP only allows a single origin in the header (or `*`). You cannot send `Access-Control-Allow-Origin: https://a.com, https://b.com`. If you need multiple origins, check the request's `Origin` against your allowlist server-side and echo back the matched origin (not the request origin directly — validate and then return it).
>
> **Follow-up:** "Does CORS protect against CSRF?" — No. CORS is a browser mechanism that restricts JavaScript from reading responses across origins. It does not prevent a form submission (`<form action="POST">`) which can still send requests to your API without reading the response. CSRF requires separate protections (SameSite cookies, CSRF tokens).
>
> **Follow-up:** "Should all responses include CORS headers?" — Only responses to cross-origin requests need CORS headers. Same-origin requests ignore them. It's safe to always include them (with validation) — it simplifies infrastructure. Just ensure preflight handling is efficient.

---

## 9. OpenAPI / Swagger

### OpenAPI Specification (v3.x)

OpenAPI is a language-agnostic standard for describing HTTP APIs. It covers endpoints, request/response schemas, parameters, authentication, and error responses.

```yaml
openapi: 3.0.3
info:
  title: Inventory API
  version: 2.0.0
  description: Multi-tenant inventory management API

servers:
  - url: https://api.example.com/v2
    description: Production

paths:
  /products:
    get:
      summary: List products
      operationId: listProducts
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 25
        - name: cursor
          in: query
          schema:
            type: string
      responses:
        "200":
          description: Product list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ProductList"
        "429":
          $ref: "#/components/responses/RateLimited"
```

### Documenting Endpoints

**Request body:**

```yaml
paths:
  /products:
    post:
      summary: Create a product
      operationId: createProduct
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateProductRequest"
      responses:
        "201":
          description: Product created
          headers:
            Location:
              schema:
                type: string
                format: uri
              description: URL of created product
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Product"
```

**Response schemas:**

```yaml
components:
  schemas:
    Product:
      type: object
      required:
        - id
        - name
        - price
      properties:
        id:
          type: string
          format: uuid
          example: "550e8400-e29b-41d4-a716-446655440000"
        name:
          type: string
          example: "Wireless Charger"
        price:
          type: number
          format: float
          example: 29.99
        description:
          type: string
          nullable: true
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    CreateProductRequest:
      type: object
      required:
        - name
        - price
      properties:
        name:
          type: string
          maxLength: 255
          example: "Wireless Charger"
        price:
          type: number
          minimum: 0.01
          example: 29.99
        description:
          type: string
          maxLength: 5000

    ProductList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: "#/components/schemas/Product"
        meta:
          type: object
          properties:
            total:
              type: integer
              nullable: true
              example: 1542
        links:
          type: object
          properties:
            next:
              type: string
              format: uri
              nullable: true
            prev:
              type: string
              format: uri
              nullable: true
```

**Error responses:**

```yaml
components:
  responses:
    ValidationError:
      description: Validation failed
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                $ref: "#/components/schemas/ErrorObject"

    RateLimited:
      description: Rate limit exceeded
      headers:
        Retry-After:
          schema:
            type: integer
          description: Seconds to wait before retrying
        X-RateLimit-Limit:
          schema:
            type: integer
        X-RateLimit-Remaining:
          schema:
            type: integer
        X-RateLimit-Reset:
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/RateLimitError"

  schemas:
    ErrorObject:
      type: object
      properties:
        id:
          type: string
          example: "err_abc123"
        code:
          type: string
          example: "VALIDATION_ERROR"
        message:
          type: string
          example: "Validation failed for the request."
        errors:
          type: array
          items:
            $ref: "#/components/schemas/FieldError"

    FieldError:
      type: object
      properties:
        field:
          type: string
          example: "email"
        message:
          type: string
          example: "must be a valid email address"
        code:
          type: string
          example: "INVALID_FORMAT"

    RateLimitError:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              example: "RATE_LIMIT_EXCEEDED"
            message:
              type: string
              example: "Too many requests. Please try again later."
            retry_after_seconds:
              type: integer
```

**Authentication:**

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: "JWT access token obtained from /oauth/token"

    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: "Legacy API key for machine-to-machine"

security:
  - BearerAuth: []
```

### Design-First vs Code-First

| Approach | Workflow | Pros | Cons |
|---|---|---|---|
| **Design-first** | Write OpenAPI YAML → Generate server stub/client | Contract is source of truth, language-agnostic, reviewable by non-devs | Code generation may not match idiomatic patterns |
| **Code-first** | Write code → Generate OpenAPI from annotations/reflection | Always in sync, uses native patterns, faster iteration | May expose internal details, annotation clutter |

**Recommendation for senior roles:** Design-first for public-facing APIs where contract stability matters. Code-first for internal microservices where speed matters. For critical APIs, do design-first and add contract tests that verify the implementation matches the spec.

### API Documentation Tools

| Tool | Type | Use Case |
|---|---|---|
| Swagger UI | Interactive docs | Developer portals, try-it-out |
| Redoc | Static docs | Clean, searchable reference docs |
| Stoplight | Full lifecycle | Design, mock, document, test |
| Postman | Client + docs | Manual testing, collections |
| SwaggerHub | Collaborative platform | Teams, versioning, hosting |

### Contract Testing with OpenAPI

```python
# Using pytest + openapi-core (Python example)
from openapi_core import create_spec
from openapi_core.validation.request.validators import RequestValidator
from openapi_core.validation.response.validators import ResponseValidator

spec = create_spec(openapi_yaml_content)
request_validator = RequestValidator(spec)
response_validator = ResponseValidator(spec)

def test_create_product_response():
    request = make_test_request("POST", "/products", body={"name": "Test", "price": 10})
    response = app.handle(request)

    # Validate response against spec
    result = response_validator.validate(request, response)
    assert not result.errors, f"Response violates spec: {result.errors}"
```

### Spectral for API Linting

```yaml
# .spectral.yaml
extends: spectral:oas

rules:
  # Custom rules
  operation-operationId:
    message: "Every operation must have an operationId"
    given: "$.paths[*][*]"
    then:
      field: operationId
      function: defined

  no-ambiguous-paths:
    message: "Paths must use kebab-case"
    given: "$.paths[*]~"
    then:
      function: pattern
      functionOptions:
        match: "^/[-a-z0-9/{}]+$"
```

> **Trap:** Out-of-date documentation. The spec says `POST /v1/products` returns `{id, name, price}`, but the actual API returns `{id, title, cost}`. Clients build against the spec and break in production. Solution: make spec verification part of CI. Fail the build if the spec doesn't match a recorded response (contract testing).
>
> **Trap:** Not documenting error responses. The spec has beautiful schemas for 200 responses but not a single 4xx or 5xx definition. Clients have no idea what error format to parse. Always document at least the common error responses: 400, 401, 403, 404, 422, 429, 500.
>
> **Trap:** Overly permissive schemas — `type: object` with no `properties`. This accepts any JSON object, defeating the purpose of a spec. Be explicit: list every property, mark required fields, define types, provide examples. `additionalProperties: false` prevents unexpected fields from passing validation.

> **Follow-up:** "How do you version your OpenAPI spec?" — Store spec alongside code in the same repository. Tag the spec file with the API version in `info.version`. Use git tags or branches for major versions. Generate human-readable changelogs from spec diffs between versions.
>
> **Follow-up:** "How do you handle breaking changes detected in OpenAPI spec review?" — Break review into two stages: (1) automated — use OpenAPI diff tools (openapi-diff, oasdiff) to flag breaking changes in CI, (2) manual — human reviews the diff and decides if the breaking change is worth a version bump or can be done additively.
>
> **Follow-up:** "How would you use OpenAPI for mock servers during development?" — Tools like Prism (from Stoplight) read the OpenAPI spec and generate realistic mock responses using `example` values or pattern-based fake data. Frontend teams can develop against mocks before the backend is complete. This is the "contract-first" promise in practice.

---

## 10. Tier 2 Q&A Drill

Cover your screen. Answer each question out loud. Check your answer against the hidden text. Mark any answer that was vague.

---

**Q1:** You're designing a new API. What versioning strategy do you choose and why?

<details>
<summary>Answer</summary>

URL path versioning (`/v1/`). It's explicit, trivial to route at the gateway, easy to test, and visible in every log and analytics tool. Header versioning is more REST-pure but creates debugging friction. Query param versioning is fragile for caching. Default-version strategy: if no version is specified, return the latest stable version with a `Warning` header advising the client to pin a version. Whatever you choose, the critical thing is to version from day one — adding versioning later requires either breaking every client or supporting an unversioned path indefinitely.
</details>

---

**Q2:** When should you use cursor pagination instead of offset pagination? Give a concrete example.

<details>
<summary>Answer</summary>

Use cursor pagination whenever the dataset is:
1. Large and deep (offset 100K is slow)
2. Frequently written to (new records shift pages)
3. A real-time feed (notifications, activity logs, messages)

Example: A notification feed. User has 5000 notifications. With offset pagination, inserting 10 new notifications between page loads causes duplicates and missed notifications. With cursor pagination (`WHERE id > :last_notification_id`), the user always sees new items appended correctly regardless of writes. Additionally, an activity feed viewed by 10K concurrent users would cause database strain with `SELECT count(*)` on every page load — cursor pagination avoids this entirely.
</details>

---

**Q3:** A client sends `?fields=id,email,password_hash`. What happens?

<details>
<summary>Answer</summary>

It should return 400 Bad Request or ignore the `password_hash` field. Never pass user-requested field names directly to the serialization layer. Every endpoint must have an allowlist of exposed fields. In this case, `password_hash` is not in the allowlist, so it's either omitted (safe) or rejected with a validation error. The response should include only `id` and `email`. Additionally, this is a red flag — someone is probing the API for sensitive fields. Log the request and investigate.
</details>

---

**Q4:** Design an error response structure for a validation endpoint. Then explain how it differs between development and production.

<details>
<summary>Answer</summary>

Structure:

```json
{
  "error": {
    "id": "err_uuid",
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "errors": [
      {
        "field": "email",
        "message": "must be a valid email",
        "code": "INVALID_FORMAT"
      }
    ]
  }
}
```

Development: Include a `detail` field with the full stack trace, query that failed, and input context. Production: Only return the structured error. Never expose internals. The `id` field links to server logs where the full trace is stored. Also, never differentiate "user not found" from "wrong password" in production — both return "Invalid credentials."
</details>

---

**Q5:** Explain the difference between HS256 and RS256 for JWT signing. When would you choose one over the other?

<details>
<summary>Answer</summary>

HS256: Symmetric algorithm. Same secret signs and verifies tokens. Fast, simple, but both parties must share the secret. Use for single-service or internal APIs where both issuer and verifier are in your control.

RS256: Asymmetric algorithm. Private key signs, public key verifies. Slower (RSA operations are expensive), but anyone can verify without holding the signing key. Use when multiple services need to verify tokens, or when third parties need to verify (public key can be published via a JWKS endpoint).

Key difference: HS256 key compromise means every verifier is compromised. RS256 key compromise of the private key is catastrophic but the public key is safe to distribute. For microservice architectures with many verifiers, RS256 is preferred.
</details>

---

**Q6:** A client is rate-limited and receives a 429. What headers must the response include?

<details>
<summary>Answer</summary>

Required:
- `Retry-After`: Seconds (integer) or HTTP-date the client must wait before retrying.
- `X-RateLimit-Limit`: Maximum requests allowed in the window.
- `X-RateLimit-Remaining`: Requests remaining in the current window (should be 0).
- `X-RateLimit-Reset`: Unix timestamp when the window resets.

Also recommended: a JSON body with the same info so clients that parse the body don't need to read headers.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 45
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1700003600
```

Missing `Retry-After` is a common trap — clients retry immediately and get another 429, creating a retry storm.
</details>

---

**Q7:** You're implementing caching for a product listing endpoint. The response includes user-specific pricing. What Cache-Control directive do you use?

<details>
<summary>Answer</summary>

`Cache-Control: private, max-age=60`

The `private` directive ensures the response is cached only on the client's browser, not on shared caches (CDNs, proxies). The pricing is user-specific (discount tiers, tax rates), so caching it publicly would leak one user's pricing to another. `max-age=60` allows the browser to reuse the response for 60 seconds without re-fetching.

If the endpoint had no user-specific data, `public, max-age=300, s-maxage=600` would be appropriate for CDN caching.
</details>

---

**Q8:** Your SPA at `https://app.example.com` can't make API requests to `https://api.example.com`. What's wrong and how do you fix it?

<details>
<summary>Answer</summary>

CORS is not configured on the API server. The browser is blocking the cross-origin request. Fix:

1. Add CORS headers to API responses: `Access-Control-Allow-Origin: https://app.example.com`
2. Handle OPTIONS preflight requests for non-simple requests (custom headers like `Authorization`, or `Content-Type: application/json`)
3. Set `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` to match what the client needs
4. Set `Access-Control-Max-Age` to cache preflight (86400 seconds)
5. If sending credentials/cookies, use explicit origin (not `*`) and set `Access-Control-Allow-Credentials: true`

Common mistakes: reflecting `Origin` without validation, not caching preflight, using `*` with credentials.
</details>

---

**Q9:** What belongs in an OpenAPI spec beyond endpoint definitions?

<details>
<summary>Answer</summary>

1. **Info metadata:** title, version, description, contact, license
2. **Server URLs:** base URLs for different environments (production, staging)
3. **Schemas:** request and response body schemas with proper types, required fields, examples
4. **Parameters:** query parameters, path parameters, headers — including default values and constraints
5. **Error responses:** every 4xx and 5xx status code with full error object schemas
6. **Security schemes:** authentication methods (Bearer JWT, API key, OAuth2), security requirements per endpoint
7. **Tags:** grouping endpoints for organization
8. **External docs:** links to detailed documentation, migration guides, changelogs
9. **Components:** reusable schemas, parameters, responses, request bodies
10. **Examples:** request/response examples for each endpoint to aid understanding

A complete spec serves as both documentation and the source of truth for code generation, contract testing, and client SDK generation.
</details>

---

**Q10:** A client is migrating from v1 to v2 of your API. What do you return when they call a deprecated v1 endpoint?

<details>
<summary>Answer</summary>

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Content-Type: application/json

{
  "data": { ... },
  "meta": {
    "api_version": "1",
    "deprecation_notice": {
      "message": "API version 1 is deprecated. Please migrate to version 2.",
      "migration_url": "https://docs.example.com/migration-v1-to-v2",
      "sunset_date": "2025-12-31",
      "changelog_url": "https://docs.example.com/changelog"
    }
  }
}
```

The endpoint still works (200, not 410) but returns deprecation headers and a notice body. Monitor the `Deprecation` header in your analytics to track which clients haven't migrated. After the sunset date, return 410 Gone with a link to the migration guide. Never break a client without warning — the deprecation headers give them time to migrate.
</details>

---

> **End of Tier 2.** You should be able to explain every trade-off in this document without looking. Vagueness on any topic means you need to re-study that section. Next: [Tier 3 — Senior (Advanced Topics)](./03-senior.md).
