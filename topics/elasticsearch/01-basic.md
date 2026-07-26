# Elasticsearch — Basic

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Elasticsearch fundamentals — inverted index, documents, mappings, search basics, query DSL, relevance, aggregation basics  
> **Trap:** Most candidates treat ES like a database with a search feature. At the senior level, you must understand the *inverted index* and *distributed nature* as the foundation for everything else.

---

## 1. Core concepts

### What is Elasticsearch?

Elasticsearch is a **distributed, RESTful search and analytics engine** built on Apache Lucene. Key properties:

- **Document-oriented**: Stores data as JSON documents
- **Schema-flexible**: Mappings can be dynamic or explicit
- **Near-real-time**: Documents become searchable within ~1 second (configurable refresh_interval)
- **Distributed**: Shards are distributed across nodes automatically
- **Full-text search**: Powered by Lucene's inverted index + BM25 ranking
- **Analytics**: Rich aggregation framework for real-time analytics
- **REST API**: All operations via HTTP JSON

### Core components

| Term | Definition | Analogy |
|------|------------|---------|
| **Cluster** | A collection of connected nodes | A distributed database |
| **Node** | A single Elasticsearch instance | A database server |
| **Index** | A collection of documents | A database/table |
| **Shard** | A Lucene index — the unit of distribution | A partition |
| **Primary shard** | The authoritative copy of data | Write partition |
| **Replica shard** | A copy of a primary shard | Read replica |
| **Document** | A JSON object (stored in an index) | A row |
| **Field** | A key-value pair in a document | A column |
| **Mapping** | The schema definition | Table schema |
| **Analysis** | Process of converting text into tokens | Tokenization |

### Inverted index — the foundation

An **inverted index** maps terms (tokens) to the documents that contain them. This is the opposite of a "forward index" (which maps documents to their terms).

```
Document 1: "The quick brown fox"
Document 2: "The lazy dog"

Inverted index:
"brown"  → [Doc1]
"dog"    → [Doc2]
"fox"    → [Doc1]
"lazy"   → [Doc2]
"quick"  → [Doc1]
"the"    → [Doc1, Doc2]
```

**How search works:**
1. Query is analyzed (tokenized + filtered)
2. For each token, Lucene looks up the term dictionary → postings list
3. Postings list gives document IDs containing the term
4. Results are scored (BM25), combined (bool logic), and returned

**Trap:** Unlike a B-tree index (MySQL/PostgreSQL), an inverted index does not store the original field value. For "fetch" operations, ES reads from the stored `_source` field (or doc_values). The inverted index only handles search.

### Cluster health

```bash
GET _cluster/health

# Response:
{
  "cluster_name": "mycluster",
  "status": "yellow",          # green, yellow, or red
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 2,
  "active_primary_shards": 10,
  "active_shards": 20,
  "unassigned_shards": 2,      # replicas that can't be assigned
  "delayed_unassigned_shards": 0
}
```

| Status | Meaning | Action |
|--------|---------|--------|
| **Green** | All primary + replica shards are active | OK |
| **Yellow** | All primaries active, some replicas unassigned | Normal during node loss. Fix: add/recover node |
| **Red** | Some primaries unassigned | Data loss or node failure. Fix: restore/reallocate |

---

## 2. REST API basics

All operations are via HTTP:

```bash
# Index a document (create or update)
PUT /products/_doc/1
{
  "name": "Widget",
  "price": 29.99,
  "category": "tools",
  "tags": ["metal", "small"],
  "in_stock": true,
  "created_at": "2024-01-15"
}

# Get document
GET /products/_doc/1

# Check document exists
HEAD /products/_doc/1

# Update document (partial)
POST /products/_update/1
{
  "doc": { "price": 24.99 }
}

# Delete document
DELETE /products/_doc/1

# Create (fail if exists)
PUT /products/_create/1
{ "name": "New Widget" }

# Bulk operations
POST /_bulk
{"index": {"_index": "products", "_id": "2"}}
{"name": "Gadget", "price": 49.99}
{"index": {"_index": "products", "_id": "3"}}
{"name": "Doohickey", "price": 9.99}

# Search
GET /products/_search
{
  "query": {
    "match": { "name": "widget" }
  }
}

# Count
GET /products/_count
{
  "query": { "match_all": {} }
}
```

**Trap:** `PUT /products/_doc/1` is **idempotent** — it replaces the entire document. `POST /products/_update/1` is partial. `PUT /products/_create/1` fails if the document already exists.

### Response structure

```json
{
  "took": 5,              // milliseconds
  "timed_out": false,
  "_shards": {
    "total": 2,
    "successful": 2,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"    // eq = exact, gte = approximate (for large sets)
    },
    "max_score": 1.2,
    "hits": [
      {
        "_index": "products",
        "_id": "1",
        "_score": 1.2,
        "_source": { "name": "Widget", "price": 29.99 }
      }
    ]
  }
}
```

---

## 3. Mappings

Mappings define how documents and their fields are stored and indexed.

### Dynamic mapping

By default, ES detects field types automatically:

```
"name"   : "hello"       → text (+ keyword sub-field)
"price"  : 29.99         → float
"count"  : 100           → long
"date"   : "2024-01-15"  → date
"active" : true          → boolean
"tags"   : ["a", "b"]    → text (each element analyzed)
```

**Trap:** Dynamic mapping can cause type conflicts. For example, if first document has `"price": "29.99"` (string), and second has `"price": 30` (number), ES will reject the second.

### Explicit mapping

```bash
PUT /products
{
  "mappings": {
    "properties": {
      "name":        { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "description": { "type": "text", "analyzer": "english" },
      "price":       { "type": "double" },
      "category":    { "type": "keyword" },
      "tags":        { "type": "keyword" },
      "in_stock":    { "type": "boolean" },
      "created_at":  { "type": "date", "format": "yyyy-MM-dd" },
      "location":    { "type": "geo_point" },
      "attributes":  { "type": "object" },  # default for objects
      "variants":    { "type": "nested" }   # preserves relationships
    }
  }
}
```

### Field data types

| Type | Description | Use case |
|------|-------------|----------|
| `text` | Full-text, analyzed | Searchable content |
| `keyword` | Exact value, not analyzed | Filtering, sorting, aggregations |
| `long` / `integer` / `short` / `byte` | Integer types | Numeric fields |
| `double` / `float` / `half_float` | Floating point | Prices, scores |
| `boolean` | true/false | Flags |
| `date` | Date (millis since epoch or formatted string) | Timestamps |
| `ip` | IPv4/IPv6 | IP addresses |
| `geo_point` | Latitude/longitude | Location |
| `geo_shape` | Complex shapes | Polygons |
| `nested` | Array of objects (separate Lucene docs) | Preserves object relationships |
| `object` | JSON object (default) | Flattened at indexing |
| `binary` | Base64-encoded binary | Store but not search |
| `join` | Parent-child relation | Document hierarchy |

### Multi-fields

A field can have multiple sub-fields for different purposes:

```bash
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" },  # for exact match/sort
          "english":  { "type": "text", "analyzer": "english" },  # stemming
          "ngram":    { "type": "text", "analyzer": "ngram_analyzer" }  # autocomplete
        }
      }
    }
  }
}
```

### Dynamic templates

```bash
PUT /logs
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword" }
        }
      },
      {
        "longs_as_integer": {
          "match_mapping_type": "long",
          "mapping": { "type": "integer" }
        }
      }
    ]
  }
}
```

---

## 4. Analysis

Analysis is the process of converting text into tokens for the inverted index.

### Analysis pipeline

```
Raw text → Character filters → Tokenizer → Token filters → Index tokens
```

```
Character filter: Strip HTML tags
         ↓
Tokenizer: Split on whitespace/punctuation
         ↓
Token filter: Lowercase, remove stop words, stem
         ↓
Index tokens: ["quick", "brown", "fox"]
```

### Built-in analyzers

| Analyzer | Behavior |
|----------|----------|
| `standard` | Tokenizes on word boundaries, lowercases, removes most punctuation |
| `simple` | Tokenizes on non-letters, lowercases |
| `whitespace` | Tokenizes on whitespace only |
| `keyword` | No tokenization (the whole text is one token) |
| `pattern` | Tokenizes using a regex pattern |
| `language` | Language-specific (english, french, etc.) — applies stemming |
| `fingerprint` | Creates a fingerprint for deduplication |

### Custom analyzer

```bash
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_strip": { "type": "html_strip" }
      },
      "filter": {
        "my_stopwords": {
          "type": "stop",
          "stopwords": ["the", "a", "an", "is"]
        },
        "my_synonyms": {
          "type": "synonym",
          "synonyms": ["quick, fast", "big, large, huge"]
        },
        "my_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "ngram_filter": {
          "type": "ngram",
          "min_gram": 2,
          "max_gram": 3
        }
      },
      "tokenizer": {
        "my_tokenizer": { "type": "standard" }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "my_tokenizer",
          "filter": ["lowercase", "my_stopwords", "my_synonyms", "my_stemmer"]
        },
        "autocomplete": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "ngram_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "my_analyzer" },
      "title_search": { "type": "text", "analyzer": "my_analyzer" },
      "title_suggest": { "type": "text", "analyzer": "autocomplete" }
    }
  }
}
```

### Test an analyzer

```bash
POST /_analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown Foxes!"
}
# Returns: ["the", "quick", "brown", "foxes"]

POST /my_index/_analyze
{
  "field": "title",
  "text": "The Quick Brown Foxes!"
}
# Returns tokens using the field's analyzer

POST /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "stemmer"],
  "text": "Foxes running quickly"
}
# Returns: ["fox", "run", "quickli"]
```

**Trap:** The analyzer used at index time and search time must be consistent. If they differ, you may get unexpected results. `match` queries use the search analyzer from the field mapping; `term` queries use no analyzer (exact match on the indexed token).

---

## 5. Query DSL

### Query vs Filter context

| Context | Scoring | Caching | Use |
|---------|---------|---------|-----|
| **Query** | Yes (calculates `_score`) | Not cached (but results can be) | Full-text search, relevance |
| **Filter** | No (`_score` = 0) | Cached in memory | Structured conditions, yes/no |

```bash
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "widget" } }       # Query context — affects score
      ],
      "filter": [
        { "term": { "category": "tools" } },    # Filter context — no scoring, cached
        { "range": { "price": { "gte": 10, "lte": 100 } } }  # Filter context
      ]
    }
  }
}
```

**Trap:** Filters are cached at the segment level. If a segment changes (new documents), the cache is invalidated and rebuilt. Rarely-changed segments benefit most from filter caching.

### Leaf queries

#### Full-text queries

```bash
# match — analyzes the query text, searches for matches
GET /products/_search
{
  "query": { "match": { "name": "quick brown fox" } }
}

# match_phrase — requires terms in order
GET /products/_search
{
  "query": { "match_phrase": { "description": "quick brown fox" } }
}

# match_phrase_prefix — last term can be a prefix
GET /products/_search
{
  "query": { "match_phrase_prefix": { "name": "quick bro" } }
}

# multi_match — search across multiple fields
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "widget",
      "fields": ["name^3", "description", "category"]  # name gets 3x boost
    }
  }
}

# query_string — powerful but error-prone
GET /products/_search
{
  "query": {
    "query_string": {
      "query": "name:widget AND price:[10 TO 100]"
    }
  }
}

# simple_query_string — safer version
GET /products/_search
{
  "query": {
    "simple_query_string": {
      "query": "widget tools",
      "fields": ["name", "category"],
      "default_operator": "and"
    }
  }
}
```

#### Term-level queries (exact match)

```bash
# term — exact match on the inverted index (use with keyword fields)
GET /products/_search
{
  "query": { "term": { "category": "tools" } }
}

# terms — match any of multiple values
GET /products/_search
{
  "query": { "terms": { "category": ["tools", "hardware"] } }
}

# range
GET /products/_search
{
  "query": { "range": { "price": { "gte": 10, "lte": 100 } } }
}

# exists — field has a non-null value
GET /products/_search
{
  "query": { "exists": { "field": "tags" } }
}

# prefix
GET /products/_search
{
  "query": { "prefix": { "name.keyword": "Wid" } }  # keyword sub-field
}

# wildcard
GET /products/_search
{
  "query": { "wildcard": { "name.keyword": "W*dget" } }
}

# regexp
GET /products/_search
{
  "query": { "regexp": { "name.keyword": "Wid.*" } }
}

# fuzzy — Levenshtein distance
GET /products/_search
{
  "query": { "fuzzy": { "name": "widget" } }
}

# IDs
GET /products/_search
{
  "query": { "ids": { "values": ["1", "2", "3"] } }
}
```

**Trap:** `term` queries on `text` fields usually fail because the query text is NOT analyzed. The query "Quick" won't match the indexed token "quick" (lowercased). Always use `match` for full-text fields and `term` for `keyword` fields.

### Compound queries

```bash
# bool — must, should, filter, must_not
GET /products/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "name": "widget" } } ],
      "should":   [ { "match": { "description": "premium" } } ],  # boosts score
      "filter":   [ { "term": { "category": "tools" } } ],
      "must_not": [ { "term": { "in_stock": false } } ],
      "minimum_should_match": 1  # at least 1 should clause must match
    }
  }
}

# boosting — reduce score of matching docs
GET /products/_search
{
  "query": {
    "boosting": {
      "positive": { "match": { "name": "widget" } },
      "negative": { "term": { "category": "obsolete" } },
      "negative_boost": 0.5
    }
  }
}

# constant_score — wrap a filter in a query context with constant score
GET /products/_search
{
  "query": {
    "constant_score": {
      "filter": { "term": { "category": "tools" } },
      "boost": 1.2
    }
  }
}

# dis_max — take the max score from multiple queries (union)
GET /products/_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "name": "widget" } },
        { "match": { "description": "widget" } }
      ],
      "tie_breaker": 0.3  # add scores from other matching queries
    }
  }
}

# function_score — custom scoring
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "widget" } },
      "functions": [
        { "filter": { "range": { "price": { "lte": 20 } } },
          "weight": 2 },
        { "gauss": {
            "created_at": {
              "origin": "2024-06-01",
              "scale": "30d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply"
    }
  }
}
```

### Pagination

```bash
# from + size — standard pagination (not for deep pagination!)
GET /products/_search
{
  "from": 0,
  "size": 10,
  "query": { "match_all": {} }
}

# search_after — for deep pagination (cursor-based)
GET /products/_search
{
  "size": 10,
  "sort": [ { "price": "asc" }, { "_id": "asc" } ],
  "query": { "match_all": {} }
}
# Use last sort values in next request:
GET /products/_search
{
  "size": 10,
  "search_after": [29.99, "product_42"],
  "sort": [ { "price": "asc" }, { "_id": "asc" } ],
  "query": { "match_all": {} }
}
```

**Trap:** `from + size` has a default limit of 10,000 (index.max_result_window). For deep pagination (e.g., "page 1000"), use `search_after` or `scroll` (for batch processing).

### Sorting

```bash
GET /products/_search
{
  "sort": [
    { "price": { "order": "asc" } },
    { "created_at": { "order": "desc" } },
    "_score"  # default sort
  ],
  "query": { "match": { "name": "widget" } }
}
```

**Trap:** Sorting on `text` fields is not possible (they're analyzed). Use the `.keyword` sub-field or disable `fielddata` on `text`. `doc_values` on `keyword` fields is used for sorting.

---

## 6. Relevance scoring (BM25)

### BM25 formula

Elasticsearch 5.0+ uses **BM25** (Okapi BM25) for relevance scoring:

```
Score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D| / avgdl))
```

Where:
- `f(qi, D)` = term frequency in document
- `k1` = saturation point (default: 1.2)
- `b` = length normalization (default: 0.75)
- `|D|` = document length
- `avgdl` = average document length
- `IDF(qi)` = inverse document frequency

### Explain API

```bash
GET /products/_explain/1
{
  "query": { "match": { "name": "widget" } }
}
# Returns detailed scoring breakdown:
# - score contribution per term
# - IDF, TF, field length normalization
# - query boost
```

### Tuning BM25

```bash
PUT /products/_settings
{
  "index": {
    "similarity": {
      "default": {
        "type": "BM25",
        "k1": 1.2,      # higher = TF has more impact
        "b": 0.75       # higher = more length normalization
      }
    }
  }
}
```

**Trap:** BM25 parameters should be tuned based on your corpus. Short fields (titles): lower `k1`. Long fields (descriptions): higher `k1`. For your product catalog, the defaults work well for most cases.

---

## 7. Basic aggregations

### Metric aggregations

```bash
# Single value
GET /products/_search
{
  "size": 0,
  "aggs": {
    "avg_price":  { "avg": { "field": "price" } },
    "min_price":  { "min": { "field": "price" } },
    "max_price":  { "max": { "field": "price" } },
    "sum_prices": { "sum": { "field": "price" } },
    "count":      { "value_count": { "field": "price" } }
  }
}

# Stats (all in one)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_stats": { "stats": { "field": "price" } }
  }
}

# Extended stats (includes variance, std_dev)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_extended": { "extended_stats": { "field": "price" } }
  }
}

# Percentiles
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_percentiles": {
      "percentiles": {
        "field": "price",
        "percents": [50, 95, 99]
      }
    }
  }
}

# Cardinality (unique count — approximate)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "unique_categories": {
      "cardinality": { "field": "category" }
    }
  }
}
```

**Trap:** `cardinality` is approximate (HyperLogLog-based, ~5% error by default). For exact counts, use a `terms` aggregation with `size` set high (but memory costly).

### Bucket aggregations

```bash
# Terms — group by unique values (like GROUP BY)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category",
        "size": 10,
        "order": { "_count": "desc" }
      }
    }
  }
}

# Range — custom ranges
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "budget", "to": 20 },
          { "key": "mid", "from": 20, "to": 100 },
          { "key": "premium", "from": 100 }
        ]
      }
    }
  }
}

# Date histogram
GET /products/_search
{
  "size": 0,
  "aggs": {
    "products_over_time": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      }
    }
  }
}

# Histogram (numeric)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "price",
        "interval": 10
      }
    }
  }
}
```

### Nested aggregations

```bash
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "top_products": {
          "top_hits": {
            "size": 3,
            "sort": [{ "price": "desc" }]
          }
        }
      }
    }
  }
}
```

---

## 8. Index management

### Create index with settings

```bash
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "max_result_window": 10000
  },
  "mappings": { ... }
}
```

### Index templates

```bash
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "30s"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" }
      }
    }
  },
  "priority": 100
}
```

### Aliases

```bash
# Create alias for single index
PUT /products/_alias/products_read

# Create alias for multiple indices (search across)
POST /_aliases
{
  "actions": [
    { "add": { "index": "products_v1", "alias": "products" } },
    { "add": { "index": "products_v2", "alias": "products" } }
  ]
}

# Filtering alias — search only active products
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "products",
        "alias": "active_products",
        "filter": { "term": { "status": "active" } }
      }
    }
  ]
}

# Zero-downtime reindexing
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}
```

### Reindexing

```bash
POST /_reindex
{
  "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" },
  "script": {
    "source": "ctx._source.price = ctx._source.price * 1.1"
  }
}
```

---

## 9. Node discovery and cluster formation

### Node roles

```yaml
# elasticsearch.yml
node.roles: ["master", "data", "ingest"]  # all roles (default)
node.roles: ["master"]                      # dedicated master
node.roles: ["data"]                        # dedicated data
node.roles: ["ingest"]                      # dedicated ingest pipeline
node.roles: ["ml"]                          # machine learning
node.roles: []                              # coordinating-only (load balancer)
```

### Minimum master nodes

```yaml
# Prevent split-brain — must be majority of master-eligible nodes
discovery.zen.minimum_master_nodes: 2      # for 3 master-eligible nodes
```

### Discovery (7.x+)

```yaml
discovery.seed_hosts:
  - 192.168.1.10:9300
  - 192.168.1.11:9300
  - 192.168.1.12:9300
cluster.initial_master_nodes:
  - node-1
  - node-2
  - node-3
```

---

## 10. Important gotchas for senior interviews

### 1. "Elasticsearch is a database"

It's a **search and analytics engine**, not a primary database. It's eventually consistent (near-real-time), has no transactions, and uses the `_source` field for document storage (not optimized for read-heavy workloads like PostgreSQL).

### 2. "DELETE is immediate"

Deletes are **soft** — documents are marked as deleted and removed during segment merging. Disk space is not released immediately.

### 3. "More shards = more parallelism"

Too many shards = overhead (each shard is a Lucene index with its own memory, file handles). Too few = poor parallelism. Rule: 20-50 GB per shard, max ~1,000 shards per node.

### 4. "Elasticsearch is fast"

ES is fast for full-text search and aggregations on indexed fields. It's **slow** for:
- Fetching many individual documents by ID (use a database)
- Complex joins (avoid — denormalize)
- Point-in-time lookups on non-indexed fields

---

## 11. Practical Drills

### Drill 1 — Create a product index

Create an index for products with: name (searched), description (searched), price (range + sort), category (filtering), tags (filtering), created_at (date histogram), in_stock (filtering).

<details>
<summary>Answer</summary>

```bash
PUT /products
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name":        { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "description": { "type": "text", "analyzer": "english" },
      "price":       { "type": "double" },
      "category":    { "type": "keyword" },
      "tags":        { "type": "keyword" },
      "in_stock":    { "type": "boolean" },
      "created_at":  { "type": "date" }
    }
  }
}
```
</details>

### Drill 2 — Write a bool query

Find products that:
- Must match "widget" in name
- Should match "premium" in description
- Filter: category = "tools"
- Filter: price between 10 and 100
- Must_not: in_stock = false
- Boost products created in the last 30 days

<details>
<summary>Answer</summary>

```bash
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "widget" } }
      ],
      "should": [
        { "match": { "description": "premium" } },
        {
          "range": {
            "created_at": {
              "gte": "now-30d/d"
            }
          }
        }
      ],
      "filter": [
        { "term": { "category": "tools" } },
        { "range": { "price": { "gte": 10, "lte": 100 } } }
      ],
      "must_not": [
        { "term": { "in_stock": false } }
      ]
    }
  }
}
```
</details>

### Drill 3 — Aggregation: products by category with stats

Return: products grouped by category, with avg price, count, and top 3 most expensive products in each category.

<details>
<summary>Answer</summary>

```bash
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "product_count": { "value_count": { "field": "price" } },
        "top_products": {
          "top_hits": {
            "size": 3,
            "sort": [{ "price": { "order": "desc" } }],
            "_source": { "includes": ["name", "price"] }
          }
        }
      }
    }
  }
}
```
</details>

---

## Interview traps cheatsheet — Basic

| Trap | The truth |
|------|-----------|
| "term query works the same as match query" | `term` is exact (no analyzer). `match` analyzes the query text. `term` on a `text` field usually finds nothing. |
| "More shards = better performance" | Each shard has overhead. Rule: 20-50 GB per shard. Too many shards hurts performance. |
| "Dynamic mapping is fine for production" | Can cause type conflicts, mapping explosions (too many fields). Always use explicit mapping. |
| "Elasticsearch is ACID" | No. Near-real-time, eventually consistent. No transactions. |
| "DELETE immediately frees disk" | Soft delete. Space reclaimed during segment merge. |
| "text fields are good for sorting" | Can't sort on analyzed fields. Use keyword sub-field. |
| "from + size works for infinite pagination" | Limit: 10,000 by default. Use search_after for deep pagination. |
| "Elasticsearch can replace PostgreSQL" | ES is for search/analytics. Not for transactional data. |
| "score is always meaningful across queries" | BM25 scores are relative to the query and corpus. Not comparable across different queries. |
| "nested objects work the same as objects" | `nested` preserves relationships (separate Lucene docs). `object` is flattened. |
</details>
