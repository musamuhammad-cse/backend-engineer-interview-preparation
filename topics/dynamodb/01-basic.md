# DynamoDB — Basic

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** DynamoDB fundamentals — tables, items, primary key, CRUD, RCU/WCU, query & scan, basic indexes  
> **Trap:** Most candidates treat DynamoDB like a traditional database. At the senior level, you must understand that **DynamoDB requires you to design for access patterns first**, not schema first.

---

## 1. Core concepts

### What is DynamoDB?

DynamoDB is a fully managed NoSQL key-value and document database by AWS. Key properties:

- **Serverless**: No servers to manage (fully managed)
- **Key-value + document**: Items are key-value pairs or JSON documents (max 400 KB)
- **Multi-AZ by default**: Data replicated across 3 AZs in a region
- **Single-digit millisecond latency**: For eventually consistent reads
- **Automatic scaling**: Partitions are managed transparently
- **Strong consistency option**: For reads that need the latest data

### Core components

| Term | Definition |
|------|------------|
| **Table** | A collection of items (like a table in SQL, but schema-flexible) |
| **Item** | A single record (like a row, but different items can have different attributes) |
| **Attribute** | A key-value pair (like a column) |
| **Primary key** | Uniquely identifies each item (like a primary key in SQL) |
| **Partition key** | (Hash key) — used to distribute data across partitions |
| **Sort key** | (Range key) — items with same partition key are sorted by sort key |
| **Local Secondary Index (LSI)** | Same PK, different SK. Created at table creation. |
| **Global Secondary Index (GSI)** | Different PK + SK. Can be created anytime. |

### Primary key types

**Option 1: Partition key only (simple primary key)**
```
Table: Users
Partition key: user_id (String)
Items are uniquely identified by partition key alone.

Example:
{ "user_id": "u1001", "name": "Alice", "email": "alice@example.com" }
{ "user_id": "u1002", "name": "Bob",   "email": "bob@example.com" }
```

**Option 2: Partition key + Sort key (composite primary key)**
```
Table: Orders
Partition key: user_id (String)
Sort key: order_date (String)
Items with same user_id are sorted by order_date.

Example:
{ "user_id": "u1001", "order_date": "2024-01-15", "order_id": "ORD-001", "total": 100 }
{ "user_id": "u1001", "order_date": "2024-06-01", "order_id": "ORD-002", "total": 50 }
{ "user_id": "u1002", "order_date": "2024-03-10", "order_id": "ORD-003", "total": 75 }
```

**Trap:** The **partition key** is the only field that determines which partition an item lives on. If you use only a partition key (no sort key), that partition key must be **unique** across the table.

---

## 2. CRUD operations (AWS SDK Node.js examples)

```js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb')
const { DynamoDBDocumentClient, PutCommand, GetCommand, UpdateCommand, DeleteCommand, QueryCommand, ScanCommand } = require('@aws-sdk/lib-dynamodb')

const client = new DynamoDBClient({ region: 'us-east-1' })
const docClient = DynamoDBDocumentClient.from(client)
```

### Create / PutItem

```js
// PutItem — creates or replaces an item
await docClient.send(new PutCommand({
  TableName: 'Users',
  Item: {
    user_id: 'u1001',
    name: 'Alice',
    email: 'alice@example.com',
    role: 'engineer',
    created_at: '2024-01-15'
  }
}))

// PutItem with condition — only write if item doesn't exist
await docClient.send(new PutCommand({
  TableName: 'Users',
  Item: { user_id: 'u1001', name: 'Alice' },
  ConditionExpression: 'attribute_not_exists(user_id)'
}))

// PutItem with return values
const result = await docClient.send(new PutCommand({
  TableName: 'Users',
  Item: { user_id: 'u1001', name: 'Alice' },
  ReturnValues: 'ALL_OLD'  // returns previous item if it existed
}))
```

### Read / GetItem

```js
// GetItem — get by primary key
const result = await docClient.send(new GetCommand({
  TableName: 'Users',
  Key: { user_id: 'u1001' }
}))
console.log(result.Item)  // { user_id: 'u1001', name: 'Alice', ... }

// GetItem with specific attributes
const result = await docClient.send(new GetCommand({
  TableName: 'Users',
  Key: { user_id: 'u1001' },
  ProjectionExpression: 'user_id, name, email'
}))

// GetItem with strong consistency
const result = await docClient.send(new GetCommand({
  TableName: 'Users',
  Key: { user_id: 'u1001' },
  ConsistentRead: true  // strongly consistent (default: eventually consistent)
}))
```

### Update / UpdateItem

```js
// UpdateItem — partial update (creates item if doesn't exist)
await docClient.send(new UpdateCommand({
  TableName: 'Users',
  Key: { user_id: 'u1001' },
  UpdateExpression: 'SET role = :role, updated_at = :updated',
  ExpressionAttributeValues: {
    ':role': 'senior_engineer',
    ':updated': '2024-06-01'
  }
}))

// Update with condition — only update if role is currently 'engineer'
await docClient.send(new UpdateCommand({
  TableName: 'Users',
  Key: { user_id: 'u1001' },
  UpdateExpression: 'SET role = :new_role',
  ConditionExpression: 'role = :current_role',
  ExpressionAttributeValues: {
    ':new_role': 'senior_engineer',
    ':current_role': 'engineer'
  }
}))

// Atomic counter increment
await docClient.send(new UpdateCommand({
  TableName: 'Orders',
  Key: { order_id: 'ORD-001' },
  UpdateExpression: 'ADD version_count :inc',
  ExpressionAttributeValues: {
    ':inc': 1
  }
}))
```

### Delete / DeleteItem

```js
// DeleteItem
await docClient.send(new DeleteCommand({
  TableName: 'Users',
  Key: { user_id: 'u1001' }
}))

// Delete with condition
await docClient.send(new DeleteCommand({
  TableName: 'Orders',
  Key: { user_id: 'u1001', order_date: '2024-01-15' },
  ConditionExpression: 'attribute_exists(order_id)'
}))
```

### Batch operations

```js
// BatchGetItem — get multiple items (up to 100 items or 16 MB)
const result = await docClient.send(new BatchGetCommand({
  RequestItems: {
    Users: {
      Keys: [
        { user_id: 'u1001' },
        { user_id: 'u1002' },
        { user_id: 'u1003' }
      ],
      ProjectionExpression: 'user_id, name'
    }
  }
}))
console.log(result.Responses.Users)

// BatchWriteItem — batch put/delete (up to 25 items or 16 MB)
await docClient.send(new BatchWriteCommand({
  RequestItems: {
    Users: [
      { PutRequest: { Item: { user_id: 'u1004', name: 'Charlie' } } },
      { PutRequest: { Item: { user_id: 'u1005', name: 'Diana' } } },
      { DeleteRequest: { Key: { user_id: 'u1001' } } }
    ]
  }
}))
```

---

## 3. Query

Query returns items with the **same partition key**, optionally filtered/sorted by sort key.

```js
// Basic query by partition key
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid',
  ExpressionAttributeValues: {
    ':uid': 'u1001'
  }
}))

// Query with sort key conditions
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid AND order_date BETWEEN :start AND :end',
  ExpressionAttributeValues: {
    ':uid': 'u1001',
    ':start': '2024-01-01',
    ':end': '2024-12-31'
  }
}))

// Sort key functions:
//   =, <, <=, >, >=, BETWEEN, BEGINS_WITH
//   sort_key = 'value'
//   sort_key BETWEEN 'a' AND 'z'
//   begins_with(sort_key, :prefix)

// Query with filter (applied AFTER data is read — does not reduce RCU)
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid',
  FilterExpression: 'total > :min_total',
  ExpressionAttributeValues: {
    ':uid': 'u1001',
    ':min_total': 50
  }
}))

// Query pagination
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid',
  Limit: 10,
  ExclusiveStartKey: lastEvaluatedKey  // from previous response
}))
// Response includes LastEvaluatedKey for pagination

// Query with sort order
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid',
  ScanIndexForward: false  // descending order (default: true = ascending)
}))

// Query with index
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  IndexName: 'StatusIndex',  // GSI name
  KeyConditionExpression: 'status = :status',
  ExpressionAttributeValues: {
    ':status': 'shipped'
  }
}))
```

**Trap:** `FilterExpression` is applied after reading data. It does **not** reduce RCU consumption. You pay to read items, then filter them client-side. If you need to reduce RCU on heavily filtered queries, consider a GSI.

---

## 4. Scan

Scan reads every item in a table/index. Expensive — avoid in production for large tables.

```js
// Full scan (consumes RCU for every item)
const result = await docClient.send(new ScanCommand({
  TableName: 'Orders'
}))

// Scan with filter (still reads all items, only returns matching)
const result = await docClient.send(new ScanCommand({
  TableName: 'Orders',
  FilterExpression: 'total > :min',
  ExpressionAttributeValues: {
    ':min': 100
  }
}))

// Scan with limit + pagination
const result = await docClient.send(new ScanCommand({
  TableName: 'Orders',
  Limit: 100,
  ExclusiveStartKey: lastEvaluatedKey
}))

// Parallel scan — for large tables, use segments
const result = await docClient.send(new ScanCommand({
  TableName: 'Orders',
  TotalSegments: 10,     // 10 parallel workers
  Segment: 0              // this worker processes segment 0
}))
```

**Trap:** A Scan on a large table can consume all provisioned RCU, throttling other operations. Use Scans sparingly, prefer GSIs for alternative access patterns. If you need to scan, use parallel scans with segments.

---

## 5. Capacity — RCU, WCU

### Read Capacity Unit (RCU)

| Read type | Item size | RCU consumed |
|-----------|-----------|--------------|
| Eventually consistent | 4 KB | 0.5 RCU |
| Strongly consistent | 4 KB | 1 RCU |
| Transactional | 4 KB | 2 RCU |

**Formula:** `RCU = ceil(item_size / 4 KB) × (1 for strong, 0.5 for eventual, 2 for transactional)`

Example: Reading a 10 KB item strongly consistent:
- Ceil(10 / 4) = 3 × 1 = **3 RCU**

### Write Capacity Unit (WCU)

| Write type | Item size | WCU consumed |
|------------|-----------|--------------|
| Standard write | 1 KB | 1 WCU |
| Transactional | 1 KB | 2 WCU |

**Formula:** `WCU = ceil(item_size / 1 KB)`

Example: Writing a 2.5 KB item:
- Ceil(2.5 / 1) = **3 WCU**

### Provisioned vs On-demand capacity

| Feature | Provisioned | On-demand |
|---------|-------------|-----------|
| Pricing | Pay per RCU/WCU per hour | Pay per request |
| Scaling | Manual or auto-scaling | Automatic (2x per minute) |
| Throttling | If exceeded | Never (within soft limits) |
| Cost | Lower for predictable workloads | Higher for predictable, better for spiky |
| Burst capacity | Yes (5 min accumulated unused) | N/A |

```bash
# Create table with provisioned capacity
aws dynamodb create-table \
  --table-name Orders \
  --key-schema AttributeName=user_id,KeyType=HASH AttributeName=order_date,KeyType=RANGE \
  --attribute-definitions AttributeName=user_id,AttributeType=S AttributeName=order_date,AttributeType=S \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5

# Update capacity
aws dynamodb update-table \
  --table-name Orders \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10
```

### Burst capacity

Unused RCU/WCU accumulates up to **5 minutes** of capacity:

```
If provisioned = 100 RCU, and you use 60 RCU for 5 minutes:
Burst = (100 - 60) × 300 seconds = 12,000 burst RCU
```

**Trap:** Burst capacity is a buffer, not a solution. If you consistently exceed provisioned capacity, you'll drain burst and get throttled.

---

## 6. Basic indexes

### Local Secondary Index (LSI)

- **Same partition key**, **different sort key**
- Must be created at **table creation time**
- Max **5 LSIs** per table
- Shares throughput with base table (no separate capacity)
- Items with the same partition key but different sort key views

```bash
aws dynamodb create-table \
  --table-name Orders \
  --key-schema AttributeName=user_id,KeyType=HASH AttributeName=order_date,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=user_id,AttributeType=S \
    AttributeName=order_date,AttributeType=S \
    AttributeName=status,AttributeType=S \
  --local-secondary-indexes \
    "[{\"IndexName\": \"OrderStatusIndex\", \"KeySchema\": [{\"AttributeName\":\"user_id\",\"KeyType\":\"HASH\"},{\"AttributeName\":\"status\",\"KeyType\":\"RANGE\"}], \"Projection\": {\"ProjectionType\":\"ALL\"}}]"
```

### Global Secondary Index (GSI)

- **Different partition key** (and optionally different sort key)
- Can be created **anytime** (even after table creation)
- Max **20 GSIs** per table default (can request more)
- **Separate throughput** from base table (incur additional cost)
- Enables query on attributes that aren't the table's primary key

```js
// Create GSI via AWS CLI
aws dynamodb update-table \
  --table-name Orders \
  --attribute-definitions AttributeName=status,AttributeType=S AttributeName=order_date,AttributeType=S \
  --global-secondary-index-updates \
    "[{\"Create\": {\"IndexName\": \"StatusByDateIndex\", \"KeySchema\": [{\"AttributeName\":\"status\",\"KeyType\":\"HASH\"},{\"AttributeName\":\"order_date\",\"KeyType\":\"RANGE\"}], \"Projection\": {\"ProjectionType\":\"ALL\"}, \"ProvisionedThroughput\": {\"ReadCapacityUnits\":5, \"WriteCapacityUnits\":5}}}]"
```

### LSI vs GSI comparison

| Feature | LSI | GSI |
|---------|-----|-----|
| Partition key | **Must be same** as table's PK | Can be different |
| Sort key | Different from table's SK | Can be different |
| Created | At table creation only | Anytime |
| Max count | 5 | 20 (default) |
| Throughput | Shares table's capacity | Separate (provisioned) |
| Item limit | Same partition = 10 GB | No limit |
| Consistency | Eventually + Strongly consistent | Eventually consistent only |
| Cost | Included with table RCU/WCU | Additional RCU/WCU |

### Projection types

```bash
# KEYS_ONLY — only partition + sort keys of the index
"Projection": { "ProjectionType": "KEYS_ONLY" }

# INCLUDE — keys + specific attributes
"Projection": { "ProjectionType": "INCLUDE", "NonKeyAttributes": ["name", "status"] }

# ALL — all attributes (default)
"Projection": { "ProjectionType": "ALL" }
```

**Trap:** Using `ALL` projection for a GSI costs more (writes to GSI include all attributes). If you only need a few attributes, use `INCLUDE` to reduce write costs.

---

## 7. Pagination

```js
// First page
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid',
  Limit: 10
}))
// result.Items => array of 0-10 items
// result.LastEvaluatedKey => undefined if no more pages, otherwise key for next page

// Subsequent page
const nextResult = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid',
  Limit: 10,
  ExclusiveStartKey: result.LastEvaluatedKey
}))
```

---

## 8. Conditional expressions

Conditions are evaluated **before** the write operation:

```js
// Only create if item doesn't exist
ConditionExpression: 'attribute_not_exists(user_id)'

// Only update if item exists
ConditionExpression: 'attribute_exists(user_id)'

// Only update if attribute equals value
ConditionExpression: 'status = :current_status'
ExpressionAttributeValues: { ':current_status': 'pending' }

// Only update if attribute > value
ConditionExpression: 'version = :expected_version'
ExpressionAttributeValues: { ':expected_version': 3 }

// Compound conditions
ConditionExpression: 'status = :s AND attribute_exists(created_at)'

// Check attribute types
ConditionExpression: 'attribute_type(price, :num_type)'
ExpressionAttributeValues: { ':num_type': 'N' }

// Condition on size
ConditionExpression: 'size(description) < :max'
ExpressionAttributeValues: { ':max': 100 }

// IF_NOT_EXISTS in update expression
UpdateExpression: 'SET #c = if_not_exists(#c, :default)'
```

**Trap:** If a conditional write fails, DynamoDB returns a `ConditionalCheckFailedException`. It does **not** consume write capacity (WCU), but it does consume **read capacity** for the condition check.

---

## 9. ReturnValues

```js
// Default — returns no attributes
ReturnValues: 'NONE'

// Returns item before modification
ReturnValues: 'ALL_OLD'
// UpdateCommand response: { Attributes: { user_id: 'u1', name: 'Alice', role: 'engineer' } }

// Returns updated attributes (before update)
ReturnValues: 'UPDATED_OLD'

// Returns item after modification
ReturnValues: 'ALL_NEW'

// Returns updated attributes (after update)
ReturnValues: 'UPDATED_NEW'
```

---

## 10. Item size and attribute values

- **Maximum item size**: 400 KB (including attribute names)
- Attribute names count toward the 400 KB limit

```js
// Good — short attribute names save space
{ "uid": "u1001", "fn": "Alice", "ln": "Smith" }

// Bad — long attribute names waste space
{ "user_id": "u1001", "first_name": "Alice", "last_name": "Smith" }
```

**Trap:** In single-table design (covered in Intermediate/Senior), attribute names like `PK`, `SK`, `GSI1PK`, `GSI1SK` are common to save space and maintain consistency.

---

## 11. Practical Drills

### Drill 1 — Calculate capacity

You have a table with the following:
- Item size: 6 KB (including attribute names)
- 100 reads/second (eventually consistent)
- 50 writes/second

How many RCU and WCU do you need?

<details>
<summary>Answer</summary>

**Reads:**
- Item size = 6 KB → ceil(6/4) = 2 RCU per strongly consistent read
- Eventually consistent = 0.5 × 2 = 1 RCU per read
- 100 reads/sec = 100 RCU

**Writes:**
- Item size = 6 KB → ceil(6/1) = 6 WCU per write
- 50 writes/sec = 300 WCU

**Total: 100 RCU, 300 WCU**
</details>

### Drill 2 — Query vs Scan

You need to find all orders for user `u1001` from January 2024 with total > $50. Which approach is better?

<details>
<summary>Answer</summary>

**Best approach: Query with filter**

```js
const result = await docClient.send(new QueryCommand({
  TableName: 'Orders',
  KeyConditionExpression: 'user_id = :uid AND order_date BETWEEN :start AND :end',
  FilterExpression: 'total > :min_total',
  ExpressionAttributeValues: {
    ':uid': 'u1001',
    ':start': '2024-01-01',
    ':end': '2024-01-31',
    ':min_total': 50
  }
}))
```

Query reads only items with `user_id = u1001` (efficient partition access). Filter removes low-value items from results. A Scan would read every item in the table — expensive.
</details>

### Drill 3 — Create a table with LSI

Create a table `Orders` with partition key `user_id` and sort key `order_date`. Add an LSI `StatusIndex` with sort key `status`.

<details>
<summary>Answer</summary>

```bash
aws dynamodb create-table \
  --table-name Orders \
  --key-schema AttributeName=user_id,KeyType=HASH AttributeName=order_date,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=user_id,AttributeType=S \
    AttributeName=order_date,AttributeType=S \
    AttributeName=status,AttributeType=S \
  --local-secondary-indexes \
    "[{\"IndexName\":\"StatusIndex\",\"KeySchema\":[{\"AttributeName\":\"user_id\",\"KeyType\":\"HASH\"},{\"AttributeName\":\"status\",\"KeyType\":\"RANGE\"}],\"Projection\":{\"ProjectionType\":\"INCLUDE\",\"NonKeyAttributes\":[\"total\"]}}]" \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10
```
</details>

---

## Interview traps cheatsheet — Basic

| Trap | The truth |
|------|-----------|
| "DynamoDB is just a key-value store" | It supports both key-value (simple PK) and document (PK + SK) models with rich query capabilities. |
| "Scan is fine for small tables" | Scan reads every item and consumes RCU for each. Even on small tables, it's wasteful. Prefer Query. |
| "FilterExpression reduces cost" | Filter is applied after reading data. You still pay for all read RCU. |
| "PutItem always replaces the item" | It does, but `ConditionExpression: attribute_not_exists(pk)` makes it fail if item exists. |
| "Batch operations are atomic" | BatchGetItem and BatchWriteItem are **not** atomic. Some operations can fail, others succeed. |
| "LSI and GSI are the same" | LSI: same PK, created at table, shares throughput, strong consistent reads are possible. GSI: different PK, anytime creation, separate throughput, eventually consistent only. |
| "Projection ALL is always best" | ALL projection writes all attributes to the index → higher WCU. Use INCLUDE if you only need a few fields. |
| "DynamoDB is cheap" | At scale, on-demand can be expensive. Provisioned with auto-scaling is more cost-effective for predictable loads. |
| "Strongly consistent reads are always better" | Strong reads use 2x RCU and can fail during partition moves. Use eventual for most cases. |
| "I can add a sort key later" | No. Partition key and sort key are fixed at table creation. You can add GSIs for new access patterns. |
</details>
