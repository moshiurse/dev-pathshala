# 🚀 Redis Stack — RediSearch, RedisJSON ও Module Ecosystem Deep Dive

> Redis Stack হলো Redis-এর উপর বানানো একটি **module-powered data platform** যেখানে একই Redis server-এর ভেতর full-text search, JSON document storage, graph query, time-series aggregation, probabilistic filtering, autocomplete, geo filtering, vector search — সবকিছু একসাথে চালানো যায়।
>
> Daraz product search, Prothom Alo archive search, Pathao driver discovery, Chaldal catalog filtering, GP metrics, bKash fraud heuristics — বাংলাদেশি scale-এর অনেক সমস্যা Redis Stack খুব ছোট latency-তে সমাধান করতে পারে।

---

## 📌 সংজ্ঞা

### 1. Redis Stack — সংজ্ঞা

Redis Stack হলো **Redis OSS core + একগুচ্ছ modules**-এর একটি bundle।

সহজ ভাষায়:

- Redis core দেয় key-value engine
- RediSearch দেয় search index + query engine
- RedisJSON দেয় JSON document storage + path-based update
- RedisGraph দেয় graph query capability
- RedisTimeSeries দেয় time-series ingestion + aggregation
- RedisBloom দেয় probabilistic data structure family

এক লাইনের সংজ্ঞা:

> **Redis Stack = "একই Redis server-এ cache + document store + search engine + analytics primitives"**

#### Redis Stack-এর module list

| Module | কী কাজ করে | বাংলাদেশ context example |
|---|---|---|
| Redis Core | string, hash, list, set, zset, stream, pub/sub | Foodpanda session cache |
| RediSearch | full-text search, aggregation, faceting, autocomplete, vector search | Daraz product search |
| RedisJSON | nested JSON document store | user profile, cart, order snapshot |
| RedisGraph | graph traversal, recommendation, relationship query | social graph, mutual friend |
| RedisTimeSeries | metrics, counters, retention, downsampling | GP tower metrics, app latency |
| RedisBloom | Bloom/Cuckoo/TopK/CMSketch | fraud signals, duplicate detection |

#### Redis Stack বনাম Redis OSS বনাম Redis Enterprise

| দিক | Redis OSS | Redis Stack | Redis Enterprise |
|---|---|---|---|
| Core key-value | ✅ | ✅ | ✅ |
| RediSearch | ❌ default-এ নেই | ✅ bundled | ✅ managed/optimized |
| RedisJSON | ❌ default-এ নেই | ✅ bundled | ✅ |
| Graph/TimeSeries/Bloom | ❌ | ✅ | ✅ |
| Single binary install experience | মাঝারি | ভালো | fully managed |
| Multi-model workload | সীমিত | শক্তিশালী | সবচেয়ে production-friendly |
| Managed HA/observability | self-manage | self-manage | enterprise features |
| Cost | কম | কম/মাঝারি | বেশি |
| Best fit | cache, queue, simple state | app + search + JSON | large mission-critical platform |

#### Redis Stack কেন আলাদা

অনেক টিম এই pattern দেখে:

```text
Application
 ├─ MySQL/PostgreSQL  -> source of truth
 ├─ Redis             -> cache/session
 ├─ Elasticsearch     -> search
 ├─ MongoDB           -> JSON document
 └─ Kafka/TSDB        -> metrics/stream
```

Redis Stack এই বিচ্ছিন্ন architecture-এর কিছু অংশ compact করে:

```text
Application
 ├─ PostgreSQL/MySQL -> transactional truth
 └─ Redis Stack      -> cache + search + JSON + metrics helper + filters
```

সব use case-এ না, কিন্তু অনেক **operationally simple** workload-এ Redis Stack strong choice।

#### Redis Stack-এর architecture overview

```text
                  ┌──────────────────────────────────────┐
                  │              Application             │
                  │  PHP / Node.js / Go / Python / Java  │
                  └──────────────────┬───────────────────┘
                                     │ RESP / TLS
                                     ▼
        ┌────────────────────────────────────────────────────────────┐
        │                       Redis Server                         │
        │                                                            │
        │  ┌──────────────────────────────────────────────────────┐  │
        │  │                  Redis Core Engine                   │  │
        │  │  event loop | allocator | persistence | replication │  │
        │  └──────────────┬─────────────┬─────────────┬──────────┘  │
        │                 │             │             │             │
        │        ┌────────▼───┐ ┌──────▼─────┐ ┌─────▼─────┐       │
        │        │ RediSearch │ │ RedisJSON  │ │ TimeSeries │       │
        │        │ inverted   │ │ JSON tree  │ │ samples    │       │
        │        │ index      │ │ + paths    │ │ + rules    │       │
        │        └────────────┘ └────────────┘ └───────────┘       │
        │                 │             │             │             │
        │        ┌────────▼───┐ ┌──────▼─────┐ ┌─────▼─────┐       │
        │        │ RedisGraph │ │ RedisBloom │ │  Vectors   │       │
        │        │ nodes/edge │ │ BF/CF/CMS  │ │ in index   │       │
        │        └────────────┘ └────────────┘ └───────────┘       │
        └────────────────────────────────────────────────────────────┘
```

#### Module internals: high-level view

- Redis core keyspace handle করে key lifecycle
- প্রতিটি module Redis Module API ব্যবহার করে commands register করে
- Module নিজস্ব data type allocate করতে পারে
- Persistence-এর সময় module তার state encode/decode করে
- Replication/AOF rewrite-এ module commands বা module-specific serialization ব্যবহার হয়
- RediSearch secondary index maintain করে
- RedisJSON tree/path structure maintain করে
- Search query runtime-এ RediSearch JSON values extract করতে পারে

#### Redis Stack কোথায় ভালো ফিট করে

- ultra-low latency search
- application-side secondary index
- hybrid cache + search
- moderate dataset যেখানে RAM budget আছে
- operational simplicity গুরুত্বপূর্ণ
- polyglot lookup/query একই engine-এ চাই

#### Redis Stack কোথায় fit নাও করতে পারে

- 10TB archive search যেখানে disk-first storage দরকার
- heavy analytical OLAP query
- long-term cheap cold storage
- multi-node complex enterprise governance যেখানে Redis Enterprise বা Elasticsearch ecosystem better
- massive suffix/wildcard-only search workload

### 2. RediSearch — Full-Text Search Deep Dive

#### কেন RediSearch?

RediSearch হলো Redis-এর জন্য **secondary indexing + full-text search + aggregation engine**।

কেন এটা দরকার?

- Redis core নিজে `GET`, `HGET`, `ZRANGE` দ্রুত করে
- কিন্তু `"সব Apple mobile যাদের price 50k-100k আর rating 4+"` ধরনের query core Redis পারে না
- RediSearch text tokenization, inverted index, numeric range, geo radius, facets, highlighting, autocomplete, vector KNN দেয়

#### Elasticsearch-এর alternative হিসেবে কেন ভাবা হয়

| প্রশ্ন | RediSearch | Elasticsearch |
|---|---|---|
| Deployment size | ছোট/সহজ | বড়/ভারি |
| Latency | খুব কম, same-memory | ভালো, কিন্তু JVM/segment overhead আছে |
| Same server as cache | ✅ | ❌ |
| Search + app state together | ✅ | ❌ |
| Huge archive search | সীমিত | শক্তিশালী |
| Ops complexity | কম | বেশি |
| Near real-time | sub-ms to low-ms | low-ms to 100ms |

বাংলাদেশ context:

- **Daraz**-এর hot catalog subset Redis Stack-এ থাকলে search suggestion খুব দ্রুত হয়
- **Prothom Alo** breaking news headlines Redis Stack-এ থাকলে homepage search instant হয়
- **Pathao**-এর driver/job lookup location + status সহ Redis-এ থাকলে rider matching fast হয়

#### RediSearch indexing model

RediSearch সাধারণত দুই ধরনের source থেকে index বানায়:

1. HASH-based index
2. JSON-based index

##### HASH-based index

যখন data Redis Hash-এ stored:

```text
HSET product:1001 title "iPhone 15 Pro" brand "Apple" category "mobile" price 189999 stock 12
HSET product:1002 title "Galaxy S24 Ultra" brand "Samsung" category "mobile" price 164999 stock 8
```

তখন:

```bash
FT.CREATE idx:products ON HASH PREFIX 1 product: SCHEMA \
  title TEXT WEIGHT 5.0 \
  brand TAG SORTABLE \
  category TAG SORTABLE \
  price NUMERIC SORTABLE \
  stock NUMERIC SORTABLE
```

##### JSON-based index

যখন data JSON document আকারে stored:

```json
{
  "sku": "SKU-1001",
  "title": "iPhone 15 Pro",
  "brand": "Apple",
  "category": "mobile",
  "price": 189999,
  "stock": 12,
  "warehouse": {
    "city": "Dhaka",
    "zone": "Tejgaon"
  }
}
```

তখন:

```bash
FT.CREATE idx:products ON JSON PREFIX 1 product: SCHEMA \
  $.title AS title TEXT WEIGHT 5.0 \
  $.brand AS brand TAG SORTABLE \
  $.category AS category TAG SORTABLE \
  $.price AS price NUMERIC SORTABLE \
  $.stock AS stock NUMERIC SORTABLE \
  $.warehouse.city AS city TAG
```

#### HASH বনাম JSON index

| দিক | HASH | JSON |
|---|---|---|
| Simple flat fields | খুব সহজ | সহজ |
| Nested data | সীমিত | খুব শক্তিশালী |
| Partial update | field-level | path-level |
| Array/object query | awkward | natural |
| App payload shape | flat | document-centric |
| Best fit | small flat entities | user profile, product doc, order snapshot |

#### RediSearch schema field types

| Field type | কী কাজ | উদাহরণ |
|---|---|---|
| TEXT | full-text indexing | title, description, article body |
| TAG | exact filter, faceting | brand, category, district, status |
| NUMERIC | range filter | price, rating, stock |
| GEO | radius search | lat/lon of Pathao driver |
| VECTOR | embedding similarity | semantic product/article search |
|

##### TEXT field

TEXT field tokenize হয়, stemming/stopword apply হতে পারে, relevance score পায়।

```bash
FT.CREATE idx:news ON JSON PREFIX 1 article: SCHEMA \
  $.title AS title TEXT WEIGHT 5.0 \
  $.body AS body TEXT WEIGHT 1.0
```

##### TAG field

TAG exact-match filtering-এর জন্য।

- brand
- seller
- category
- district
- payment_method

```bash
FT.SEARCH idx:products "@brand:{Apple} @category:{mobile}"
```

##### NUMERIC field

```bash
FT.SEARCH idx:products "@price:[50000 200000] @stock:[1 +inf]"
```

##### GEO field

```bash
FT.CREATE idx:drivers ON JSON PREFIX 1 driver: SCHEMA \
  $.name AS name TEXT \
  $.location AS location GEO \
  $.status AS status TAG
```

Search within radius:

```bash
FT.SEARCH idx:drivers "@location:[90.4125 23.8103 5 km] @status:{available}"
```

##### VECTOR field

Semantic search use case:

```bash
FT.CREATE idx:articles ON JSON PREFIX 1 article: SCHEMA \
  $.title AS title TEXT \
  $.body AS body TEXT \
  $.embedding AS embedding VECTOR HNSW 6 \
    TYPE FLOAT32 DIM 384 DISTANCE_METRIC COSINE M 16 EF_CONSTRUCTION 200
```

এখানে:

- `VECTOR` = embedding index
- `HNSW` = approximate nearest neighbor algorithm
- `DIM 384` = vector dimension
- `COSINE` = similarity metric

#### FT.CREATE deep dive

`FT.CREATE` হলো index create command।

এর গুরুত্বপূর্ণ options:

| Option | অর্থ | কেন ব্যবহার করবেন |
|---|---|---|
| ON HASH / ON JSON | source model | data shape অনুযায়ী |
| PREFIX | কোন keys index হবে | key filtering |
| FILTER | extra expression | selective indexing |
| LANGUAGE | stemming language | English default |
| LANGUAGE_FIELD | document-level language | multilingual docs |
| SCORE | default score | manual boost |
| SCORE_FIELD | doc score field | ranking boost |
| PAYLOAD_FIELD | custom payload | advanced ranking |
| MAXTEXTFIELDS | many text fields | internal limit handling |
| TEMPORARY | temporary index | short-lived workload |
| NOOFFSETS | less memory | no highlighting |
| NOFIELDS | less memory | field bits disable |
| NOFREQS | less memory | term frequency disable |
| STOPWORDS | custom stop words | domain control |
| SKIPINITIALSCAN | lazy build | faster create on huge data |

##### WEIGHT

`WEIGHT` TEXT field-এর গুরুত্ব বাড়ায়।

```bash
FT.CREATE idx:products ON JSON PREFIX 1 product: SCHEMA \
  $.title AS title TEXT WEIGHT 5.0 \
  $.brand AS brand TEXT WEIGHT 2.0 \
  $.description AS description TEXT WEIGHT 1.0
```

এতে title match description-এর চেয়ে বেশি score পাবে।

##### SORTABLE

```bash
$.price AS price NUMERIC SORTABLE
$.created_at AS created_at NUMERIC SORTABLE
```

এই field-এ `SORTBY` দ্রুত করা যায়।

##### NOINDEX

Field return করতে চান, but searchable না:

```bash
$.image_url AS image_url TEXT NOINDEX
```

Use case:

- listing response-এ দরকার
- search tokenization চাই না
- RAM save করতে চান

##### PHONETIC

Bangla transliteration search বা phonetic similarity-এর ক্ষেত্রে useful হতে পারে।

```bash
$.name AS name TEXT PHONETIC dm:en
```

Practical note:

- built-in phonetic algorithm English-centric
- Bangla product search-এর জন্য transliteration field আলাদা রাখা better
- যেমন `"শাড়ি"`, `"saree"`, `"sari"`-কে normalized field-এ map করা যেতে পারে

##### WITHSUFFIXTRIE

Suffix/pattern use case-এ text field-এ suffix trie build করা যায়।

```bash
$.sku AS sku TEXT WITHSUFFIXTRIE
```

এটা wildcard/prefix-like cases-এ সাহায্য করে, কিন্তু memory overhead বাড়ায়।

#### Query model: FT.SEARCH

`FT.SEARCH index query [options...]`

সাধারণ full-text:

```bash
FT.SEARCH idx:products "iphone 15"
```

Field specific:

```bash
FT.SEARCH idx:products "@title:(iphone 15)"
```

##### Boolean queries

```bash
FT.SEARCH idx:products "(@brand:{Apple} @category:{mobile})"
```

এর মানে:

- brand অবশ্যই Apple
- category অবশ্যই mobile

OR query:

```bash
FT.SEARCH idx:products "(@brand:{Apple|Samsung}) @category:{mobile}"
```

NOT query:

```bash
FT.SEARCH idx:products "iphone -@brand:{RefurbishedStore}"
```

বাংলাদেশ context:

```bash
FT.SEARCH idx:news "(@section:{sports|cricket}) -@district:{international}"
```

##### TAG filter syntax

```bash
@field:{value}
@field:{value1|value2}
```

Examples:

```bash
FT.SEARCH idx:products "@brand:{Apple}"
FT.SEARCH idx:products "@district:{dhaka|chattogram}"
FT.SEARCH idx:orders "@payment_status:{paid} @channel:{bkash}"
```

##### Prefix search

```bash
FT.SEARCH idx:products "iph*"
```

Use cases:

- search-as-you-type
- SKU prefix
- brand prefix

##### Suffix search

True suffix search Redis-এ natural না; সাধারণত pattern-heavy search memory expensive।

Options:

- `WITHSUFFIXTRIE` enable করা
- n-gram field বানানো
- reverse string field index করা
- বা dedicated search engine use করা

Example idea:

```text
Original title: iphone-case
Reverse field : esac-enohpi
Suffix: case   -> Reverse prefix: esac*
```

##### Fuzzy search

```bash
FT.SEARCH idx:products "%iphnoe%"
```

বা multiple edits:

```bash
FT.SEARCH idx:products "%%samzung%%"
```

Use case:

- user typo করেছে
- mobile keyboard mistake
- Banglish product নাম ভুল লিখেছে

##### Wildcard queries

```bash
FT.SEARCH idx:products "w'*pro*'"
```

Important note:

- wildcard query powerful but expensive
- large corpus-এ uncontrolled wildcard avoid করুন
- autocomplete-এর জন্য `FT.SUGGET` অনেক ভালো option

##### Numeric range filter

```bash
FT.SEARCH idx:products "@price:[1000 5000] @stock:[1 +inf]"
```

Open range:

```bash
FT.SEARCH idx:products "@price:[50000 +inf]"
FT.SEARCH idx:products "@rating:[(4 5]"
```

##### Geo filtering

Pathao example:

```bash
FT.SEARCH idx:drivers "@location:[90.4125 23.8103 3 km] @status:{available}"
```

Foodpanda example:

```bash
FT.SEARCH idx:restaurants "@location:[90.4074 23.7278 2 km] @is_open:{true}"
```

##### Pagination

```bash
FT.SEARCH idx:products "@category:{mobile}" LIMIT 0 20
FT.SEARCH idx:products "@category:{mobile}" LIMIT 20 20
```

Tip:

- deep pagination expensive হতে পারে
- cursor-like experience দরকার হলে sortable id/date সঙ্গে range pagination ভাবুন

##### Sorting

```bash
FT.SEARCH idx:products "@category:{mobile}" SORTBY price ASC LIMIT 0 10
FT.SEARCH idx:articles "bangladesh budget" SORTBY published_at DESC LIMIT 0 20
```

##### RETURN fields

```bash
FT.SEARCH idx:products "iphone" RETURN 3 title price brand
```

##### Highlighting and summarization

```bash
FT.SEARCH idx:news "election" \
  HIGHLIGHT FIELDS 2 title body TAGS "<mark>" "</mark>" \
  SUMMARIZE FIELDS 1 body LEN 20
```

Prothom Alo search result page-এ এটা headline snippet generate করতে পারে।

##### SCORING এবং relevance

RediSearch relevance সাধারণত TF-IDF/BM25-style scoring model follow করে।

High-level intuition:

- document-এ term বেশি থাকলে score বাড়ে
- rare term হলে score বাড়ে
- shorter/more focused field boost পায়
- field weight score-কে influence করে

Example:

```text
Query: "iphone 15 pro"

Doc A title: "iPhone 15 Pro Max"
Doc B title: "Apple Phone Cover for iPhone 15"

Result:
- Doc A score বেশি
- কারণ exact phrase proximity + title weight + better term density
```

##### EXPLAINSCORE

```bash
FT.SEARCH idx:products "iphone 15" WITHSCORES EXPLAINSCORE
```

Use case:

- ranking বুঝতে
- wrong result debug করতে
- boost tuning validate করতে

##### Slop / phrase behavior

Phrase query:

```bash
FT.SEARCH idx:news '"স্মার্ট বাংলাদেশ"'
```

Phrase proximity concept বুঝুন:

- `smart bangladesh plan` phrase near হলে better
- unrelated দূরে থাকলে lower relevance

##### Stemming and stop words

Stemming example:

- `running` → `run`
- `phones` → `phone`
- `delivering` → `deliver`

Bangla challenge:

- Bangla morphology built-inভাবে perfect না
- Bangla search-এর জন্য অনেক টিম normalized field রাখে
- যেমন `প্রযুক্তি`, `প্রযুক্তির`, `প্রযুক্তিতে` → normalized root form

Stop words example:

- the
- is
- a
- of

বাংলা domain-specific stopwords হতে পারে:

- এই
- ওই
- এবং
- কিন্তু
- জন্য

তবে overly aggressive stopwords relevance নষ্ট করতে পারে।

##### Synonyms

Chaldal example:

- `চিনি`
- `sugar`
- `chini`

এই তিনটাকে synonym group-এ map করা যায়।

```bash
FT.SYNUPDATE idx:products syn:groceries "চিনি" "sugar" "chini"
FT.SYNUPDATE idx:products syn:mobile "mobile" "phone" "smartphone" "মোবাইল"
```

Why powerful:

- bilingual catalog search
- Banglish input tolerance
- local product naming variance

##### Phonetic/normalization for Bangla product search

Practical pattern:

```text
User types         Normalized field
──────────         ────────────────
shari              saree
sari               saree
শাড়ি              saree
sharee             saree
```

তারপর index করুন:

- original title
- normalized search_terms
- synonyms

##### Auto-complete

Autocomplete-এর জন্য RediSearch suggestion dictionary আছে।

```bash
FT.SUGADD ac:products "iphone 15 pro" 100 PAYLOAD "SKU-1001"
FT.SUGADD ac:products "iphone 15 case" 50 PAYLOAD "SKU-2010"
FT.SUGADD ac:products "samsung s24 ultra" 90 PAYLOAD "SKU-3010"
```

Query:

```bash
FT.SUGGET ac:products "iph" MAX 5 WITHSCORES WITHPAYLOADS
FT.SUGGET ac:products "iphoen" FUZZY MAX 5
```

Daraz-like UX:

- user `iph` লিখল
- instant suggestion list এল
- click করলে search page open

##### FT.AGGREGATE

Aggregation faceting/reporting-এর জন্য use হয়।

Category count example:

```bash
FT.AGGREGATE idx:products "@brand:{Apple|Samsung}" \
  GROUPBY 1 @category \
  REDUCE COUNT 0 AS total \
  SORTBY 2 @total DESC
```

Average price by brand:

```bash
FT.AGGREGATE idx:products "@category:{mobile}" \
  GROUPBY 1 @brand \
  REDUCE AVG 1 @price AS avg_price \
  REDUCE MIN 1 @price AS min_price \
  REDUCE MAX 1 @price AS max_price \
  SORTBY 2 @avg_price DESC
```

Prothom Alo example: per section article count

```bash
FT.AGGREGATE idx:articles "budget" \
  GROUPBY 1 @section \
  REDUCE COUNT 0 AS matched_articles
```

##### APPLY and expressions

```bash
FT.AGGREGATE idx:products "*" \
  APPLY "@price * 0.9" AS discounted_price \
  FILTER "@discounted_price < 100000" \
  SORTBY 2 @discounted_price ASC
```

##### Cursor-based aggregation

Large aggregate result set হলে:

```bash
FT.AGGREGATE idx:events "*" WITHCURSOR COUNT 1000
```

Useful for:

- reporting dashboard
- backoffice analytics
- bulk export

##### Dialect support

নতুন query syntax features-এর জন্য dialect version গুরুত্বপূর্ণ।

```bash
FT.SEARCH idx:products "@brand:{Apple}" DIALECT 2
```

Production note:

- app code-এ fixed dialect use করুন
- upgrade-এর পর query behavior drift কমবে

#### RediSearch index management

##### FT.INFO

```bash
FT.INFO idx:products
```

এখানে দেখতে পাবেন:

- number of docs
- index size
- inverted index size
- average bytes per record
- cursor stats
- percent indexed

##### FT.ALTER

নতুন field add:

```bash
FT.ALTER idx:products SCHEMA ADD $.seller_rating AS seller_rating NUMERIC SORTABLE
```

##### FT.DROPINDEX

```bash
FT.DROPINDEX idx:products
```

Documents রেখে drop:

```bash
FT.DROPINDEX idx:products KEEPDOCS
```

##### Reindex pattern

Schema change বড় হলে common pattern:

```text
idx:products:v1   -> live
idx:products:v2   -> build
application alias -> switch
old index         -> drop
```

#### RediSearch performance tuning

##### Memory trade-off

RediSearch RAM-heavy হতে পারে কারণ:

- inverted index postings
- term dictionary
- sortable values
- offsets/highlight metadata
- JSON source document
- vector index graph (HNSW)

##### Memory বাঁচানোর tricks

- unnecessary field index করবেন না
- `NOINDEX` use করুন display-only field-এ
- `NOOFFSETS` যদি highlighting না লাগে
- `NOFREQS` যদি tf scoring দরকার না হয়
- narrow PREFIX use করুন
- TAG field-এ giant free-text দেবেন না
- deep wildcard avoid করুন

##### Throughput vs latency

| Pattern | ভালো practice |
|---|---|
| Bulk load | pipeline + `SKIPINITIALSCAN` + batch insert |
| Heavy updates | small JSON partial update |
| Frequent sort | only needed fields sortable |
| Autocomplete | FT.SUGGET, not wildcard search |
| Geo lookup | separate geo-focused index |

#### RediSearch vs Elasticsearch — performance intuition

| বিষয় | RediSearch | Elasticsearch |
|---|---|---|
| Data path | RAM-first | segment/disk/JVM mixed |
| Hot subset search | অসাধারণ | ভালো |
| Large corpus archive | ব্যয়বহুল RAM | বেশি efficient |
| Write amplification | index updates in memory | segment merge overhead |
| Deployment | single server possible | usually cluster-centric |
| Facets/aggregate | strong | very strong |
| Ecosystem | narrower | richer |
| Same-app low-latency | strong advantage | weaker |

##### Daraz product search-এর জন্য intuition

যদি use case হয়:

- active catalog 2-5 million SKU
- autocomplete 5-20 ms দরকার
- filters: brand, category, district, seller, price
- cart/session/cache একই Redis infra-তে already আছে

তাহলে Redis Stack compelling।

যদি use case হয়:

- 500 million SKU history
- log analytics + index lifecycle + cold tier
- very complex multilingual analyzer chain
- data lake integration

তাহলে Elasticsearch/OpenSearch stronger choice।

#### Prothom Alo article search example

Article doc shape:

```json
{
  "id": "article:2025:budget-01",
  "title": "জাতীয় বাজেটে প্রযুক্তি খাতে নতুন বরাদ্দ",
  "body": "অর্থমন্ত্রী আজ সংসদে...",
  "section": "economy",
  "tags": ["budget", "technology", "bangladesh"],
  "published_at": 1751328000,
  "district": "dhaka"
}
```

Index:

```bash
FT.CREATE idx:articles ON JSON PREFIX 1 article: SCHEMA \
  $.title AS title TEXT WEIGHT 5.0 \
  $.body AS body TEXT WEIGHT 1.0 \
  $.section AS section TAG SORTABLE \
  $.tags[*] AS tags TAG \
  $.published_at AS published_at NUMERIC SORTABLE \
  $.district AS district TAG
```

Query:

```bash
FT.SEARCH idx:articles "budget @section:{economy} @published_at:[1751241600 1753920000]" \
  SORTBY published_at DESC \
  HIGHLIGHT FIELDS 2 title body TAGS "<em>" "</em>" \
  SUMMARIZE FIELDS 1 body LEN 18
```

### 3. RedisJSON — Deep Dive

#### কেন RedisJSON?

RedisJSON হলো Redis-এ JSON document store করার module।

কেন useful:

- nested object store করা যায়
- array/object path-এ partial update করা যায়
- full document rewrite লাগে না
- atomic numeric/array operations করা যায়
- RediSearch দিয়ে সেই JSON index করা যায়

#### Redis core string বনাম hash বনাম JSON

| Model | সুবিধা | সীমাবদ্ধতা |
|---|---|---|
| String | simple, fast | nested update awkward |
| Hash | flat field update easy | deep nested object নেই |
| JSON | document natural, path update | metadata overhead বেশি হতে পারে |

#### JSON.SET

Full document set:

```bash
JSON.SET user:1001 $ '{
  "id": 1001,
  "name": "Rahim Uddin",
  "phone": "017XXXXXXXX",
  "district": "Dhaka",
  "preferences": {
    "language": "bn",
    "payment": "bKash"
  },
  "addresses": [
    {
      "label": "home",
      "area": "Mirpur DOHS"
    }
  ]
}'
```

Path-specific set:

```bash
JSON.SET user:1001 $.preferences.language '"en"'
```

#### JSON.GET

```bash
JSON.GET user:1001
JSON.GET user:1001 $.preferences
JSON.GET user:1001 $.addresses[0].area
```

#### JSON.MGET

```bash
JSON.MGET user:1001 user:1002 user:1003 $.district
```

Use case:

- Chaldal multiple user district resolve
- campaign targeting
- personalization batch read

#### JSONPath query

RedisJSON path syntax powerful:

```bash
JSON.GET store:1 '$.store.book[*].author'
```

আরও examples:

```bash
JSON.GET product:1001 '$.variants[*].color'
JSON.GET order:5001 '$.items[*].sku'
JSON.GET user:1001 '$..area'
```

#### Atomic operations

##### JSON.NUMINCRBY

```bash
JSON.NUMINCRBY wallet:bkash:1001 $.balance 500
```

Use case:

- reward point increment
- page view counter
- wallet snapshot adjustment

##### JSON.ARRAPPEND

```bash
JSON.ARRAPPEND cart:1001 $.items '{"sku":"RICE-25KG","qty":1,"price":2950}'
```

##### JSON.ARRPOP

```bash
JSON.ARRPOP cart:1001 $.items -1
```

##### JSON.DEL

```bash
JSON.DEL user:1001 $.preferences.old_flag
```

##### JSON.TOGGLE

```bash
JSON.TOGGLE user:1001 $.marketing_opt_in
```

#### Nested document update without full rewrite

Traditional string pattern:

```text
GET full JSON -> app modify -> SET full JSON back
```

Problems:

- network overhead বেশি
- race condition ঝুঁকি
- large document update expensive

RedisJSON pattern:

```text
JSON.SET key $.path newValue
```

Example:

```bash
JSON.SET order:9001 $.delivery.status '"assigned"'
JSON.SET order:9001 $.delivery.driver_id '"DRV-8891"'
```

Pathao/Foodpanda delivery workflow-এ খুব useful।

#### Combining RedisJSON + RediSearch

এটাই Redis Stack-এর killer combo।

Flow:

1. document JSON-এ store
2. RediSearch JSON paths index করে
3. user full-text + filters দিয়ে query করে
4. same key থেকে structured data ফেরত পায়

```text
JSON Document
   │
   ├─ $.title         -> TEXT field
   ├─ $.category      -> TAG field
   ├─ $.price         -> NUMERIC field
   ├─ $.location      -> GEO field
   └─ $.embedding     -> VECTOR field
   │
   ▼
RediSearch secondary index
```

#### Product catalog example

```json
{
  "sku": "DARAZ-IPHONE-15-BLUE-256",
  "title": "iPhone 15 Pro Max 256GB Blue",
  "title_bn": "আইফোন ১৫ প্রো ম্যাক্স ২৫৬ জিবি নীল",
  "brand": "Apple",
  "category": "mobile",
  "subcategories": ["smartphone", "ios"],
  "price": 194999,
  "discount_price": 188999,
  "rating": 4.8,
  "stock": 11,
  "districts": ["dhaka", "chattogram"],
  "seller": {
    "id": "SELLER-77",
    "name": "Daraz Mall BD",
    "score": 4.9
  },
  "attributes": {
    "color": "blue",
    "storage": "256GB",
    "network": "5G"
  }
}
```

এই document থেকে search, filter, sort, autocomplete — সব চালানো যায়।

#### Memory efficiency vs HASH

গুরুত্বপূর্ণ সত্য:

- ছোট flat object-এর জন্য Hash বেশি memory efficient হতে পারে
- nested object, arrays, dynamic structure-এর জন্য JSON development-wise cleaner
- memory cost evaluate না করে blindly JSON use করবেন না

Rule of thumb:

| Scenario | Better choice |
|---|---|
| 10-15 flat fields | Hash |
| deeply nested profile | JSON |
| array-heavy order snapshot | JSON |
| extreme memory sensitivity | Hash first evaluate |
| search over nested docs | JSON + RediSearch |

#### User profile use case

```json
{
  "id": "USR-1001",
  "name": "Nusrat Jahan",
  "phone": "88017xxxxxxx",
  "city": "Dhaka",
  "district": "Dhaka",
  "loyalty_points": 1250,
  "preferred_payment": "bKash",
  "saved_addresses": [
    {"label": "home", "area": "Dhanmondi", "lat": 23.746, "lon": 90.374},
    {"label": "office", "area": "Gulshan 1", "lat": 23.780, "lon": 90.416}
  ],
  "tags": ["vip", "express-customer"]
}
```

Search use cases:

- `vip` users
- Dhaka district users
- bKash-preferring customers
- points > 1000

#### Order details use case

```json
{
  "order_id": "ORD-5001",
  "user_id": "USR-1001",
  "channel": "app",
  "payment": {
    "method": "bKash",
    "status": "paid",
    "trx_id": "9BX33221"
  },
  "items": [
    {"sku": "RICE-25KG", "qty": 1, "price": 2950},
    {"sku": "OIL-5L", "qty": 2, "price": 860}
  ],
  "delivery": {
    "status": "packed",
    "warehouse": "Tejgaon",
    "slot": "evening"
  }
}
```

Partial update:

```bash
JSON.SET order:5001 $.delivery.status '"out_for_delivery"'
JSON.SET order:5001 $.delivery.agent '"RIDER-221"'
```

#### JSON + search example

```bash
FT.CREATE idx:users ON JSON PREFIX 1 user: SCHEMA \
  $.name AS name TEXT WEIGHT 3.0 \
  $.district AS district TAG \
  $.preferred_payment AS preferred_payment TAG \
  $.loyalty_points AS loyalty_points NUMERIC SORTABLE \
  $.tags[*] AS tags TAG
```

Search VIP Dhaka customers:

```bash
FT.SEARCH idx:users "@district:{Dhaka} @tags:{vip} @loyalty_points:[1000 +inf]"
```

### 4. RedisGraph (brief)

RedisGraph historically Redis ecosystem-এর graph module।

মূল ধারণা:

- nodes
- edges
- properties
- pattern matching
- Cypher query language

Example domain:

- social network
- mutual connections
- recommendation graph
- fraud ring analysis

Cypher example:

```bash
GRAPH.QUERY social "
  MATCH (u:User {id:'USR-1001'})-[:FRIEND]->(f:User)
  RETURN f.name
"
```

Recommendation example:

```bash
GRAPH.QUERY commerce "
  MATCH (u:User {id:'USR-1001'})-[:BOUGHT]->(p:Product)<-[:BOUGHT]-(other:User)-[:BOUGHT]->(rec:Product)
  RETURN rec.title, count(*) AS strength
  ORDER BY strength DESC
  LIMIT 10
"
```

Reality check:

- graph use case Redis core-এর primary focus না
- very graph-centric system হলে Neo4j/FalkorDB evaluate করুন
- কিন্তু low-latency graph traversal niche cases-এ useful হতে পারে

### 5. RedisTimeSeries (brief)

RedisTimeSeries time-stamped samples efficiently store করে।

Use cases:

- GP tower CPU/traffic metrics
- API latency time series
- IoT sensor data
- e-commerce order/minute dashboard

Core ideas:

- append-only samples
- retention policy
- labels
- aggregation rules
- downsampling

Create series:

```bash
TS.CREATE ts:api:latency RETENTION 604800000 LABELS service checkout env prod
```

Add samples:

```bash
TS.ADD ts:api:latency * 121
TS.ADD ts:api:latency * 98
TS.ADD ts:api:latency * 140
```

Query range:

```bash
TS.RANGE ts:api:latency - + AGGREGATION avg 60000
```

Usefulness:

- raw data + aggregated rollup
- dashboard refresh fast
- Grafana-style integrations possible

### 6. RedisBloom (brief)

RedisBloom probabilistic data structures-এর family।

#### Bloom Filter

প্রশ্নের উত্তর দেয়:

- `এটা সম্ভবত আগে দেখা হয়েছে কি?`

Properties:

- false positive হতে পারে
- false negative হয় না
- memory efficient

Example:

```bash
BF.RESERVE bf:seen_orders 0.01 1000000
BF.ADD bf:seen_orders ORD-1001
BF.EXISTS bf:seen_orders ORD-1001
```

Use case:

- duplicate event detection
- repeated coupon claim screening
- fraud pre-filter

#### Cuckoo Filter

Deletion support better।

```bash
CF.RESERVE cf:fraud 100000
CF.ADD cf:fraud DEVICE-991
CF.EXISTS cf:fraud DEVICE-991
```

#### Top-K

সবচেয়ে frequent items track করে।

```bash
TOPK.RESERVE topk:searches 20 2000 7 0.9
TOPK.ADD topk:searches "iphone" "iphone" "samsung" "airpods"
TOPK.LIST topk:searches
```

Use case:

- trending search
- Prothom Alo most searched topic
- Daraz hot keywords

#### Count-Min Sketch

Approximate frequency count:

```bash
CMS.INITBYPROB cms:keywords 0.001 0.99
CMS.INCRBY cms:keywords iphone 1 samsung 1 iphone 1
CMS.QUERY cms:keywords iphone samsung
```

### 7. Redis Stack Architecture

#### Modules কীভাবে Redis core-এ plug করে

```text
Command arrives: FT.SEARCH idx:products "iphone"
        │
        ▼
Redis command table lookup
        │
        ├─ GET         -> core string handler
        ├─ HSET        -> core hash handler
        ├─ JSON.SET    -> RedisJSON module handler
        ├─ FT.SEARCH   -> RediSearch module handler
        └─ TS.ADD      -> TimeSeries module handler
```

Module API capabilities:

- custom commands register
- custom key data type register
- RDB save/load hooks
- AOF rewrite hooks
- memory accounting hooks
- notification hooks

#### Memory management with modules

সব module RAM consume করে, কিন্তু pattern আলাদা:

| Component | Memory driver |
|---|---|
| Redis JSON docs | raw document + tree overhead |
| RediSearch index | term dictionary + postings + sortable values |
| GEO/TAG | index structure |
| VECTOR HNSW | graph adjacency layers |
| TimeSeries | compressed chunks |
| Bloom/CMS | pre-allocated probabilistic structure |

Production implication:

- শুধু key count না, index footprint estimate করুন
- JSON document 1KB হলেও search index extra RAM লাগবে
- vector dimension × document count × algorithm বড় factor

#### Cluster compatibility

Redis Stack cluster mode-এ চলতে পারে, কিন্তু design understand করা জরুরি।

Important points:

- index and documents sharding-aware হতে হবে
- prefix strategy consistent হওয়া দরকার
- multi-key operations hash slot aware হতে হবে
- module support/version cluster compatibility check করতে হবে
- client library cluster module command support test করতে হবে

#### High-level cluster picture

```text
                 Redis Cluster (16384 hash slots)

   ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
   │ Node A             │  │ Node B             │  │ Node C             │
   │ slots 0-5460       │  │ slots 5461-10922   │  │ slots 10923-16383  │
   │ product:* subset   │  │ product:* subset   │  │ product:* subset   │
   │ idx metadata share │  │ idx metadata share │  │ idx metadata share │
   └────────────────────┘  └────────────────────┘  └────────────────────┘
```

#### Operational architecture pattern

Typical production:

```text
                        ┌────────────────────┐
                        │  App Servers       │
                        │ PHP-FPM / Node.js  │
                        └─────────┬──────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │ Redis Stack Primary/Shard │
                    │ search + json + cache     │
                    └───────┬──────────┬────────┘
                            │          │
                            ▼          ▼
                     replica/read   persistence
                     query assist   AOF/RDB
```

#### Failure considerations

- Redis memory full হলে writes fail করতে পারে
- search index build time CPU spike হতে পারে
- huge wildcard query other traffic impact করতে পারে
- replica lag stale search result দিতে পারে
- persistence fork memory pressure create করতে পারে

### 11. RediSearch vs Elasticsearch Decision Table

| Criteria | RediSearch বেছে নিন যখন | Elasticsearch বেছে নিন যখন |
|---|---|---|
| Latency | 1-10ms class response চাই | 20-100ms acceptable |
| Dataset size | RAM-এ hot dataset ফিট করে | disk/cold tiers দরকার |
| Simplicity | এক infra-তে cache+search চাই | dedicated search platform okay |
| Query pattern | app search + filters + autocomplete | complex enterprise search + observability |
| Ranking tuning | moderate | very advanced search stack |
| Analytics | lightweight aggregation | massive analytical aggregations |
| Team skill | ছোট DevOps team | search platform team আছে |
| Cost profile | RAM budget acceptable | large corpus disk economics better |
| Update frequency | real-time app updates | near-real-time large pipelines |
| Ecosystem integrations | app-centric | SIEM/ELK/log pipelines |

Quick decision shortcut:

```text
Hot operational search?      -> RediSearch
Large archive search?        -> Elasticsearch
Need cache + JSON + search?  -> Redis Stack
Need observability stack?    -> Elasticsearch/OpenSearch
```

### 12. Anti-patterns and Best Practices

#### Anti-patterns

1. **সব data Redis Stack-এ ঢুকিয়ে source of truth বানানো**
2. **massive cold archive RAM-এ রাখার চেষ্টা**
3. **display-only field-ও index করা**
4. **autocomplete-এর জন্য wildcard search চালানো**
5. **huge document full rewrite করা যখন path update possible**
6. **sortable field অতিরিক্ত ব্যবহার করা**
7. **vector search চালানো without memory planning**
8. **Bangla normalization ছাড়া bilingual search expect করা**
9. **deep pagination unlimited রাখা**
10. **query dialect/version pin না করা**

#### Best practices

1. hot data subset Redis Stack-এ রাখুন
2. JSON + RediSearch combo use করুন nested search use case-এ
3. autocomplete আলাদা suggestion dictionary-তে রাখুন
4. search index schema lean রাখুন
5. write path-এ idempotency ব্যবহার করুন
6. periodic `FT.INFO` metrics collect করুন
7. large rebuild-এর জন্য blue/green index versioning করুন
8. Bangla + Banglish normalization field রাখুন
9. filters-কে TAG/NUMERIC field-এ map করুন
10. operational dashboard-এ memory fragmentation monitor করুন

#### Best practice checklist

```text
[ ] Only necessary fields indexed?
[ ] JSON paths named consistently?
[ ] Autocomplete separated?
[ ] Reindex plan documented?
[ ] Memory headroom > 30%?
[ ] Highlighting really needed?
[ ] Synonym groups maintained?
[ ] Query latency benchmarked?
[ ] Deep pagination capped?
[ ] Fallback DB/source-of-truth clear?
```

---

## 🌍 বাস্তব উদাহরণ

### 8. Production Patterns

### 8.1 Daraz product search with filters, facets, autocomplete

Problem:

- কোটি কোটি search request
- brand, category, seller, district, price filter
- typo tolerance দরকার
- autocomplete দরকার
- hot products instantly searchable হতে হবে

Recommended pattern:

```text
MySQL/PostgreSQL -> canonical product data
        │
        ├─ CDC / app write
        ▼
RedisJSON product document
        │
        ├─ RediSearch index
        ├─ FT.SUGADD keyword dictionary
        └─ optional vector embedding index
```

Query flow:

1. user homepage search box-এ `iph` লিখল
2. `FT.SUGGET ac:products iph`
3. user `iphone 15 pro` click করল
4. `FT.SEARCH idx:products "iphone 15 pro @category:{mobile}"`
5. sidebar facets-এর জন্য `FT.AGGREGATE`
6. sort by price/rating/seller score

Facet examples:

- brand counts
- storage counts
- district availability
- seller type (mall/non-mall)

### 8.2 Prothom Alo news search with date filtering

Use case:

- breaking news homepage search
- archive search by section/date/tag
- highlighted snippets
- trending keyword discovery

Pattern:

- JSON doc: title, body, section, district, published_at, tags
- RediSearch index over title/body/tags/date
- `FT.AGGREGATE` for trending topic by section
- `TOPK` for hottest search keywords

Example queries:

```bash
FT.SEARCH idx:articles "budget @section:{economy} @published_at:[1748736000 1751328000]"
FT.SEARCH idx:articles "cricket @tags:{bangladesh|worldcup}" SORTBY published_at DESC
```

### 8.3 Pathao driver search within radius

Use case:

- rider pickup point-এর কাছাকাছি available driver খুঁজতে হবে
- filter by vehicle type, rating, zone
- rapid update frequency

Pattern:

- JSON driver state document
- fields: `status`, `vehicle_type`, `rating`, `location`
- RediSearch GEO search
- availability update via `JSON.SET`

Query:

```bash
FT.SEARCH idx:drivers "@location:[90.4125 23.8103 2 km] @status:{available} @vehicle_type:{bike}" \
  SORTBY rating DESC \
  LIMIT 0 20
```

### 8.4 Session + user profile

Use case:

- logged-in session দ্রুত lookup
- preference document store
- VIP / district / payment preference search

Pattern:

- `session:<id>` -> string/hash TTL
- `user:<id>` -> JSON profile
- `idx:users` -> search index

Practical benefit:

- cache + user document same place
- personalization দ্রুত
- marketing segmentation সহজ

### 8.5 Chaldal catalog and grocery synonym search

Use case:

- `চিনি`, `sugar`, `chini` এক জিনিস
- `আটা` vs `flour`
- pack size filter
- brand filter

Pattern:

- JSON product doc
- synonym groups
- normalized search terms
- autocomplete dictionary

### 8.6 Foodpanda restaurant discovery

Use case:

- area-based restaurant search
- cuisine filter
- open-now filter
- delivery fee range

Redis Stack fit:

- geo + numeric + tag + text একই query-তে
- trending cuisines Top-K দিয়ে
- user taste profile JSON-এ

### 8.7 bKash / fraud screening helper

Note:

Redis Stack primary ledger হওয়া উচিত না।

কিন্তু helper use cases:

- seen device Bloom filter
- frequent merchant Top-K
- suspicious keyword search on notes/support tickets
- user/device profile JSON doc

### 8.8 GP metrics / observability helper

Use case:

- latency metrics
- API error count
- near-real-time dashboard

Pattern:

- RedisTimeSeries for metrics
- RediSearch for incident ticket search
- RedisJSON for service metadata

---

## 🧭 ASCII Diagrams

### Diagram 1: Redis Stack module layering

```text
┌────────────────────────────────────────────────────────────────────┐
│                         Client Applications                        │
│    Daraz API   |   Prothom Alo CMS   |   Pathao Dispatch API      │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                           Redis Stack                              │
├────────────────────────────────────────────────────────────────────┤
│ Redis Core                                                         │
│ ├─ keyspace                                                         │
│ ├─ replication                                                      │
│ ├─ AOF/RDB                                                          │
│ └─ event loop                                                       │
├────────────────────────────────────────────────────────────────────┤
│ Modules                                                             │
│ ├─ RediSearch   -> text, tag, numeric, geo, vector                 │
│ ├─ RedisJSON    -> nested JSON docs + JSONPath                     │
│ ├─ TimeSeries   -> timestamped metrics                             │
│ ├─ Bloom        -> BF/CF/CMS/TopK                                  │
│ └─ Graph        -> nodes/edges                                     │
└────────────────────────────────────────────────────────────────────┘
```

### Diagram 2: RediSearch inverted index

```text
Documents
─────────
product:1 -> "iPhone 15 Pro Max"
product:2 -> "iPhone 15 Case"
product:3 -> "Samsung Galaxy S24"
product:4 -> "Pro Max Charger"

Inverted Index
──────────────
iphone  -> [1, 2]
15      -> [1, 2]
pro     -> [1, 4]
max     -> [1, 4]
case    -> [2]
samsung -> [3]
galaxy  -> [3]
s24     -> [3]
charger -> [4]

Query: "iphone pro"
  iphone -> [1,2]
  pro    -> [1,4]
  AND    -> [1]
```

### Diagram 3: JSON partial update flow

```text
Without RedisJSON path update
─────────────────────────────
App -> GET full document -> modify -> SET full document
       20KB payload           race risk   20KB payload

With RedisJSON path update
──────────────────────────
App -> JSON.SET order:5001 $.delivery.status "assigned"
       only changed path sent over network
```

### Diagram 4: Daraz search request lifecycle

```text
User types: iph
    │
    ▼
FT.SUGGET ac:products iph
    │
    ▼
Suggestions:
  - iphone 15 pro max
  - iphone 15 case
  - iphone charger
    │
    ▼
User selects "iphone 15 pro max"
    │
    ▼
FT.SEARCH idx:products "iphone 15 pro max @category:{mobile} @price:[50000 +inf]"
    │
    ├─ MATCH text tokens
    ├─ FILTER brand/category/price
    ├─ SORTBY relevance / price
    └─ RETURN title price image_url seller
    │
    ▼
Product list page
```

### Diagram 5: Prothom Alo news search

```text
Reporter publishes article
        │
        ▼
JSON.SET article:2025:001 $ {...}
        │
        ▼
RediSearch indexes:
  - title
  - body
  - tags
  - section
  - published_at
        │
        ▼
Reader searches: "budget technology"
        │
        ▼
FT.SEARCH + HIGHLIGHT + SUMMARIZE
        │
        ▼
"জাতীয় বাজেটে <mark>প্রযুক্তি</mark> খাতে..."
```

### Diagram 6: Pathao geo radius search

```text
                       Rider pickup: Dhanmondi
                              (23.7465, 90.3760)
                                      │
             ┌────────────────────────┼────────────────────────┐
             │                        │                        │
             ▼                        ▼                        ▼
        Driver A                 Driver B                 Driver C
     (23.7470,90.3770)       (23.7600,90.3900)       (23.7000,90.4200)
       status=available         status=busy           status=available
       bike                     bike                  car

FT.SEARCH idx:drivers "@location:[90.3760 23.7465 2 km] @status:{available}"

Result:
  Driver A -> inside radius + available
  Driver B -> inside-ish but busy => excluded
  Driver C -> outside radius => excluded
```

### Diagram 7: Search + JSON combo

```text
JSON Document (product:1001)
┌─────────────────────────────────────────────┐
│ title        = "iPhone 15 Pro Max"         │
│ brand        = "Apple"                     │
│ category     = "mobile"                    │
│ price        = 194999                       │
│ seller.score = 4.9                          │
│ stock        = 11                           │
│ attrs.color  = "blue"                      │
└─────────────────────────────────────────────┘
            │
            ├─ $.title       -> TEXT
            ├─ $.brand       -> TAG
            ├─ $.category    -> TAG
            ├─ $.price       -> NUMERIC
            ├─ $.seller.score-> NUMERIC
            └─ $.stock       -> NUMERIC
            │
            ▼
      RediSearch secondary index
```

### Diagram 8: Vector + keyword hybrid search

```text
User query: "best phone for photography"
        │
        ├─ keyword tokens  -> title/body match
        └─ embedding model -> query vector
                  │
                  ▼
       FT.SEARCH / hybrid KNN query on VECTOR field
                  │
                  ├─ semantic similarity
                  ├─ category filter mobile
                  └─ price filter
                  ▼
      ranked product suggestions
```

### Diagram 9: Time-series downsampling

```text
Raw latency samples (per second)
100 110 130 120 125 150 90 88 115 123 140 160 ...
 │
 ├─ keep raw for 1 day
 └─ rule: avg per minute -> ts:api:latency:1m

Downsampled series
minute_1 -> 117
minute_2 -> 122
minute_3 -> 111
```

### Diagram 10: Bloom filter use in fraud screening

```text
Incoming device_id -> BF.EXISTS bf:known_devices DEVICE-991
      │
      ├─ yes -> maybe seen before
      │        check deeper rules
      │
      └─ no  -> definitely unseen
               add and mark as new device
```

### Diagram 11: Cluster-aware search architecture

```text
                    ┌───────────────────────────┐
                    │ Application Query Layer   │
                    └────────────┬──────────────┘
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                 ▼                 ▼
         ┌───────────┐     ┌───────────┐     ┌───────────┐
         │ Redis A   │     │ Redis B   │     │ Redis C   │
         │ keys+idx  │     │ keys+idx  │     │ keys+idx  │
         └───────────┘     └───────────┘     └───────────┘
```

### Diagram 12: End-to-end operational pattern

```text
                         PostgreSQL / MySQL
                               │
                       source of truth
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
         CDC / App write                 Backoffice jobs
                │                             │
                └──────────────┬──────────────┘
                               ▼
                          Redis Stack
               ┌──────────────┼───────────────┐
               │              │               │
               ▼              ▼               ▼
           RedisJSON      RediSearch      RedisBloom/TS
               │              │               │
               └─────> API / Search / Metrics responses
```

---

## 🐘 PHP Code

### PHP setup notes

Packages:

```bash
composer require predis/predis
```

অথবা phpredis extension ব্যবহার করতে পারেন।

এই section-এ দুই ধরনের example আছে:

- `phpredis` rawCommand-based
- `Predis` executeRaw-based

### 9.1 Product index creation with phpredis

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

// পুরনো index থাকলে drop
try {
    $redis->rawCommand('FT.DROPINDEX', 'idx:products', 'KEEPDOCS');
} catch (Throwable $e) {
    // index না থাকলে ignore
}

$response = $redis->rawCommand(
    'FT.CREATE',
    'idx:products',
    'ON', 'JSON',
    'PREFIX', '1', 'product:',
    'SCHEMA',
    '$.title', 'AS', 'title', 'TEXT', 'WEIGHT', '5.0',
    '$.title_bn', 'AS', 'title_bn', 'TEXT', 'WEIGHT', '4.0',
    '$.search_terms', 'AS', 'search_terms', 'TEXT', 'WEIGHT', '3.0',
    '$.brand', 'AS', 'brand', 'TAG', 'SORTABLE',
    '$.category', 'AS', 'category', 'TAG', 'SORTABLE',
    '$.districts[*]', 'AS', 'districts', 'TAG',
    '$.price', 'AS', 'price', 'NUMERIC', 'SORTABLE',
    '$.rating', 'AS', 'rating', 'NUMERIC', 'SORTABLE',
    '$.stock', 'AS', 'stock', 'NUMERIC', 'SORTABLE',
    '$.location', 'AS', 'location', 'GEO'
);

var_dump($response);
```

### 9.2 Seed JSON documents with phpredis

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$products = [
    'product:sku:iphone-15-pro-max' => [
        'sku' => 'DARAZ-1001',
        'title' => 'iPhone 15 Pro Max 256GB Blue',
        'title_bn' => 'আইফোন ১৫ প্রো ম্যাক্স ২৫৬ জিবি নীল',
        'search_terms' => 'iphone আইফোন phone smartphone ios apple',
        'brand' => 'Apple',
        'category' => 'mobile',
        'districts' => ['dhaka', 'chattogram'],
        'price' => 194999,
        'rating' => 4.8,
        'stock' => 11,
        'location' => '90.4125,23.8103',
    ],
    'product:sku:galaxy-s24-ultra' => [
        'sku' => 'DARAZ-1002',
        'title' => 'Samsung Galaxy S24 Ultra 256GB',
        'title_bn' => 'স্যামসাং গ্যালাক্সি এস২৪ আল্ট্রা ২৫৬ জিবি',
        'search_terms' => 'samsung galaxy phone smartphone android mobile',
        'brand' => 'Samsung',
        'category' => 'mobile',
        'districts' => ['dhaka', 'khulna'],
        'price' => 164999,
        'rating' => 4.7,
        'stock' => 7,
        'location' => '90.4125,23.8103',
    ],
];

foreach ($products as $key => $doc) {
    $redis->rawCommand(
        'JSON.SET',
        $key,
        '$',
        json_encode($doc, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES)
    );
}

echo "Seeded products\n";
```

### 9.3 Product search with filters (phpredis)

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$query = '(@brand:{Apple|Samsung} @category:{mobile} @districts:{dhaka} @price:[100000 200000] @stock:[1 +inf]) (iphone|galaxy)';

$result = $redis->rawCommand(
    'FT.SEARCH',
    'idx:products',
    $query,
    'SORTBY', 'rating', 'DESC',
    'RETURN', '6', 'title', 'title_bn', 'brand', 'price', 'rating', 'stock',
    'LIMIT', '0', '10',
    'DIALECT', '2'
);

print_r($result);
```

### 9.4 JSON document CRUD (phpredis)

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$userKey = 'user:1001';

// Create full document
$redis->rawCommand(
    'JSON.SET',
    $userKey,
    '$',
    json_encode([
        'id' => 1001,
        'name' => 'Nusrat Jahan',
        'district' => 'Dhaka',
        'preferred_payment' => 'bKash',
        'loyalty_points' => 1200,
        'tags' => ['vip', 'express-customer'],
        'preferences' => [
            'language' => 'bn',
            'notifications' => true,
        ],
    ], JSON_UNESCAPED_UNICODE)
);

// Read full document
$full = $redis->rawCommand('JSON.GET', $userKey, '$');
echo $full . PHP_EOL;

// Partial update
$redis->rawCommand('JSON.SET', $userKey, '$.preferences.language', '"en"');

// Atomic increment
$redis->rawCommand('JSON.NUMINCRBY', $userKey, '$.loyalty_points', '50');

// Append tag
$redis->rawCommand('JSON.ARRAPPEND', $userKey, '$.tags', '"dhaka-priority"');

// Read path only
$district = $redis->rawCommand('JSON.GET', $userKey, '$.district');
echo $district . PHP_EOL;
```

### 9.5 Autocomplete implementation (phpredis)

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$redis->rawCommand('FT.SUGADD', 'ac:products', 'iphone 15 pro max', '100', 'PAYLOAD', 'DARAZ-1001');
$redis->rawCommand('FT.SUGADD', 'ac:products', 'iphone 15 case', '60', 'PAYLOAD', 'DARAZ-2001');
$redis->rawCommand('FT.SUGADD', 'ac:products', 'samsung galaxy s24 ultra', '90', 'PAYLOAD', 'DARAZ-1002');
$redis->rawCommand('FT.SUGADD', 'ac:products', 'smart watch', '40', 'PAYLOAD', 'DARAZ-5001');

$suggestions = $redis->rawCommand(
    'FT.SUGGET',
    'ac:products',
    'iph',
    'MAX', '5',
    'WITHSCORES',
    'WITHPAYLOADS'
);

print_r($suggestions);
```

### 9.6 Aggregation pipeline (phpredis)

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$aggregate = $redis->rawCommand(
    'FT.AGGREGATE',
    'idx:products',
    '@category:{mobile}',
    'GROUPBY', '1', '@brand',
    'REDUCE', 'COUNT', '0', 'AS', 'total_products',
    'REDUCE', 'AVG', '1', '@price', 'AS', 'avg_price',
    'REDUCE', 'MAX', '1', '@rating', 'AS', 'max_rating',
    'SORTBY', '2', '@total_products', 'DESC'
);

print_r($aggregate);
```

### 9.7 Predis example

```php
<?php

declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

use Predis\Client;

$client = new Client([
    'scheme' => 'tcp',
    'host' => '127.0.0.1',
    'port' => 6379,
]);

$client->executeRaw([
    'JSON.SET',
    'article:2025:budget-01',
    '$',
    json_encode([
        'title' => 'জাতীয় বাজেটে প্রযুক্তি খাতে নতুন বরাদ্দ',
        'body' => 'অর্থমন্ত্রী সংসদে নতুন প্রস্তাব ঘোষণা করেছেন...',
        'section' => 'economy',
        'district' => 'dhaka',
        'published_at' => 1751328000,
        'tags' => ['budget', 'technology', 'bangladesh'],
    ], JSON_UNESCAPED_UNICODE),
]);

$results = $client->executeRaw([
    'FT.SEARCH',
    'idx:articles',
    'budget @section:{economy}',
    'SORTBY', 'published_at', 'DESC',
    'RETURN', '3', 'title', 'section', 'published_at',
]);

print_r($results);
```

### 9.8 PHP helper class example

```php
<?php

declare(strict_types=1);

final class RedisProductSearchService
{
    public function __construct(private Redis $redis)
    {
    }

    public function search(string $keyword, array $filters = []): array
    {
        $parts = [];

        if ($keyword !== '') {
            $parts[] = $keyword;
        }

        if (!empty($filters['brand'])) {
            $brands = implode('|', array_map([$this, 'escapeTagValue'], $filters['brand']));
            $parts[] = "@brand:{${brands}}";
        }

        if (!empty($filters['category'])) {
            $parts[] = '@category:{' . $this->escapeTagValue($filters['category']) . '}';
        }

        if (!empty($filters['district'])) {
            $parts[] = '@districts:{' . $this->escapeTagValue($filters['district']) . '}';
        }

        $minPrice = $filters['min_price'] ?? '-inf';
        $maxPrice = $filters['max_price'] ?? '+inf';
        $parts[] = "@price:[{$minPrice} {$maxPrice}]";
        $parts[] = '@stock:[1 +inf]';

        $query = trim(implode(' ', $parts));

        return $this->redis->rawCommand(
            'FT.SEARCH',
            'idx:products',
            $query,
            'SORTBY', 'rating', 'DESC',
            'LIMIT', '0', '20',
            'RETURN', '6', 'title', 'brand', 'category', 'price', 'rating', 'stock',
            'DIALECT', '2'
        );
    }

    private function escapeTagValue(string $value): string
    {
        return str_replace(['-', ' '], ['\\-', '\\ '], $value);
    }
}
```

### 9.9 PHP session + user profile pattern

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$sessionId = 'sess:abc123';
$userId = '1001';

$redis->setex($sessionId, 3600, json_encode([
    'user_id' => $userId,
    'device' => 'android',
    'logged_in_at' => time(),
]));

$redis->rawCommand('JSON.SET', 'user:1001', '$', json_encode([
    'id' => 1001,
    'name' => 'Nusrat Jahan',
    'district' => 'Dhaka',
    'preferred_payment' => 'bKash',
    'tags' => ['vip'],
], JSON_UNESCAPED_UNICODE));

$profile = $redis->rawCommand('JSON.GET', 'user:1001', '$');
$session = $redis->get($sessionId);

echo $session . PHP_EOL;
echo $profile . PHP_EOL;
```

### 9.10 PHP geo search for Pathao-style driver lookup

```php
<?php

declare(strict_types=1);

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$redis->rawCommand('JSON.SET', 'driver:7001', '$', json_encode([
    'id' => 'DRV-7001',
    'name' => 'Rakib',
    'status' => 'available',
    'vehicle_type' => 'bike',
    'rating' => 4.9,
    'location' => '90.3760,23.7465',
]));

$search = $redis->rawCommand(
    'FT.SEARCH',
    'idx:drivers',
    '@location:[90.3760 23.7465 2 km] @status:{available} @vehicle_type:{bike}',
    'SORTBY', 'rating', 'DESC',
    'RETURN', '4', 'name', 'status', 'vehicle_type', 'rating'
);

print_r($search);
```

---

## 🟨 Node.js / JavaScript Code

### Setup

```bash
npm install redis @redis/search @redis/json ioredis
```

Practical note:

- modern `redis` package-এর `client.ft` API মূলত `@redis/search` capability expose করে
- modern `redis` package-এর `client.json` API মূলত `@redis/json` capability expose করে
- raw command flexibility দরকার হলে `ioredis` খুব handy

Node Redis client module commands support করে, আর ioredis raw command দিয়ে flexible।

### 10.1 Product index creation with `redis` client

```js
import { createClient, SchemaFieldTypes } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });

client.on('error', (err) => {
  console.error('Redis Client Error', err);
});

await client.connect();

try {
  await client.ft.dropIndex('idx:products');
} catch (error) {
  // ignore when index does not exist
}

await client.ft.create(
  'idx:products',
  {
    title: {
      type: SchemaFieldTypes.TEXT,
      AS: 'title',
      WEIGHT: 5,
    },
    title_bn: {
      type: SchemaFieldTypes.TEXT,
      AS: 'title_bn',
      WEIGHT: 4,
    },
    brand: {
      type: SchemaFieldTypes.TAG,
      AS: 'brand',
      SORTABLE: true,
    },
    category: {
      type: SchemaFieldTypes.TAG,
      AS: 'category',
      SORTABLE: true,
    },
    price: {
      type: SchemaFieldTypes.NUMERIC,
      AS: 'price',
      SORTABLE: true,
    },
    rating: {
      type: SchemaFieldTypes.NUMERIC,
      AS: 'rating',
      SORTABLE: true,
    },
    stock: {
      type: SchemaFieldTypes.NUMERIC,
      AS: 'stock',
      SORTABLE: true,
    },
    location: {
      type: SchemaFieldTypes.GEO,
      AS: 'location',
    },
  },
  {
    ON: 'JSON',
    PREFIX: 'product:',
  }
);

console.log('idx:products created');
await client.quit();
```

### 10.2 Seed JSON documents with `redis` client

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

await client.json.set('product:sku:iphone-15-pro-max', '$', {
  sku: 'DARAZ-1001',
  title: 'iPhone 15 Pro Max 256GB Blue',
  title_bn: 'আইফোন ১৫ প্রো ম্যাক্স ২৫৬ জিবি নীল',
  brand: 'Apple',
  category: 'mobile',
  price: 194999,
  rating: 4.8,
  stock: 11,
  search_terms: 'iphone আইফোন smartphone phone ios apple',
  location: '90.4125,23.8103',
  districts: ['dhaka', 'chattogram'],
});

await client.json.set('product:sku:galaxy-s24-ultra', '$', {
  sku: 'DARAZ-1002',
  title: 'Samsung Galaxy S24 Ultra 256GB',
  title_bn: 'স্যামসাং গ্যালাক্সি এস২৪ আল্ট্রা ২৫৬ জিবি',
  brand: 'Samsung',
  category: 'mobile',
  price: 164999,
  rating: 4.7,
  stock: 8,
  search_terms: 'samsung galaxy android phone mobile smartphone',
  location: '90.4125,23.8103',
  districts: ['dhaka', 'khulna'],
});

await client.quit();
```

### 10.3 Product search with filters using `redis` client

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

const query = '(@brand:{Apple|Samsung} @category:{mobile} @price:[100000 200000] @stock:[1 +inf]) (iphone|galaxy)';

const result = await client.ft.search('idx:products', query, {
  LIMIT: {
    from: 0,
    size: 10,
  },
  SORTBY: {
    BY: 'rating',
    DIRECTION: 'DESC',
  },
  RETURN: ['title', 'title_bn', 'brand', 'price', 'rating', 'stock'],
  DIALECT: 2,
});

console.dir(result, { depth: null });
await client.quit();
```

### 10.4 JSON CRUD using `redis` client

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

await client.json.set('user:1001', '$', {
  id: 1001,
  name: 'Nusrat Jahan',
  district: 'Dhaka',
  preferred_payment: 'bKash',
  loyalty_points: 1200,
  tags: ['vip', 'express-customer'],
  preferences: {
    language: 'bn',
    notifications: true,
  },
});

const fullDoc = await client.json.get('user:1001');
console.log(fullDoc);

await client.json.set('user:1001', '$.preferences.language', 'en');
await client.json.numIncrBy('user:1001', '$.loyalty_points', 100);
await client.json.arrAppend('user:1001', '$.tags', 'dhaka-priority');

const district = await client.json.get('user:1001', {
  path: '$.district',
});
console.log(district);

await client.quit();
```

### 10.5 Autocomplete with `redis` client

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

await client.ft.sugAdd('ac:products', 'iphone 15 pro max', 100, {
  PAYLOAD: 'DARAZ-1001',
});

await client.ft.sugAdd('ac:products', 'iphone 15 case', 60, {
  PAYLOAD: 'DARAZ-2001',
});

await client.ft.sugAdd('ac:products', 'samsung galaxy s24 ultra', 90, {
  PAYLOAD: 'DARAZ-1002',
});

const suggestions = await client.ft.sugGet('ac:products', 'iph', {
  MAX: 5,
  WITHSCORES: true,
  WITHPAYLOADS: true,
});

console.dir(suggestions, { depth: null });
await client.quit();
```

### 10.6 Aggregation with `redis` client

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

const aggregate = await client.ft.aggregate('idx:products', '@category:{mobile}', {
  STEPS: [
    {
      type: 'GROUPBY',
      properties: ['@brand'],
      REDUCE: [
        {
          type: 'COUNT',
          AS: 'total_products',
        },
        {
          type: 'AVG',
          property: '@price',
          AS: 'avg_price',
        },
        {
          type: 'MAX',
          property: '@rating',
          AS: 'max_rating',
        },
      ],
    },
  ],
});

console.dir(aggregate, { depth: null });
await client.quit();
```

### 10.7 ioredis raw command example

```js
import Redis from 'ioredis';

const redis = new Redis('redis://127.0.0.1:6379');

await redis.call(
  'JSON.SET',
  'article:2025:budget-01',
  '$',
  JSON.stringify({
    title: 'জাতীয় বাজেটে প্রযুক্তি খাতে নতুন বরাদ্দ',
    body: 'অর্থমন্ত্রী সংসদে নতুন প্রস্তাব ঘোষণা করেছেন...',
    section: 'economy',
    district: 'dhaka',
    published_at: 1751328000,
    tags: ['budget', 'technology', 'bangladesh'],
  })
);

const results = await redis.call(
  'FT.SEARCH',
  'idx:articles',
  'budget @section:{economy} @published_at:[1748736000 1751328000]',
  'SORTBY', 'published_at', 'DESC',
  'RETURN', '3', 'title', 'section', 'published_at'
);

console.dir(results, { depth: null });
redis.disconnect();
```

### 10.8 Node.js service class example

```js
import { createClient } from 'redis';

export class ProductSearchService {
  constructor(client) {
    this.client = client;
  }

  async search({ keyword = '', brand = [], category = '', district = '', minPrice = '-inf', maxPrice = '+inf' }) {
    const parts = [];

    if (keyword) {
      parts.push(keyword);
    }

    if (brand.length) {
      parts.push(`@brand:{${brand.join('|')}}`);
    }

    if (category) {
      parts.push(`@category:{${category}}`);
    }

    if (district) {
      parts.push(`@districts:{${district}}`);
    }

    parts.push(`@price:[${minPrice} ${maxPrice}]`);
    parts.push('@stock:[1 +inf]');

    const query = parts.join(' ');

    return this.client.ft.search('idx:products', query, {
      SORTBY: {
        BY: 'rating',
        DIRECTION: 'DESC',
      },
      LIMIT: {
        from: 0,
        size: 20,
      },
      RETURN: ['title', 'brand', 'category', 'price', 'rating', 'stock'],
      DIALECT: 2,
    });
  }
}

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

const service = new ProductSearchService(client);
const results = await service.search({
  keyword: 'iphone',
  brand: ['Apple'],
  category: 'mobile',
  district: 'dhaka',
  minPrice: 100000,
  maxPrice: 250000,
});

console.dir(results, { depth: null });
await client.quit();
```

### 10.9 Pathao-style geo search in Node.js

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

await client.json.set('driver:7001', '$', {
  id: 'DRV-7001',
  name: 'Rakib',
  status: 'available',
  vehicle_type: 'bike',
  rating: 4.9,
  location: '90.3760,23.7465',
});

const drivers = await client.ft.search(
  'idx:drivers',
  '@location:[90.3760 23.7465 2 km] @status:{available} @vehicle_type:{bike}',
  {
    SORTBY: {
      BY: 'rating',
      DIRECTION: 'DESC',
    },
    RETURN: ['name', 'status', 'vehicle_type', 'rating'],
  }
);

console.dir(drivers, { depth: null });
await client.quit();
```

### 10.10 News search with highlighting pattern in Node.js

```js
import Redis from 'ioredis';

const redis = new Redis('redis://127.0.0.1:6379');

const result = await redis.call(
  'FT.SEARCH',
  'idx:articles',
  'budget @section:{economy}',
  'HIGHLIGHT', 'FIELDS', '2', 'title', 'body', 'TAGS', '<mark>', '</mark>',
  'SUMMARIZE', 'FIELDS', '1', 'body', 'LEN', '18',
  'SORTBY', 'published_at', 'DESC',
  'LIMIT', '0', '10'
);

console.dir(result, { depth: null });
redis.disconnect();
```

### 10.11 Session + profile pattern in Node.js

```js
import { createClient } from 'redis';

const client = createClient({ url: 'redis://127.0.0.1:6379' });
await client.connect();

await client.set('session:abc123', JSON.stringify({
  userId: 1001,
  device: 'android',
  loginAt: Date.now(),
}), {
  EX: 3600,
});

await client.json.set('user:1001', '$', {
  id: 1001,
  name: 'Nusrat Jahan',
  district: 'Dhaka',
  preferred_payment: 'bKash',
  loyalty_points: 1500,
  tags: ['vip'],
});

const [session, profile] = await Promise.all([
  client.get('session:abc123'),
  client.json.get('user:1001'),
]);

console.log(session);
console.log(profile);
await client.quit();
```

---

## ✅ কখন ব্যবহার করবেন

### Redis Stack use করবেন যখন

- আপনার hot operational dataset RAM-এ রাখা সম্ভব
- cache, JSON document, search একসাথে দরকার
- autocomplete + filters + low latency চাই
- same app server থেকে simple infra দিয়ে powerful query দরকার
- e-commerce/catalog/news/search features দ্রুত ship করতে চান
- nested profile/order snapshot path-based update দরকার
- geo + numeric + tag filter combine করতে চান
- small/medium ops team দিয়ে maintain করতে চান

### Redis Stack use না-ও করতে পারেন যখন

- archive search dataset খুব বড় এবং disk-based search engine better
- complex NLP/analyzer ecosystem primary requirement
- observability/search cluster already Elasticsearch-ভিত্তিক
- RAM cost unacceptable
- source-of-truth transactional durability strongest concern
- full BI/OLAP query দরকার

### Practical decision table

| Use case | Recommendation |
|---|---|
| Daraz hot product search | Redis Stack strong |
| Prothom Alo homepage + recent archive search | Redis Stack strong |
| Prothom Alo 15 বছরের full archive | Elasticsearch stronger |
| Pathao nearby driver lookup | Redis Stack excellent |
| Chaldal user cart/profile | RedisJSON strong |
| GP real-time service latency | RedisTimeSeries strong |
| Fraud seen-device filter | RedisBloom strong |
| Social recommendation graph heavy workload | dedicated graph DB evaluate |

### Production best practices recap

- source-of-truth DB আলাদা রাখুন
- Redis Stack-কে serving/indexing layer ভাবুন
- memory capacity plan করুন
- index schema lean রাখুন
- JSON document size সীমিত রাখুন
- autocomplete dictionary আলাদা রাখুন
- blue/green reindex process রাখুন
- monitoring-এ latency + memory + index size ধরুন

### Red flags

- সব article body, image metadata, comments, audit log একসাথে Redis-এ dump করা
- `*foo*bar*` wildcard query user-facing box-এ allow করা
- 50 fields sortable করা
- synonym governance না রাখা
- language normalization ছাড়া Bangla/Banglish search বানানো

---

## 🔑 Key Takeaways

- Redis Stack হলো Redis core-এর উপর multi-model module ecosystem
- RediSearch full-text, filter, aggregate, geo, autocomplete, vector search দেয়
- RedisJSON nested document store ও path-based atomic update দেয়
- RedisJSON + RediSearch একসাথে ব্যবহার করলে powerful application search layer বানানো যায়
- Redis Stack low-latency operational search-এর জন্য অসাধারণ
- Elasticsearch-এর সাথে competition আছে, কিন্তু use case ভিন্ন
- hot subset search, autocomplete, geo lookup, profile search, catalog filtering-এ Redis Stack shines
- huge archive search, heavy analytics, disk-first large corpus-এ Elasticsearch/OpenSearch better হতে পারে
- Bangladesh context-এ Daraz, Prothom Alo, Pathao, Chaldal, Foodpanda, bKash, GP—সবখানেই module-specific use case কল্পনা করা যায়
- anti-pattern এড়িয়ে memory-aware schema design করলে Redis Stack production-এ অত্যন্ত কার্যকর

### শেষ কথা

Redis Stack-কে শুধু “আরেকটা cache” ভাবলে ভুল হবে।

এটা আসলে:

- cache
- serving database
- search helper
- JSON document layer
- metrics helper
- probabilistic toolkit

— সবকিছুর একটি compact operational platform।

যদি আপনার system-এ **speed**, **simple deployment**, **rich read/query capability**, এবং **hot data serving** সবচেয়ে গুরুত্বপূর্ণ হয়, তবে Redis Stack খুব শক্তিশালী অস্ত্র।
