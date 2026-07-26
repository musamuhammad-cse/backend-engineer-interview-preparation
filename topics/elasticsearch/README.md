# Elasticsearch — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** ELK Stack (Elasticsearch, Logstash, Kibana — listed in observability skills), multi-tenant SaaS (product search, log aggregation), trading platform (trade history search), Chronos (job logs search)  
> **Note:** Elasticsearch at the senior level tests whether you understand the inverted index, cluster architecture, shard strategy, and when to use it vs. traditional databases or other search solutions.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the query examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what kills senior loops | 5 min |

**The senior signal is shard strategy, cluster sizing, and knowing when Elasticsearch is the wrong tool.** Full-text search is table stakes — understanding the distributed nature, the near-real-time semantics, and the GC/pressure behavior of the JVM is what separates senior candidates.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Inverted index (how ES indexes and searches), documents & indices, mappings (dynamic/explicit), basic CRUD via REST API, query DSL (term, match, bool, range, exists), filter vs query context, relevance scoring (TF-IDF, BM25), analyzers (standard, simple, custom), basic aggregations (metrics, buckets) | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Advanced queries (constant_score, function_score, boosting, dis_max, multi_match), nested & parent-child queries, aggregations deep dive (pipeline, nested, geohash grid), index templates & dynamic templates, aliases, reindexing, shard & routing (routing field, custom routing), cluster architecture (node roles, discovery, Zen/FTR), snapshot & restore | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Performance tuning (shard sizing, segment merging, indexing buffer, refresh interval, translog), deep Elasticsearch cluster (hot-warm-cold architecture, ILM policies, cross-cluster search), tuning for search vs indexing throughput, JVM tuning (heap sizing, GC tuning, circuit breakers), production ops (rolling restarts, cluster upgrade, monitoring), ELK Stack (Logstash pipelines, Filebeat, Kibana), security (RBAC, TLS, audit), Elasticsearch vs alternatives (Solr, Meilisearch, Typesense, Atlas Search) | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 140+ interview questions, code puzzles (query writing, aggregation design), debugging scenarios (yellow cluster, slow searches, high JVM heap, no space left), system design prompts | Ongoing drill |

---

## Coverage map

### Core concepts
- Inverted index: term dictionary, postings list, segment structure
- Cluster, node, index, shard (primary + replica), document, field
- REST API (all operations via HTTP JSON)
- Mappings: dynamic vs explicit, data types (text, keyword, date, long, double, boolean, ip, geo_point, nested, object)
- Analysis: character filters, tokenizer, token filters; built-in analyzers (standard, simple, whitespace, keyword, pattern, language)
- Query DSL: leaf queries (term, match, range, exists, prefix, wildcard, regexp), compound queries (bool, boosting, constant_score, dis_max, function_score)
- Filter vs query context (scoring vs non-scoring, caching)
- Relevance scoring: TF-IDF (pre-5.0), BM25 (5.0+), explain API
- Aggregations: metric (avg, sum, min, max, stats, cardinality, percentiles), bucket (terms, range, date_histogram, histogram, geohash_grid, nested, filter), pipeline (avg_bucket, cumulative_sum, derivative, moving_avg, bucket_script)

### Indexing and mapping
- Index settings: number_of_shards, number_of_replicas, refresh_interval
- Dynamic templates: mapping for fields matching a pattern
- Index templates: apply settings + mappings to new indices matching pattern
- Data stream: append-only time-series indices (Elasticsearch 7.9+)
- Component templates: reusable building blocks for index templates
- Reindexing: `_reindex` API for copying data between indices
- Update by query: `_update_by_query`
- Delete by query: `_delete_by_query`
- Aliases: zero-downtime reindexing, multi-index search, filtering alias

### Cluster and shards
- Node roles: master-eligible, data, ingest, machine learning, coordinating
- Discovery: Zen (pre-7.x), cluster formation module (7.x+)
- Shard allocation: awareness, filtering, balancing
- Rack/zone awareness: `cluster.routing.allocation.awareness.attributes`
- Shard sizing: 20-50 GB per shard, 1 shard per GB of heap
- Segment merging: tiered merge policy, `_forcemerge` for read-only indices
- Cluster health: green, yellow, red — meaning and remediation

### Performance
- Refresh interval: near-real-time vs indexing throughput trade-off
- Translog: durability vs performance (`async`, `sync`, `request`)
- Indexing buffer (`indices.memory.index_buffer_size`)
- Fielddata vs doc_values (aggregation on text vs keyword)
- Circuit breakers: `indices.breaker.request`, `parent`, `fielddata`
- JVM tuning: heap size (< 50% RAM, < 32 GB), GC (G1GC preferred)
- Slow logs: search slow log, indexing slow log
- Hot threads: identify CPU hotspots

### Production
- Rolling restart procedure
- Full cluster restart
- Snapshot and restore (repository-s3, shared FS)
- Cross-cluster search (CCS)
- Cross-cluster replication (CCR)
- Index Lifecycle Management (ILM): hot → warm → cold → delete/frozen
- Monitoring: `_cat`, `_cluster/stats`, `_nodes/stats`, `_cluster/health`
- Elastic APM for application performance monitoring

### ELK Stack
- Logstash: input, filter (grok, mutate, date), output to Elasticsearch
- Filebeat: lightweight log shipper
- Kibana: visualization, dashboards, Discover, Dev Tools, Machine Learning
- Beats: Filebeat, Metricbeat, Heartbeat, Auditbeat

### Search features
- Full-text search: `match`, `match_phrase`, `multi_match`, `query_string`, `simple_query_string`
- Term-level queries: `term`, `terms`, `range`, `exists`, `prefix`, `wildcard`, `regexp`, `fuzzy`
- Compound queries: `bool` (must, should, filter, must_not), `boosting`, `constant_score`, `dis_max`, `function_score`
- Joins: `nested` (for nested objects), `has_child`, `has_parent` (for parent-child)
- geo: `geo_distance`, `geo_bounding_box`, `geo_shape`
- Specialized: `more_like_this`, `knn` (vector search, 8.0+), `percolate`
- Suggestion: `term suggester`, `phrase suggester`, `completion suggester`
- Highlighting: `plain`, `unified`, `fvh`
- Rescoring: `rescore` phase for top-N results

### Elasticsearch vs alternatives
| Feature | Elasticsearch | Solr | Meilisearch | Typesense |
|---------|--------------|------|-------------|-----------|
| Full-text search | Excellent | Excellent | Very good | Good |
| Analytics/aggregations | Excellent | Good | Limited | Limited |
| Distributed | Yes | Yes | Limited | Limited |
| Ops complexity | High | High | Low | Low |
| Time-series | Good (with ILM) | Limited | No | No |
| Vector search | Yes (8.0+) | No | No | No |

---

## Study order recommendation

Elasticsearch is likely less familiar than databases/Redis, so focus on fundamentals first.

```
Week 1:  01-basic.md          + Basic Q&A drill (query DSL from memory)
Week 2:  02-intermediate.md   + Intermediate Q&A drill (aggregation writing)
Week 3:  03-senior.md         + Senior Q&A drill (shard strategy, cluster design)
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** DynamoDB.
