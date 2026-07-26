# MongoDB — Basic

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** MongoDB fundamentals — document model, CRUD, query operators, basic indexing, aggregation pipeline basics  
> **Trap:** Most candidates treat MongoDB like SQL with different syntax. At the senior level, you must articulate the *paradigm shift* from relational to document.

---

## 1. MongoDB document model

### Documents and BSON

MongoDB stores data as **documents** in **collections** (analogous to rows in tables, but schemaless). Internally, documents are serialized as **BSON** (Binary JSON), a binary-encoded serialization of JSON-like documents.

```
Analogy:  Database  →  Collection  →  Document
          MySQL:    Database  →  Table       →  Row
```

A document is a set of key-value pairs:

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Alice",
  "email": "alice@example.com",
  "address": {
    "city": "New York",
    "zip": "10001"
  },
  "tags": ["engineering", "senior"]
}
```

### BSON data types

| Type | Alias | Example |
|------|-------|---------|
| Double | `double` | `3.14159` |
| String | `string` | `"hello"` |
| Object | `object` | `{ "a": 1 }` |
| Array | `array` | `[1, 2, 3]` |
| Binary Data | `binData` | `BinData(0, "e8...")` |
| ObjectId | `objectId` | `ObjectId("...")` |
| Boolean | `bool` | `true` |
| Date | `date` | `ISODate("2024-01-01")` |
| Null | `null` | `null` |
| 32-bit Integer | `int` | `NumberInt(42)` |
| 64-bit Integer | `long` | `NumberLong(42)` |
| Decimal128 | `decimal` | `NumberDecimal("10.99")` |
| Timestamp | `timestamp` | `Timestamp(123, 456)` |

**Trap:** There is no `integer` type in BSON — numbers are either `Double`, `Int32`, or `Int64`. If you insert `{ x: 42 }`, the shell treats it as `Double`. Use `NumberInt()` or `NumberLong()` for explicit int types.

**BSON document size limit:** 16 MB. This affects schema design — you must not embed unbounded arrays inside documents.

### The `_id` field

Every document must have a unique `_id` field, which acts as the primary key:
- If not provided, MongoDB generates an `ObjectId` (12 bytes: 4-byte timestamp + 5-byte random + 3-byte increment)
- Can be any BSON type (UUID, integer, string — but must be unique)
- `_id` is immutable after document creation
- `_id` has a unique index by default

```js
// Custom _id
db.users.insertOne({ _id: "user_alice_001", name: "Alice" })
```

**Trap:** Some candidates assume `ObjectId` is always sorted by insertion order. It's *approximately* insertion-ordered (first 4 bytes are timestamp), but within the same second the random component dominates.

### Dynamic schema vs fixed schema

MongoDB collections do not enforce a schema. Documents in the same collection can have different fields:

```js
db.users.insertMany([
  { name: "Alice", role: "engineer" },
  { name: "Bob", role: "manager", direct_reports: 5 },
  { name: "Charlie" }  // no role field at all
])
```

**Trap:** "Schemaless" does not mean "schema doesn't matter." Application code still expects certain fields. Use:
- **Document validation** (MongoDB 3.2+): `db.createCollection("users", { validator: { $jsonSchema: { ... } } })`
- **Mongoose/ODM** for application-level schema enforcement
- **Migration patterns** to handle schema evolution

At the senior level, call out that "schemaless" means *application-enforced schema* rather than *database-enforced schema*, which is a trade-off: faster iteration but weaker data integrity guarantees.

---

## 2. CRUD operations

### Create

```js
// Insert one document
db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  role: "engineer",
  createdAt: new Date()
})

// Insert many documents
db.users.insertMany([
  { name: "Bob", email: "bob@example.com" },
  { name: "Charlie", email: "charlie@example.com" }
])
```

`insertOne` returns the inserted `_id`. `insertMany` returns an array of `_id`s.

**Trap:** Insert operations have ordering:
- Ordered (default): MongoDB inserts in order, stops at first error → some documents may not be inserted
- Unordered: MongoDB inserts in parallel, skips errors, continues with remaining documents

```js
db.users.insertMany([doc1, doc2, doc3], { ordered: false })
```

### Read — find()

```js
// Find all documents
db.users.find()

// Find with filter
db.users.find({ role: "engineer" })

// Find one document
db.users.findOne({ email: "alice@example.com" })

// Projection — only return specific fields
db.users.find(
  { role: "engineer" },
  { name: 1, email: 1, _id: 0 }  // 1=include, 0=exclude
)
```

### Sort, skip, limit

```js
db.users.find()
  .sort({ name: 1 })    // 1=ascending, -1=descending
  .skip(10)
  .limit(20)
```

**Trap:** The order of `.sort()`, `.skip()`, `.limit()` in the method chain does not affect the execution order. MongoDB applies them in a fixed order: `sort → skip → limit`. But `sort + skip + limit` can produce unexpected results if you skip across documents with equal sort values.

**Memory limit:** `sort` without an index uses 32 MB of RAM, then errors. Use `.allowDiskUse(true)` to spill to disk (slow), or better, add an index.

### Update

```js
// Update one document
db.users.updateOne(
  { email: "alice@example.com" },
  { $set: { role: "senior_engineer" } }
)

// Update many documents
db.users.updateMany(
  { role: "junior" },
  { $set: { role: "engineer" } }
)

// Replace one document (completely replaces, except _id)
db.users.replaceOne(
  { email: "alice@example.com" },
  { name: "Alice", email: "alice@example.com", role: "senior_engineer", updatedAt: new Date() }
)
```

#### Update operators

| Operator | Description | Example |
|----------|-------------|---------|
| `$set` | Set a field value | `{ $set: { name: "Alice" } }` |
| `$unset` | Remove a field | `{ $unset: { oldField: "" } }` |
| `$inc` | Increment a numeric field | `{ $inc: { score: 5 } }` |
| `$mul` | Multiply a numeric field | `{ $mul: { price: 1.1 } }` |
| `$min` | Update only if value is less | `{ $min: { discount: 0.1 } }` |
| `$max` | Update only if value is greater | `{ $max: { price: 100 } }` |
| `$rename` | Rename a field | `{ $rename: { old: "new" } }` |
| `$push` | Add element to array | `{ $push: { tags: "new" } }` |
| `$push` + `$each` | Add multiple to array | `{ $push: { tags: { $each: ["a", "b"] } } }` |
| `$pop` | Remove first/last from array | `{ $pop: { tags: 1 } }` (last) or `{ tags: -1 }` (first) |
| `$pull` | Remove matching elements | `{ $pull: { tags: "old" } }` |
| `$pullAll` | Remove all matching | `{ $pullAll: { tags: ["a", "b"] } }` |
| `$addToSet` | Add to array if not present | `{ $addToSet: { tags: "new" } }` |
| `$currentDate` | Set to current date | `{ $currentDate: { updatedAt: true } }` |

### Upsert

```js
// updateOne with upsert: true creates document if no match found
db.users.updateOne(
  { email: "new@example.com" },
  { $set: { name: "New User" } },
  { upsert: true }
)
```

**Trap:** `upsert` can create a document based on the query filter plus the update clause. If the filter contains immutable fields like `_id`, include them in `$setOnInsert`:

```js
db.users.updateOne(
  { email: "new@example.com" },
  {
    $set: { name: "New User" },
    $setOnInsert: { createdAt: new Date() }
  },
  { upsert: true }
)
```

### Delete

```js
db.users.deleteOne({ email: "old@example.com" })
db.users.deleteMany({ role: "inactive" })
db.collection.drop()     // Drop entire collection + indexes
db.dropDatabase()         // Drop the database
```

**Trap:** `remove({})` (old API) is deprecated in favor of `deleteMany({})`. Both remove all documents but not the collection itself or its indexes.

---

## 3. Query operators

### Comparison operators

```js
// $eq, $ne, $gt, $gte, $lt, $lte
db.products.find({ price: { $gte: 10, $lte: 50 } })

// $in, $nin
db.products.find({ category: { $in: ["electronics", "books"] } })
```

**Trap:** `$in` uses an index efficiently. But `$nin` does not — it typically does a collection scan or index scan over everything not matched.

### Logical operators

```js
// $and — implicit (comma-separated filters are ANDed)
db.users.find({ role: "engineer", status: "active" })

// $and — explicit (useful when same field in multiple conditions)
db.users.find({
  $and: [
    { price: { $gte: 10 } },
    { price: { $lte: 50 } }
  ]
})

// $or
db.users.find({
  $or: [
    { role: "engineer" },
    { role: "manager" }
  ]
})

// $not
db.users.find({ price: { $not: { $gte: 100 } } })

// $nor — neither condition matches
db.users.find({
  $nor: [
    { status: "archived" },
    { deleted: true }
  ]
})
```

**Trap:** `$or` with indexes: if each clause of `$or` can use an index, MongoDB processes them in parallel and merges. If no index covers all clauses, it does a collection scan.

### Element operators

```js
// $exists — field exists (even with null value)
db.users.find({ middle_name: { $exists: false } })

// $type — match by BSON type
db.users.find({ age: { $type: "int" } })  // Only int32, not double
db.users.find({ age: { $type: ["int", "double"] } })
```

### Evaluation operators

```js
// $regex
db.users.find({ name: { $regex: /^alice/i } })

// $expr — use aggregation expressions within query
db.orders.find({
  $expr: { $gt: ["$total", "$limit"] }
})

// $mod — modulo
db.orders.find({ qty: { $mod: [4, 0] } })  // qty % 4 == 0
```

**Trap:** `$regex` with case-insensitive flag (`/i`) uses indexes only if the index is recognized as having the same collation. Unanchored regex patterns (`/pattern/` without `^`) cannot use indexes efficiently. Prefer text indexes for full-text search.

### Array operators

```js
// $all — array contains all specified elements
db.posts.find({ tags: { $all: ["mongodb", "database"] } })

// $elemMatch — array contains element matching all conditions
db.products.find({
  sizes: { $elemMatch: { size: "M", qty: { $gt: 5 } } }
})

// $size — array has exact length
db.posts.find({ tags: { $size: 3 } })
```

**Trap:** `$size` cannot use an index. If you need to query by array length, maintain a derived `tags_count` field and index that.

### Projection operators

```js
// $ — project first matching array element
db.students.find(
  { grades: { $gte: 90 } },
  { "grades.$": 1 }
)

// $elemMatch — project first matching array element with conditions
db.students.find(
  {},
  { grades: { $elemMatch: { $gte: 90 } } }
)

// $slice — project a slice of array
db.posts.find({}, { comments: { $slice: 10 } })        // first 10
db.posts.find({}, { comments: { $slice: -5 } })        // last 5
db.posts.find({}, { comments: { $slice: [20, 10] } })  // skip 20, take 10
```

---

## 4. Basic aggregation pipeline

The aggregation pipeline processes documents through a sequence of stages. Each stage transforms the documents passing through it.

```js
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$amount" },
      orderCount: { $count: {} }
    }
  },
  { $sort: { totalSpent: -1 } },
  { $limit: 10 }
])
```

### Core stages

#### `$match`
Filters documents (like `find()`). Always put `$match` as early as possible in the pipeline to reduce the number of documents flowing to subsequent stages.

**Trap:** `$match` after `$group` must operate on grouped fields. A `$match` before `$group` uses indexes; after `$group` it does not.

#### `$project`
Reshapes documents — include, exclude, add, or rename fields.

```js
db.users.aggregate([
  { $project: {
      _id: 0,
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      isSenior: { $gte: ["$yearsOfExperience", 8] }
    }
  }
])
```

#### `$group`
Groups documents by a specified expression and produces one output document per group.

```js
db.orders.aggregate([
  { $group: {
      _id: { year: { $year: "$createdAt" }, month: { $month: "$createdAt" } },
      totalRevenue: { $sum: "$amount" },
      avgOrderValue: { $avg: "$amount" },
      orders: { $push: "$orderId" },
      firstOrder: { $first: "$createdAt" },
      lastOrder: { $last: "$createdAt" }
    }
  }
])
```

Accumulators: `$sum`, `$avg`, `$min`, `$max`, `$push` (array), `$addToSet` (unique array), `$first`, `$last`, `$stdDevPop`, `$stdDevSamp`.

**Trap:** `$push` on a large group can produce arrays that exceed the 16 MB document limit. Use `$addToSet` or limit with `$slice` in `$project`.

#### `$sort`
Sorts documents. Uses 32 MB RAM limit, spill to disk with `allowDiskUse`.

```js
db.orders.aggregate([
  { $sort: { totalSpent: -1 } }
], { allowDiskUse: true })
```

#### `$limit`, `$skip`
Limit/skip documents in the pipeline.

```js
{ $skip: 100 },
{ $limit: 20 }
```

#### `$unwind`
Deconstructs an array field into multiple documents (one per array element).

```js
// Input: { name: "Alice", tags: ["mongodb", "nosql"] }
// After $unwind: two documents
// { name: "Alice", tags: "mongodb" }
// { name: "Alice", tags: "nosql" }
db.users.aggregate([
  { $unwind: { path: "$tags", preserveNullAndEmptyArrays: true } }
])
```

**Trap:** `$unwind` on a field that doesn't exist or is null removes the document by default. Always check if `preserveNullAndEmptyArrays` is needed.

#### `$lookup`
Left outer join with another collection (akin to SQL JOIN).

```js
db.orders.aggregate([
  { $lookup: {
      from: "customers",          // target collection
      localField: "customerId",   // field in orders
      foreignField: "_id",        // field in customers
      as: "customer"              // output array
    }
  },
  { $unwind: "$customer" }       // convert array to embedded object
])
```

**Trap:** `$lookup` is powerful but expensive — it does one query per document from the pipeline. For large datasets, it can be very slow. Use sparingly, and ensure indexes on `foreignField`. Consider embedding rather than referencing if you always need the joined data.

**`$lookup` with pipeline (MongoDB 3.6+):**

```js
db.orders.aggregate([
  { $lookup: {
      from: "reviews",
      let: { productId: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$productId", "$$productId"] } } },
        { $sort: { createdAt: -1 } },
        { $limit: 3 }
      ],
      as: "recentReviews"
    }
  }
])
```

#### `$bucket`, `$bucketAuto`
Categorize documents into buckets (histogram).

```js
db.products.aggregate([
  { $bucket: {
      groupBy: "$price",
      boundaries: [0, 10, 50, 100, 500, 1000],
      default: "Other",
      output: {
        count: { $sum: 1 },
        products: { $push: "$name" }
      }
    }
  }
])
```

#### `$facet`
Run multiple pipelines in parallel on the same input documents.

```js
db.orders.aggregate([
  { $facet: {
      "byCategory": [
        { $group: { _id: "$category", count: { $sum: 1 } } }
      ],
      "byStatus": [
        { $group: { _id: "$status", count: { $sum: 1 } } }
      ],
      "totals": [
        { $group: { _id: null, total: { $sum: "$amount" } } }
      ]
    }
  }
])
```

**Trap:** `$facet` can produce very large intermediate documents; it's limited by the 16 MB document limit. Use projections early to reduce field sizes.

#### `$addFields`, `$set`
Add new fields or modify existing fields.

```js
db.orders.aggregate([
  { $addFields: {
      totalWithTax: { $multiply: ["$total", 1.08] },
      orderYear: { $year: "$createdAt" }
    }
  }
])
```

`$set` is an alias for `$addFields` (MongoDB 4.2+).

#### `$out`, `$merge`
Write pipeline results to a collection.

```js
// $out — replace entire collection
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $out: "customer_totals" }
])

// $merge — merge into existing collection (insert, merge, replace, keepExisting, fail)
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $merge: {
      into: "customer_totals",
      on: "_id",
      whenMatched: "merge",
      whenNotMatched: "insert"
    }
  }
])
```

---

## 5. Basic indexing

### Why indexes in MongoDB work differently

MongoDB stores documents in BSON format. Indexes are B-tree structures that store references to documents. Without an index, MongoDB does a **collection scan** (COLLSCAN) — scanning every document in the collection.

```js
// Create an index
db.users.createIndex({ email: 1 })  // 1 = ascending, -1 = descending

// Create a compound index
db.users.createIndex({ role: 1, status: 1 })

// See indexes
db.users.getIndexes()

// Drop index
db.users.dropIndex({ email: 1 })
```

### Single field index

```js
db.users.createIndex({ email: 1 })
```
Supports queries on `email`, sort on `email`, and distinct on `email`.

### Compound index

```js
db.users.createIndex({ role: 1, status: 1, createdAt: -1 })
```
Supports queries on `role`, `role + status`, `role + status + createdAt`. Does NOT support queries on `status` alone or `status + createdAt` without `role`.

**Trap:** The order of fields in a compound index matters. MongoDB applies the **ESR rule** (Equality, Sort, Range):
- Fields tested for equality first
- Fields used for sorting next
- Fields used for range queries last

```js
// Query: find engineers with salary > 100000, ordered by name
// Index: { role: 1, name: 1, salary: 1 }
//        ^equal  ^sort     ^range
db.users.find({ role: "engineer", salary: { $gt: 100000 } }).sort({ name: 1 })
```

### Unique index

```js
db.users.createIndex({ email: 1 }, { unique: true })
```

**Trap:** Unique indexes treat `null` as a value. If you insert multiple documents with `{ email: null }` or missing the `email` field, only the first succeeds.

```js
// Solution: partial + unique
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { email: { $exists: true, $ne: null } } }
)
```

### Index properties

```js
// Background index build (non-blocking)
db.users.createIndex({ name: 1 }, { background: true })

// TTL index — auto-delete documents
db.sessions.createIndex({ lastAccessed: 1 }, { expireAfterSeconds: 3600 })

// Sparse index — only index documents containing the field
db.users.createIndex({ twitterHandle: 1 }, { sparse: true })

// Partial index — index based on a filter
db.orders.createIndex(
  { status: 1, createdAt: 1 },
  { partialFilterExpression: { status: "pending" } }
)
```

### explain() basics

```js
db.users.find({ email: "alice@example.com" }).explain("executionStats")
```

Key metrics:
- `winningPlan`: The plan MongoDB chose
- `rejectedPlans`: Plans that were considered but rejected
- `executionTimeMillis`: How long it took
- `totalDocsExamined`: Documents scanned
- `totalKeysExamined`: Index keys scanned
- `nReturned`: Documents returned

**Trap:** `executionStats` only shows the winning plan's execution. Use `allPlansExecution` to see details for all plans.

### Explain output — COLLSCAN vs IXSCAN

```
COLLSCAN  — collection scan: scanned all documents (bad)
IXSCAN    — index scan: scanned index keys (good)
FETCH     — fetched documents from heap (needed when index doesn't cover query)
SORT      — in-memory sort (bad when slow — needs index)
```

---

## 6. MongoDB vs Relational databases

| Aspect | MongoDB (Document) | PostgreSQL (Relational) |
|--------|-------------------|------------------------|
| Schema | Dynamic (app-enforced) | Fixed (DB-enforced) |
| Query | JSON-like query language, aggregation pipeline | SQL |
| Joins | `$lookup` (slow, one query per doc) | JOIN (optimized) |
| Indexes | B+Tree based | B+Tree, Hash, GiST, GIN, BRIN |
| Transactions | Multi-document ACID (4.0+) | ACID from start |
| Modeling | Embed or reference | Normalize with foreign keys |
| Scalability | Native sharding (horizontal) | Partitioning + read replicas (vertical first) |
| Consistency | Tunable (write concern) | Strong consistency |
| Use case | Flexible schema, embedded data, write-heavy, rapid prototyping | Relational data, complex queries, joins, consistency-critical |

---

## 7. Drivers and connection handling

### Node.js driver

```js
const { MongoClient } = require('mongodb')

const client = new MongoClient('mongodb://localhost:27017', {
  maxPoolSize: 50,        // connection pool
  minPoolSize: 5,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  retryWrites: true,
  retryReads: true
})

async function run() {
  try {
    await client.connect()
    const db = client.db('myapp')
    const users = db.collection('users')
    const user = await users.findOne({ email: 'alice@example.com' })
    console.log(user)
  } finally {
    await client.close()
  }
}
```

### Go driver (mongo-go-driver)

```go
import (
    "context"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

client, err := mongo.Connect(context.Background(), options.Client().
    ApplyURI("mongodb://localhost:27017").
    SetMaxPoolSize(50))

collection := client.Database("myapp").Collection("users")

var user struct {
    ID    primitive.ObjectID `bson:"_id"`
    Name  string             `bson:"name"`
    Email string             `bson:"email"`
}

err = collection.FindOne(context.Background(), bson.M{"email": "alice@example.com"}).Decode(&user)
```

### Connection pooling

Default pool size in MongoDB drivers:
- Node.js: `maxPoolSize: 100` (4.0+)
- Go: `SetMaxPoolSize(100)`

**Trap:** Connection pool exhaustion behaves differently from PostgreSQL — MongoDB returns an error when all connections are in use (no queuing). Always set `waitQueueTimeoutMS` (default: 0 = no timeout = infinite wait).

```js
const client = new MongoClient(uri, {
  maxPoolSize: 50,
  waitQueueTimeoutMS: 5000  // wait max 5 seconds for a connection
})
```

---

## 8. Important gotchas for senior interviews

### 1. "MongoDB is eventually consistent" — it depends

- **By default:** reads go to primary (strong consistency)
- **With secondary read preference:** eventually consistent (lag behind primary)
- **With `w: "majority"` write concern:** safe, acknowledged by majority of replica set
- **With `w: 1` write concern + primary read preference:** strong consistency for that session, but write may not survive a primary election

### 2. MongoDB transactions are not free

Multi-document ACID transactions exist since 4.0, but:
- They require replica sets
- They have a 60-second default timeout
- They lock all involved documents
- Throughput is significantly lower than single-document operations
- The WiredTiger cache must be large enough to hold all modified documents

### 3. Aggregation pipeline memory

All stages combined must fit within 100 MB of RAM by default (set with `allowDiskUse: true` to bypass).

### 4. `$lookup` vs embedding

At senior level, the question isn't "how do I join in MongoDB?" but "should I even need to join?" If you're joining frequently, you might have the wrong schema — or the wrong database.

### 5. "MongoDB is web scale" stereotype

Be prepared for the anti-MongoDB bias in FAANG interviews. Many interviewers experienced MongoDB in its early days (pre-3.0) when it had serious issues (global lock, no transactions, mmap storage engine, data loss under replication). Acknowledge the history, then articulate how much has changed (WiredTiger, document-level locking, ACID, change streams, mature sharding).

---

## 9. Practical Drills

### Drill 1 — Write a query

You have a `products` collection:
```json
{ "_id": 1, "name": "Widget", "category": "tools", "price": 29.99, "tags": ["metal", "small"], "stock": { "warehouseA": 50, "warehouseB": 10 } }
```

Write queries for:
1. All products in "tools" category with price > 20
2. Products tagged with both "metal" and "small"
3. Products where stock in warehouseA is less than warehouseB
4. Products sorted by price descending, skip 20, limit 10

<details>
<summary>Answers</summary>

```js
// 1
db.products.find({ category: "tools", price: { $gt: 20 } })

// 2
db.products.find({ tags: { $all: ["metal", "small"] } })

// 3
db.products.find({
  $expr: { $lt: ["$stock.warehouseA", "$stock.warehouseB"] }
})

// 4
db.products.find().sort({ price: -1 }).skip(20).limit(10)
```
</details>

### Drill 2 — Aggregation pipeline

You have an `orders` collection:
```json
{ "_id": 1, "customerId": "C1", "items": [ { "product": "Widget", "qty": 2, "price": 20 } ], "total": 40, "status": "shipped", "createdAt": ISODate("2024-01-15") }
```

Write an aggregation that:
1. Filters orders from January 2024 with status "shipped"
2. Unwinds items
3. Groups by product to get total quantity sold
4. Sorts by total quantity descending
5. Returns top 5 products

<details>
<summary>Answer</summary>

```js
db.orders.aggregate([
  { $match: {
      status: "shipped",
      createdAt: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2024-02-01")
      }
    }
  },
  { $unwind: "$items" },
  { $group: {
      _id: "$items.product",
      totalQty: { $sum: "$items.qty" }
    }
  },
  { $sort: { totalQty: -1 } },
  { $limit: 5 }
])
```
</details>

### Drill 3 — Index selection

Given:
```js
db.orders.createIndex({ customerId: 1, createdAt: -1, status: 1 })
```

Which of these queries can use the index efficiently?

```js
// A
db.orders.find({ customerId: "C1" }).sort({ createdAt: -1 })

// B
db.orders.find({ status: "shipped" }).sort({ createdAt: -1 })

// C
db.orders.find({ customerId: "C1", status: "shipped" }).sort({ createdAt: -1 })

// D
db.orders.find({ customerId: "C1", createdAt: { $gt: ISODate("2024-01-01") } })
```

<details>
<summary>Answer</summary>

- **A**: Yes — uses prefix `customerId` for equality, `createdAt` for sort (ESR)
- **B**: No — `status` is not a prefix of the index; would need separate index on `{ status, createdAt }`
- **C**: Yes — `customerId` (equality) → `status` (equality) → `createdAt` (sort). All three match ESR.
- **D**: Yes — `customerId` (equality) → `createdAt` (range)
</details>

---

## Interview traps cheatsheet

| Trap | The truth |
|------|-----------|
| "MongoDB is schemaless, so I don't need to plan schema" | Schema is enforced at app level. You still need to design it. |
| "ObjectIds are always sortable" | Only approximately — same-second inserts have random ordering. |
| "Compound indexes automatically support all prefix combinations" | True — but suffix-only queries cannot use the index. |
| "$lookup is like SQL JOIN" | It's per-document — much slower for large result sets. |
| "MongoDB can replace PostgreSQL" | Wrong tool for relational data with complex joins. |
| "16 MB limit means I can't store large data" | 16 MB per document; reference large data via GridFS. |
| "Transactions in MongoDB work just like SQL" | 60s timeout, document-level locking, WiredTiger cache dependent. |
| "Replica sets automatically handle all failover" | Rollback can occur; clients may see stale data during election. |
| "I don't need indexes for my aggregation pipeline" | $match + $sort without indexes uses 32 MB RAM limit. |
| "MongoDB is slower than PostgreSQL" | For embedded data access patterns, MongoDB can be faster — no joins needed. |
</details>
