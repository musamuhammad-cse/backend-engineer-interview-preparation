# Elasticsearch — Intermediate

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Advanced queries, aggregations deep dive, nested & parent-child, index templates, shard & routing, cluster architecture  
> **Real-world anchor:** Multi-tenant SaaS (multi-tenant search, product catalog), trading platform (trade search, aggregation), ELK Stack (log aggregation with Logstash)

---

## 1. Advanced queries

### Multi-match query types

```bash
# best_fields — uses the highest score from any matching field (default)
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "widget premium",
      "fields": ["name^3", "description"],
      "type": "best_fields",
      "tie_breaker": 0.3
    }
  }
}

# most_fields — combines scores from all matching fields
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "widget",
      "fields": ["name", "name.english", "name.ngram"],
      "type": "most_fields"
    }
  }
}

# cross_fields — treats fields as one big field (for cross-field search)
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "Will Smith",
      "fields": ["first_name", "last_name"],
      "type": "cross_fields",
      "operator": "and"
    }
  }
}

# phrase — match_phrase across multiple fields
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "quick brown fox",
      "fields": ["title", "body"],
      "type": "phrase"
    }
  }
}

# phrase_prefix — match_phrase_prefix across multiple fields
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "quick bro",
      "fields": ["title", "body"],
      "type": "phrase_prefix"
    }
  }
}

# bool_prefix — combines match_bool_prefix across multiple fields
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "quick brown f",
      "fields": ["title", "body"],
      "type": "bool_prefix"
    }
  }
}
```

### Function score query — advanced scoring

```bash
# Score based on recency, price, and popularity
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "widget" } },
      "functions": [
        # Newer products get higher score
        {
          "gauss": {
            "created_at": {
              "origin": "2024-06-15",
              "scale": "30d",
              "decay": 0.5
            }
          }
        },
        # Products with high sales get boost
        {
          "field_value_factor": {
            "field": "sales_count",
            "factor": 0.1,
            "modifier": "log1p"
          }
        },
        # Random score for A/B testing
        {
          "random_score": {
            "seed": "user_123",
            "field": "_seq_no"
          }
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply",
      "max_boost": 20.0
    }
  }
}
```

### Named queries

```bash
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": { "query": "widget", "_name": "name_match" } } }
      ],
      "filter": [
        { "term": { "category": { "value": "tools", "_name": "tools_filter" } } }
      ]
    }
  }
}
# Response includes matched_queries per hit
```

### Minimum should match

```bash
# Control precision/recall
GET /products/_search
{
  "query": {
    "match": {
      "name": {
        "query": "premium titanium widget",
        "minimum_should_match": "2<75%"  # 2 required, beyond that 75%
      }
    }
  }
}
```

### Rescore

```bash
# Re-score top-k results with an expensive query
GET /products/_search
{
  "query": {
    "match": { "name": "widget" }
  },
  "rescore": {
    "window_size": 50,
    "query": {
      "rescore_query": {
        "match_phrase": {
          "name": "widget premium"
        }
      },
      "query_weight": 0.7,
      "rescore_query_weight": 1.2
    }
  }
}
```

### Collapsing (field collapsing)

```bash
# Deduplicate results by a field
GET /products/_search
{
  "query": { "match": { "category": "tools" } },
  "collapse": {
    "field": "brand.keyword"
  },
  "sort": ["price"],
  "aggregations": {
    "total_brands": {
      "cardinality": { "field": "brand.keyword" }
    }
  }
}
```

---

## 2. Nested and parent-child queries

### Nested objects

Regular `object` fields flatten data — relationships between fields are lost:

```bash
PUT /orders
{
  "mappings": {
    "properties": {
      "items": { "type": "object" }  # default
    }
  }
}

PUT /orders/_doc/1
{
  "items": [
    { "name": "Widget", "price": 29.99, "qty": 2 },
    { "name": "Gadget", "price": 49.99, "qty": 1 }
  ]
}
```

**Trap:** With `object` type, a query for `price: 49.99 AND qty: 2` would match this document (because the fields are flattened: `items.price = [29.99, 49.99]`, `items.qty = [1, 2]`). It doesn't enforce that they belong to the same object.

Use **nested** to preserve relationships:

```bash
PUT /orders_nested
{
  "mappings": {
    "properties": {
      "items": { "type": "nested" }
    }
  }
}

# Query nested objects
GET /orders_nested/_search
{
  "query": {
    "nested": {
      "path": "items",
      "query": {
        "bool": {
          "must": [
            { "term": { "items.name": "Gadget" } },
            { "range": { "items.qty": { "gte": 1 } } }
          ]
        }
      },
      "inner_hits": {  # return matching nested docs
        "size": 10
      }
    }
  }
}
```

### Nested aggregations

```bash
GET /orders_nested/_search
{
  "size": 0,
  "aggs": {
    "items": {
      "nested": { "path": "items" },
      "aggs": {
        "avg_price": { "avg": { "field": "items.price" } },
        "top_products": {
          "terms": { "field": "items.name" }
        }
      }
    }
  }
}
```

### Parent-child (join field)

```bash
PUT /company
{
  "mappings": {
    "properties": {
      "join_field": {
        "type": "join",
        "relations": {
          "company": "employee"
        }
      }
    }
  }
}

# Index parent
PUT /company/_doc/c1
{
  "name": "Acme Corp",
  "join_field": "company"
}

# Index child
PUT /company/_doc/e1?routing=c1
{
  "name": "Alice",
  "join_field": {
    "name": "employee",
    "parent": "c1"
  }
}

# Find employees in a company
GET /company/_search
{
  "query": {
    "has_parent": {
      "parent_type": "company",
      "query": { "match": { "name": "Acme" } }
    }
  }
}

# Find companies with specific employees
GET /company/_search
{
  "query": {
    "has_child": {
      "type": "employee",
      "query": { "match": { "name": "Alice" } },
      "score_mode": "max"  # or "avg", "sum", "none"
    }
  }
}
```

**Trap:** Parent-child is expensive — each child needs routing to the parent's shard. Prefer denormalization or nested objects unless you have a clear need for separate parent/child indexing.

---

## 3. Aggregations — deep dive

### Pipeline aggregations

```bash
# Moving average (7-day rolling)
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "day"
      },
      "aggs": {
        "daily_sales": { "sum": { "field": "amount" } },
        "moving_avg": {
          "moving_fn": {
            "buckets_path": "daily_sales",
            "window": 7,
            "script": "MovingFunctions.unweightedAvg(values)"
          }
        }
      }
    }
  }
}

# Cumulative sum
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "over_time": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_sales": { "sum": { "field": "amount" } },
        "cumulative_sales": {
          "cumulative_sum": { "buckets_path": "monthly_sales" }
        }
      }
    }
  }
}

# Bucket script — calculate ratio
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "total_sales": { "sum": { "field": "amount" } }
      }
    },
    "total_all": {
      "sum_bucket": { "buckets_path": "by_category>total_sales" }
    },
    "category_percentages": {
      "terms": { "field": "category" },
      "aggs": {
        "total_sales": { "sum": { "field": "amount" } },
        "percentage": {
          "bucket_script": {
            "buckets_path": {
              "category_sales": "total_sales",
              "all_sales": "total_all"
            },
            "script": "params.category_sales / params.all_sales * 100"
          }
        }
      }
    }
  }
}
```

### Geohash grid aggregation

```bash
GET /places/_search
{
  "size": 0,
  "aggs": {
    "by_location": {
      "geohash_grid": {
        "field": "location",
        "precision": 5  # 1-12, higher = more granular
      }
    }
  }
}
```

### Filters aggregation

```bash
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "order_filters": {
      "filters": {
        "filters": {
          "high_value": { "range": { "total": { "gte": 1000 } } },
          "recent":     { "range": { "created_at": { "gte": "now-7d" } } },
          "pending":    { "term": { "status": "pending" } }
        }
      },
      "aggs": {
        "avg_total": { "avg": { "field": "total" } }
      }
    }
  }
}
```

### Significant terms (interesting/unusual terms)

```bash
# Find categories that are unusually common in high-value orders
GET /orders/_search
{
  "size": 0,
  "query": { "range": { "total": { "gte": 1000 } } },
  "aggs": {
    "significant_categories": {
      "significant_terms": {
        "field": "category",
        "min_doc_count": 5,
        "mutual_information": {}  # scoring method
      }
    }
  }
}
```

### Composite aggregation (paginated aggregation)

```bash
# Paginated terms aggregation (for high-cardinality fields)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "composite": {
        "size": 100,
        "sources": [
          { "category": { "terms": { "field": "category" } } }
        ]
      }
    }
  }
}
# Use "after_key" from response for pagination
```

### Time-series with date_histogram

```bash
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "daily": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day",
        "format": "yyyy-MM-dd",
        "min_doc_count": 0,        # include empty buckets
        "extended_bounds": {
          "min": "2024-01-01",
          "max": "2024-01-31"
        }
      },
      "aggs": {
        "revenue": { "sum": { "field": "total" } },
        "order_count": { "value_count": { "field": "_id" } }
      }
    }
  }
}
```

---

## 4. Index templates and dynamic templates

### Index template (legacy)

```bash
# Legacy (pre-7.8) — _template
PUT /_template/logs_template
{
  "index_patterns": ["logs-*"],
  "settings": {
    "number_of_shards": 2,
    "refresh_interval": "10s"
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "message": { "type": "text", "analyzer": "english" },
      "level":    { "type": "keyword" },
      "service":  { "type": "keyword" }
    }
  }
}
```

### Composable index template (7.8+)

```bash
# Component template (reusable block)
PUT /_component_template/logs_settings
{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "refresh_interval": "10s",
      "index.lifecycle.name": "logs_policy"
    }
  }
}

PUT /_component_template/logs_mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text", "analyzer": "english" },
        "level":    { "type": "keyword" },
        "service":  { "type": "keyword" }
      }
    }
  }
}

# Composable index template
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*", "syslog-*"],
  "composed_of": ["logs_settings", "logs_mappings"],
  "priority": 100
}
```

### Dynamic templates — field type inference

```bash
PUT /my_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword" }
        }
      },
      {
        "longs_as_integers": {
          "match_mapping_type": "long",
          "mapping": { "type": "integer" }
        }
      },
      {
        "created_at_as_date": {
          "match": "created_*",
          "unmatch": "created_by",
          "mapping": { "type": "date" }
        }
      },
      {
        "runtime_fields": {
          "path_match": "runtime.*",
          "mapping": { "type": "runtime" }
        }
      }
    ]
  }
}
```

### Runtime fields (computed at query time)

```bash
PUT /products
{
  "mappings": {
    "runtime": {
      "discounted_price": {
        "type": "double",
        "script": {
          "source": "if (doc['price'].value > 100) emit(doc['price'].value * 0.9); else emit(doc['price'].value)"
        }
      }
    }
  }
}

# Query using runtime field
GET /products/_search
{
  "query": {
    "range": { "discounted_price": { "gte": 50 } }
  }
}
```

---

## 5. Shard and routing

### How routing works

When indexing a document, Elasticsearch determines which shard it goes to:

```bash
shard = hash(_routing) % number_of_primary_shards
```

By default, `_routing` = `_id`. This means documents with the same `_id` pattern go to the same shard.

### Custom routing

```bash
# Index with custom routing (all orders for org_42 go to same shard)
PUT /orders/_doc/order_001?routing=org_42
{
  "org_id": "org_42",
  "order_number": "ORD-001"
}

# Search with routing (query only the relevant shard)
GET /orders/_search?routing=org_42
{
  "query": { "term": { "org_id": "org_42" } }
}
```

**Benefits:**
- All documents for a tenant go to the same shard (fewer shards to query)
- Parent-child requires routing (child must be on same shard as parent)

**Trap:** If you use custom routing, you must specify routing on ALL operations (index, get, delete, update, search). If you mix routed and non-routed operations, you get inconsistent results.

### Shard allocation awareness

```bash
# elasticsearch.yml
node.attr.zone: us-east-1a

# Configure allocation awareness
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: us-east-1a,us-east-1b,us-east-1c
```

This ensures primary and replica shards are not allocated to nodes in the same zone.

### Shard filtering

```bash
# Allocate specific indices to specific nodes
PUT /hot_logs/_settings
{
  "index.routing.allocation.require.zone": "hot"
}

PUT /cold_logs/_settings
{
  "index.routing.allocation.require.zone": "cold"
}
```

---

## 6. Cluster architecture

### Node roles and cluster topology

| Node type | Roles | Purpose |
|-----------|-------|---------|
| **Master** | `master` | Cluster management (lightweight) |
| **Data** | `data` | Store + search data (heavy) |
| **Ingest** | `ingest` | Pre-process documents before indexing |
| **Coordinating** | (none) | Load balancer — route requests, merge results |
| **Machine Learning** | `ml` | ML capabilities |
| **Transform** | `transform` | Transform management |

### Recommended cluster topology

```
Small cluster (dev/test):
  3 nodes: master + data + ingest (all roles)

Medium cluster (prod):
  3 dedicated master nodes
  3-10 data nodes (hot)
  N data nodes (warm/cold)
  2-3 coordinating nodes (if high search volume)

Large cluster:
  3 dedicated master nodes
  10-50 data nodes (hot, warm, cold tiers)
  3+ coordinating nodes
  N ingest nodes (if heavy ingestion pipeline)
```

### Cluster discovery (7.x+)

```yaml
# elasticsearch.yml — all nodes
discovery.seed_hosts:
  - 10.0.0.1:9300
  - 10.0.0.2:9300
  - 10.0.0.3:9300

# Only on initial master-eligible nodes
cluster.initial_master_nodes:
  - master-1
  - master-2
  - master-3
```

### Cluster health and monitoring

```bash
# Quick health
GET /_cat/health?v

# Node info
GET /_cat/nodes?v                    # node list with roles
GET /_cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,load_1m
GET /_nodes/stats                    # detailed per-node stats

# Index info
GET /_cat/indices?v                  # all indices
GET /_cat/shards?v                   # shard distribution

# Cluster state (large — use with care)
GET /_cluster/state?filter_path=metadata.templates

# Pending tasks
GET /_cluster/pending_tasks
```

### Hot-thread analysis

```bash
# Identify CPU-intensive threads
GET /_nodes/hot_threads
# Returns stack traces for the busiest threads — useful for debugging slowdowns
```

---

## 7. Snapshot and restore

### Register repository

```bash
# S3 repository
PUT /_snapshot/my_backup
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "us-east-1",
    "base_path": "es-snapshots"
  }
}

# Shared filesystem repository
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups",
    "compress": true
  }
}
```

### Create and manage snapshots

```bash
# Create snapshot of all indices
PUT /_snapshot/my_backup/snapshot_20240115

# Create snapshot of specific indices
PUT /_snapshot/my_backup/snapshot_20240115
{
  "indices": "logs-2024.01.15,products",
  "ignore_unavailable": true,
  "include_global_state": true  # includes cluster settings, templates
}

# List snapshots
GET /_snapshot/my_backup/_all

# Delete snapshot
DELETE /_snapshot/my_backup/snapshot_20240115

# Restore
POST /_snapshot/my_backup/snapshot_20240115/_restore
{
  "indices": "logs-2024.01.*",
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored-logs-$1"
}
```

**Trap:** Snapshots are **incremental** — only the first snapshot is full; subsequent ones store changed segments. Deleting an old snapshot doesn't save much space because segments are shared across snapshots.

---

## 8. Data streams (7.9+)

Data streams simplify time-series data management:

```bash
# Create index template with data stream
PUT /_index_template/logs_stream
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": { "number_of_shards": 2 },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" }
      }
    }
  }
}

# Index into the data stream
POST /logs-stream/_doc
{
  "@timestamp": "2024-01-15T10:00:00Z",
  "message": "Server started",
  "service": "web"
}

# Search across all backing indices
GET /logs-stream/_search

# Roll over manually (or auto via ILM)
POST /logs-stream/_rollover

# Delete old data by deleting the backing index
DELETE /logs-stream-000002
```

---

## 9. Cross-cluster search

```bash
# Configure remote cluster
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote": {
      "remote_cluster_1": {
        "seeds": ["10.0.0.1:9300", "10.0.0.2:9300"]
      }
    }
  }
}

# Search across clusters
GET /remote_cluster_1:logs,local_logs/_search
{
  "query": { "match_all": {} }
}
```

---

## 10. Practical drills

### Drill 1 — Multi-tenant search

Design a search architecture for your SaaS where each org has its own product catalog. Users search across their org's products.

<details>
<summary>Answer</summary>

**Option 1: One index per tenant (pro: isolation, con: many indices)**
```bash
PUT /products_org_42
# ... mappings
PUT /products_org_99
# ... mappings
```
Pro: Full isolation, can scale per tenant. Con: cluster metadata grows with each tenant.

**Option 2: Single index with tenant filter**
```bash
PUT /products
{
  "mappings": {
    "properties": {
      "org_id": { "type": "keyword" },
      "name":   { "type": "text" }
    }
  }
}

# Search for tenant
GET /products/_search
{
  "query": {
    "bool": {
      "filter": [ { "term": { "org_id": "org_42" } } ],
      "must":   [ { "match": { "name": "widget" } } ]
    }
  }
}
```
Pro: Simple management, good for < 10K tenants. Con: No per-tenant tuning.

**Option 3: Custom routing**
```bash
# Route by org_id so one org's data is colocated
PUT /products/_doc/prod_001?routing=org_42
{
  "org_id": "org_42",
  "name": "Widget"
}

# Search with routing
GET /products/_search?routing=org_42
{
  "query": { "match": { "name": "widget" } }
}
```
Pro: Fewer shards queried, better performance. Con: Routing must be consistent.
</details>

### Drill 2 — Aggregation for trading dashboard

You have trades indexed in Elasticsearch. Design aggregations for a real-time trading dashboard showing:
- OHLCV per symbol per minute
- Top traded symbols by volume
- Trader ranking by P&L
- Trade count per second (for heatmap)

<details>
<summary>Answer</summary>

```bash
# OHLCV per minute
GET /trades/_search
{
  "size": 0,
  "aggs": {
    "per_symbol": {
      "terms": { "field": "symbol" },
      "aggs": {
        "per_minute": {
          "date_histogram": {
            "field": "timestamp",
            "fixed_interval": "1m"
          },
          "aggs": {
            "open":  { "min": { "field": "price" } },  # first trade of minute
            "high":  { "max": { "field": "price" } },
            "low":   { "min": { "field": "price" } },
            "close": { "max": { "field": "price" } },   # last trade
            "volume": { "sum": { "field": "qty" } }
          }
        }
      }
    }
  }
}

# Top symbols by volume
GET /trades/_search
{
  "size": 0,
  "aggs": {
    "by_symbol": {
      "terms": {
        "field": "symbol",
        "size": 10,
        "order": { "volume": "desc" }
      },
      "aggs": {
        "volume": { "sum": { "field": "qty" } }
      }
    }
  }
}
```

**Note for your scenario:** For real-time dashboards at 20K+ DAU, consider using date_histogram with `min_doc_count: 0` and `extended_bounds` for complete time series, and cache frequent aggregations.
</details>

### Drill 3 — Index template for application logs

Design an index template for application logs with the following requirements:
- Index per day (logs-2024.01.15)
- message field analyzed with english analyzer
- level, service, host as keyword
- @timestamp as date
- Dynamic string fields should be keyword (not text)
- Refresh interval every 30 seconds
- Roll over after 50 GB or 30 days

<details>
<summary>Answer</summary>

```bash
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "30s",
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    },
    "mappings": {
      "dynamic_templates": [
        {
          "strings_as_keywords": {
            "match_mapping_type": "string",
            "mapping": { "type": "keyword", "ignore_above": 256 }
          }
        }
      ],
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text", "analyzer": "english" },
        "level":      { "type": "keyword" },
        "service":    { "type": "keyword" },
        "host":       { "type": "keyword" }
      }
    }
  }
}

# ILM policy
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": { "max_size": "50GB", "max_age": "30d" },
          "set_priority": { "priority": 100 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```
</details>

---

## Interview traps cheatsheet — Intermediate

| Trap | The truth |
|------|-----------|
| "nested objects and objects are the same" | `nested` preserves relationships (separate Lucene docs). `object` flattens arrays — query can match across elements. |
| "Parent-child is always better than nested" | Parent-child is expensive (needs routing). Use nested for small arrays, parent-child for frequently updated children. |
| "Composite aggregation is faster than terms" | Composite paginates through results, not faster. Use for high-cardinality fields where you need paginated results. |
| "Custom routing is always beneficial" | Must specify routing on ALL operations. Missing routing on search returns incorrect results. |
| "Snapshot restore is instant" | Must reindex all data — takes as long as the original indexing. |
| "Index templates update existing indices" | Templates only apply to NEW indices. Existing indices keep their current settings/mappings. |
| "Data streams are just aliases" | Data streams manage backing indices automatically with rollover — more than aliases. |
| "Runtime fields are as fast as indexed fields" | Runtime fields are computed at query time — no index, no doc_values. Slower but no storage cost. |
| "Minimum should match controls recall" | Yes — `2<75%` means 2 clauses required beyond that at least 75% must match. Good for precision/recall tuning. |
| "Rescore runs on all matching docs" | Rescore only runs on the top `window_size` documents from the initial query. |
</details>
