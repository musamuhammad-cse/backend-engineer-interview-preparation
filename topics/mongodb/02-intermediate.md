# MongoDB — Intermediate

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Advanced queries, aggregation pipeline deep dive, indexing strategies, multi-document transactions, schema design patterns, change streams, performance optimization  
> **Real-world anchor:** Multi-tenant SaaS, trading platform (high-throughput writes), Chronos (job status/document storage)

---

## 1. Advanced queries

### Text search

MongoDB supports text search with a `text` index:

```js
// Create text index on one or more fields
db.articles.createIndex({ title: "text", body: "text" })

// Or create a wildcard text index (covers all string fields)
db.articles.createIndex({ "$**": "text" })
```

Query with `$text`:

```js
db.articles.find(
  { $text: { $search: "mongodb aggregation performance" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

**Text search operators:**

```js
// Exact phrase
{ $text: { $search: "\"aggregation pipeline\"" } }

// Exclusion
{ $text: { $search: "mongodb -sql" } }  // contains "mongodb" but not "sql"

// Weighted fields
db.articles.createIndex(
  { title: "text", body: "text" },
  { weights: { title: 10, body: 1 } }
)
```

**Trap:** A collection can have at most **one** text index. If you need text search across different field combinations, use Atlas Search (Lucene-based) instead.

**Trap:** `$text` is not case-sensitive by default. Use `$caseSensitive: true` to make it case-sensitive (but then it won't use the text index for stemming).

### Geospatial queries

```js
// Create 2dsphere index (for GeoJSON objects)
db.places.createIndex({ location: "2dsphere" })

// GeoJSON point
{
  name: "Central Park",
  location: {
    type: "Point",
    coordinates: [-73.97, 40.77]  // [longitude, latitude]
  }
}

// $near — nearest points
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-73.97, 40.77] },
      $maxDistance: 1000,  // meters
      $minDistance: 10
    }
  }
})

// $geoWithin — points within a polygon
db.places.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[[0,0], [0,1], [1,1], [1,0], [0,0]]]
      }
    }
  }
})

// $geoIntersects — geometries intersecting an area
db.places.find({
  location: {
    $geoIntersects: {
      $geometry: {
        type: "Polygon",
        coordinates: [[[0,0], [0,2], [2,2], [2,0], [0,0]]]
      }
    }
  }
})
```

### Array query patterns

#### Matching exact array

```js
// Exact match (order + elements must match)
db.posts.find({ tags: ["mongodb", "nosql"] })
```

#### Matching any element

```js
// Array contains at least one matching element
db.posts.find({ tags: "mongodb" })
```

#### Matching all elements

```js
// Array contains all specified elements
db.posts.find({ tags: { $all: ["mongodb", "nosql"] } })
```

#### Matching with conditions on array elements

```js
// Element matching multiple conditions on same array index
db.products.find({
  sizes: { $elemMatch: { size: "M", qty: { $gt: 5 } } }
})

// Without $elemMatch — conditions applied across different elements
// This matches if ANY size= element "M" exists AND ANY size element has qty>5
// Not the same as "one element that satisfies both!"
db.products.find({ "sizes.size": "M", "sizes.qty": { $gt: 5 } })
```

**Trap:** This is one of the most common MongoDB interview traps. `$elemMatch` ensures both conditions apply to the same array element; without it, they can match different elements.

### Positional operator ($)

The positional `$` operator identifies the first matching element of an array in a query:

```js
db.students.updateOne(
  { _id: 1, "grades.grade": { $gte: 90 } },
  { $set: { "grades.$.honors": true } }
  // Only updates the first matching array element
)
```

### Positional all ($[])

Updates all array elements:

```js
// Reset all grades to incomplete
db.students.updateOne(
  { _id: 1 },
  { $set: { "grades.$[].status": "incomplete" } }
)
```

### Positional filtered ($[identifier])

Updates array elements matching a filter:

```js
db.students.updateOne(
  { _id: 1 },
  { $set: { "grades.$[elem].status": "passed" } },
  { arrayFilters: [{ "elem.grade": { $gte: 60 } }] }
)
```

---

## 2. Aggregation pipeline — deep dive

### Pipeline optimization principles

1. **`$match` early** — reduces documents flowing through the pipeline; uses indexes if at the start
2. **`$project` early** — reduces document size; but `$match` should still come first
3. **`$sort` before `$group`** — enables `$first`/`$last` without separate sorting
4. **Avoid `$unwind` + `$group`** — use `$reduce` or `$map` instead if possible
5. **Use `$lookup` sparingly** — one query per input document
6. **Use `allowDiskUse` for large sorts/groups** — default 100 MB RAM limit

### Pipeline stage order for optimal performance

```
$match → $sort → $project → $lookup → $unwind → $group → $project → $sort → $limit
```

### Advanced stages

#### `$lookup` with pipeline (MongoDB 3.6+)

```js
db.orders.aggregate([
  { $match: { status: "active" } },
  { $lookup: {
      from: "payments",
      let: { orderId: "$_id", customerId: "$customerId" },
      pipeline: [
        { $match: {
            $expr: {
              $and: [
                { $eq: ["$orderId", "$$orderId"] },
                { $eq: ["$customerId", "$$customerId"] }
              ]
            }
        }},
        { $sort: { createdAt: -1 } },
        { $limit: 1 }
      ],
      as: "latestPayment"
    }
  },
  { $unwind: { path: "$latestPayment", preserveNullAndEmptyArrays: true } }
])
```

#### `$lookup` with `$mergeObjects` to flatten

```js
db.orders.aggregate([
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerObj"
    }
  },
  { $replaceRoot: {
      newRoot: {
        $mergeObjects: ["$$ROOT", { $arrayElemAt: ["$customerObj", 0] }]
      }
    }
  },
  { $project: { customerObj: 0 } }
])
```

#### `$graphLookup` — recursive graph traversal

```js
// Find all employees under a manager (recursive hierarchy)
db.employees.aggregate([
  { $match: { name: "Alice" } },
  { $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "reportsTo",
      connectToField: "_id",
      as: "subordinates",
      maxDepth: 10,
      depthField: "level"
    }
  }
])
```

#### `$facet` — parallel processing

```js
db.orders.aggregate([
  { $match: { createdAt: { $gte: ISODate("2024-01-01") } } },
  { $facet: {
      "totals": [
        { $group: {
            _id: null,
            totalRevenue: { $sum: "$total" },
            avgOrderValue: { $avg: "$total" },
            orderCount: { $sum: 1 }
        }}
      ],
      "byStatus": [
        { $group: { _id: "$status", count: { $sum: 1 } } }
      ],
      "topCustomers": [
        { $group: { _id: "$customerId", total: { $sum: "$total" } } },
        { $sort: { total: -1 } },
        { $limit: 10 }
      ],
      "monthlyTrend": [
        { $group: {
            _id: { year: { $year: "$createdAt" }, month: { $month: "$createdAt" } },
            revenue: { $sum: "$total" }
        }},
        { $sort: { "_id.year": 1, "_id.month": 1 } }
      ]
    }
  }
])
```

**Trap:** `$facet` output stage is limited to 16 MB. If your facets are large, run them separately.

#### `$bucket` and `$bucketAuto`

```js
// Manual buckets
db.products.aggregate([
  { $bucket: {
      groupBy: "$price",
      boundaries: [0, 50, 100, 200, 500],
      default: "500+",
      output: {
        count: { $sum: 1 },
        avgPrice: { $avg: "$price" }
      }
    }
  }
])

// Auto buckets — MongoDB automatically determines boundaries
db.products.aggregate([
  { $bucketAuto: {
      groupBy: "$price",
      buckets: 5,  // target number of buckets
      output: {
        count: { $sum: 1 },
        avgPrice: { $avg: "$price" }
      }
    }
  }
])
```

#### `$replaceRoot` and `$replaceWith`

```js
// Promote a subdocument to root
db.orders.aggregate([
  { $replaceRoot: { newRoot: "$shipping" } }
])

// $replaceWith (alias for $replaceRoot in 4.2+)
db.orders.aggregate([
  { $replaceWith: "$shipping" }
])
```

#### Window functions (MongoDB 5.0+)

```js
db.sales.aggregate([
  { $setWindowFields: {
      partitionBy: "$productId",
      sortBy: { date: 1 },
      output: {
        cumulativeSales: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        },
        movingAvg7Day: {
          $avg: "$amount",
          window: { documents: [-6, "current"] }
        }
      }
    }
  }
])
```

### Conditional logic in pipelines

```js
db.orders.aggregate([
  { $project: {
      orderId: 1,
      total: 1,
      tier: {
        $switch: {
          branches: [
            { case: { $gte: ["$total", 1000] }, then: "platinum" },
            { case: { $gte: ["$total", 500] }, then: "gold" },
            { case: { $gte: ["$total", 100] }, then: "silver" }
          ],
          default: "bronze"
        }
      },
      discountApplied: {
        $cond: {
          if: { $gte: ["$total", 500] },
          then: { $multiply: ["$total", 0.9] },
          else: "$total"
        }
      }
    }
  }
])
```

### Array expressions

```js
db.products.aggregate([
  { $project: {
      // Filter array elements
      inStock: {
        $filter: {
          input: "$warehouses",
          as: "wh",
          cond: { $gt: ["$$wh.qty", 0] }
        }
      },
      // Get first element
      firstWarehouse: { $first: "$warehouses" },
      // Get element at index
      primaryWarehouse: { $arrayElemAt: ["$warehouses", 0] },
      // Check if array contains value
      hasChicago: { $in: ["chicago", "$warehouseNames"] },
      // Reduce array to single value
      totalStock: {
        $reduce: {
          input: "$warehouses.qty",
          initialValue: 0,
          in: { $add: ["$$value", "$$this"] }
        }
      },
      // Map over array
      warehouseCities: {
        $map: {
          input: "$warehouses",
          as: "wh",
          in: "$$wh.city"
        }
      }
    }
  }
])
```

### Date operations

```js
db.orders.aggregate([
  { $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" },
        day: { $dayOfMonth: "$createdAt" },
        week: { $isoWeek: "$createdAt" },
        dayOfWeek: { $dayOfWeek: "$createdAt" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])

// Date math
db.orders.aggregate([
  { $match: {
      createdAt: {
        $gte: { $dateSubtract: { startDate: "$$NOW", unit: "day", amount: 30 } }
      }
    }
  }
])
```

### String operations

```js
db.users.aggregate([
  { $project: {
      upperName: { $toUpper: "$name" },
      emailDomain: { $substrCP: ["$email", { $indexOfCP: ["$email", "@"] }, -1] },
      nameParts: { $split: ["$name", " "] },
      redacted: { $concat: [ { $substrCP: ["$name", 0, 2] }, "***" ] },
      hasEmail: { $regexMatch: { input: "$email", regex: /@/ } }
    }
  }
])
```

### Type conversion

```js
db.orders.aggregate([
  { $addFields: {
      priceDecimal: { $convert: { input: "$price", to: "decimal", onError: 0, onNull: 0 } },
      createdAtDate: { $toDate: "$createdAt" }
    }
  }
])
```

---

## 3. Indexing strategies

### Index basics

Each index consumes disk and memory (in WiredTiger cache). Every insert/update/delete must update all indexes on the collection.

**General rules:**
- Index fields used in `$match`, `$sort`, `$group` (if it's the first stage)
- Prefer equality-test fields → sort fields → range-test fields (ESR rule)
- Index selectivity matters — highly selective indexes (few documents per key) are more efficient
- Avoid building indexes on fields with low cardinality for leading position (e.g., `gender`)

### Compound index design

```js
// For query: find({ customerId: "C1", status: "active" }).sort({ createdAt: -1 })
// Index: { customerId: 1, status: 1, createdAt: -1 }
// ESR:   ^equality       ^equality     ^sort
```

### ESR rule in detail

```js
// Poor index — range before sort prevents sort optimization
db.orders.createIndex({ customerId: 1, createdAt: 1, status: 1 })

// Better — equality, sort, range
db.orders.createIndex({ customerId: 1, status: 1, createdAt: 1 })

// Query: find engineers in NY earning > $100K, sorted by name
db.users.find({
  role: "engineer",           // equality
  city: "NY",                 // equality
  salary: { $gte: 100000 }    // range
}).sort({ name: 1 })         // sort

// Ideal index: { role: 1, city: 1, name: 1, salary: 1 }
//               ^eq      ^eq      ^sort    ^range
```

### Covering indexes

A covering index contains all fields required by the query — MongoDB never fetches the document:

```js
// Query: find({ email: "alice@example.com" }, { name: 1, email: 1, _id: 0 })
// Index on { email: 1, name: 1 } covers this query
// explain() shows: IXSCAN, no FETCH stage
```

**Trap:** If you include `_id: 1` in projection, `_id` must be in the index for coverage. Exclude `_id` if you want coverage and didn't index `_id` (you can't — `_id` always has an index, so excluding `_id` from projection doesn't affect coverage).

### Multikey indexes

Indexes on array fields:

```js
// Document: { tags: ["mongodb", "database", "nosql"] }
db.posts.createIndex({ tags: 1 })
// Creates index entries for "mongodb", "database", "nosql" — each pointing to same document
```

**Trap:** A compound index can have at most **one** array field. If you index two array fields, the index becomes a cross-product (each tag × each category), which can be huge.

```js
// This creates a COMPOUND multikey index — each pair (tag, category) is indexed
db.posts.createIndex({ tags: 1, categories: 1 })
// If a post has 10 tags and 5 categories, that's 50 index entries
```

### Partial indexes

```js
// Index only orders with status: "pending"
db.orders.createIndex(
  { createdAt: 1, customerId: 1 },
  { partialFilterExpression: { status: "pending" } }
)

// The index is only used when the query also includes the filter
db.orders.find({ status: "pending", createdAt: { $lt: ISODate("2024-01-01") } })
```

### Sparse indexes

Only index documents that contain the indexed field:

```js
db.users.createIndex({ twitterHandle: 1 }, { sparse: true })
// Does not index users without twitterHandle
```

**Trap:** `sparse` + `unique` creates a unique index that allows multiple documents missing the field:

```js
// Both will succeed
db.users.insertOne({ name: "Alice" })
db.users.insertOne({ name: "Bob" })
```

### TTL indexes

Auto-expire documents after a specified duration:

```js
// Delete sessions older than 24 hours
db.sessions.createIndex(
  { lastAccessed: 1 },
  { expireAfterSeconds: 86400 }
)
```

**Trap:** The indexed field must be a date or array of dates. TTL runs every 60 seconds. Document deletion can lag by up to 60 seconds.

### Index intersection

MongoDB can use multiple indexes to fulfill a query:

```js
// Indexes:
db.orders.createIndex({ status: 1 })
db.orders.createIndex({ customerId: 1 })

// Query: find({ status: "shipped", customerId: "C1" })
// MongoDB may scan both indexes and intersect
```

**Trap:** Index intersection is less efficient than a single compound index. Only use separate indexes when query patterns vary so widely that a compound index can't serve all of them.

### explain("executionStats") interpretation

```
Query: db.orders.find({ status: "shipped", customerId: "C1" }).sort({ createdAt: -1 })

Stage-wise:
1. IXSCAN (index: { customerId: 1, createdAt: 1, status: 1 })
   - keysExamined: 10
   - docsExamined: 10
   - nReturned: 10
   - executionTimeMillisEstimate: 2

Total: executionTimeMillis: 2
       totalDocsExamined: 10
       totalKeysExamined: 10
       nReturned: 10
```

**Key ratios for senior interview:**
- `totalDocsExamined ≈ nReturned` — efficient (using indexes well)
- `totalDocsExamined >> nReturned` — many documents scanned but few returned (poor selectivity)
- `totalKeysExamined >> nReturned` — many index keys scanned (index not selective enough)

### Planning index strategy

For a multi-tenant SaaS app (your experience):

```js
// Every query has organization_id filter
// Lead with equality → sort → range

// Orders query: all orders for org, sorted by date
db.orders.createIndex({ organization_id: 1, created_at: -1 })

// Orders query by status: org + status + date
db.orders.createIndex({ organization_id: 1, status: 1, created_at: -1 })

// Customer search within org
db.orders.createIndex({ organization_id: 1, customer_email: 1 })

// Avoid: index on (organization_id, created_at, status) when both work
// Prefer the more selective 'status' field over 'created_at' in compound
```

---

## 4. Multi-document transactions

### ACID transactions in MongoDB

Since MongoDB 4.0 (replica sets) and 4.2 (sharded clusters):

```js
const session = client.startSession()

try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
  })

  const accounts = db.collection("accounts")

  await accounts.updateOne(
    { _id: "A", balance: { $gte: 100 } },
    { $inc: { balance: -100 } },
    { session }
  )

  await accounts.updateOne(
    { _id: "B" },
    { $inc: { balance: 100 } },
    { session }
  )

  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()
  throw err
} finally {
  session.endSession()
}
```

### Transaction constraints

| Constraint | Detail |
|------------|--------|
| Timeout | 60 seconds default (`transactionLifetimeLimitSeconds`) |
| Locking | All involved documents are locked; concurrent access to same documents blocked |
| Oplog | Must fit within 10% of oplog size |
| WiredTiger cache | Modified documents held in cache until commit |
| Write concern | Can be overridden per-statement; default: "majority" |
| Read concern | `local` (default), `majority`, `snapshot` |
| Retry on conflict | Write conflicts cause `TransientTransactionError` — application must retry |

### When to use transactions

**Use transactions when you need atomic updates across multiple documents/collections.** But prefer single-document operations — they are atomic by default:

```js
// Single document atomic update — no transaction needed
db.accounts.updateOne(
  { _id: "A", balance: { $gte: 100 } },
  {
    $inc: { balance: -100 },
    $push: { transactions: { to: "B", amount: 100, date: new Date() } }
  }
)
```

**Trap:** Many developers overuse transactions in MongoDB. The document model allows embedded arrays and nested objects, so many "multi-table" operations can be single-document operations. Don't reach for transactions when a well-structured document would suffice.

---

## 5. Schema design patterns

### Core trade-off: Embedding vs Referencing

| | Embedding | Referencing |
|---|---|---|
| Read performance | Fast — single read gets all related data | Slower — requires `$lookup` or multiple queries |
| Write performance | Slower if embedded data changes frequently | Faster — update only the referenced document |
| Document size | Limited to 16 MB | No size constraint on references |
| Data consistency | Atomic within single document | Requires transactions for atomic updates |
| Queryability | Can query on embedded fields directly | Must `$lookup` to join |
| Data duplication | High — same data embedded in multiple docs | Low — normalized |

### Common schema patterns

#### 1. One-to-one (embed)

```json
{
  "_id": "user_001",
  "name": "Alice",
  "profile": {
    "bio": "Engineer",
    "avatar": "url.jpg"
  }
}
```

#### 2. One-to-few (embed, array of subdocuments)

```json
{
  "_id": "user_001",
  "name": "Alice",
  "addresses": [
    { "type": "home", "city": "NY" },
    { "type": "work", "city": "SF" }
  ]
}
```

**When to embed:** Few subdocuments, always accessed with parent, rarely updated independently.

#### 3. One-to-many (referencing — child points to parent)

```json
// Order document
{ "_id": "order_001", "customerId": "user_001", "total": 100 }

// User document
{ "_id": "user_001", "name": "Alice" }
```

**When to reference:** Many subdocuments, accessed independently, 16 MB limit concerns.

#### 4. One-to-many — array of references (parent points to children)

```json
{ "_id": "user_001", "name": "Alice", "orderIds": ["order_001", "order_002"] }
```

**Trap:** Unbounded arrays — if a user can have unlimited orders, the array grows unbounded. Prefer child referencing (order points to user).

#### 5. Many-to-many

```json
// Student
{ "_id": "student_001", "name": "Alice", "courseIds": ["course_001", "course_002"] }

// Course
{ "_id": "course_001", "name": "MongoDB 101", "studentIds": ["student_001"] }
```

**Better for large M:N:** Use a junction collection:

```json
// Enrollment
{ "_id": "enr_001", "studentId": "student_001", "courseId": "course_001", "enrolledAt": "2024-01-01" }
```

#### 6. Polymorphic schema (discriminator)

```json
// Different document types in same collection
// Type A — User
{ "_id": "u1", "type": "user", "name": "Alice", "email": "alice@example.com" }

// Type B — Organization
{ "_id": "o1", "type": "org", "name": "Acme Corp", "domain": "acme.com" }
```

**Use case:** When different entities share a common query pattern (e.g., search across users and organizations).

#### 7. Bucket pattern (time-series optimization)

```json
// Instead of one document per reading:
// Group sensor readings into hourly buckets
{
  "sensorId": "sensor_001",
  "hour": ISODate("2024-01-01T00:00:00Z"),
  "readings": [
    { "t": ISODate("2024-01-01T00:01:00Z"), "v": 23.5 },
    { "t": ISODate("2024-01-01T00:02:00Z"), "v": 23.7 }
  ],
  "count": 120,
  "avg": 23.6
}
```

**Benefit:** Fewer documents, fewer index entries, better read locality.

#### 8. Computed pattern

Pre-compute derived data and store it:

```json
{
  "_id": "order_001",
  "items": [{ "product": "Widget", "price": 20, "qty": 2 }],
  "total": 40,
  "totalComputed": 40
}
```

**Use case:** Avoid aggregating the same data repeatedly.

### Schema anti-patterns

| Anti-pattern | Problem | Solution |
|---|---|---|
| Massive (unbounded) arrays | Document exceeds 16 MB, slows reads | Reference instead of embed, or paginate |
| Bloated documents | Unnecessary fields in every query | Use projections, split collections |
| Unnecessary indexes on every field | Slow writes, large index size | Only index query patterns, not all fields |
| Overly deep nesting (> 4-5 levels) | Hard to query, update, validate | Flatten or reference |
| Always using `$lookup` (treated as RDBMS) | Poor performance | Embed frequently accessed data |
| All collections with generic `_id` | Missing natural primary keys | Use meaningful IDs where appropriate |

---

## 6. Schema validation

MongoDB 3.2+ supports document validation:

```js
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email"],
      properties: {
        _id: { bsonType: "objectId" },
        name: {
          bsonType: "string",
          description: "Must be a string and is required"
        },
        email: {
          bsonType: "string",
          pattern: "^.+@.+$",
          description: "Must be a valid email"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        role: {
          enum: ["engineer", "manager", "admin"]
        },
        tags: {
          bsonType: "array",
          uniqueItems: true,
          items: { bsonType: "string" }
        }
      }
    }
  },
  validationAction: "error"  // or "warn" (MongoDB 3.6+)
})
```

**Trap:** Validation is only applied on insert and update, not on existing documents. Use `db.collection.aggregate([{ $match: { $nor: [validation] } }])` to find invalid documents.

---

## 7. Aggregation pipeline optimization — benchmark approach

For your trading platform (20K+ DAU), an aggregation like this:

```js
db.trades.aggregate([
  { $match: { symbol: "AAPL", timestamp: { $gte: start, $lt: end } } },
  { $sort: { timestamp: -1 } },
  { $group: {
      _id: { $dateTrunc: { date: "$timestamp", unit: "minute" } },
      open: { $first: "$price" },
      close: { $last: "$price" },
      high: { $max: "$price" },
      low: { $min: "$price" },
      volume: { $sum: "$qty" }
    }
  }
])
```

**Optimization tips:**
1. Index on `{ symbol: 1, timestamp: -1 }` — covers `$match` and `$sort`
2. Use `$match` before `$sort` to reduce documents for sorting
3. If the range is always the latest N minutes, consider a bucketed time-series schema
4. For real-time OHLCV, use change streams + pre-aggregation

---

## 8. WiredTiger storage engine

### Key features

- **Document-level concurrency**: Multiple documents can be read/written concurrently
- **MVCC (Multi-Version Concurrency Control)**: Readers don't block writers, writers don't block readers
- **Snappy compression** (default): Collection data compressed on disk
- **Prefix compression** on indexes: Common index key prefixes compressed
- **Cache**: Default 50% of (RAM - 1 GB), or 256 MB minimum
- **Journal**: Write-ahead log for crash recovery (60ms commit interval by default)

### Cache management

```
Read:  If data is in cache → immediate
       If data is not in cache → page fault, read from disk into cache (evicts other data)

Write: If data is in cache → update in cache, journal commit
       If data is not in cache → page fault, load page, update, journal commit
```

**Trap:** If working set does not fit in cache, MongoDB becomes disk-bound. Monitor `wiredTiger.cache.tracked dirty bytes in the cache`. If this grows over the cache size, you need more RAM or a bigger cache.

### Compression

```js
// Collection-level compression
db.createCollection("logs", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
})
```

Compression ratios: zstd > zlib > snappy (in compression) but snappy > zlib > zstd (in speed). Default: snappy.

### Journal

```js
// Journal commit interval — default 100ms, configurable
// Lower = better durability, higher = better write throughput
```

---

## 9. Aggregation pipeline memory management

### default limits

| Parameter | Default | Max |
|-----------|---------|-----|
| Pipeline RAM | 100 MB | unlimited with `allowDiskUse: true` |
| Single `$sort` RAM | 32 MB | same |
| `$group` RAM | 100 MB | same |
| Single document size | 16 MB | 16 MB (hard limit) |
| Pipeline result size | 16 MB per document | same |

### Operations that trigger spill to disk

- `$sort` without index
- `$group` with large number of buckets
- `$bucketAuto`
- `$facet`

Enable disk use:

```js
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } }
], { allowDiskUse: true })
```

---

## 10. Practical drills

### Drill 1 — Schema design

You're designing the trade collection for your trading platform (20K+ DAU, millions of trades/day). Each trade has: symbol, price, qty, side (buy/sell), timestamp, orderId, userId. Users query:
1. Recent trades for a symbol (last N seconds)
2. OHLCV candles per minute/hour/day
3. A user's trade history

Design the document model and indexes.

<details>
<summary>Answer</summary>

**Option 1: Flat documents** — one document per trade

```json
{
  "_id": ObjectId("..."),
  "symbol": "AAPL",
  "price": 150.50,
  "qty": 100,
  "side": "buy",
  "timestamp": ISODate("2024-01-15T10:30:00Z"),
  "orderId": "ORD001",
  "userId": "USR001"
}
```

Indexes:
```js
// Symbol + timestamp for recent trades and OHLCV
db.trades.createIndex({ symbol: 1, timestamp: -1 })
// User + timestamp for user trade history
db.trades.createIndex({ userId: 1, timestamp: -1 })
```

**Option 2: Bucketed schema** — group trades into second/minute buckets

```json
{
  "_id": { "symbol": "AAPL", "time": ISODate("2024-01-15T10:30:00Z") },
  "trades": [
    { "price": 150.50, "qty": 100, "side": "buy", "userId": "USR001" }
  ],
  "ohlcv": {
    "open": 150.00, "high": 151.00, "low": 149.50, "close": 150.50, "volume": 5000
  },
  "tradeCount": 42
}
```

This reduces document count and enables fast OHLCV queries but makes per-user queries harder.
</details>

### Drill 2 — Query performance

Given this query that is slow on a 10M-document collection:

```js
db.orders.find({
  organization_id: "org_123",
  status: { $in: ["pending", "processing"] },
  created_at: { $gte: ISODate("2024-01-01") }
}).sort({ created_at: -1 })
```

Design the optimal index.

<details>
<summary>Answer</summary>

ESR analysis:
- `organization_id` — equality → first
- `status` — equality (`$in` is multiple equality values, still considered equality) → second
- `created_at` — range ($gte) → third, and also used for sort

Index: `{ organization_id: 1, status: 1, created_at: -1 }`

The sort direction (-1) matches the sort order in the query, so MongoDB reads the index in order and avoids an in-memory sort.
</details>

### Drill 3 — Find the bug

```js
// Find orders where total exceeds credit limit
db.customers.createIndex({ creditLimit: 1 })
db.customers.find({ creditLimit: { $lt: "$total" } })
```

<details>
<summary>Answer</summary>

`$lt` compares against the literal string `"$total"`, not the `total` field value. Use `$expr`:

```js
db.customers.find({
  $expr: { $lt: ["$creditLimit", "$total"] }
})
```

But this requires an index on both fields. If frequent, add a computed field or use aggregation.
</details>

---

## Interview traps cheatsheet — Intermediate

| Trap | The truth |
|------|-----------|
| "$elemMatch and regular array queries are equivalent" | `$elemMatch` ensures conditions apply to same array element; without it, they can match different elements. |
| "Compound indexes can include two array fields" | Only one array field per compound index (cartesian product explosion). |
| "Text search in MongoDB is as powerful as Elasticsearch" | No stemming, fuzzy search, or relevance tuning beyond weights. Use Atlas Search for advanced text. |
| "Transactions solve all atomicity problems" | Transactions are expensive, have 60s timeout, and require thoughtful design. Single-document ops are free atomicity. |
| "Aggregation pipeline is always faster than application-level processing" | Not always — `$lookup` for every document can be very slow. Sometimes in-app join is faster. |
| "allowDiskUse makes large aggregations fast" | It avoids RAM errors but disk spill is very slow. Optimize to fit in RAM. |
| "All upserts are safe" | Upsert + unique index can fail if the insert key already exists (race condition on read-then-write). |
| "PartialFilterExpression filters always apply" | The partial filter expression must be part of the query for the index to be used. |
| "$project at start of pipeline always helps" | $project before $match prevents index usage on filtered fields. Always $match first. |
</details>
