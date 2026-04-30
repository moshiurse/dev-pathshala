# 🟠 Apache Solr — Enterprise-grade Search Platform

> Solr এবং Elasticsearch — দুজনই Lucene-এর সন্তান। কিন্তু চরিত্র ভিন্ন। Solr শান্ত, প্রাচীন, enterprise; Elasticsearch তরুণ, দ্রুত, hyped। বাংলাদেশের অনেক সরকারি ও enterprise সিস্টেমে এখনো Solr চলছে।

> 🔗 Inverted index, full-text search-এর basics-এর জন্য `full-text-search.md` পড়ুন। Elasticsearch তুলনার জন্য `elasticsearch.md`।

---

## 📖 সূচিপত্র

- [Solr-এর ইতিহাস ও Lucene সম্পর্ক](#-solr-এর-ইতিহাস-ও-lucene-সম্পর্ক)
- [SolrCloud Architecture](#-solrcloud-architecture)
- [Schema vs Schemaless Mode](#-schema-vs-schemaless-mode)
- [Field Types, copyField, dynamicField](#-field-types-copyfield-dynamicfield)
- [Analyzers ও Bangla Handling](#-analyzers-ও-bangla-handling)
- [Query Parsers](#-query-parsers)
- [Faceting, Highlighting, Spell Check, Suggester](#-faceting-highlighting-spell-check-suggester)
- [Updates: Atomic, Optimistic Concurrency, Commit Strategy](#-updates-atomic-optimistic-concurrency-commit-strategy)
- [Replication: Legacy Master-Slave vs SolrCloud](#-replication-legacy-master-slave-vs-solrcloud)
- [Performance: Caches, Warming, Tlog](#-performance-caches-warming-tlog)
- [Security](#-security)
- [Client Libraries (PHP Solarium + Node)](#-client-libraries-php-solarium--node)
- [Solr vs Elasticsearch — Comprehensive Comparison](#-solr-vs-elasticsearch--comprehensive-comparison)
- [কখন Solr বেছে নিবেন](#-কখন-solr-বেছে-নিবেন)
- [Migration Patterns Solr↔ES](#-migration-patterns-solres)
- [BD Real-world Case Study](#-bd-real-world-case-study)
- [Common Pitfalls](#-common-pitfalls)
- [Checklist](#-checklist)

---

## 📜 Solr-এর ইতিহাস ও Lucene সম্পর্ক

```
2004  Yonik Seeley (CNET) Solr বানালেন in-house — Lucene-এর উপর HTTP wrapper
2006  Apache Foundation-এ donate
2010  Apache top-level project
2011  Solr 3 — distributed search experiment
2013  Solr 4 + SolrCloud (ZooKeeper, real distributed)
2017  Solr 7 — autoscaling, time routed alias
2021  Solr 8.x — JSON Request API, streaming expressions
2023  Solr 9 — Java 11+, dropped legacy Master-Slave defaults
2024  Solr 9.4+ — vector search (HNSW) native
```

### Lucene ↔ Solr ↔ Elasticsearch

```
                ┌─────────────────────────────────────────────┐
                │              Apache Lucene                  │
                │  (Inverted Index, BM25, Analyzers, HNSW)   │
                │  Java library, embedded                     │
                └────────────────┬────────────────────────────┘
                                 │
                  ┌──────────────┴───────────────┐
                  ▼                              ▼
        ┌──────────────────┐            ┌──────────────────┐
        │   Apache Solr    │            │  Elasticsearch   │
        │ (HTTP server +   │            │ (HTTP + cluster) │
        │  SolrCloud)      │            │                  │
        │ Apache 2.0       │            │ SSPL/Elastic 2.0 │
        │ ZooKeeper-based  │            │ self-cluster     │
        └──────────────────┘            └──────────────────┘
```

**Lucene** = engine। **Solr** এবং **Elasticsearch** = এই engine-এর উপর distributed HTTP service। দুজনই inverted index, BM25, analyzer infrastructure share করে। পার্থক্য মূলত cluster coordination, API design, ecosystem।

---

## 🏛️ SolrCloud Architecture

### Pre-SolrCloud (Legacy "Standalone")

Master-Slave replication; manual sharding; SPOF master।

### SolrCloud (Production-ready since 4.x)

```
                    ┌────────────────────────────┐
                    │      ZooKeeper Ensemble     │
                    │  (3 or 5 nodes, quorum)     │
                    │ - Cluster state              │
                    │ - Leader election            │
                    │ - Configuration store        │
                    └────────────┬────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌────────────┐     ┌────────────┐     ┌────────────┐
       │ Solr Node 1│     │ Solr Node 2│     │ Solr Node 3│
       │            │     │            │     │            │
       │ ┌────────┐ │     │ ┌────────┐ │     │ ┌────────┐ │
       │ │collection_products              shard1 - shard2  shard3│
       │ │  shard1│ │     │ │  shard2│ │     │ │  shard3│ │
       │ │(leader)│ │     │ │(leader)│ │     │ │(leader)│ │
       │ │  shard2│ │     │ │  shard3│ │     │ │  shard1│ │
       │ │(replica│ │     │ │(replica│ │     │ │(replica│ │
       │ └────────┘ │     │ └────────┘ │     │ └────────┘ │
       └────────────┘     └────────────┘     └────────────┘
```

### Key Concepts

| Term | অর্থ |
|------|------|
| **Collection** | logical index (ES-এর index-এর সমান) |
| **Shard** | collection-এর partition (consistent hash by `id`) |
| **Replica** | shard-এর copy (NRT / TLOG / PULL) |
| **Leader** | shard-এর primary replica — write এখানে আগে |
| **Core** | একটি Solr node-এ একটি replica = একটি core |
| **ZooKeeper** | cluster metadata, leader election, config |
| **Overseer** | ZooKeeper-elected node, collection-level admin task |

### Replica Types

```
NRT (Near Real-Time) :  default; নিজে index করে ও সার্চ-ও করে
TLOG (Transaction)   :  index করে কিন্তু search করে না; failover-এ leader হতে পারে
PULL                 :  শুধু index file copy; leader হতে পারে না; read-heavy scale
```

Read-heavy scenario: 1 NRT leader + 1 TLOG + 6 PULL = high read throughput, low write impact।

### ZooKeeper

ZooKeeper-এ থাকে:
- `/clusterstate.json` — কোন node-এ কোন replica
- `/configs/<name>` — schema, solrconfig.xml
- `/aliases.json`
- live nodes ephemeral znodes

**Production rule:** ZooKeeper minimum 3-node ensemble (5 better)। SolrCloud ZK ছাড়া চলে না।

---

## 📐 Schema vs Schemaless Mode

### Classic Schema Mode (recommended for production)

`schema.xml` (অথবা newer `managed-schema`) — সব field, fieldType আগেই declare।

```xml
<schema name="products" version="1.6">
  <field name="id"          type="string" indexed="true" stored="true" required="true" multiValued="false"/>
  <field name="title"       type="text_general"  indexed="true" stored="true"/>
  <field name="title_bn"    type="text_bangla"   indexed="true" stored="true"/>
  <field name="price_bdt"   type="plong"         indexed="true" stored="true"/>
  <field name="category"    type="string"        indexed="true" stored="true" multiValued="true"/>
  <field name="seller_id"   type="plong"         indexed="true" stored="true"/>
  <field name="seller_name" type="string"        indexed="true" stored="true"/>
  <field name="rating"      type="pfloat"        indexed="true" stored="true"/>
  <field name="created_at"  type="pdate"         indexed="true" stored="true"/>
  <field name="location"    type="location"      indexed="true" stored="true"/>
  <field name="embedding"   type="knn_vector_768" indexed="true" stored="true"/>

  <field name="_text_"      type="text_general"  indexed="true" stored="false" multiValued="true"/>

  <copyField source="title"     dest="_text_"/>
  <copyField source="title_bn"  dest="_text_"/>
  <copyField source="seller_name" dest="_text_"/>

  <uniqueKey>id</uniqueKey>
</schema>
```

### Schemaless Mode

প্রথম document index করার সময় auto field detection (string → date → number etc.)। **Production-এ এড়িয়ে চলুন** — Elasticsearch dynamic mapping-এর মতোই বিপদ।

### Managed Schema API

REST API দিয়ে runtime schema modify:

```bash
curl -X POST http://localhost:8983/solr/products/schema -H 'Content-Type: application/json' -d '{
  "add-field": { "name":"discount_pct", "type":"pint", "indexed":true, "stored":true }
}'
```

ZooKeeper সাথে সাথে replicate করে।

---

## 🧩 Field Types, copyField, dynamicField

### fieldType

```xml
<fieldType name="text_general" class="solr.TextField" positionIncrementGap="100">
  <analyzer type="index">
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.StopFilterFactory" words="stopwords.txt"/>
    <filter class="solr.PorterStemFilterFactory"/>
  </analyzer>
  <analyzer type="query">
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.SynonymGraphFilterFactory" synonyms="synonyms.txt" expand="true"/>
    <filter class="solr.PorterStemFilterFactory"/>
  </analyzer>
</fieldType>

<fieldType name="knn_vector_768" class="solr.DenseVectorField" vectorDimension="768"
           similarityFunction="cosine"/>
```

### copyField

একটি field-এর content আরেক field-এ duplicate (ভিন্ন analyzer-এ index করার জন্য)।

```xml
<copyField source="title" dest="title_exact" maxChars="256"/>
<copyField source="title" dest="_text_"/>
```

ব্যবহার:
- Searchable catch-all field
- Same data ভিন্ন analyzer-এ (exact + stemmed + ngram)
- Suggester-এর জন্য আলাদা field

### dynamicField

Naming pattern-এ field type assign — schemaless-এর controlled version।

```xml
<dynamicField name="*_i"  type="pint"     indexed="true" stored="true"/>
<dynamicField name="*_s"  type="string"   indexed="true" stored="true"/>
<dynamicField name="*_t"  type="text_general" indexed="true" stored="true"/>
<dynamicField name="*_dt" type="pdate"    indexed="true" stored="true"/>
<dynamicField name="*_ss" type="string"   indexed="true" stored="true" multiValued="true"/>
```

```json
// document
{ "id":"123", "color_s":"red", "weight_i":250, "tags_ss":["new","sale"] }
```

ভাল practice; convention দিয়ে schema-less-এর flexibility পান।

---

## 🔠 Analyzers ও Bangla Handling

Solr-এর analyzer Lucene-এর সরাসরি wrapper, তাই ES-এর মতোই concept।

### Built-in Tokenizers / Filters (selected)

```
Tokenizers   : Standard, Whitespace, NGram, EdgeNGram, ICU,
               Pattern, PathHierarchy, ClassicTokenizer
Filters      : LowerCase, Stop, Stemmer (Porter, KStem, Snowball),
               Synonym, ASCIIFolding, ICUNormalizer, NGram,
               WordDelimiterGraph, ShingleFilter, PhoneticFilter
```

### Bangla support — সীমাবদ্ধতা

Solr-এ built-in `BengaliAnalyzer` (Lucene contrib) **আছে** কিন্তু:
- `BengaliStemFilter` rule-based, modern terms-এ দুর্বল
- ICU plugin manually setup করতে হয়
- Synonym list manual

### Custom Bangla fieldType

```xml
<fieldType name="text_bangla" class="solr.TextField" positionIncrementGap="100">
  <analyzer type="index">
    <charFilter class="solr.MappingCharFilterFactory" mapping="bn_digits.txt"/>
    <tokenizer class="solr.ICUTokenizerFactory"/>
    <filter class="solr.ICUNormalizer2FilterFactory" name="nfkc_cf" mode="compose"/>
    <filter class="solr.StopFilterFactory" words="bn_stopwords.txt" ignoreCase="true"/>
    <filter class="solr.BengaliNormalizationFilterFactory"/>
    <filter class="solr.BengaliStemFilterFactory"/>
  </analyzer>
  <analyzer type="query">
    <charFilter class="solr.MappingCharFilterFactory" mapping="bn_digits.txt"/>
    <tokenizer class="solr.ICUTokenizerFactory"/>
    <filter class="solr.ICUNormalizer2FilterFactory" name="nfkc_cf" mode="compose"/>
    <filter class="solr.SynonymGraphFilterFactory" synonyms="bn_synonyms.txt" expand="true"/>
    <filter class="solr.StopFilterFactory" words="bn_stopwords.txt"/>
    <filter class="solr.BengaliNormalizationFilterFactory"/>
    <filter class="solr.BengaliStemFilterFactory"/>
  </analyzer>
</fieldType>
```

`bn_digits.txt`:
```
"০" => "0"
"১" => "1"
"২" => "2"
...
"৯" => "9"
```

`bn_synonyms.txt`:
```
চিনি, sugar
ডিম, egg
মোবাইল, ফোন, phone, mobile
টিভি, tv, television
```

> 🛠️ ICU support-এর জন্য `analysis-extras` jar `server/solr/lib/`-এ copy করতে হবে।

---

## 🔍 Query Parsers

Solr-এ একাধিক query parser; query string-এ `{!parser}` দিয়ে নির্বাচন।

### ১. Standard Lucene Parser (`q=`)

```
q=title:iphone AND price_bdt:[50000 TO 250000]
```

Strict syntax; non-tech user input direct দেওয়া বিপজ্জনক।

### ২. DisMax — multi-field "for users"

```
q=iphone 15 pro
defType=dismax
qf=title^3 title_bn^3 description seller_name
mm=2<-1 5<80%        # minimum should match
pf=title^5            # phrase boost on full match
ps=2                  # phrase slop
```

### ৩. Edismax (Extended DisMax) — সবচেয়ে ব্যবহৃত

DisMax-এর সব + Lucene syntax allow + per-field boost + stopwords sensible:

```
defType=edismax
q=আইফোন pro
qf=title^3 title_bn^3 description tags^2
mm=2<-1 5<80%
pf=title^5 title_bn^5
pf2=title^3
pf3=title^2
bf=recip(ms(NOW,created_at),3.16e-11,1,1)   # recency boost
boost=product(rating, 1.2)
fq=in_stock:true
fq=category:Mobile
fq={!field f=price_bdt}[50000 TO 250000]
```

`fq` (filter query) — cached, score-এ অবদান নেই, fast।

### ৪. JSON Request API (modern)

```bash
curl http://localhost:8983/solr/products/select -d '{
  "query": {
    "edismax": {
      "query": "আইফোন pro",
      "qf": "title^3 title_bn^3 description",
      "pf": "title^5"
    }
  },
  "filter": [
    "in_stock:true",
    "category:Mobile",
    "price_bdt:[50000 TO 250000]"
  ],
  "sort": "score desc, rating desc",
  "limit": 20,
  "fields": "id,title,price_bdt,seller_name,score",
  "facet": {
    "by_brand": { "type":"terms", "field":"brand", "limit":10 },
    "price_hist": { "type":"range", "field":"price_bdt",
                    "start":0, "end":300000, "gap":50000 }
  }
}'
```

### ৫. Block-Join — parent/child

E-commerce-এ rare; nested document use করলে।

```
q={!parent which="doc_type:product"}+sku:VARIANT-RED +stock:[1 TO *]
```

### ৬. Complex Phrase

```
q={!complexphrase}"iphone 15 *"~3
```

---

## 🎛️ Faceting, Highlighting, Spell Check, Suggester

### Faceting — Solr-এর strongest feature

```
facet=true
facet.field=brand
facet.field=category
facet.range=price_bdt
f.price_bdt.facet.range.start=0
f.price_bdt.facet.range.end=500000
f.price_bdt.facet.range.gap=10000
facet.pivot=category,brand           # nested facet
```

### JSON Facet API — modern, recommended

```json
"facet": {
  "categories": {
    "type": "terms", "field": "category", "limit": 20,
    "facet": {
      "avg_price": "avg(price_bdt)",
      "by_brand": { "type":"terms","field":"brand","limit":5 }
    }
  },
  "price_stats": "stats(price_bdt)",
  "monthly": {
    "type":"range","field":"created_at",
    "start":"NOW/MONTH-12MONTHS","end":"NOW/MONTH","gap":"+1MONTH"
  }
}
```

JSON Facet **Elasticsearch aggregation-এর সমান শক্তিশালী**, অনেক ক্ষেত্রে আরও দ্রুত (যেমন high-cardinality terms)।

### Highlighting

```
hl=true
hl.fl=title,description
hl.simple.pre=<em>
hl.simple.post=</em>
hl.fragsize=120
hl.method=unified            # fastest, recommended
```

### Spell Check

`solrconfig.xml`-এ:

```xml
<searchComponent name="spellcheck" class="solr.SpellCheckComponent">
  <lst name="spellchecker">
    <str name="name">default</str>
    <str name="field">_text_</str>
    <str name="classname">solr.DirectSolrSpellChecker</str>
    <str name="distanceMeasure">internal</str>
    <float name="accuracy">0.5</float>
  </lst>
</searchComponent>
```

Query: `spellcheck=true&spellcheck.collate=true` → "iphoen" → "iphone" সাজেশন।

### Suggester (Autocomplete)

```xml
<searchComponent name="suggest" class="solr.SuggestComponent">
  <lst name="suggester">
    <str name="name">productSuggester</str>
    <str name="lookupImpl">AnalyzingInfixLookupFactory</str>
    <str name="dictionaryImpl">DocumentDictionaryFactory</str>
    <str name="field">title</str>
    <str name="weightField">popularity</str>
    <str name="suggestAnalyzerFieldType">text_general</str>
  </lst>
</searchComponent>
```

```
/suggest?q=আইফো&suggest.dictionary=productSuggester&suggest.count=5
```

---

## ✏️ Updates: Atomic, Optimistic Concurrency, Commit Strategy

### Atomic Update — partial document

পুরো doc reindex না করে নির্দিষ্ট field পরিবর্তন:

```json
[
  { "id":"SKU-IPHONE-15-256",
    "stock":     { "set": 11 },
    "price_bdt": { "set": 192000 },
    "tags":      { "add": "discount" } }
]
```

⚠️ Atomic update-এর জন্য সব field `stored="true"` অথবা `docValues="true"` থাকতে হবে — internally Solr পুরো doc reconstruct করে reindex করে।

### Optimistic Concurrency (`_version_`)

Solr প্রতিটি doc-এ auto `_version_` field রাখে।

```json
[ { "id":"SKU-1", "_version_": 1683929292929292929,
    "price_bdt": { "set": 195000 } } ]
```

- `_version_ > 0` : exactly এই version আছে কিনা; mismatch → 409 Conflict
- `_version_ = -1`: doc অবশ্যই থাকবে
- `_version_ = 1` : doc থাকবে না (insert only)

### Commit Strategy — Solr-এর সবচেয়ে গুরুত্বপূর্ণ tuning

```
Hard Commit  : tlog flush + new segment + (openSearcher হলে searchable)
              - durability guarantee
              - expensive

Soft Commit  : doc searchable হয় (Lucene reopen searcher)
              - কোনো disk fsync নেই
              - মাঝারি cost (cache invalidate)
              - এটাই ES-এর "refresh"-এর সমতুল্য
```

`solrconfig.xml`:

```xml
<updateHandler class="solr.DirectUpdateHandler2">
  <autoCommit>
    <maxTime>60000</maxTime>          <!-- hard commit every 60s -->
    <maxDocs>10000</maxDocs>
    <openSearcher>false</openSearcher> <!-- durability only, no search visibility -->
  </autoCommit>
  <autoSoftCommit>
    <maxTime>2000</maxTime>            <!-- search-visible after 2s -->
  </autoSoftCommit>
  <updateLog>
    <str name="dir">${solr.ulog.dir:}</str>
  </updateLog>
</updateHandler>
```

> ❌ Production-এ প্রতিটি update-এ `?commit=true` করবেন না — তীব্র GC pressure, segment explosion।
> ✅ অটো commit + soft commit interval tune করুন।

### Tlog (Transaction Log)

Durability + recovery + atomic update-এর backbone। Hard commit-এ truncate হয়। Tlog ভর্তি হলে replay slow।

---

## 🔄 Replication: Legacy Master-Slave vs SolrCloud

### Legacy Master-Slave (Standalone Solr)

```
Master         →  Slave 1
   (writes)      (read-only)
   index files   periodic pull
                 ↓
              Slave 2
              Slave 3
```

- Master SPOF
- Eventual consistency
- Manual failover
- Solr 9-এ deprecated (এখনো configurable)

### SolrCloud — automatic

- Leader election via ZooKeeper
- Real-time sync (NRT replicas)
- Automatic failover
- Distributed updates with versioning

```
Update flow (SolrCloud)
─────────────────────────
Client → any node (SmartClient route)
   ↓
Identify shard by hash(id)
   ↓
Forward to shard leader
   ↓
Leader: write tlog, apply to local Lucene
   ↓
Replicate to all replicas (parallel, sync)
   ↓
ACK to client when min_rf met
```

`min_rf` (minimum replication factor) — অনেকটা Cassandra-র `QUORUM`-এর মতো।

---

## ⚡ Performance: Caches, Warming, Tlog

### Solr Caches (per searcher per core)

| Cache | কী store | কখন hit |
|-------|----------|---------|
| **filterCache** | bitset of docs matching `fq` | পুনরাবৃত্ত filter (category=Mobile) |
| **queryResultCache** | top-N doc IDs for full query | exact query repeat |
| **documentCache** | stored field of recently fetched doc | doc list page render |
| **fieldValueCache** | uninverted multivalued field | facet on multivalued |
| **perSegFilter** | segment-level filter | NRT scenarios |

```xml
<query>
  <filterCache         size="2048"  initialSize="512"  autowarmCount="256"/>
  <queryResultCache    size="2048"  initialSize="512"  autowarmCount="128"/>
  <documentCache       size="4096"  initialSize="1024" autowarmCount="0"/>
  <maxBooleanClauses>4096</maxBooleanClauses>
</query>
```

### Warming Queries

Cold cache slow; new searcher open হলে আগেই common queries চালান:

```xml
<listener event="newSearcher" class="solr.QuerySenderListener">
  <arr name="queries">
    <lst><str name="q">*:*</str><str name="rows">0</str>
         <str name="facet">true</str><str name="facet.field">category</str></lst>
    <lst><str name="q">iphone</str><str name="rows">10</str></lst>
  </arr>
</listener>
```

### JVM / OS

- Heap ৮–৩১GB; > 32GB মানে compressed oops break
- G1GC default in Solr 9
- OS file cache-এর জন্য বাকি RAM ছাড়ুন (Lucene mmap)
- `ulimit -n 65535`
- `vm.max_map_count = 262144`

### Streaming Expressions

বড় export, joins, aggregations:

```
expr=search(products,
            q="*:*",
            fl="id,seller_id,price_bdt",
            sort="seller_id asc",
            qt="/export")
```

`/export` handler: streaming, no top-N limit, hundreds of millions doc।

---

## 🔐 Security

### BasicAuth + RBAC

`security.json` (ZooKeeper-এ upload):

```json
{
  "authentication": {
    "blockUnknown": true,
    "class": "solr.BasicAuthPlugin",
    "credentials": {
      "solr": "<bcrypt_hash>",
      "search_app": "<bcrypt_hash>"
    }
  },
  "authorization": {
    "class": "solr.RuleBasedAuthorizationPlugin",
    "permissions": [
      { "name":"read",  "role":"reader" },
      { "name":"update","role":"writer" },
      { "name":"all",   "role":"admin"  }
    ],
    "user-role": {
      "solr":"admin",
      "search_app":["reader","writer"]
    }
  }
}
```

### SSL / TLS

`bin/solr -Dsolr.ssl.enabled=true ...`; certificate keystore configure।

### Kerberos / SPNEGO

Hadoop-heavy enterprise-এ — KDC ticket দিয়ে auth। Yarn/HDFS integration-এ ব্যাপক।

### Audit Logging

`AuditLoggerPlugin` configurable — request, user, IP, response code।

---

## 💻 Client Libraries (PHP Solarium + Node)

### PHP — Solarium

```bash
composer require solarium/solarium
```

```php
<?php
use Solarium\Client;

$config = [
    'endpoint' => [
        'localhost' => [
            'host' => 'solr.daraz.local',
            'port' => 8983,
            'path' => '/',
            'collection' => 'products',
            'username' => 'search_app',
            'password' => getenv('SOLR_PASS'),
        ]
    ]
];

$client = new Client(new \Solarium\Core\Client\Adapter\Curl(),
                     new \Symfony\Component\EventDispatcher\EventDispatcher(),
                     $config);

// 1. Index a Daraz product (Bangla)
$update = $client->createUpdate();
$doc = $update->createDocument();
$doc->id          = 'SKU-IPHONE-15-256';
$doc->title       = 'iPhone 15 Pro Max 256GB';
$doc->title_bn    = 'আইফোন ১৫ প্রো ম্যাক্স ২৫৬জিবি';
$doc->price_bdt   = 195000;
$doc->stock       = 12;
$doc->category    = ['Mobile','Smartphone','Apple'];
$doc->seller_id   = 42;
$doc->seller_name = 'Daraz Mall';
$doc->rating      = 4.7;
$doc->created_at  = gmdate('Y-m-d\TH:i:s\Z');
$doc->location    = '23.7515,90.3782';

$update->addDocument($doc);
$update->addCommit(false, false, true);   // softCommit
$client->update($update);

// 2. Edismax search — Bangla + English mixed
$query = $client->createSelect();
$query->setQuery('আইফোন pro');

$edismax = $query->getEDisMax();
$edismax->setQueryFields('title^3 title_bn^3 description tags^2');
$edismax->setMinimumMatch('2<-1 5<80%');
$edismax->setPhraseFields('title^5 title_bn^5');
$edismax->setBoostFunctions('recip(ms(NOW,created_at),3.16e-11,1,1)');

$query->addFilterQuery(['key'=>'cat','query'=>'category:Mobile']);
$query->addFilterQuery(['key'=>'price','query'=>'price_bdt:[50000 TO 250000]']);
$query->addFilterQuery(['key'=>'stock','query'=>'stock:[1 TO *]']);

$query->addSort('score','desc');
$query->setRows(20);
$query->setFields(['id','title','price_bdt','seller_name','rating','score']);

// Faceting
$facet = $query->getFacetSet();
$facet->createFacetField('brand')->setField('brand')->setLimit(10);
$facet->createFacetRange('price')->setField('price_bdt')
      ->setStart(0)->setEnd(300000)->setGap(50000);

// Highlighting
$hl = $query->getHighlighting();
$hl->setFields('title,title_bn');
$hl->setSimplePrefix('<em>');
$hl->setSimplePostfix('</em>');

$result = $client->select($query);

printf("পাওয়া গেছে: %d\n", $result->getNumFound());
foreach ($result as $d) {
    printf("%-50s ৳%d ★%.1f\n",
        $d->title, $d->price_bdt, $d->rating);
}

foreach ($result->getFacetSet()->getFacet('brand') as $brand => $cnt) {
    echo "$brand : $cnt\n";
}

// 3. Atomic update — stock decrement
$update = $client->createUpdate();
$doc = $update->createDocument();
$doc->setKey('id');
$doc->id = 'SKU-IPHONE-15-256';
$doc->setField('stock',  11, null, 'set');
$doc->setField('tags',   'discount', null, 'add');
$update->addDocument($doc);
$update->addCommit(false, false, true);
$client->update($update);

// 4. kNN vector search (Solr 9.4+)
$query = $client->createSelect();
$vector = '[' . implode(',', $embedding768) . ']';
$query->setQuery('{!knn f=embedding topK=30}' . $vector);
$query->addFilterQuery(['key'=>'stock','query'=>'stock:[1 TO *]']);
$query->setFields(['id','title','price_bdt','score']);
$result = $client->select($query);
```

### Node.js — `solr-client`

```bash
npm install solr-client
```

```ts
import { createClient } from 'solr-client';

const solr = createClient({
  host: 'solr.daraz.local',
  port: 8983,
  core: 'products',
  secure: false,
  basicAuth: { username:'search_app', password: process.env.SOLR_PASS! }
});

// Add
await new Promise<void>((res, rej) =>
  solr.add(
    [{ id:'SKU-IPHONE-15-256', title:'iPhone 15 Pro Max',
       title_bn:'আইফোন ১৫', price_bdt: 195000, stock:12 }],
    (err) => err ? rej(err) : res()
  )
);
await new Promise<void>((r,e)=>solr.softCommit(err=>err?e(err):r()));

// Search
const q = solr.createQuery()
  .q({ title:'iphone', title_bn:'আইফোন' })
  .qop('AND')
  .fq([
    { field:'category', value:'Mobile' },
    { field:'price_bdt', value:'[50000 TO 250000]' },
  ])
  .sort({ score:'desc', rating:'desc' })
  .fl(['id','title','price_bdt','seller_name','score'])
  .rows(20)
  .facet({ on:true, field:['brand','category'] });

const result: any = await new Promise((res, rej) =>
  solr.search(q, (err, r) => err ? rej(err) : res(r))
);

console.log(result.response.numFound, result.response.docs);
```

Solarium-এর সমান feature-rich JS client কম; অনেক team direct HTTP (axios + JSON Request API) ব্যবহার করে।

---

## ⚖️ Solr vs Elasticsearch — Comprehensive Comparison

| বিষয় | **Apache Solr** | **Elasticsearch** |
|------|-----------------|-------------------|
| **Underlying** | Apache Lucene | Apache Lucene |
| **License** | Apache 2.0 ✅ (true OSS) | SSPL/Elastic 2.0 (since 7.11), AGPL option 8.x |
| **Vendor** | Apache Foundation | Elastic NV (commercial) |
| **Cluster coordination** | Apache ZooKeeper (external) | Built-in (Zen → since 7.0 internal Raft-like) |
| **Deployment** | একটু ভারী (ZK setup) | Self-contained, easier |
| **Schema** | schema.xml / managed-schema (explicit-first) | Mapping JSON (dynamic-first) |
| **Schemaless mode** | আছে but discouraged | Default behavior |
| **Query DSL** | URL params, JSON Request API, Lucene/edismax | JSON Query DSL (cleaner, modern) |
| **Default analyzer** | text_general | standard |
| **Faceting** | Facet API + JSON Facet — **mature, fast on high cardinality** | Aggregations |
| **Aggregation/Analytics** | JSON Facet, Streaming Expressions | Pipeline aggregations, Composite agg |
| **Real-time / NRT** | Soft commits (~1-2s) | Refresh interval (1s default) |
| **Atomic updates** | Native partial update | `update` with script/doc, less ergonomic |
| **Optimistic concurrency** | `_version_` field native | `_seq_no` + `_primary_term`, `if_seq_no` param |
| **Security** | BasicAuth, Kerberos, RBAC, SSL — free | Basic free, advanced (FLS/DLS) Platinum |
| **Highlighting** | Unified, Original, FastVector | Plain, Postings, FVH |
| **Spell check / Suggest** | Multiple suggesters (analyzing infix, fuzzy) | Phrase suggester, completion suggester |
| **Vector search (HNSW)** | Solr 9.0+ | 8.0+ (more tuning options) |
| **Hybrid search** | Manual query combination | Native RRF retriever |
| **ML** | Limited; Streaming + LTR plugin | X-Pack ML, ELSER, anomaly det. |
| **Cross-cluster search** | আছে (newer) | Mature (CCS, CCR) |
| **Time-series / logs** | Time Routed Aliases | ILM, Data Streams (purpose-built) |
| **Cloud managed** | AWS (limited), Bitnami, self-host | Elastic Cloud, AWS, GCP, Azure |
| **Admin UI** | Solr Admin (functional, plain) | Kibana (rich, dashboards) |
| **Ecosystem** | Hadoop, HDFS, Tika, Nutch, Apache stack | Beats, Logstash, Kibana, APM, Fleet |
| **Client libs** | SolrJ (Java best), Solarium (PHP), pysolr | Best-in-class for ALL languages |
| **Documentation** | Detailed reference guide | Polished, marketing-rich |
| **Community momentum** | Stable, slower | Faster, larger |
| **Performance** | Faceting on high-cardinality faster (often) | General search latency similar |
| **Enterprise legacy** | বেশি (banks, gov, Hadoop shops) | বাড়ছে কিন্তু license সমস্যা |
| **Learning curve** | Steeper (XML, ZK, more knobs) | Gentler |
| **Best at** | Complex faceting, structured search, OSS-strict, Hadoop integration | Modern dev experience, observability, ML, semantic |

### সংক্ষিপ্ত decision

```
চাই strict OSS (Apache 2.0)              → Solr (অথবা OpenSearch)
চাই hyper-rich faceting                  → Solr
আছে Hadoop/Java enterprise               → Solr
চাই best dev UX, modern API              → Elasticsearch
চাই log/metrics/APM ecosystem            → Elasticsearch + Kibana
চাই managed cloud, fast adoption         → Elasticsearch (Elastic Cloud)
চাই latest semantic/vector ML            → Elasticsearch
ZooKeeper avoid করতে চান                  → Elasticsearch
SSPL license-এ আপত্তি নাই                → Elasticsearch
```

---

## 🎯 কখন Solr বেছে নিবেন

### ১. Apache 2.0 license requirement

Government, public sector, certain commercial clauses-এ SSPL/Elastic license unacceptable। OpenSearch একটি বিকল্প, কিন্তু Solr-এর maturity ১৫+ বছর।

### ২. Existing Hadoop / Java ecosystem

- Cloudera / Hortonworks / EMR cluster-এ Solr (Cloudera Search) integration ready
- HDFS-এ index store
- Spark + SolrJ batch indexing
- Kerberos SSO

### ৩. Heavy faceting / structured search

E-commerce filter, library catalog, scientific dataset — JSON Facet API ES-এর aggregation-এর চেয়ে দ্রুত high-cardinality terms-এ। Pivot facet, range facet, query facet — খুব expressive।

### ৪. Strict schema / structured data

Library OPAC (Online Public Access Catalog), bibliographic — explicit schema, controlled vocabulary; Solr-এর schema.xml fit।

### ৫. Streaming Expressions

বড় export / cross-collection join / aggregation — Solr-এর parallel SQL-like streaming।

### ৬. Cost-sensitive self-hosting

ZooKeeper + Solr — সব OSS, পুরো cluster self-host করতে free; ES-এ basic license-এও কিছু feature পরিবর্তিত হতে পারে future-এ।

---

## 🔄 Migration Patterns Solr↔ES

### Solr → Elasticsearch (more common)

```
1. Schema mapping
   schema.xml fieldType → ES mapping JSON
   - text_general    → text + standard analyzer
   - string          → keyword
   - plong/pint      → long/integer
   - pdate           → date
   - location        → geo_point
   - copyField       → multi-fields ("fields":{"raw":...})

2. Analyzer migration
   Solr filter chain ↔ ES analyzer chain (most equivalent)

3. Data migration
   Option A: Re-index from source (RDBMS) to ES
   Option B: Solr /export (streaming) → Logstash solr_input → ES
   Option C: Custom script: SolrJ deep paging → bulk ES

4. Query rewrite
   edismax    → multi_match best_fields
   fq         → bool.filter
   facet      → aggs.terms / range
   highlight  → highlight (mostly compatible)

5. Dual-write phase
   App writes both Solr + ES; query Solr; verify ES result quality
   Switch query traffic gradually (5% → 25% → 100%)
```

### Elasticsearch → Solr

বেশ rare; license driver হলে OpenSearch-এ যাওয়া সাধারণত সহজ। Solr-এ গেলে:

```
1. ES mapping → schema.xml + dynamicField
2. Query DSL → Solr edismax + JSON Request API
3. Aggregations → JSON Facet API
4. ES dump (elasticdump / scroll) → Solr Bulk JSON
5. Kibana → Banana (Lucidworks) বা Grafana
```

---

## 🇧🇩 BD Real-world Case Study

### ১. National e-GP (e-Government Procurement) — Solr

বাংলাদেশের electronic government procurement system জাতীয় পর্যায়ে tender/contract data search করতে Solr ব্যবহার করে (representative — অনেক সরকারি ও SOE system এই ধরনের stack-এ আছে):

- **কেন Solr?** Apache 2.0 strict, vendor-neutral procurement requirement; existing Java/JBoss stack
- **Volume:** ~12M tender doc, 50M activity log
- **Schema:** Heavy structured (tender_id, ministry, category, status, dates, amount range)
- **Faceting:** ministry, category, year, status — JSON facet pivot
- **Bangla:** Custom analyzer (ICU + Bengali stem) tender description-এ
- **Hosting:** On-prem 3-node SolrCloud + 3 ZooKeeper

### ২. Prothom Alo Archive — Solr legacy → ES migration

Bangladeshi news archive ১৫+ বছরের article (millions); প্রথমে Solr 4.x-এ ছিল। ২০২২-এ ES 7-এ migrate (CMS team আপগ্রেড সিদ্ধান্ত)। Lessons:

- Bangla analyzer migration time-consuming (synonym, stem)
- Highlight markup parity test critical
- Click-through metrics +12% after BM25 tuning + recency boost

### ৩. Bank — Loan/Compliance Search on Solr

Universal Bank Bangladesh-জাতীয় enterprise (representative) compliance search:

- Hadoop-এ raw transaction data; Solr-এ derived index
- Kerberos SSO via AD
- Strict audit logging requirement
- Regulator data export → streaming expressions
- ES যেখানে fits না: তাদের stack pure Apache

### ৪. Modern startup (Foodpanda BD-জাতীয়) — Elasticsearch

Restaurant + dish search:
- ES 8 + Bangla analyzer
- Geo + price + cuisine facet
- Vector search (semantic dish match)
- Kibana ops dashboards
- Managed Elastic Cloud

> **Pattern:** এন্টারপ্রাইজ legacy + government → Solr। Modern startup, observability-heavy → ES।

---

## 🚨 Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `?commit=true` প্রতি update-এ | GC storm, segment explosion | autoCommit + autoSoftCommit |
| Schemaless production | Field type drift | Explicit schema, dynamicField conventions |
| ZooKeeper 1-node | Cluster down যদি ZK fail | Min 3-node ensemble |
| Heap > 32GB | Compressed oops loss | ≤ 31GB |
| `*` leading wildcard | Slow scan | EdgeNGram field |
| Atomic update missing stored fields | Update fails / data loss | All fields stored or docValues |
| filterCache too small | Cache miss, slow re-queries | Tune size; eviction monitor |
| autowarmCount very high | Slow searcher open | Balance warm vs latency |
| `tlog` huge (no hard commit) | Slow recovery on restart | autoCommit hard ৬০s |
| `docValues=false` on facet field | Fielddata, OOM | docValues=true on keyword/numeric facet field |
| Query without `fq` for filters | filterCache miss | Use `fq` for non-scoring constraints |
| ZooKeeper connect timeout | Cluster ops fail | jute.maxbuffer, network reliability |
| Multi-tenant single core | Noisy neighbor | Collection-per-tenant or alias routing |
| Stale `clusterstate.json` | Outdated routes | Solr 7+ uses per-collection state — verify |

---

## ✅ Checklist

### Architecture
- [ ] SolrCloud, no legacy master-slave for new system
- [ ] ZooKeeper 3 (or 5) node ensemble, separate from Solr
- [ ] Replica strategy chosen (NRT default; PULL for read-heavy)
- [ ] Collection alias for blue/green reindex
- [ ] Backup/snapshot strategy (HDFS / S3 via repository)

### Schema
- [ ] Explicit schema (managed-schema), not schemaless
- [ ] `_version_`, `_root_`, `_text_` standard fields present
- [ ] copyField → catch-all `_text_` for naive search
- [ ] dynamicField conventions documented
- [ ] docValues=true on all facet/sort/group fields
- [ ] stored=true on fields used in atomic update / highlight
- [ ] Bangla fieldType (ICU + Bengali stem + synonyms) configured

### Indexing
- [ ] autoCommit hard ~60s, openSearcher=false
- [ ] autoSoftCommit ~1-5s based on freshness need
- [ ] Bulk update via `/update` JSON; never per-doc commit
- [ ] Tlog directory on fast disk
- [ ] Optimistic concurrency for update conflicts

### Query
- [ ] edismax default; raw Lucene parser only for power users
- [ ] `fq` used for all non-scoring filters
- [ ] Result fields explicit (`fl=`)
- [ ] Pagination cursor (`cursorMark`) for deep results
- [ ] JSON Facet API preferred over old facet params
- [ ] Highlighting method = unified

### Performance
- [ ] filterCache, queryResultCache tuned + warming queries
- [ ] Heap ≤ 31GB; G1GC; OS file cache plentiful
- [ ] `ulimit -n 65535`; `vm.max_map_count = 262144`
- [ ] Slow query log (`<requestHandler>` slow threshold)
- [ ] Metrics: Prometheus exporter / JMX → Grafana

### Security
- [ ] BasicAuth + RBAC enabled
- [ ] TLS on inter-node + client
- [ ] `security.json` in ZooKeeper, no anonymous access
- [ ] Audit logging enabled
- [ ] Network: VPC/private subnet, firewall

### Operations
- [ ] Snapshot/restore tested
- [ ] Rolling upgrade procedure documented
- [ ] Disaster recovery plan (ZooKeeper + collection backup)
- [ ] Monitoring alert: heap usage, query latency p95, indexing lag, ZK connection

---

> **চূড়ান্ত কথা:** Solr হলো বুদ্ধিমান, পরিণত, কখনো ক্লান্তিকর কিন্তু dependable engineer। Elasticsearch হলো brilliant, fast-moving, কখনো temperamental superstar। আপনার দলের চরিত্র, license-এর constraint, ecosystem (Hadoop / observability), এবং ০-৩ বছরের roadmap বুঝে বেছে নিন। Lucene-এর core বুঝলে দুটোই আপনার অস্ত্রভাণ্ডারে থাকবে।
