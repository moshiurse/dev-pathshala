# 🔎 Elasticsearch — Distributed Search ও Analytics Engine

> "If your product has a search box, you'll eventually need Elasticsearch."
>
> Daraz catalog 50M product, Pathao support ticket, Prothom Alo news archive — সব Elasticsearch-এ। এই ডকুমেন্ট shallow তত্ত্ব নয়; production cluster চালানোর depth।

> 🔗 Full-text search ও inverted index-এর মূল ধারণার জন্য আগে `full-text-search.md` পড়ুন। এই ফাইল শুধু **Elasticsearch-নির্দিষ্ট** গভীর বিষয়।

---

## 📖 সূচিপত্র

- [Elasticsearch কী এবং কেন](#-elasticsearch-কী-এবং-কেন)
- [Cluster Architecture ও Node Roles](#-cluster-architecture-ও-node-roles)
- [Shards, Replicas, Segments, Lucene](#-shards-replicas-segments-lucene)
- [Index, Document, Mapping](#-index-document-mapping)
- [Field Types](#-field-types)
- [Analyzers ও Tokenizers (Bangla সহ)](#-analyzers-ও-tokenizers-bangla-সহ)
- [Indexing Pipeline ও Ingest Processors](#-indexing-pipeline-ও-ingest-processors)
- [Search APIs](#-search-apis)
- [Aggregations](#-aggregations)
- [Relevance: TF-IDF, BM25, Boosting](#-relevance-tf-idf-bm25-boosting)
- [Vector Search ও Hybrid](#-vector-search-ও-hybrid)
- [Performance Tuning](#-performance-tuning)
- [Cluster Sizing, JVM Heap, ILM, Snapshots](#-cluster-sizing-jvm-heap-ilm-snapshots)
- [Security](#-security)
- [Client Libraries (PHP + Node)](#-client-libraries-php--node)
- [Elasticsearch vs OpenSearch](#-elasticsearch-vs-opensearch)
- [BD Case Study — Daraz Product Search](#-bd-case-study--daraz-product-search)
- [Common Pitfalls](#-common-pitfalls)
- [Checklist](#-checklist)

---

## 📌 Elasticsearch কী এবং কেন

Elasticsearch (ES) হলো **Apache Lucene**-এর উপর তৈরি একটি **distributed, RESTful, JSON-based search এবং analytics engine**। Shay Banon ২০১০ সালে শুরু করেন। আজ Logging (ELK), e-commerce search, observability, security analytics, vector search — সব ক্ষেত্রে industry-standard।

### কী সমাধান করে?

```
সমস্যা                                      RDBMS                  Elasticsearch
─────────────────────────────────────────────────────────────────────────────────
"আইফোন ১৫" সার্চে "iPhone 15"               LIKE '%...%' স্লো       Match query, instant
Typo ("iphoen" → "iPhone")                  ❌                     Fuzzy match
Relevance ranking                            ❌                     BM25 scoring
50M doc-এ <100ms latency                     ❌ (full scan)         ✅
Real-time aggregation (top categories)       Slow GROUP BY          Aggregations API
Bangla + English mixed text                  Tokenization কঠিন       ICU + custom analyzer
Vector similarity (semantic search)          ❌                     dense_vector + kNN
Geo search ("৫km within Dhanmondi")          PostGIS                geo_point built-in
Log analytics (TB-scale)                     ❌                     ELK stack
```

### Use cases at a glance

- **Daraz** — product catalog search, faceted browsing
- **Pathao** — driver/parcel ticket full-text search
- **Prothom Alo / Bdnews24** — article search archive
- **bKash fraud team** — log analytics
- **Foodpanda** — restaurant + dish search
- **Chaldal** — grocery search with synonyms ("চিনি" = "sugar")

---

## 🏛️ Cluster Architecture ও Node Roles

```
                     ┌──────────────────────────────────────┐
                     │         Elasticsearch Cluster        │
                     │  cluster.name = daraz-search-prod    │
                     └────────┬─────────┬──────────┬────────┘
                              │         │          │
                ┌─────────────┘         │          └────────────────┐
                ▼                       ▼                            ▼
        ┌───────────────┐       ┌────────────────┐         ┌───────────────────┐
        │ Master Node   │       │   Data Nodes   │         │ Coordinator Node  │
        │ (3 dedicated) │       │ (hot/warm/cold)│         │  (REST gateway)   │
        │ cluster state │       │  shards live   │         │ scatter-gather    │
        └───────────────┘       └────────────────┘         └───────────────────┘
                                         │
                              ┌──────────┴──────────┐
                              ▼                     ▼
                       ┌─────────────┐       ┌─────────────┐
                       │ Ingest Node │       │  ML Node    │
                       │ pipelines   │       │ anomaly det.│
                       └─────────────┘       └─────────────┘
```

### Node Roles

| Role | দায়িত্ব | Production সাইজিং |
|------|---------|---------------------|
| **master** (eligible) | cluster state, shard allocation, mapping update | ৩টি dedicated, lightweight (4 vCPU, 8GB) |
| **data_hot** | new index, frequent write+read | NVMe SSD, বড় RAM (64GB+) |
| **data_warm** | কিছু পুরনো, read-mostly | SSD, মাঝারি RAM |
| **data_cold** | আরও পুরনো, rare read | HDD/snapshot mounted |
| **data_frozen** | searchable snapshot, S3-backed | minimal local |
| **ingest** | indexing-time pipeline (enrich, transform) | CPU-heavy, optional separate |
| **coordinator** (only) | query route + reduce; কোনো data নেই | API gateway-এর মতো |
| **ml** | anomaly detection, NLP inference | GPU/CPU heavy |
| **remote_cluster_client** | CCS — cross-cluster search | network |

### Quorum এবং Split-Brain

Master election Raft-like protocol। `discovery.seed_hosts` + `cluster.initial_master_nodes`। কখনো ২টি master দেবেন না — split-brain হয়। অন্তত ৩।

```
quorum = (master-eligible / 2) + 1
3 → 2     5 → 3     7 → 4
```

---

## 🧱 Shards, Replicas, Segments, Lucene

### Shard

প্রতিটি index N টি **primary shard**-এ বিভক্ত। প্রতিটি shard আসলে একটি independent **Lucene index**।

```
products index (primary=5, replica=1)
─────────────────────────────────────
Node A: P0, P3, R1, R4
Node B: P1, P4, R2, R0
Node C: P2,     R3
```

### Lucene Segment

একটি shard ভিতরে অনেকগুলো immutable segment। প্রতি `refresh_interval` (default 1s) -এ in-memory buffer → new segment ফাইল। Background-এ merge হয় (small segments → large)।

```
Shard (Lucene Index)
┌──────────────────────────────────────────────────┐
│ segment_1.cfs (immutable)   2GB                  │
│ segment_2.cfs (immutable)   1.8GB                │
│ segment_3.cfs (immutable)   500MB                │
│ segment_4.cfs (immutable)   100MB                │
│ in-memory buffer  (translog backed)              │
└──────────────────────────────────────────────────┘
        │
        └─→ merge → segment_5.cfs (new, deletes purge)
```

**Translog** — durability-এর জন্য WAL। `index.translog.durability: request` (every op fsync) বনাম `async`।

### Refresh vs Flush vs Commit

| Operation | কী হয় | কখন |
|-----------|--------|------|
| **refresh** | in-memory buffer → searchable (new segment), no disk fsync | প্রতি 1s (default) |
| **flush** | translog fsync + Lucene commit | প্রতি 30 min বা 512MB translog |
| **force merge** | অনেক segment → কম segment | manual, off-hour |

### Shard sizing rules of thumb

```
✅ Shard আকার  : 10GB – 50GB
✅ Heap-এ shard : প্রতি 1GB heap → ~20 shard (overhead-এর কারণে)
❌ অতি ছোট shard (1MB×1000) — overhead ভয়ঙ্কর
❌ অতি বড় shard (200GB) — relocation/recovery বিপদ
```

ILM rollover: `max_primary_shard_size: 50gb`, `max_age: 7d` — যা আগে আসে।

---

## 📂 Index, Document, Mapping

### Index, Document

Index ≈ DB-তে table; Document ≈ row (JSON object)।

```bash
PUT /products/_doc/SKU-IPHONE-15-256
{
  "title": "iPhone 15 Pro Max 256GB",
  "title_bn": "আইফোন ১৫ প্রো ম্যাক্স ২৫৬জিবি",
  "price_bdt": 195000,
  "stock": 12,
  "seller": { "id": 42, "name": "Daraz Mall", "rating": 4.7 },
  "category": ["Mobile","Smartphone","Apple"],
  "tags": ["new","featured","5g"],
  "created_at": "2024-09-22T10:15:00Z",
  "location": { "lat": 23.7515, "lon": 90.3782 },
  "embedding": [0.012, -0.045, ...]   // 768 dim
}
```

### Mapping

Schema; field → type, analyzer, indexing options।

```bash
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s",
    "analysis": { ... }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "title":      { "type": "text", "analyzer": "english_custom",
                      "fields": { "keyword": { "type":"keyword","ignore_above":256 }}},
      "title_bn":   { "type": "text", "analyzer": "bangla_custom" },
      "price_bdt":  { "type": "long" },
      "stock":      { "type": "integer" },
      "seller": {
        "properties": {
          "id":     { "type": "long" },
          "name":   { "type": "keyword" },
          "rating": { "type": "float" }
        }
      },
      "category":   { "type": "keyword" },
      "tags":       { "type": "keyword" },
      "created_at": { "type": "date" },
      "location":   { "type": "geo_point" },
      "embedding":  { "type": "dense_vector", "dims": 768,
                      "index": true, "similarity": "cosine" }
    }
  }
}
```

### Dynamic Mapping — সাবধান!

ডিফল্টে নতুন field auto-mapped হয়। বিপদ:
- "id" field "1234" string হিসেবে index হলে পরে "1234.5" এলে failure
- **Mapping explosion** — random key (UUID) field → ১০০০+ field, heap বিপদ

```bash
"dynamic": "strict"   // unknown field → error
"dynamic": "false"    // unknown field stored, not indexed
"dynamic": true       // default — সাবধান
```

### Reindex API

Mapping পরিবর্তন → নতুন index তৈরি → `_reindex` → alias swap।

```bash
POST /_reindex
{ "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" } }

POST /_aliases
{ "actions": [
  { "remove": { "index":"products_v1", "alias":"products" } },
  { "add":    { "index":"products_v2", "alias":"products" } }
]}
```

---

## 🔢 Field Types

| Type | উদাহরণ | কখন |
|------|--------|------|
| `text` | "iPhone 15 Pro" | full-text search, analyzed |
| `keyword` | "category:Mobile" | exact match, aggregate, sort, term |
| `long`, `integer`, `short`, `byte` | id, count | numeric range |
| `float`, `double`, `half_float` | rating, price (paisa হলে long ভাল) | numeric |
| `scaled_float` | price (scaling_factor=100) | money — recommended |
| `boolean` | featured | true/false |
| `date` | "2024-09-22T10:00:00Z" | range, date_histogram |
| `date_nanos` | high-precision time | trading, telemetry |
| `ip` | "203.190.13.12" | range query |
| `geo_point` | { lat, lon } | distance, bounding box |
| `geo_shape` | polygon | complex geo |
| `dense_vector` | [0.1, -0.2, ...] | kNN semantic search |
| `sparse_vector` | learned sparse | ELSER / hybrid |
| `nested` | array of objects (independent) | review array, line items |
| `object` (default) | nested JSON, flattened | simple nested |
| `flattened` | unbounded keys | meta tags |
| `join` | parent-child | rare; usually denormalize |
| `runtime` | computed at query time | schema-less columns |
| `completion` | autocomplete suggester | search-as-you-type |
| `search_as_you_type` | edge ngram পদ্ধতি | autocomplete |

> ⚠️ **Money:** `float` ব্যবহার করবেন না। `scaled_float` (scaling 100) অথবা `long paisa` ব্যবহার করুন।

### `text` vs `keyword` — সবচেয়ে সাধারণ ভুল

```json
"title": {
  "type":"text",                                   // search-এর জন্য
  "fields":{
    "keyword":{ "type":"keyword","ignore_above":256 } // sort/agg-এর জন্য
  }
}
```

`title.keyword` ব্যবহার করুন সর্ট/aggregation-এ; `title` ব্যবহার করুন match query-তে।

---

## 🔠 Analyzers ও Tokenizers (Bangla সহ)

Analyzer = Char filter → Tokenizer → Token filter chain।

```
Input  : "Apple iPhone-15 Pro Max!!"
─────────────────────────────────────
char_filter (HTML strip, mapping):
   "Apple iPhone-15 Pro Max"

tokenizer (standard):
   ["Apple", "iPhone", "15", "Pro", "Max"]

token_filter (lowercase, stop, stemmer):
   ["appl", "iphon", "15", "pro", "max"]

→ inverted index-এ সংরক্ষিত
```

### Built-in analyzers

| Analyzer | কী করে |
|----------|--------|
| `standard` | Unicode tokenization + lowercase |
| `simple` | non-letter-এ split + lowercase |
| `whitespace` | শুধু whitespace split (case রাখে) |
| `keyword` | পুরো string একটি token (no analyze) |
| `english`, `french`, ... | language-specific stemmer + stopwords |
| `pattern` | regex-based |
| `fingerprint` | dedup, sort tokens — fuzzy matching |

### N-gram ও Edge N-gram (autocomplete)

```
"chaldal"  edge_ngram(2,8) →
   "ch","cha","chal","chald","chalda","chaldal"
```

প্রথম অক্ষর থেকে prefix index → instant autocomplete।

```json
"settings": {
  "analysis": {
    "tokenizer": {
      "edge_ng_tokenizer": {
        "type":"edge_ngram", "min_gram":2, "max_gram":15,
        "token_chars":["letter","digit"]
      }
    },
    "analyzer": {
      "autocomplete": { "tokenizer":"edge_ng_tokenizer", "filter":["lowercase"] },
      "autocomplete_search": { "tokenizer":"lowercase" }
    }
  }
}
```

### Bangla Analyzer

ES-এ built-in `bengali` analyzer আছে (Lucene 6.4+) — stemming + stopwords। কিন্তু production-এ প্রায়ই custom লাগে কারণ:

- Mixed script ("আইফোন iPhone")
- Bangla numerals ("১৫" → normalize "15")
- ICU normalization দরকার

```json
"settings": {
  "analysis": {
    "char_filter": {
      "bn_digit_to_en": {
        "type":"mapping",
        "mappings":["০=>0","১=>1","২=>2","৩=>3","৪=>4",
                    "৫=>5","৬=>6","৭=>7","৮=>8","৯=>9"]
      }
    },
    "filter": {
      "bn_stop": { "type":"stop", "stopwords":["এর","ও","একটি","এই","এবং","হল"] },
      "bn_stem": { "type":"stemmer", "language":"bengali" },
      "icu_norm":{ "type":"icu_normalizer", "name":"nfkc_cf", "mode":"compose" }
    },
    "analyzer": {
      "bangla_custom": {
        "char_filter":["bn_digit_to_en"],
        "tokenizer":"icu_tokenizer",
        "filter":["icu_norm","bn_stop","bn_stem"]
      }
    }
  }
}
```

> 🛠️ Plugin: `analysis-icu`, `analysis-phonetic` install করতে হবে।

### Synonyms

`চিনি` ↔ `sugar`, `ডিম` ↔ `egg` — Chaldal-এর জন্য জরুরি।

```json
"filter": {
  "bn_synonyms": {
    "type":"synonym_graph",
    "synonyms":[
      "চিনি, sugar",
      "ডিম, egg",
      "মোবাইল, phone, ফোন",
      "টিভি, tv, television"
    ]
  }
}
```

`synonym_graph` (multi-token aware) preferred over old `synonym`। **Search analyzer-এ apply করুন**, index analyzer-এ নয় — তাহলে synonyms পরিবর্তন করতে reindex লাগবে না।

---

## 🚰 Indexing Pipeline ও Ingest Processors

Ingest pipeline = indexing-এর আগেই document transformation।

```bash
PUT /_ingest/pipeline/product_enrich
{
  "processors": [
    { "lowercase": { "field": "title", "target_field": "title_normalized" } },
    { "set":       { "field": "indexed_at", "value": "{{_ingest.timestamp}}" } },
    { "script":    { "source":
        "ctx.price_bdt_range = ctx.price_bdt < 5000 ? 'low' : (ctx.price_bdt < 50000 ? 'mid' : 'high');" }},
    { "geoip":     { "field": "client_ip", "target_field": "geo" } },
    { "remove":    { "field": "client_ip" } }
  ]
}

PUT /products/_doc/1?pipeline=product_enrich
{ "title":"iPhone 15", "price_bdt":195000, "client_ip":"203.190.13.12" }
```

Painless = ES-এর secure scripting language (Java-like)।

---

## 🔍 Search APIs

### Match Query (full-text)

```bash
GET /products/_search
{
  "query": {
    "match": {
      "title_bn": {
        "query":"আইফোন প্রো",
        "operator":"and",
        "fuzziness":"AUTO"
      }
    }
  }
}
```

### Term Query (exact, no analysis)

```bash
{ "query": { "term": { "category": "Mobile" } } }
```

> ❌ `term` কখনো `text` field-এ ব্যবহার করবেন না — analyzed আকারে index হয়, যা আপনি জানেন না।

### Bool Query — must / should / must_not / filter

```bash
GET /products/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "title_bn":"আইফোন" } } ],
      "should":   [ { "match": { "tags":"featured" }, "boost":2 } ],
      "filter":   [
        { "term":  { "category":"Mobile" } },
        { "range": { "price_bdt": { "gte":50000, "lte":250000 }}},
        { "term":  { "in_stock": true } }
      ],
      "must_not": [ { "term": { "seller.banned": true } } ]
    }
  },
  "size": 20
}
```

`filter` clause score-এ অংশ নেয় না, **filter cache** ব্যবহার করে — fast।

### Multi-match

```json
{ "multi_match": {
    "query":"iphone 15",
    "fields":["title^3","title_bn^3","description","tags^2"],
    "type":"best_fields",
    "fuzziness":"AUTO"
}}
```

`type`:
- `best_fields` (default) — সেরা একটি field-এর score
- `most_fields` — সব field combine
- `cross_fields` — বহু field-এ ছড়ানো term (e.g. firstname+lastname)
- `phrase` — phrase match
- `phrase_prefix` — autocomplete

### Range, Geo

```json
{ "range": { "created_at": { "gte":"now-30d/d" } } }

{ "geo_distance": { "distance":"5km", "location":{ "lat":23.78,"lon":90.40 } } }
```

### Function Score / Scripted Score

Custom ranking — যেমন seller rating + recency boost:

```json
{ "function_score": {
    "query": { "match": { "title":"iphone" } },
    "functions": [
      { "field_value_factor": { "field":"seller.rating","factor":1.2,"modifier":"sqrt" } },
      { "gauss": { "created_at": { "origin":"now","scale":"30d","decay":0.5 } } },
      { "filter": { "term":{"featured":true} }, "weight":1.5 }
    ],
    "score_mode":"sum","boost_mode":"multiply"
}}
```

### More Like This (MLT) — "similar products"

```json
{ "more_like_this": {
    "fields":["title","description"],
    "like":[ { "_index":"products","_id":"SKU-IPHONE-15-256" } ],
    "min_term_freq":1,"max_query_terms":12
}}
```

### Pagination — search_after, NOT deep `from`

```
❌ from:10000, size:20    → ১০০২০ doc fetch + sort, প্রতিটি shard থেকে
                            → max_result_window=10000 hit
✅ search_after: [last_score, last_id]   → cursor, infinite scroll
✅ scroll API : bulk export-এ
✅ point_in_time (PIT) + search_after : modern best
```

```bash
POST /products/_pit?keep_alive=1m
GET /_search
{
  "size":50,
  "pit":{ "id":"<pit_id>", "keep_alive":"1m" },
  "sort":[ {"created_at":"desc"}, {"_id":"asc"} ],
  "search_after":[ "2024-09-22T10:00:00Z", "SKU-..." ]
}
```

---

## 📊 Aggregations

ES = real-time analytics engine। Aggregation type:

| Type | কী | উদাহরণ |
|------|----|--------|
| **Bucket** | group by-এর মতো | `terms`, `range`, `date_histogram`, `geo_grid` |
| **Metric** | numeric calculation | `avg`, `sum`, `min`, `max`, `cardinality`, `percentiles`, `stats` |
| **Pipeline** | aggregation-এর উপর aggregation | `bucket_script`, `derivative`, `moving_avg` |
| **Matrix** | multiple field correlation | `matrix_stats` |

### Daraz dashboard — top categories + price percentiles

```json
GET /products/_search
{
  "size":0,
  "query":{ "term":{"in_stock":true} },
  "aggs":{
    "by_category":{
      "terms":{ "field":"category","size":10 },
      "aggs":{
        "price_pct":{ "percentiles":{ "field":"price_bdt","percents":[50,90,99] } },
        "unique_sellers":{ "cardinality":{ "field":"seller.id" } }
      }
    },
    "sales_per_day":{
      "date_histogram":{ "field":"created_at","calendar_interval":"day" },
      "aggs":{
        "revenue":{ "sum":{ "field":"price_bdt" } },
        "growth": { "derivative":{ "buckets_path":"revenue" } }
      }
    }
  }
}
```

### Cardinality — unique count

`cardinality` = HyperLogLog++ approximation। Exact count চাইলে `terms` size big, কিন্তু expensive।

---

## 🎯 Relevance: TF-IDF, BM25, Boosting

### BM25 (Default since 5.0)

ES-এ default scoring। Lucene's BM25 implementation:

```
score(D,Q) = Σ IDF(q) · (f(q,D)·(k1+1)) / (f(q,D) + k1·(1 - b + b·|D|/avgdl))

f(q,D)  : term frequency in document
|D|     : document length
avgdl   : average document length in field
k1=1.2  : term frequency saturation
b=0.75  : length normalization
```

**Tuning:** Per-field similarity:

```json
"similarity": {
  "bm25_short": { "type":"BM25","k1":0.8,"b":0.3 }
},
"mappings": {
  "properties":{
    "title":{ "type":"text","similarity":"bm25_short" }
  }
}
```

### Boosting

| কোথায় | উদাহরণ |
|--------|--------|
| Field boost | `"title^3"` |
| Query boost | `{ "match":{ "title":{ "query":"x","boost":2 } }}` |
| Index boost | multi-index search |
| Function score | popularity, recency |

### Rescore — দ্বিতীয় round expensive scoring

প্রথম stage: BM25 top 1000; দ্বিতীয় stage: ML model বা heavy script top 1000-এ চালান।

```json
{ "query":{...},
  "rescore":{
    "window_size":1000,
    "query":{
      "score_mode":"multiply",
      "rescore_query":{ "function_score":{...} }
    }
  }
}
```

### Explain API — score debug

```bash
GET /products/_search?explain=true
```

Response-এ field-by-field BM25 calculation breakdown। Production-এ off; debug-এ অমূল্য।

---

## 🧠 Vector Search ও Hybrid

### dense_vector + kNN (HNSW)

8.0+ থেকে native HNSW। সেমান্টিক সার্চ — embedding দিয়ে।

```json
PUT /products
{
  "mappings":{
    "properties":{
      "embedding":{
        "type":"dense_vector",
        "dims":768,
        "index":true,
        "similarity":"cosine",
        "index_options":{ "type":"hnsw","m":16,"ef_construction":100 }
      }
    }
  }
}
```

```bash
GET /products/_search
{
  "knn":{
    "field":"embedding",
    "query_vector":[0.012,-0.045,...],
    "k":50,
    "num_candidates":500
  }
}
```

### Hybrid — Lexical + Vector + RRF

Reciprocal Rank Fusion (8.8+):

```bash
GET /products/_search
{
  "retriever":{
    "rrf":{
      "retrievers":[
        { "standard":{ "query":{ "match":{ "title":"iphone 15" } } } },
        { "knn":{ "field":"embedding","query_vector":[...],"k":50,"num_candidates":500 } }
      ],
      "rank_window_size":50,
      "rank_constant":60
    }
  }
}
```

> 🔗 Embedding generation, vector DB tradeoff (Pinecone, Weaviate, pgvector vs ES) — পৃথক ফাইল `21-ai-engineering/vector-databases.md`-এ।

### sparse_vector / ELSER

ES নিজস্ব sparse encoder — neural ranking, no embedding API call। Daraz Bangla query-তে BM25-এর পরিপূরক।

---

## ⚡ Performance Tuning

### Indexing-side

```
✅ Bulk API ব্যবহার   → 5-15MB per request, 1000-5000 doc
✅ refresh_interval বাড়ান → "30s" বা bulk import-এ "-1"
✅ replica বন্ধ → number_of_replicas:0; পরে চালু
✅ translog async → durability:async (data loss tolerance থাকলে)
✅ doc-এ _id auto generate → routing fast
❌ একটি একটি করে index PUT — slow
```

```bash
POST /products/_bulk
{ "index":{ "_id":"SKU-1" } }
{ "title":"...", "price_bdt":195000 }
{ "index":{ "_id":"SKU-2" } }
{ "title":"...", "price_bdt":85000 }
```

### Search-side

```
✅ filter clause    → cached, fast
✅ source filtering : "_source":["id","title"]   → kB save
✅ doc_values      : sort/agg-এর জন্য already on
✅ keyword fields  : eager_global_ordinals → terms agg fast
✅ index sort      : "index.sort.field" — top-N early termination
✅ aggregations-এ size:0 → hit return-এর cost বাঁচে
✅ profile API     : query bottleneck identify
```

### Query Profiling

```bash
GET /products/_search
{ "profile":true, "query":{...} }
```

Time per shard, per query node — optimization-এর সোনার খনি।

### Common slow query patterns

```
❌ wildcard "*foo*"          → leading wildcard, no index use; reverse field দিয়ে
❌ regexp                     → CPU intensive
❌ deep from:10000            → max_result_window
❌ aggregations on text       → fielddata explosion; keyword field use
❌ script_score everything    → cache miss
```

---

## 📐 Cluster Sizing, JVM Heap, ILM, Snapshots

### JVM Heap Rules

```
✅ Heap ≤ 32GB  (compressed oops boundary)
✅ Heap ≤ 50% of physical RAM (rest = OS file cache for Lucene)
✅ Use G1GC (default 8.0+)
❌ 64GB heap → বরং 31GB heap রাখুন; বাকি RAM Lucene-এর জন্য
```

### Cluster Sizing — Daraz example

```
Data size : 50M product × 2KB avg = 100GB
Replica   : ×2 = 200GB
Overhead  : ×1.4 = 280GB

Shard size target 30GB → 280/30 ≈ 10 primary shard
Replica 1 → 20 total shard

Node count: 5 data nodes (warm spread)
RAM per node: 64GB (heap 31GB + OS cache 33GB)
Disk: NVMe 500GB usable per node
```

### ILM (Index Lifecycle Management)

Hot → Warm → Cold → Frozen → Delete।

```bash
PUT /_ilm/policy/logs_policy
{
  "policy":{
    "phases":{
      "hot":   { "actions":{ "rollover":{ "max_primary_shard_size":"50gb","max_age":"7d" } } },
      "warm":  { "min_age":"7d",  "actions":{ "shrink":{ "number_of_shards":1 },
                                              "forcemerge":{ "max_num_segments":1 },
                                              "allocate":{ "include":{ "data":"warm" } } } },
      "cold":  { "min_age":"30d", "actions":{ "searchable_snapshot":{ "snapshot_repository":"s3-mumbai" } } },
      "frozen":{ "min_age":"90d", "actions":{ "searchable_snapshot":{ "snapshot_repository":"s3-mumbai" } } },
      "delete":{ "min_age":"365d","actions":{ "delete":{} } }
    }
  }
}
```

### Snapshot to S3

```bash
PUT /_snapshot/s3-mumbai
{ "type":"s3",
  "settings":{ "bucket":"daraz-es-backup","region":"ap-south-1","base_path":"prod" } }

PUT /_snapshot/s3-mumbai/snap-2024-09-22?wait_for_completion=false
{ "indices":"products,orders", "include_global_state":false }

POST /_snapshot/s3-mumbai/snap-2024-09-22/_restore
```

Searchable snapshots → cold/frozen tier-এ S3 থেকে on-demand mount। বিশাল cost saving।

---

## 🔐 Security

### Authentication

- **Native realm** (built-in user store)
- **LDAP / AD**
- **SAML / OIDC** (enterprise SSO)
- **API keys** — service-to-service
- **PKI** (client cert)

### Authorization — RBAC

```bash
POST /_security/role/daraz_search_app
{
  "indices":[
    { "names":["products","categories"], "privileges":["read","view_index_metadata"] },
    { "names":["search_logs"], "privileges":["create"] }
  ],
  "applications":[]
}

POST /_security/user/app_search
{ "password":"...", "roles":["daraz_search_app"], "full_name":"Daraz search service" }
```

### Field & Document Level Security (Platinum/Enterprise)

```json
{ "indices":[{
    "names":["users"],
    "privileges":["read"],
    "field_security":{ "grant":["id","name","email"], "except":["password","ssn"] },
    "query":"{\"term\":{\"tenant_id\":\"daraz\"}}"
}]}
```

### Audit logging

`xpack.security.audit.enabled: true` — auth event, index access, mapping change।

### Network

- TLS on transport (node↔node) **mandatory** in production
- TLS on HTTP layer
- IP filter / VPC private subnet

---

## 💻 Client Libraries (PHP + Node)

### PHP — `elasticsearch/elasticsearch` (official)

```bash
composer require elasticsearch/elasticsearch:^8.0
```

```php
<?php
use Elastic\Elasticsearch\ClientBuilder;

$client = ClientBuilder::create()
    ->setHosts(['https://es.daraz.local:9200'])
    ->setBasicAuthentication('app_search', getenv('ES_PASS'))
    ->setCABundle('/etc/ssl/es-ca.crt')
    ->build();

// 1. Index a Daraz product (Bangla title)
$response = $client->index([
    'index' => 'products',
    'id'    => 'SKU-IPHONE-15-256',
    'body'  => [
        'title'     => 'iPhone 15 Pro Max 256GB',
        'title_bn'  => 'আইফোন ১৫ প্রো ম্যাক্স ২৫৬জিবি',
        'price_bdt' => 195000,
        'stock'     => 12,
        'seller'    => ['id'=>42,'name'=>'Daraz Mall','rating'=>4.7],
        'category'  => ['Mobile','Smartphone','Apple'],
        'tags'      => ['new','featured','5g'],
        'created_at'=> date('c'),
        'location'  => ['lat'=>23.7515,'lon'=>90.3782],
    ],
]);

// 2. Bulk index
$bulk = ['body' => []];
foreach ($products as $p) {
    $bulk['body'][] = [ 'index' => [ '_index'=>'products','_id'=>$p['sku'] ] ];
    $bulk['body'][] = $p;
    if (count($bulk['body']) >= 1000) {
        $client->bulk($bulk);
        $bulk = ['body'=>[]];
    }
}
if (! empty($bulk['body'])) $client->bulk($bulk);

// 3. BM25 search — Bangla + English mixed
$result = $client->search([
    'index' => 'products',
    'body'  => [
        'query' => [
            'bool' => [
                'must' => [[
                    'multi_match' => [
                        'query'  => 'আইফোন pro',
                        'fields' => ['title^3','title_bn^3','description'],
                        'type'   => 'best_fields',
                        'fuzziness' => 'AUTO',
                    ]
                ]],
                'filter' => [
                    [ 'term'  => [ 'category' => 'Mobile' ] ],
                    [ 'range' => [ 'price_bdt' => [ 'gte'=>50000, 'lte'=>250000 ] ] ],
                ],
            ],
        ],
        'sort' => [ '_score', [ 'seller.rating' => 'desc' ] ],
        'size' => 20,
        '_source' => ['title','price_bdt','seller.name','stock'],
    ]
]);

foreach ($result['hits']['hits'] as $hit) {
    printf("%-50s ৳%d  ★%.1f\n",
        $hit['_source']['title'],
        $hit['_source']['price_bdt'],
        $hit['_source']['seller']['rating']
    );
}

// 4. kNN semantic search
$knn = $client->search([
    'index' => 'products',
    'body'  => [
        'knn' => [
            'field'         => 'embedding',
            'query_vector'  => $embeddingFromOpenAi,    // 768-d
            'k'             => 30,
            'num_candidates'=> 300,
            'filter' => [ 'term' => [ 'in_stock' => true ] ],
        ],
        '_source' => ['title','price_bdt'],
    ]
]);

// 5. Aggregation — top selling categories last 30d
$agg = $client->search([
    'index' => 'orders',
    'body'  => [
        'size' => 0,
        'query' => [ 'range' => [ 'created_at' => [ 'gte' => 'now-30d/d' ] ] ],
        'aggs' => [
            'by_cat' => [
                'terms' => [ 'field' => 'category', 'size' => 10 ],
                'aggs'  => [ 'revenue' => [ 'sum' => [ 'field' => 'amount_bdt' ] ] ]
            ]
        ]
    ]
]);
```

### Node.js — `@elastic/elasticsearch`

```ts
import { Client } from '@elastic/elasticsearch';

const es = new Client({
  node: 'https://es.daraz.local:9200',
  auth: { username: 'app_search', password: process.env.ES_PASS! },
  tls:  { ca: fs.readFileSync('/etc/ssl/es-ca.crt') },
});

// Index
await es.index({
  index: 'products',
  id:    'SKU-IPHONE-15-256',
  document: {
    title: 'iPhone 15 Pro Max 256GB',
    title_bn: 'আইফোন ১৫ প্রো ম্যাক্স',
    price_bdt: 195000,
    seller: { id: 42, rating: 4.7 },
    embedding: embedding768,
  }
});

// Bulk via helpers
import { Client as Cli } from '@elastic/elasticsearch';
await es.helpers.bulk({
  datasource: products,                      // async iterable
  onDocument: (doc) => ({ index:{ _index:'products', _id: doc.sku } }),
  flushBytes: 5_000_000,
  concurrency: 4,
});

// Search with filter + multi_match + sort
const res = await es.search<ProductDoc>({
  index: 'products',
  query: {
    bool: {
      must: [{ multi_match: { query:'আইফোন pro', fields:['title^3','title_bn^3'], fuzziness:'AUTO' } }],
      filter: [
        { term:  { category: 'Mobile' } },
        { range: { price_bdt: { gte:50000, lte:250000 } } },
      ],
    }
  },
  sort: ['_score', { 'seller.rating': 'desc' }],
  size: 20,
  _source: ['title','price_bdt','seller.name'],
});

console.log(res.hits.hits.map(h => h._source));

// search_after pagination
let after: any[] | undefined;
while (true) {
  const page = await es.search({
    index: 'products',
    size: 100,
    sort: [ { created_at:'desc' }, { _id:'asc' } ],
    search_after: after,
    track_total_hits: false,
  });
  if (!page.hits.hits.length) break;
  after = page.hits.hits.at(-1)!.sort;
  // process page
}
```

---

## 🔄 Elasticsearch vs OpenSearch

```
2010  Elasticsearch born (Apache 2.0)
2015  AWS Elasticsearch Service launches
2019  Amazon Elasticsearch (Open Distro) — adds free security, alerting
2021  Elastic relicenses 7.11 → SSPL + Elastic License (NOT open source-OSI)
       AWS forks → OpenSearch (Apache 2.0)
2024  Elasticsearch 8.x adds AGPL option; OpenSearch 2.x mature
```

### সাদৃশ্য

কোর Lucene, query DSL ৯৫% compatible; নবীনতম ES specific feature (ELSER, semantic_text, certain ML jobs) OpenSearch-এ নেই।

### পার্থক্য

| বিষয় | Elasticsearch | OpenSearch |
|------|---------------|------------|
| License | SSPL/Elastic 2.0 (non-OSI) / AGPL option | Apache 2.0 (OSI) |
| Vendor | Elastic NV | AWS-led foundation |
| Security free? | হ্যাঁ (basic) | হ্যাঁ (full RBAC default) |
| ELSER, semantic_text | ✅ | ❌ |
| Anomaly detection | Platinum | Free |
| Vector / kNN | Native HNSW | Native HNSW (k-NN plugin) |
| Cloud | Elastic Cloud, AWS, GCP, Azure | AWS managed (cheaper), self-host |
| Client compatibility | Official ES client | OpenSearch client; ES 7.10 client OK |
| Roadmap | Elastic-driven, ML-heavy | AWS-driven, OSS-purist |
| BD adoption | বেশি (legacy + modern) | বাড়ছে (cost-conscious startup) |

**পছন্দের গাইড:** Apache 2.0 license MUST, AWS-only — OpenSearch। Latest ML/semantic feature MUST — Elastic। অনিশ্চিত — query DSL দুটোতেই কাজ করে।

---

## 🇧🇩 BD Case Study — Daraz Product Search

### Scale

- 50M product, 200K seller
- 1500 RPS peak (campaign), 300 RPS baseline
- p99 target: 200ms
- ৩ language: English, বাংলা, mixed

### Cluster

```
Region   : AWS Mumbai (ap-south-1)
Type     : Elasticsearch 8.13 (Elastic Cloud Hot-Warm)

Master   : 3 × c6g.large (dedicated)
Hot data : 5 × r6gd.2xlarge (64GB RAM, 475GB NVMe)
Warm data: 3 × r6g.xlarge (32GB RAM, EBS gp3 1TB)
Coord    : 2 × c6g.xlarge (search-only)
ML       : 1 × m6g.large (anomaly detection on click stream)

Index    : products-v3
Shards   : 10 primary × 1 replica = 20
Per-shard: ~28GB
Refresh  : 5s (campaign-time 30s)
```

### Search architecture

```
User → CDN → API GW → Node.js coord svc → ES coord nodes → data nodes
                            │
                            ├─ Redis cache (top queries 30s)
                            └─ Embedding service (Sagemaker) for kNN
```

### Query strategy

1. **Fast path:** BM25 multi_match (title^3, title_bn^3, brand, tags)
2. **Personalization:** function_score with user's category affinity boost
3. **Hybrid (A/B test):** RRF[BM25 + kNN(title_emb)] — +6.2% CTR
4. **Filter chain:** category, price_range, seller_rating, in_stock
5. **Aggregation panel:** brand, price_range_buckets, rating

### Indexing pipeline

```
MySQL CDC (Debezium) → Kafka → flink job (enrich seller info, embed)
                                        ↓
                               ES Bulk API (5MB chunks)
```

p99 freshness: ~7 seconds new product → searchable।

### Lessons

- প্রথমে dynamic mapping ছিল → 4500 field, heap pressure → strict mapping reindex
- Deep `from:10000` থেকে search_after-এ migrate → p99 ৩.৫s → ১৮০ms
- Bangla synonym list manually curated, weekly review
- Force merge campaign-এর আগে off-hour-এ → stable latency

---

## 🚨 Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Dynamic mapping ON | Field count blow up, heap CG | `dynamic:strict`, schema review |
| Float for money | Precision drift | `scaled_float` or `long paisa` |
| `text` field-এ aggregation | Fielddata, OOM | `keyword` sub-field |
| Deep pagination | Slow / max_result_window | search_after + PIT |
| 1 huge shard 200GB | Slow recovery | Rollover, ILM |
| 1000 tiny shards | Heap overhead | `_shrink`, force merge |
| `wildcard:*x*` | Full scan | `ngram`, reverse field |
| Replica = 0 in prod | Data loss on node fail | replica ≥ 1 |
| Master + data on same node small cluster | Cluster brown-out under load | dedicated masters |
| Heap > 32GB | Compressed oops broken | ≤ 31GB |
| No snapshot | Disaster | S3 ILM snapshot policy |
| `match_all` unbounded | OOM client | always `size` cap |
| Reindex without alias | Downtime during cutover | aliased index pattern |

---

## ✅ Checklist

### Design
- [ ] Mapping `dynamic:strict` (অথবা explicit), ICU plugin installed
- [ ] Money = scaled_float / long
- [ ] `text` + `keyword` সাব-field সব searchable string-এ
- [ ] Bangla analyzer (ICU + stop + stem) configured
- [ ] Synonym list version-controlled
- [ ] Index alias pattern (`products` → `products-v3`)

### Cluster
- [ ] ৩ dedicated master nodes
- [ ] Heap ≤ 31GB, ≤ 50% RAM, G1GC
- [ ] Hot/warm/cold tier roles assigned
- [ ] TLS transport + HTTP enabled
- [ ] RBAC + API keys, no `elastic` super-user in app
- [ ] Audit log on
- [ ] Snapshot repository (S3) + ILM rollover policy

### Indexing
- [ ] Bulk API (5–15MB), retries with backoff
- [ ] Refresh interval tuned (5–30s)
- [ ] Replica 0 during initial bulk; 1+ before serving
- [ ] Translog durability matched to RPO

### Search
- [ ] `filter` clause used for non-scoring constraints
- [ ] `_source` filtered to required fields
- [ ] `search_after` + PIT for deep pagination
- [ ] Query profiling sampled
- [ ] Track p50/p95/p99, slowlog enabled

### Vector / Hybrid
- [ ] dense_vector dim, similarity matched embedding model
- [ ] HNSW m / ef_construction tuned
- [ ] num_candidates ≥ 10×k
- [ ] RRF (or LTR) for hybrid lexical+vector

### Operations
- [ ] APM / metrics (Elastic Stack monitoring or Prometheus exporter)
- [ ] Slowlog for index + search
- [ ] Disk watermarks alert (low/high/flood_stage)
- [ ] Snapshot test-restore quarterly
- [ ] Upgrade plan (rolling restart procedure)

---

> **চূড়ান্ত কথা:** Elasticsearch একটি database নয়, একটি **search/analytics engine**। Source-of-truth হিসেবে RDBMS-ই রাখুন, ES-এ index/replicate করুন CDC দিয়ে। Mapping carefully, shard wisely, aggregate fearlessly, vector search সাহস করে adopt করুন — কিন্তু সবসময় profile, monitor, snapshot।
