# Elasticsearch — Question Bank

> **Target:** Senior Backend Engineer interview preparation  
> **Format:** Rapid-fire Q&A, code puzzles, debugging scenarios, system design prompts, STAR templates  
> **Real-world anchors:** ELK Stack (observability), multi-tenant SaaS (product search), trading platform (trade search), Chronos (job logs)

---

## 1. Rapid-fire Q&A (140+ questions)

### Fundamentals (25 questions)

1. **Q:** What is an inverted index?  
   **A:** A mapping from terms (tokens) to document IDs. Enables fast full-text search.

2. **Q:** What is the difference between an index in ES and a table in SQL?  
   **A:** An ES index is a collection of documents with dynamic schema, stored across shards. A SQL table has a fixed schema.

3. **Q:** What is a shard?  
   **A:** A shard is a Lucene index — the unit of distribution. Each index is split into primary shards.

4. **Q:** What is a replica shard?  
   **A:** A copy of a primary shard. Provides high availability and read scalability.

5. **Q:** What determines the shard a document goes to?  
   **A:** `hash(_routing) % number_of_primary_shards`. Default `_routing = _id`.

6. **Q:** Can you change the number of primary shards after index creation?  
   **A:** No. You must reindex to a new index with different shard count.

7. **Q:** What is the default number of primary shards?  
   **A:** 1 (Elasticsearch 8.x) or 5 (older versions).

8. **Q:** What is the default number of replica shards?  
   **A:** 1 (one replica per primary).

9. **Q:** What is a node?  
   **A:** A running Elasticsearch instance. Part of a cluster.

10. **Q:** What node roles exist?  
    **A:** master, data, ingest, ml, coordinating (empty set), transform.

11. **Q:** What is the difference between `match` and `term` queries?  
    **A:** `match` analyzes the query text (tokenizes+lowercases). `term` does no analysis — exact match on indexed token.

12. **Q:** What is the difference between query context and filter context?  
    **A:** Query context calculates `_score`. Filter context does not score — results are cached. Use filter for yes/no conditions.

13. **Q:** What is BM25?  
    **A:** The default relevance algorithm (since ES 5.0). Parameters: `k1` (term frequency saturation) and `b` (length normalization).

14. **Q:** What is `_source`?  
    **A:** The original JSON document stored at index time. Returned in search results by default.

15. **Q:** Can you disable `_source`?  
    **A:** Yes — `"_source": { "enabled": false }`. Saves disk but you can't retrieve original documents.

16. **Q:** What is the `explain` API?  
    **A:** Shows detailed scoring breakdown for a document-query pair.

17. **Q:** What is cluster health status "yellow"?  
    **A:** All primary shards active, some replicas unassigned (e.g., a data node is down).

18. **Q:** What is cluster health status "red"?  
    **A:** Some primary shards are unassigned — data loss risk.

19. **Q:** How do you get all indices?  
    **A:** `GET /_cat/indices?v` or `GET /_all` or `GET /*`.

20. **Q:** How do you check cluster health?  
    **A:** `GET /_cluster/health`.

21. **Q:** What is `_cat` API?  
    **A:** Compact and aligned text (CAT) API — human-readable cluster info.

22. **Q:** What is the default port?  
    **A:** HTTP API: 9200, Transport (cluster): 9300.

23. **Q:** What is an analyzer?  
    **A:** Converts text into tokens: character filters → tokenizer → token filters.

24. **Q:** What is the default analyzer?  
    **A:** `standard` — tokenizes on word boundaries, lowercases, removes punctuation.

25. **Q:** What is a `keyword` field?  
    **A:** Not analyzed — stored as-is. Used for exact match, sorting, aggregations.

### Mapping and Indexing (20 questions)

26. **Q:** Can you change a field type after indexing?  
    **A:** No — you must reindex to a new mapping. You can add new fields (unless `dynamic: strict`).

27. **Q:** What is dynamic mapping?  
    **A:** ES infers field types from document values. Enabled by default.

28. **Q:** What is dynamic `false`?  
    **A:** New fields are ignored (not indexed, not stored in `_source`? Actually stored in `_source`, just not searchable).

29. **Q:** What is dynamic `strict`?  
    **A:** Rejects documents with unknown fields.

30. **Q:** What is a multi-field?  
    **A:** A field with multiple sub-fields for different analysis (e.g., `name` as `text` + `name.keyword`).

31. **Q:** What is `doc_values`?  
    **A:** Column-oriented storage on disk for sorting/aggregations. Enabled by default for keyword/numeric fields.

32. **Q:** What is `fielddata`?  
    **A:** In-memory data structure for sorting/aggregations on `text` fields (not enabled by default — heap-heavy).

33. **Q:** What is a `nested` field?  
    **A:** Array of objects preserved as separate Lucene documents. Required for correct querying on object arrays.

34. **Q:** What is the difference between `object` and `nested`?  
    **A:** `object` flattens arrays — a query can match across elements. `nested` keeps objects separate.

35. **Q:** What is a `join` field?  
    **A:** Parent-child relationship within an index. Child must be on same shard as parent (use routing).

36. **Q:** What is an index template?  
    **A:** Defines settings + mappings for indices matching a pattern (e.g., `logs-*`).

37. **Q:** What is a component template?  
    **A:** Reusable building block for composable index templates (7.8+).

38. **Q:** What is a dynamic template?  
    **A:** Controls mapping for dynamically added fields based on pattern or type.

39. **Q:** What is `_reindex`?  
    **A:** Copies documents from source to destination index. Can transform with script.

40. **Q:** What is `_update_by_query`?  
    **A:** Updates documents matching a query (e.g., add/remove field).

41. **Q:** What is `_delete_by_query`?  
    **A:** Deletes documents matching a query.

42. **Q:** What is `_bulk` API?  
    **A:** Perform many index/update/delete operations in one request. Performance: ~10-15 MB batch size.

43. **Q:** What is `refresh_interval`?  
    **A:** How frequently new segments are made searchable (default: 1s).

44. **Q:** What is the translog?  
    **A:** Write-ahead log for durability. `request` (fsync per op) or `async` (fsync every 5s).

45. **Q:** What is ILM?  
    **A:** Index Lifecycle Management — automates index transitions: hot → warm → cold → delete.

### Query DSL (20 questions)

46. **Q:** What is a bool query?  
    **A:** Compound query with `must`, `should`, `filter`, `must_not` clauses.

47. **Q:** What does `minimum_should_match` do?  
    **A:** Minimum number of `should` clauses that must match. `"2<75%"` = 2 required, beyond that 75%.

48. **Q:** What is `constant_score`?  
    **A:** Wraps a filter query with a constant score. Useful for non-scoring filters in query context.

49. **Q:** What is `function_score`?  
    **A:** Allows custom scoring with functions: field_value_factor, gauss, linear, random_score.

50. **Q:** What is `boosting` query?  
    **A:** Matches documents with positive clause but reduces score of negative clause matches.

51. **Q:** What is `dis_max`?  
    **A:** Takes the maximum score from multiple queries (union). `tie_breaker` adds scores from other matches.

52. **Q:** What is `multi_match`?  
    **A:** Search across multiple fields. Types: `best_fields`, `most_fields`, `cross_fields`, `phrase`, `phrase_prefix`.

53. **Q:** What is `match_phrase`?  
    **A:** Requires terms in order. `slop` parameter allows distance between terms.

54. **Q:** What is `match_phrase_prefix`?  
    **A:** Like match_phrase but last term can be a prefix (for search-as-you-type).

55. **Q:** What is `terms` query?  
    **A:** Matches if field equals any of the given values.

56. **Q:** What is `prefix` query?  
    **A:** Finds terms starting with a prefix. Slow on large datasets.

57. **Q:** What is `wildcard` query?  
    **A:** Pattern matching with `*` and `?`. Very slow on large datasets.

58. **Q:** What is `regexp` query?  
    **A:** Regex matching on terms. Can be very slow — use sparingly.

59. **Q:** What is `fuzzy` query?  
    **A:** Levenshtein distance-based fuzzy matching. `fuzziness: "AUTO"` recommended.

60. **Q:** What is `query_string`?  
    **A:** Powerful query syntax (AND, OR, field:value). Error-prone — prefer `bool` + `simple_query_string`.

61. **Q:** What is `simple_query_string`?  
    **A:** Safer version of query_string. Supports `+`, `|`, `-`, `"`.

62. **Q:** What is `range` query?  
    **A:** Matches values in range: `gte`, `gt`, `lte`, `lt`. Works on numeric, date, string.

63. **Q:** What is `exists` query?  
    **A:** Matches documents that have a non-null value for a field.

64. **Q:** What is `more_like_this`?  
    **A:** Finds documents similar to a given document or text.

65. **Q:** What is `script` query?  
    **A:** Custom matching logic via Painless script. Slow — avoid for high-volume queries.

### Aggregations (15 questions)

66. **Q:** What are the three types of aggregations?  
    **A:** Metric (single value), Bucket (grouping), Pipeline (based on other aggregations).

67. **Q:** What is `terms` aggregation?  
    **A:** Groups by unique field values. Like SQL `GROUP BY`. Limited to the top N terms.

68. **Q:** What is `date_histogram`?  
    **A:** Groups by date intervals (calendar: month, week, day; fixed: 5m, 2h).

69. **Q:** What is `histogram`?  
    **A:** Groups by numeric intervals.

70. **Q:** What is `range` aggregation?  
    **A:** Groups by custom ranges.

71. **Q:** What is `cardinality` aggregation?  
    **A:** Approximate unique count (HyperLogLog-based, ~5% error).

72. **Q:** What is `percentiles` aggregation?  
    **A:** Calculates percentile values. Uses TDigest algorithm.

73. **Q:** What is `top_hits` aggregation?  
    **A:** Returns top matching documents per bucket.

74. **Q:** What is `nested` aggregation?  
    **A:** Aggregates on nested documents within a path.

75. **Q:** What is `composite` aggregation?  
    **A:** Paginated aggregation for high-cardinality fields. Use `after_key` for pagination.

76. **Q:** What is `pipeline` aggregation?  
    **A:** Aggregation based on results of another aggregation (e.g., moving_avg, cumulative_sum).

77. **Q:** What is `bucket_script`?  
    **A:** Executes a script on bucket values. Useful for calculating percentages.

78. **Q:** What is `significant_terms`?  
    **A:** Finds terms that are statistically significant — unusually common in a subset.

79. **Q:** How do you paginate aggregations?  
    **A:** Use `composite` aggregation (for high cardinality) or increase `size` (for low cardinality).

80. **Q:** What is `search.max_buckets`?  
    **A:** Dynamic cluster setting (default 65536) that limits total buckets in a response.

### Cluster and Performance (20 questions)

81. **Q:** How do you add a node to a cluster?  
    **A:** Install ES on a new server with the same `cluster.name` and seed hosts — it auto-joins.

82. **Q:** How do you perform a rolling restart?  
    **A:** Disable allocation → stop node → start node → wait for green → repeat.

83. **Q:** What is split-brain in Elasticsearch?  
    **A:** Two nodes both think they're the master. Prevented by `discovery.zen.minimum_master_nodes`.

84. **Q:** What is the minimum master nodes formula?  
    **A:** `(master_eligible_nodes / 2) + 1`. For 3 master nodes: 2.

85. **Q:** What is shard allocation awareness?  
    **A:** Ensures primary + replica shards are in different zones/racks.

86. **Q:** What are disk watermarks?  
    **A:** `low` (85%) → stop allocating shards. `high` (90%) → relocate shards. `flood_stage` (95%) → block writes.

87. **Q:** What is `_forcemerge`?  
    **A:** Merges segments into a specified number (1 = one segment per shard). Expensive — only for read-only indices.

88. **Q:** What is the indexing buffer?  
    **A:** Buffer for new documents before they're written to segments. Default 10% of heap.

89. **Q:** What is `refresh_interval: -1`?  
    **A:** Disables automatic refresh. Use during bulk indexing, re-enable after.

90. **Q:** What is `wait_for_active_shards`?  
    **A:** Index request setting: wait for N shards to acknowledge before returning.

91. **Q:** What is the segment merge policy?  
    **A:** Tiered merge policy. Balances between too many small segments and too-large merge operations.

92. **Q:** What is the recommended shard size?  
    **A:** 20-50 GB per shard.

93. **Q:** How many shards per node maximum?  
    **A:** ~1,000 (including replicas). Metadata overhead per shard.

94. **Q:** What is the circuit breaker?  
    **A:** Prevents OOM by rejecting requests that exceed memory limits. Per-request, fielddata, total.

95. **Q:** What JVM GC does ES use?  
    **A:** G1GC (default since 7.x). Previous: CMS.

96. **Q:** What is the recommended heap size?  
    **A:** 50% of system RAM, maximum 31 GB.

97. **Q:** Why max 31 GB heap?  
    **A:** Above 32 GB, JVM switches from compressed OOPs (32-bit pointers) to uncompressed (64-bit) — memory waste.

98. **Q:** What is `_nodes/hot_threads`?  
    **A:** Stack traces of the busiest threads — for CPU hotspot debugging.

99. **Q:** What are search slow logs?  
    **A:** Log queries exceeding thresholds. Enable per-index.

100. **Q:** What is `_cluster/allocation/explain`?  
     **A:** Explains why a shard is unassigned. First stop for red/yellow cluster diagnosis.

### Kibana and ELK (10 questions)

101. **Q:** What is Kibana?  
     **A:** Visualization and management UI for Elasticsearch.

102. **Q:** What is Logstash?  
     **A:** Server-side data processing pipeline — input, filter (grok, mutate, date), output.

103. **Q:** What is Filebeat?  
     **A:** Lightweight log shipper. Sends logs to Logstash or directly to ES.

104. **Q:** What are the Beats?  
     **A:** Lightweight data shippers: Filebeat (logs), Metricbeat (metrics), Heartbeat (uptime), Auditbeat (audit).

105. **Q:** What is the grok filter?  
     **A:** Logstash filter for parsing unstructured data into structured fields.

106. **Q:** What is ILM?  
     **A:** Index Lifecycle Management — governs hot/warm/cold/delete phases automatically.

107. **Q:** What is a data stream?  
     **A:** Append-only time-series index management (7.9+). Automatically creates backing indices.

108. **Q:** What is cross-cluster search (CCS)?  
     **A:** Search across multiple Elasticsearch clusters.

109. **Q:** What is cross-cluster replication (CCR)?  
     **A:** Replicate indices across clusters for disaster recovery or geo-proximity.

110. **Q:** What is Elastic APM?  
     **A:** Application Performance Monitoring — traces requests through distributed systems.

### Senior (15 questions)

111. **Q:** How do you design a multi-tenant search architecture?  
     **A:** Options: (1) one index per tenant (isolated), (2) single index with tenant filter field (simpler), (3) custom routing (co-located).

112. **Q:** How do you handle mapping explosion?  
     **A:** Set `dynamic: false` or `dynamic: strict`. Use `dynamic_templates` to control new field types. Limit total fields.

113. **Q:** How do you tune for indexing throughput?  
     **A:** Increase `refresh_interval`, async translog, disable replicas during bulk, larger batch sizes, no unnecessary fields.

114. **Q:** How do you tune for search speed?  
     **A:** More replicas, force-merge read-only indices, filter caching, doc_values on needed fields, adequate heap.

115. **Q:** When should you reindex?  
     **A:** To change mappings, change shard count, upgrade between incompatible versions, apply schema changes.

116. **Q:** What is `search_as_you_type`?  
     **A:** Field type for autocomplete — creates ngrams and edge ngrams for prefix matching.

117. **Q:** What is `dense_vector`?  
     **A:** Field type for vector embeddings (8.0+). Used for kNN search.

118. **Q:** What is kNN search?  
     **A:** Approximate nearest neighbor search on dense vectors (8.0+).

119. **Q:** What is frozen tier?  
     **A:** Rarely accessed data, stored on object store (S3). Partial search support (8.0+).

120. **Q:** What is Elasticsearch's snapshot consistency model?  
     **A:** Incremental — only first snapshot is full. Subsequent snapshots store changed segments only.

121. **Q:** How do you upgrade Elasticsearch?  
     **A:** Rolling restart for minor versions. Reindex for major versions (6→7, 7→8).

122. **Q:** What is the difference between Elasticsearch and OpenSearch?  
     **A:** OpenSearch is the Apache 2.0 fork (from Elasticsearch 7.10). Functionally similar.

123. **Q:** How does Elasticsearch handle resilience?  
     **A:** Replica shards, shard allocation awareness, cross-cluster replication, snapshots.

124. **Q:** What is the cluster state?  
     **A:** Cluster metadata: indices, settings, mappings, templates, routing table. Must fit in memory on all master-eligible nodes.

125. **Q:** How do you prevent accidental deletion?  
     **A:** Use action auto-create/index deletion protection via config or restricted roles.

### ES vs Alternatives (15 questions)

126. **Q:** When would you choose Elasticsearch over Solr?  
     **A:** Distributed out of the box, stronger aggregations, simpler setup, stronger time-series support (ILM).

127. **Q:** When would you choose Solr over Elasticsearch?  
     **A:** Existing Solr infrastructure, need for advanced faceting (more mature), less concern about time-series.

128. **Q:** When would you choose Meilisearch over Elasticsearch?  
     **A:** Simple typo-tolerant front-end search, small to medium datasets, don't need aggregations.

129. **Q:** When would you choose Typesense over Elasticsearch?  
     **A:** Simple setup, fast out of box, typo-tolerant, don't need complex aggregations.

130. **Q:** When would you choose MongoDB Atlas Search over Elasticsearch?  
     **A:** Already using MongoDB, don't want a separate search infrastructure, need simpler operations.

131. **Q:** What is Elasticsearch's biggest weakness?  
     **A:** Operations complexity, JVM tuning, GC pauses, shard management, cluster state overhead at scale.

132. **Q:** What is the max cluster size?  
     **A:** ~100 nodes recommended. Larger clusters face state distribution overhead.

133. **Q:** Can Elasticsearch replace a time-series database?  
     **A:** Yes — it's commonly used for logs and metrics (ELK Stack). But InfluxDB/TimescaleDB may be better for metrics-only.

134. **Q:** What is the cold tier storage cost advantage?  
     **A:** Cold nodes can use HDD instead of SSD, reducing hardware cost by 3-5x.

135. **Q:** What is the frozen tier?  
     **A:** Only in Elasticsearch 8.0+. Stores data on object store (S3). Partial search. Minimal local storage.

136. **Q:** How does ES compare to PostgreSQL for full-text search?  
     **A:** ES: inverted index, BM25, relevance tuning, aggregations, distributed. PG: GIN/tsvector, good enough for simple search, no built-in distribution.

137. **Q:** What is the E in ELK?  
     **A:** Elasticsearch (search & analytics).

138. **Q:** What is the L in ELK?  
     **A:** Logstash (data processing pipeline).

139. **Q:** What is the K in ELK?  
     **A:** Kibana (visualization & management).

140. **Q:** What are Beats?  
     **A:** Lightweight data shippers (Filebeat, Metricbeat, Heartbeat, Auditbeat).

---

## 2. Code puzzles (10 puzzles)

### Puzzle 1 — Create a product index with search-as-you-type

Create an index for products that supports:
- Name with autocomplete (search with partial input)
- Description with English stemming
- Category filtering
- Price range + sorting

<details>
<summary>Answer</summary>

```bash
PUT /products
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1,
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      },
      "analyzer": {
        "autocomplete": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "autocomplete_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete",
        "search_analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "english"
      },
      "category": { "type": "keyword" },
      "price": { "type": "double" },
      "in_stock": { "type": "boolean" },
      "created_at": { "type": "date" }
    }
  }
}
```
</details>

### Puzzle 2 — Search with filters and scoring

Write a query that:
- Searches for "premium widget" in name
- Filters by category = "tools" AND price between $10 and $100
- Boosts products that are in stock
- Boosts recently created products (within 30 days get 2x score)

<details>
<summary>Answer</summary>

```bash
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            { "match": { "name": "premium widget" } }
          ],
          "filter": [
            { "term": { "category": "tools" } },
            { "range": { "price": { "gte": 10, "lte": 100 } } }
          ]
        }
      },
      "functions": [
        {
          "filter": { "term": { "in_stock": true } },
          "weight": 2
        },
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        }
      ],
      "boost_mode": "multiply"
    }
  }
}
```
</details>

### Puzzle 3 — Aggregation: Sales dashboard

Design an aggregation for a sales dashboard showing:
- Total revenue, order count, avg order value
- Revenue per day for the last 30 days
- Top 5 products by revenue
- Revenue breakdown by category (with percentage)

<details>
<summary>Answer</summary>

```bash
GET /orders/_search
{
  "size": 0,
  "query": {
    "range": { "created_at": { "gte": "now-30d" } }
  },
  "aggs": {
    "total_revenue": { "sum": { "field": "total" } },
    "order_count": { "value_count": { "field": "_id" } },
    "avg_order": { "avg": { "field": "total" } },
    
    "daily_revenue": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day",
        "format": "yyyy-MM-dd"
      },
      "aggs": {
        "revenue": { "sum": { "field": "total" } }
      }
    },
    
    "top_products": {
      "terms": {
        "field": "product_id",
        "size": 5,
        "order": { "product_revenue": "desc" }
      },
      "aggs": {
        "product_revenue": { "sum": { "field": "total" } },
        "product_name": {
          "top_hits": {
            "size": 1,
            "_source": { "includes": ["product_name"] }
          }
        }
      }
    },
    
    "by_category": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "revenue": { "sum": { "field": "total" } }
      }
    },
    "total_revenue_all": {
      "sum_bucket": { "buckets_path": "by_category>revenue" }
    },
    "with_percentage": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "revenue": { "sum": { "field": "total" } },
        "percentage": {
          "bucket_script": {
            "buckets_path": {
              "category_rev": "revenue",
              "total_rev": "total_revenue"
            },
            "script": "params.category_rev / params.total_rev * 100"
          }
        }
      }
    }
  }
}
```
</details>

### Puzzle 4 — Design a log index template

Design an ILM-managed log index template for:
- Index per day
- Hot → warm after 7 days (force-merge to 1 segment, 1 shard)
- Delete after 90 days
- message field with english analyzer
- level, service, host as keyword
- All string fields default to keyword

<details>
<summary>Answer</summary>

```bash
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_size": "50GB", "max_age": "1d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 },
          "allocate": { "require": { "temperature": "warm" } },
          "set_priority": { "priority": 50 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}

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
            "mapping": { "type": "keyword" }
          }
        }
      ],
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text", "analyzer": "english" },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" },
        "host": { "type": "keyword" },
        "environment": { "type": "keyword" }
      }
    }
  }
}
```
</details>

### Puzzle 5 — Multi-tenant search with routing

A SaaS with 10K tenants. Each tenant searches only their own data. Design the query that ensures tenant isolation and optimal performance.

<details>
<summary>Answer</summary>

**Index documents with routing:**
```bash
PUT /products/_doc/prod_001?routing=org_42
{
  "org_id": "org_42",
  "name": "Widget",
  "price": 29.99
}

PUT /products/_doc/prod_002?routing=org_99
{
  "org_id": "org_99",
  "name": "Gadget",
  "price": 49.99
}
```

**Search with routing:**
```bash
GET /products/_search?routing=org_42
{
  "query": {
    "bool": {
      "must": { "match": { "name": "widget" } },
      "filter": { "term": { "org_id": "org_42" } }
    }
  }
}
```

**Benefits:**
- Only queries the shard(s) that hold org_42's data
- Filter ensures isolation even if routing is wrong
- Custom routing colocates all of a tenant's documents on the same shard(s)
</details>

### Puzzle 6 — Reindex with transformation

Reindex `products_v1` to `products_v2`, applying:
- Rename field `price` to `price_usd`
- Remove field `internal_notes`
- Add field `updated_at` with current timestamp

<details>
<summary>Answer</summary>

```bash
POST /_reindex
{
  "source": { "index": "products_v1" },
  "dest": { "index": "products_v2" },
  "script": {
    "source": """
      ctx._source.price_usd = ctx._source.remove('price');
      ctx._source.remove('internal_notes');
      ctx._source.updated_at = new Date().toInstant().toString();
    """,
    "lang": "painless"
  }
}
```
</details>

### Puzzle 7 — Trade search for trading platform

Design the index mapping and query for a trading platform's trade search. Requirements:
- Search by symbol, date range, side (buy/sell)
- Aggregation: trading volume per symbol per day
- Aggregation: trader P&L
- Fast symbol + date range search

<details>
<summary>Answer</summary>

**Mapping:**
```bash
PUT /trades
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "symbol":    { "type": "keyword" },
      "price":     { "type": "double" },
      "qty":       { "type": "integer" },
      "side":      { "type": "keyword" },
      "trader_id": { "type": "keyword" },
      "timestamp": { "type": "date" },
      "order_id":  { "type": "keyword" }
    }
  }
}
```

**Index:** `{ symbol: 1, timestamp: -1 }` (via ES keyword + date — built-in)

**Query (fast symbol + date range):**
```bash
GET /trades/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "symbol": "AAPL" } },
        { "range": { "timestamp": { "gte": "now-1h" } } },
        { "term": { "side": "buy" } }
      ]
    }
  },
  "sort": [{ "timestamp": "desc" }],
  "size": 100
}
```

**Aggregation (volume per symbol per day):**
```bash
GET /trades/_search
{
  "size": 0,
  "aggs": {
    "per_symbol": {
      "terms": { "field": "symbol", "size": 20 },
      "aggs": {
        "per_day": {
          "date_histogram": {
            "field": "timestamp",
            "calendar_interval": "day"
          },
          "aggs": {
            "volume": { "sum": { "field": "qty" } },
            "total_value": { "sum": { "field": "price", "script": { "source": "doc['price'].value * doc['qty'].value" } } }
          }
        }
      }
    }
  }
}
```
</details>

### Puzzle 8 — Debug slow search

A search that used to take 20ms now takes 5 seconds. How do you diagnose?

<details>
<summary>Answer</summary>

1. **Check the profile API:**
   ```bash
   GET /products/_search?profile=true
   { "query": { ... } }
   ```
   Identifies which query clause or aggregation is slow.

2. **Check slow logs:**
   ```bash
   PUT /products/_settings
   {
     "index.search.slowlog.threshold.query.warn": "1s",
     "index.search.slowlog.threshold.query.info": "500ms"
   }
   ```

3. **Check resource pressure:**
   ```bash
   GET /_nodes/stats?filter_path=nodes.*.jvm
   GET /_nodes/hot_threads
   ```

4. **Check index state:**
   ```bash
   GET /products/_segments  # many small segments?
   GET /products/_stats     # large doc count?
   ```

5. **Common causes:**
   - Segment explosion (force-merge if read-only)
   - GC pressure (heap > 85%)
   - Data growth (shard too large — reindex with more shards)
   - New unoptimized query added (check profile)
   - Filter cache cold (after restart)
</details>

### Puzzle 9 — Hot-warm-cold architecture

Design an ILM-based hot-warm-cold architecture for logs.

<details>
<summary>Answer</summary>

```yaml
# Node configuration
# Hot nodes: fast SSD, high RAM
node.roles: ["data"]
node.attr.temperature: hot

# Warm nodes: regular SSD
node.roles: ["data"]
node.attr.temperature: warm

# Cold nodes: HDD or less RAM
node.roles: ["data"]
node.attr.temperature: cold
```

```bash
# ILM policy
PUT /_ilm/policy/logs_ilm
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": { "max_size": "50GB", "max_age": "1d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 },
          "allocate": { "require": { "temperature": "warm" } },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": { "require": { "temperature": "cold" } },
          "set_priority": { "priority": 0 }
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

### Puzzle 10 — Cluster health red

Cluster health shows `red`. How do you recover?

<details>
<summary>Answer</summary>

```bash
# 1. Identify unassigned shards
GET /_cat/shards?v&h=index,shard,prirep,state,node&s=state
# Look for UNASSIGNED primaries

# 2. Get allocation explanation
GET /_cluster/allocation/explain

# 3. Check for common causes:
# - Data node down? → Restart node
# - Disk full? → Free space or add nodes
# - Allocation disabled? → Enable
# - Too many shards per node? → Add nodes

# 4. If node is permanently lost and you have replica:
POST /_cluster/reroute
{
  "commands": [
    { "allocate_stale_primary": { "index": "my_index", "shard": 0, "node": "data-1" } }
  ]
}

# 5. If no replica, restore from snapshot:
POST /_snapshot/my_backup/snapshot_latest/_restore
{
  "indices": "my_index"
}

# 6. Last resort — allocate empty primary (DATA LOSS):
POST /_cluster/reroute
{
  "commands": [
    { "allocate_empty_primary": { "index": "my_index", "shard": 0, "node": "data-1" } }
  ]
}

# 7. Wait for green
GET /_cluster/health?wait_for_status=green&timeout=300s
```
</details>

---

## 3. Debugging scenarios (5 scenarios)

### Scenario 1 — Search latency increase

**Symptom:** Product search (previously 50ms) now takes 2-5 seconds.

**Debugging:**
1. `GET /products/_search?profile=true` for the same query
2. Check segment count: `GET /products/_segments`
3. Check heap: `GET /_nodes/stats?filter_path=nodes.*.jvm`
4. Check GC: `GET /_nodes/stats?filter_path=nodes.*.jvm.gc`
5. Check command stats: `GET /_nodes/stats?filter_path=nodes.*.indices.search`

**Likely causes:**
- **Many small segments**: Merge policy hasn't kept up. Force-merge or increase indexing throughput tuning.
- **Heap pressure**: GC consuming CPU. G1GC pauses.
- **Data growth**: Index has grown beyond optimal shard size.
- **New field with fielddata**: Someone added sorting on a text field.

**Solution:**
```bash
# If many segments (and index is read-only):
POST /products/_forcemerge?max_num_segments=5

# If heap pressure:
# Increase heap (but < 50% RAM, < 32 GB)
# Check for fielddata on text fields
# Optimize queries (use filter context)

# If data growth:
# Reindex with more shards
```

### Scenario 2 — Cluster yellow after node restart

**Symptom:** After restarting a data node, cluster stays yellow for hours.

**Debugging:**
1. `GET _cat/shards` — which shards are UNASSIGNED?
2. `GET _cluster/allocation/explain` — why?
3. Check disk usage on remaining nodes

**Likely causes:**
- **Disk watermark**: Existing nodes are above 85% — ES won't allocate replicas
- **Allocation decider**: Too many shards per node
- **Awareness**: Zone awareness configured but not enough nodes in each zone

**Solution:**
```bash
# Check disk usage
GET /_cat/allocation?v

# Temporarily increase watermarks
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
  }
}
```

### Scenario 3 — Rejected indexing requests

**Symptom:** Bulk indexing requests getting rejected. `_nodes/stats/indices/indexing` shows `throttled` or `rejected` > 0.

**Debugging:**
1. Check indexing queue: `GET /_nodes/stats?filter_path=nodes.*.indices.indexing`
2. Check thread pool: `GET /_cat/thread_pool?v`
3. Check circuit breakers: `GET /_nodes/stats?filter_path=nodes.*.breakers`

**Causes:**
- Queue full: too many concurrent indexing requests
- Circuit breaker: memory limit reached
- Node overloaded: CPU or I/O saturated

**Solutions:**
1. Throttle the client (reduce batch rate)
2. Increase `thread_pool.write.queue_size` (temporary)
3. Increase `indices.memory.index_buffer_size`
4. Increase refresh interval and use async translog
5. Add more data nodes

### Scenario 4 — Out of memory (OOM)

**Symptom:** Elasticsearch process killed by OOM killer. Check `/var/log/kern.log`.

**Debugging:**
1. Check last logs: `heap OOM` or `OutOfMemoryError`
2. Check circuit breaker trips before OOM: `breakers.tripped > 0`
3. Check fielddata, request cache, segments memory

**Common causes:**
- **Fielddata on text field**: Loading millions of unique terms into heap
- **Large aggregation**: `terms` aggregation with `size` set too high
- **Parent-child queries**: Loading entire child set
- **Mapping explosion**: Too many fields

**Solutions:**
1. Increase heap (up to 31 GB, < 50% RAM)
2. Limit aggregation size: `search.max_buckets`
3. Avoid fielddata on text fields — use keyword
4. Set `indices.fielddata.cache.size: 20%` of heap
5. Add nodes to distribute load

### Scenario 5 — Logstash pipeline slow

**Symptom:** Logs are hours behind. Filebeat sends data but Logstash can't keep up.

**Debugging:**
1. Check Logstash pipeline workers: `pipeline.workers` (default = CPU cores)
2. Check Logstash monitoring API: `GET /_node/stats/pipeline`
3. Check Elasticsearch indexing throughput: `GET /_cat/indices/logs-*`
4. Check `grok` patterns — grok is CPU-intensive

**Solutions:**
1. Increase `pipeline.workers` to match CPU cores
2. Use `grok` only when needed; use `dissect` for simpler parsing
3. Add more Logstash instances with different pipelines
4. Use Filebeat's `elasticsearch` output directly (skip Logstash)
5. Increase batch size: `pipeline.batch.size` (default 125) to 500-1000

---

## 4. System design prompts (4 prompts)

### Prompt 1 — Design a product search for a multi-tenant SaaS

Design Elasticsearch-based product search for your inventory management SaaS (10K tenants).

**Requirements:**
- Each tenant searches only their own products
- Autocomplete (search-as-you-type)
- Faceted search (by category, price range, in-stock)
- Multi-lingual support
- Real-time updates (product changes visible within 5 seconds)

**Design:**

```
Architecture:
  - Single index with org_id filter
  - Custom routing by org_id
  - ILM not needed (product data is permanent)
  - 5-10 primary shards (based on total data size)

Mapping:
  - name: search_as_you_type field (autocomplete)
  - description: text with multi-language analyzer
  - category: keyword (filtering)
  - price: double (range + sort)
  - tags: keyword (filtering)
  - in_stock: boolean (filtering)
  - org_id: keyword (routing + filter)

Query pattern:
  GET /products/_search?routing=org_42
  {
    "query": {
      "bool": {
        "must": { "multi_match": { "query": "wid", "fields": ["name", "name._2gram", "name._3gram"], "type": "bool_prefix" } },
        "filter": [
          { "term": { "org_id": "org_42" } },
          { "term": { "in_stock": true } },
          { "range": { "price": { "gte": 10, "lte": 100 } } }
        ]
      }
    },
    "aggregations": {
      "by_category": { "terms": { "field": "category" } },
      "price_ranges": { "range": { "field": "price", "ranges": [ ... ] } }
    }
  }
```

**Real-time updates:** Use `refresh_interval: 1s` (default) — changes visible within 1 second. No need for `_update_by_query` — partial updates work.

### Prompt 2 — Design a logging pipeline with ELK

Design a centralized logging pipeline for your multi-tenant SaaS.

**Requirements:**
- Collect logs from 100+ microservices
- 100 GB/day of log data
- Search logs for any tenant, any service
- Retain hot data for 7 days, warm for 30 days, cold for 90 days
- Real-time monitoring dashboard in Kibana

**Architecture:**

```
Services (Docker/K8s)
  ↓ Filebeat (log shipper, reads container logs)
  ↓
Logstash (filter: grok, mutate, date; output: ES)
  ↓
Elasticsearch (hot: 7 days, warm: 30 days, cold: 90 days)
  ↓
Kibana (dashboards, alerts, APM)

Storage:
  - Hot: 100 GB/day × 7 = 700 GB (3-5 data nodes, SSD)
  - Warm: same data, force-merged + shrunk (~200 GB total)
  - Cold: on HDD or frozen tier
```

**Index template:** See Puzzle 4.

**Kibana dashboard:**
- Log volume over time (date_histogram)
- Error rate by service (filter + terms)
- Top errors (significant_terms)
- Latency percentiles (percentiles aggregation)
- CPU/memory from Metricbeat

### Prompt 3 — Real-time trade search for trading platform

Design an Elasticsearch-based trade search for 20K+ DAU trading platform.

**Requirements:**
- Search trades by symbol, date range, side, trader
- Sub-second search latency
- 1M+ trades/day (write-heavy + search)
- Real-time aggregation (OHLCV per minute)

**Design:**

```
Index: trades-YYYY.MM.dd (daily indices)
Shards: 3 primary, 1 replica
ILM: Delete after 30 days (trading data has retention)
Refresh: Trade data needs > 99% reliability — set w=1 on index
```

**Mapping:** See Puzzle 7.

**Write pattern:** Bulk indexing with async translog.

**Search optimization:**
```bash
GET /trades-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "symbol": "AAPL" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [{ "timestamp": "desc" }],
  "size": 100
}
```

**OHLCV pre-aggregation:** Use a separate index for pre-computed OHLCV (updated every minute), query that instead of computing from raw trades.

### Prompt 4 — Search across heterogeneous document types

Your SaaS needs to search across products, orders, customers, and support tickets from a single search bar.

**Design:**

```bash
# Option 1: Single index with type discriminator
PUT /unified_search
{
  "mappings": {
    "properties": {
      "type":        { "type": "keyword" },    # "product", "order", "customer", "ticket"
      "org_id":      { "type": "keyword" },
      "name":        { "type": "search_as_you_type" },
      "description": { "type": "text" },
      "status":      { "type": "keyword" },
      "created_at":  { "type": "date" }
    }
  }
}

# Query across all types
GET /unified_search/_search
{
  "query": {
    "bool": {
      "must": { "multi_match": { "query": "widget", "fields": ["name", "description"] } },
      "filter": [
        { "term": { "org_id": "org_42" } },
        { "terms": { "type": ["product", "order", "ticket"] } }
      ]
    }
  },
  "aggregations": {
    "by_type": { "terms": { "field": "type" } }
  }
}
```

**Search experience (Kibana-like):**
- Show results grouped by type (faceted)
- Each type rendered differently (product = price + image, ticket = status + priority)
- Global search bar → unified results page

---

## 5. Key metrics to remember

| Metric | Target | Why |
|--------|--------|-----|
| Cluster health | `green` | Yellow > 5 min needs investigation |
| Heap usage | < 75% | > 90% → OOM risk |
| GC time | < 15% | > 30% → CPU wasted on GC |
| Search latency p99 | < 100ms | Slow searches degrade UX |
| Indexing rejection | 0 | Rejections = backpressure needed |
| Disk usage | < 75% | > 85% → allocation stops |
| Segment count per shard | < 50 | Many segments = slow search |
| Circuit breaker trips | 0 | Trips = memory limits hit |
| Pending tasks | 0 | Pending tasks = cluster instability |
| Merge rate | Low/steady | Sustained high merges = write pressure |

---

## 6. Interview preparation checklist

- [ ] I can explain the inverted index and how it powers full-text search
- [ ] I can write bool queries with filter and query context
- [ ] I can design explicit mappings with appropriate data types
- [ ] I can write metric, bucket, and pipeline aggregations
- [ ] I know how to size shards (20-50 GB rule)
- [ ] I can diagnose cluster health yellow/red
- [ ] I can design an ILM policy for time-series data
- [ ] I know how to tune for search vs indexing
- [ ] I understand JVM heap sizing (50% RAM, < 32 GB)
- [ ] I can explain hot-warm-cold architecture
- [ ] I can design a multi-tenant search strategy
- [ ] I know when ES is the wrong tool
- [ ] I understand the ELK Stack pipeline (Filebeat → Logstash → ES → Kibana)
- [ ] I can perform a rolling restart
- [ ] I can recover from a red cluster health
