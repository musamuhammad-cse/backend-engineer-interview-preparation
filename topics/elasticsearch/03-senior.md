# Elasticsearch — Senior

> **Target:** Senior Backend Engineer interview preparation  
> **Topic:** Performance tuning, shard strategy, ILM, ELK Stack, JVM/garbage collection, production operations, ES vs alternatives  
> **Real-world anchor:** ELK Stack for observability, SaaS product search, trading platform trade search, Chronos logs

---

## 1. Performance tuning

### Shard sizing strategy

Shard sizing is the #1 performance decision in Elasticsearch.

**Rules of thumb:**
- **20-50 GB per shard** — optimal for search performance
- **Max ~1,000 shards per node** (including replicas)
- **Max ~20,000 shards per cluster** (cluster metadata overhead)
- **1 shard per GB of heap** — if you have 30 GB heap, max ~30 shards per node
- **Minimum 1 shard per index** (even for small indices)

```bash
# Calculate shard count
# Desired shard count = (expected index size) / target shard size
# For a 500 GB index with 50 GB/shard: 500 / 50 = 10 primary shards

# Check current shard sizes
GET /_cat/shards?v&h=index,shard,prirep,store
```

**Trap:** More shards ≠ more speed. Each shard is a separate Lucene index with its own memory overhead (segments, caches, file handles). 100 GB split into 100 shards (1 GB each) performs worse than 2 shards (50 GB each) due to per-shard overhead.

### Segment merging

Elasticsearch writes segments incrementally. The **merge policy** controls how segments are merged into larger segments.

```bash
# Segment info
GET /products/_segments

# Force merge a read-only index (e.g., older log index)
POST /logs-2024.01/_forcemerge?max_num_segments=1
```

**Trap:** `_forcemerge` is an expensive operation (rewrites the entire shard). Only use on read-only indices (completed indices, older logs). Never force-merge a write-heavy index — you'll fight the merge policy.

**Tiered merge policy (default):**
- Smaller segments merge more frequently
- Larger segments merge less frequently
- Target: ~10 segments per shard for search performance

### Refresh interval

Controls how often indexed documents become searchable:

```bash
# Default: 1 second
# For indexing-heavy workloads, increase to reduce I/O:
PUT /logs-2024.01/_settings
{
  "index": { "refresh_interval": "30s" }
}

# Disable refresh during bulk indexing, re-enable after
PUT /my_index/_settings
{
  "index": { "refresh_interval": "-1" }
}
# … bulk index …
PUT /my_index/_settings
{
  "index": { "refresh_interval": "1s" }
}
```

**Trade-off:** Lower refresh interval = more real-time search, more segment overhead, more merge pressure. Higher = better indexing throughput, less real-time.

### Translog

The translog is the write-ahead log for durability (like PostgreSQL WAL or MongoDB journal):

```bash
# Default: request (fsync after every index/delete/update)
# For higher throughput, use async (fsync every 5 seconds):
PUT /my_index/_settings
{
  "index": { "translog.durability": "async", "translog.sync_interval": "5s" }
}
```

**Trade-off:** `async` gives ~5x write throughput but can lose up to 5 seconds of data on crash.

### Indexing buffer

```bash
# Percentage of heap used for indexing buffer (default 10%)
indices.memory.index_buffer_size: 10%
# Per-shard max buffer (default 512 MB)
indices.memory.min_index_buffer_size: 48mb
indices.memory.max_index_buffer_size: 512mb
```

**Trap:** If the indexing buffer fills up, segments are flushed to disk. Too small = frequent flushes (many small segments). Too large = high heap pressure.

### Search vs indexing throughput tuning

| Goal | Tuning |
|------|--------|
| **Search speed** | More replicas, `_forcemerge` read-only indices, doc_values on needed fields, filter cache sizing |
| **Indexing speed** | Higher refresh_interval, async translog, bulk indexing (batch size 1-15 MB per request), fewer replicas (0 during bulk), larger indexing buffer |
| **Both** | Hot-warm-cold architecture, appropriate shard count, SSD storage, adequate heap |

---

## 2. JVM and heap tuning

### Heap sizing

```bash
# elasticsearch.yml
-Xms16g
-Xmx16g
```

**Rules:**
- **Maximum 50% of RAM.** If server has 64 GB → max 31 GB heap. Rest goes to OS (filesystem cache, which ES heavily relies on).
- **Maximum 32 GB.** Above 32 GB, JVM switches to compressed OOPs (object pointers) → less memory-efficient.
- **Minimum 4 GB** for production (2 GB for master-only nodes).
- **Don't set min and max different** (Xms = Xmx) — prevents GC from resizing.

**Trap:** The ES documentation recommends **half of system RAM** for heap (up to 31 GB). The other half is used by the OS for filesystem cache — which is critical for Elasticsearch performance (Lucene relies on OS page cache for segment access).

### Garbage Collection

```bash
# Elasticsearch 7.x+ uses G1GC by default
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

**G1GC tuning:**
```bash
# If GC pauses > 200ms:
-XX:G1HeapRegionSize=4m         # default based on heap
-XX:G1ReservePercent=15         # default 10%
-XX:G1MixedGCLiveThresholdPercent=70
-XX:G1MixedGCCountTarget=8
-XX:G1OldCSetRegionThresholdPercent=10
```

**Monitoring GC:**
```bash
# Check GC stats
GET /_nodes/stats?filter_path=nodes.*.jvm.gc

# Long GC pauses are visible in:
# - Node response time spikes
# - Cluster state updates timing out
# - Circuits breaking
```

### Circuit breakers

Circuit breakers prevent OOM by rejecting requests that exceed memory limits:

```bash
# Circuit breaker configs (in elasticsearch.yml)
indices.breaker.request.limit: 60%        # per-request (e.g., aggregation memory)
indices.breaker.fielddata.limit: 40%      # fielddata cache
indices.breaker.total.limit: 70%          # total across all breakers
network.breaker.inflight_requests.limit: 100%  # inflight requests
```

```bash
# Monitor circuit breaker trips
GET /_nodes/stats?filter_path=nodes.*.breakers

# If "tripped" > 0, the circuit breaker is saving you from OOM
# Solutions:
# - Increase heap
# - Optimize queries (don't load too many terms)
# - Increase circuit breaker limits (only if you have headroom)
```

### Fielddata vs doc_values

```bash
# text fields use fielddata for sorting/aggregations (heap-based, expensive)
# keyword fields use doc_values (disk-based, efficient)

# Enable fielddata on a text field (NOT recommended — enable only if necessary):
PUT /my_index/_mapping
{
  "properties": {
    "description": {
      "type": "text",
      "fielddata": true  # can sort/aggregate on text
    }
  }
}
```

**Trap:** Enabling `fielddata` on `text` fields loads all unique terms into the heap. For a field with 10M unique values, this is GB of heap. Use `.keyword` sub-field (which uses doc_values) instead.

---

## 3. Hot-warm-cold architecture

Separate nodes into tiers based on data temperature:

```yaml
# hot node (elasticsearch.yml)
node.roles: ["data"]
node.attr.temperature: hot

# warm node
node.roles: ["data"]
node.attr.temperature: warm

# cold node
node.roles: ["data"]
node.attr.temperature: cold
```

```bash
# Allocate by temperature
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "index.routing.allocation.require.temperature": "hot"
    }
  }
}

# Manually move index to warm tier
PUT /logs-2024.01/_settings
{
  "index.routing.allocation.require.temperature": "warm"
}
```

### Index Lifecycle Management (ILM)

ILM automates index transitions through phases:

```bash
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          },
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
          "freeze": {},  # reduced memory usage (7.x+)
          "allocate": { "require": { "temperature": "cold" } },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**ILM lifecycle phases:**
| Phase | Action | Hardware |
|-------|--------|----------|
| **Hot** | Indexing + search | Fast SSD, high RAM |
| **Warm** | Search only (no indexing) | SSD, lower RAM |
| **Cold** | Occasional search | HDD, minimal RAM |
| **Frozen** | Rare search (partial) | S3/object store (8.0+) |
| **Delete** | Remove data | — |

---

## 4. ELK Stack (Elasticsearch, Logstash, Kibana)

### Logstash pipeline

```bash
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse Nginx access logs
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }

  # Parse custom JSON logs
  json {
    source => "message"
    skip_on_invalid_json => true
  }

  # Date parsing
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Add fields
  mutate {
    add_field => { "environment" => "production" }
    convert => { "[response][bytes]" => "integer" }
  }

  # GeoIP
  geoip {
    source => "client_ip"
    target => "geo"
  }

  # Drop sensitive fields
  mutate {
    remove_field => ["password", "credit_card"]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "password"
  }
}
```

### Filebeat

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  paths:
    - /var/log/nginx/access.log
  fields:
    service: nginx
    environment: production

- type: container
  paths:
    - /var/lib/docker/containers/*/*-json.log

output.logstash:
  hosts: ["logstash:5044"]
```

### Kibana

Kibana is the visualization layer. Key features:

| Feature | Use |
|---------|-----|
| **Discover** | Search and filter log data |
| **Dashboard** | Compose visualizations |
| **Canvas** | Custom infographics |
| **Lens** | Drag-and-drop visualizations |
| **Maps** | Geospatial data |
| **Machine Learning** | Anomaly detection, log rate analysis |
| **APM** | Application performance monitoring |
| **Uptime** | Synthetic monitoring |
| **Observability** | Logs, metrics, APM in one view |

---

## 5. Security

### Elasticsearch security (Elastic Stack)

```bash
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

# Create users
# Built-in: elastic, kibana_system, logstash_system, beats_system
# Via API:
POST /_security/user/my_user
{
  "password": "password123",
  "roles": ["logstash_writer"],
  "full_name": "My User"
}
```

### Role-based access control

```bash
# Create role for read-only user
POST /_security/role/log_reader
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"],
      "field_security": { "grant": ["@timestamp", "message", "level", "service"] },
      "query": { "term": { "environment": "production" } }
    }
  ],
  "cluster": ["monitor"]
}

# Create user with this role
POST /_security/user/my_user
{
  "password": "password",
  "roles": ["log_reader"]
}
```

### TLS configuration

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.certificate: /etc/elasticsearch/certs/es.crt
xpack.security.http.ssl.key: /etc/elasticsearch/certs/es.key
xpack.security.http.ssl.certificate_authorities: ["/etc/elasticsearch/certs/ca.crt"]

# Kibana
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]
elasticsearch.ssl.certificate: "/etc/kibana/certs/kibana.crt"
elasticsearch.ssl.key: "/etc/kibana/certs/kibana.key"
```

---

## 6. Production operations

### Rolling restart procedure

```bash
# 1. Disable shard allocation (prevent shard rebalancing)
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}

# 2. Perform sync flush (optional)
POST /_flush

# 3. Stop Elasticsearch on one node
systemctl stop elasticsearch

# 4. Perform maintenance (upgrade, config change, etc.)

# 5. Start Elasticsearch
systemctl start elasticsearch

# 6. Wait for node to join cluster
GET /_cat/nodes

# 7. Re-enable shard allocation
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}

# 8. Wait for cluster health to return to green
GET /_cluster/health?wait_for_status=green&timeout=300s

# 9. Repeat for next node
```

### Upgrade strategy

| From | To | Strategy |
|------|----|----------|
| 6.x | 7.x | **Reindex** — mapping changes are not backward compatible. Full reindex required. |
| 7.x | 7.y (minor) | Rolling upgrade. Upgrade nodes one at a time. |
| 7.x | 8.x | Rolling upgrade from 7.16+ (last 7.x compatible with 8.x). Check breaking changes. |

### Monitoring

```bash
# Cluster-level
GET /_cluster/health?wait_for_status=yellow&timeout=50s
GET /_cluster/settings
GET /_cluster/stats
GET /_cluster/state?filter_path=metadata,templates

# Node-level
GET /_nodes/stats
GET /_nodes/stats?filter_path=nodes.*.jvm
GET /_nodes/stats?filter_path=nodes.*.os
GET /_nodes/stats?filter_path=nodes.*.fs

# Index-level
GET /_cat/indices?v&s=store.size:desc
GET /_cat/shards?v&h=index,shard,prirep,state,store,node

# Performance
GET /_cat/segments?v
GET /_nodes/hot_threads

# Cluster health API (with timeout)
GET /_cluster/health?timeout=30s
```

### Key metrics to monitor

| Metric | Where | Target | Alert |
|--------|-------|--------|-------|
| Cluster health | `_cluster/health` | `green` | `yellow` > 5 min, `red` > 0 |
| JVM heap usage | `_nodes/stats/jvm` | < 75% | > 90% |
| GC time | `_nodes/stats/jvm/gc` | < 15% of time | > 30% |
| Search latency | `_nodes/stats/indices/search` | p99 < 100ms | p99 > 1s |
| Indexing rate | `_cat/indices` | Varies | Sudden drop |
| Merge rate | `_nodes/stats/indices/merges` | Low | Sustained high |
| CPU | `_nodes/stats/os` | < 70% | > 90% |
| Disk usage | `_cat/allocation` | < 75% | > 85%, > 90% (read-only) |
| Circuit breaker trips | `_nodes/stats/breakers` | 0 | > 0 |
| Fielddata cache | `_nodes/stats/indices/fielddata` | < 5% heap | Growing |
| Pending tasks | `_cluster/pending_tasks` | 0 | > 10 |

### Indexing troubleshooting

```bash
# Check for rejected indexing requests
GET /_nodes/stats?filter_path=nodes.*.indices.indexing

# Rejection reasons:
# - Queue full: too many concurrent indexing requests
# - Circuit breaker: memory limit exceeded
# - Mapping update: dynamic mapping conflict
```

---

## 7. Elasticsearch for time-series (logs, metrics)

### Time-series index pattern

```
logs-2024.01.15-000001  (hot — 50 GB, 1 day)
logs-2024.01.14-000001  (hot — 50 GB)
logs-2024.01.07-000001  (warm — force-merged, 1 shard)
logs-2023.12.01-000001  (cold — frozen)
...
```

### Best practices for time-series

1. **Use ILM for lifecycle management** — automate hot → warm → cold → delete
2. **Use `@timestamp` as the time field** — common convention
3. **Index per day or per week** — based on volume (50 GB/shard target)
4. **Use data streams** (7.9+) — simplifies index management
5. **Force-merge + shrink warm indices** — reduce segments and shards
6. **Use `_source` false for logs** (if you don't need original content) — saves disk
7. **Rollover by size, not time** — 50 GB is a good chunk size

```bash
# Disable _source for logs (saves ~50% disk)
PUT /_index_template/compact_logs
{
  "index_patterns": ["compact-logs-*"],
  "template": {
    "settings": { "number_of_shards": 2 },
    "mappings": {
      "_source": { "enabled": false },
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text", "index": false },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" }
      }
    }
  }
}
```

---

## 8. Elasticsearch vs alternatives

### Elasticsearch vs Solr

| Factor | Elasticsearch | Solr |
|--------|--------------|------|
| Ease of use | REST API, simpler setup | More config files |
| Analytics | Stronger aggregations | Good, but less versatile |
| Distributed | Native (ZEN/FTR) | Via ZooKeeper |
| Time-series | ILM + Rollup | Limited |
| Vector search | 8.0+ (dense_vector) | No |
| Monitoring | Built-in (X-Pack) | Via separate tools |
| Community | Larger | Smaller but mature |

### Elasticsearch vs Meilisearch

| Factor | Elasticsearch | Meilisearch |
|--------|--------------|-------------|
| Ease of use | Complex | Very simple |
| Full-text search | Excellent (BM25) | Excellent (custom ranking) |
| Typo tolerance | Manual (fuzzy) | Automatic |
| Filtering | Powerful DSL | Basic |
| Aggregations | Rich | None |
| Scaling | Distributed | Single-node |
| Best for | Enterprise search, logs, analytics | Front-end search, typo-tolerant |

### Elasticsearch vs Typesense

| Factor | Elasticsearch | Typesense |
|--------|--------------|-----------|
| Ease of use | Complex | Simple |
| Typo tolerance | Manual | Automatic |
| Faceting | Powerful | Limited |
| Performance | Tuning-dependent | Fast out of box |
| Vector search | Yes (8.0+) | Yes (0.24+) |
| Maturity | 10+ years | ~4 years |

### Elasticsearch vs MongoDB Atlas Search

| Factor | Elasticsearch | Atlas Search |
|--------|--------------|-------------|
| Integration | Separate stack | Built-in MongoDB |
| Full-text search | Excellent | Good (Lucene-based) |
| Aggregations | Pipeline | MongoDB aggregation |
| Operations | Self-managed or cloud | Fully managed |
| Time-series | ILM | MongoDB time-series |
| Best for | Search-first use cases | MongoDB-centric apps |

---

## 9. Common production issues and debugging

### Issue 1: Cluster health is yellow

**Symptom:** `GET _cluster/health` shows `status: yellow`.

**Diagnosis:**
- Yellow = unassigned replica shards
- `GET _cat/shards` shows shards with `state: UNASSIGNED`
- `GET _cluster/allocation/explain` for reason

**Common causes:**
- A data node is down (replicas can't be allocated)
- Disk watermark exceeded (ES won't allocate to near-full nodes)
- Shard allocation is disabled (`cluster.routing.allocation.enable: none`)
- Too many shards per node (allocation decider)

**Solutions:**
1. If node is down: restart or replace node
2. If allocation is disabled: re-enable
3. If disk is full: add nodes, free space, or increase watermark
4. If too many shards: add nodes or reduce shard count

### Issue 2: Cluster health is red

**Symptom:** `status: red` — some primary shards are unassigned.

**Diagnosis:**
- `GET _cat/shards` shows primaries with `state: UNASSIGNED`
- `GET _cluster/allocation/explain` for detailed reason

**Solutions:**
1. If a node is temporarily down: restart it
2. If a node is permanently lost: re-route shards from replicas (if available)
   ```bash
   POST /_cluster/reroute
   {
     "commands": [{ "allocate_stale_primary": { "index": "my_index", "shard": 0, "node": "my_node" } }]
   }
   ```
3. If no replica exists: restore from snapshot
4. As last resort: allocate empty primary (data loss):
   ```bash
   POST /_cluster/reroute
   {
     "commands": [{ "allocate_empty_primary": { "index": "my_index", "shard": 0, "node": "my_node" } }]
   }
   ```

### Issue 3: Search is slow

**Diagnosis:**
1. Check `_nodes/stats/indices/search` — query latency
2. Check `_nodes/hot_threads` — CPU hotspots
3. Check slow logs:
   ```bash
   PUT /my_index/_settings
   {
     "index.search.slowlog.threshold.query.warn": "1s",
     "index.search.slowlog.threshold.query.info": "500ms",
     "index.indexing.slowlog.threshold.index.warn": "10s"
   }
   ```
4. Run query with `?explain` and `?profile`:
   ```bash
   GET /my_index/_search?profile=true
   {
     "query": { ... }
   }
   ```

**Common causes:**
- No filter cache (filters not reused)
- Too many shards queried (search hits N shards)
- Wildcard/regex queries on high-cardinality fields
- Aggregations on high-cardinality fields
- Heap too small (frequent GC)
- Page cache misses (working set > RAM)

**Solutions:**
1. Use filter context for non-scoring conditions
2. Reduce shard count
3. Pre-compute aggregations
4. Increase heap (but < 50% RAM)
5. Add more nodes for parallelization

### Issue 4: High JVM heap usage

**Symptom:** `_nodes/stats/jvm` shows heap > 85%.

**Diagnosis:**
- `_nodes/stats/jvm/gc` — check GC time and collection count
- `_cat/fielddata` — fielddata cache size
- `_nodes/stats/indatives/fielddata` — fielddata evictions
- `_nodes/stats/indices/segments` — segment memory (nearly 50% of heap can be segments)

**Common causes:**
- Heavy aggregations on high-cardinality fields
- Fielddata on text fields enabled
- Too many segments per shard
- Parent-child queries

**Solutions:**
1. Increase heap (up to 31 GB, < 50% RAM)
2. Use doc_values instead of fielddata
3. Force-merge read-only indices
4. Aggregation optimization: use `execution_hint: "map"` for low-cardinality
5. Set `search.max_buckets` to limit aggregation scope
6. Use `composite` aggregation for high-cardinality terms

### Issue 5: Disk space running out

**Symptom:** ES enters read-only mode when disk > 95%.

```bash
# Check disk watermarks
GET /_cluster/settings?include_defaults=true&filter_path=*.disk.watermark

# Defaults:
# low: 85%, high: 90%, flood_stage: 95%
```

**Solutions:**
1. **Delete old indices** — `DELETE /logs-2023.01.*`
2. **Add nodes** to distribute data
3. **Shrink indices** — `POST /my_index/_shrink/my_index_small`
4. **Force-merge** to reclaim deleted document space
5. **Increase disk watermark** (temporary):
   ```bash
   PUT /_cluster/settings
   {
     "persistent": {
       "cluster.routing.allocation.disk.watermark.low": "90%",
       "cluster.routing.allocation.disk.watermark.high": "95%",
       "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
     }
   }
   ```

### Issue 6: Mapping explosion (too many fields)

**Symptom:** Cluster state size growing, slow cluster state updates, OOM on master nodes.

**Cause:** Dynamic mapping creates a field for every unique JSON key. A service logging request parameters creates a new field per parameter name → mapping explosion.

```bash
# Check mapping size
GET /my_index/_mapping?pretty
```

**Solutions:**
1. **Set `dynamic: false` or `dynamic: strict`** — don't allow dynamic fields
2. **Use `dynamic_templates`** to control how new fields are mapped
3. **`index.mapping.total_fields.limit`** — default 1000, can increase, but address the root cause
4. **Flatten nested documents** before indexing

```bash
# Limit total fields
PUT /my_index/_settings
{
  "index.mapping.total_fields.limit": 2000
}

# Disable dynamic mapping
PUT /my_index
{
  "mappings": {
    "dynamic": false,  # ignore unknown fields, don't index them
    "properties": {
      "known_field": { "type": "text" }
    }
  }
}
```

---

## 10. Elasticsearch for your SaaS

### Product search

For your multi-tenant SaaS, product search is a natural use case:

```bash
PUT /products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "product_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngram_filter"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "org_id":      { "type": "keyword" },
      "name":        { "type": "search_as_you_type" },     # autocomplete
      "description": { "type": "text", "analyzer": "english" },
      "sku":         { "type": "keyword" },
      "category":    { "type": "keyword" },
      "price":       { "type": "double" },
      "in_stock":    { "type": "boolean" },
      "tags":        { "type": "keyword" }
    }
  }
}
```

### Log aggregation

For your ELK Stack setup:

```bash
# Filebeat ships logs
# Logstash parses and enriches
# Elasticsearch indexes
# Kibana visualizes

# ILM policy for logs
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot":  { "min_age": "0ms", "actions": { "rollover": { "max_size": "50GB", "max_age": "1d" } } },
      "warm": { "min_age": "7d",  "actions": { "forcemerge": { "max_num_segments": 1 }, "shrink": { "number_of_shards": 1 }, "allocate": { "require": { "temperature": "warm" } } } },
      "cold": { "min_age": "30d", "actions": { "allocate": { "require": { "temperature": "cold" } } } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

---

## Interview traps cheatsheet — Senior

| Trap | The truth |
|------|-----------|
| "More shards = more parallelism = faster" | Each shard has ~1-2 GB of heap overhead. 1000 shards = 1-2 GB heap just for metadata. Rule: 20-50 GB/shard. |
| "More replicas = better search" | More replicas = more copies to search, yes. But also more disk and indexing overhead. Trade-off. |
| "Heap size can be > 32 GB" | Above 32 GB, JVM uses uncompressed OOPs → memory waste. Max 31 GB on 64 GB server. |
| "Force-merging always improves performance" | On read-only indices, yes. On write-indices, the merge will keep fighting. |
| "Dynamic mapping is convenient in production" | Mapping explosion risk — a malicious/misconfigured service can create 10K+ fields. |
| "Elasticsearch and PostgreSQL are interchangeable" | ES is for search/analytics. PostgreSQL is for relational data/transactions. Different tools. |
| "refresh_interval=1s means real-time search" | Near-real-time. Between refresh cycles, documents are in the buffer, not searchable. |
| "Circuit breaker = fail-safe, I can ignore them" | Circuit breakers prevent OOM. Frequent trips = you need more memory or better queries. |
| "Snapshot is a complete backup" | Only the first snapshot is full. Subsequent snapshots are incremental (stored as deltas). |
| "Elasticsearch monitors itself — no external tools needed" | Cluster health API is not enough. Monitor GC, heap, disk, circuit breakers, and slow logs externally. |
</details>
