# 🔍 ফুল-টেক্সট সার্চ (Full-Text Search & Elasticsearch)

> ফুল-টেক্সট সার্চ হলো টেক্সট ডেটার মধ্যে দ্রুত ও প্রাসঙ্গিক (relevant) অনুসন্ধান।
> Elasticsearch এই ক্ষেত্রে শিল্পের মান। Daraz এ প্রোডাক্ট সার্চ, Prothom Alo তে নিউজ সার্চ — সব Elasticsearch দিয়ে হয়।

---

## 📖 সূচিপত্র

- [ফুল-টেক্সট সার্চ কী?](#-ফুল-টেক্সট-সার্চ-কী)
- [Inverted Index](#-inverted-index)
- [Tokenization ও Analyzers](#-tokenization-ও-analyzers)
- [Elasticsearch Architecture](#-elasticsearch-architecture)
- [Mapping ও Data Types](#-mapping-ও-data-types)
- [Query DSL](#-query-dsl)
- [LIKE vs Full-Text Search](#-like-vs-full-text-search)
- [Use Cases](#-use-cases)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 ফুল-টেক্সট সার্চ কী?

সাধারণ `LIKE` query এবং ফুল-টেক্সট সার্চের মধ্যে বিশাল পার্থক্য:

```
SQL LIKE Query:
──────────────
SELECT * FROM products WHERE name LIKE '%আইফোন%';

সমস্যা:
❌ "iPhone 15" খুঁজে পাবে না (ইংরেজি/বাংলা মিক্স)
❌ "আইফোন ১৫ প্রো" লিখলে "iPhone 15 Pro Max" পাবে না
❌ Spelling mistake সহ্য করে না
❌ Relevance ranking নেই (কোনটা বেশি প্রাসঙ্গিক?)
❌ ধীর (Full table scan)

Full-Text Search:
─────────────────
GET /products/_search
{ "query": { "match": { "name": "আইফোন 15" } } }

সুবিধা:
✅ "iPhone 15", "আইফোন ১৫", "iphone15" সব পায়
✅ Fuzzy matching ("iphoen" → "iPhone" সাজেস্ট করে)
✅ Relevance scoring (সবচেয়ে প্রাসঙ্গিক আগে)
✅ Synonyms ("মোবাইল" সার্চে "phone" ও আসে)
✅ অত্যন্ত দ্রুত (Inverted Index)
```

---

## 📊 Inverted Index

ফুল-টেক্সট সার্চের মূল রহস্য হলো **Inverted Index**। বইয়ের সূচিপত্রের মতো — শব্দ দিলে কোন ডকুমেন্টে আছে বলে দেয়।

```
ডকুমেন্ট সমূহ (Daraz প্রোডাক্ট):
────────────────────────────────
Doc 1: "iPhone 15 Pro Max 256GB Blue"
Doc 2: "Samsung Galaxy S24 Ultra Blue"
Doc 3: "iPhone 15 Case Black"
Doc 4: "Samsung Galaxy Buds Pro"

Forward Index (সাধারণ):
─────────────────────────
Doc 1 → [iPhone, 15, Pro, Max, 256GB, Blue]
Doc 2 → [Samsung, Galaxy, S24, Ultra, Blue]
Doc 3 → [iPhone, 15, Case, Black]
Doc 4 → [Samsung, Galaxy, Buds, Pro]

Inverted Index (উল্টো!):
──────────────────────────
┌──────────┬─────────────────┐
│   Term   │   Documents     │
├──────────┼─────────────────┤
│ iphone   │ Doc 1, Doc 3    │
│ 15       │ Doc 1, Doc 3    │
│ pro      │ Doc 1, Doc 4    │
│ max      │ Doc 1           │
│ 256gb    │ Doc 1           │
│ blue     │ Doc 1, Doc 2    │
│ samsung  │ Doc 2, Doc 4    │
│ galaxy   │ Doc 2, Doc 4    │
│ s24      │ Doc 2           │
│ ultra    │ Doc 2           │
│ case     │ Doc 3           │
│ black    │ Doc 3           │
│ buds     │ Doc 4           │
└──────────┴─────────────────┘

Query: "iPhone Blue"
  → "iPhone" → Doc 1, Doc 3
  → "Blue"   → Doc 1, Doc 2
  → Intersection/Scoring: Doc 1 (উভয় match!) > Doc 2, Doc 3

  Result:
  1. Doc 1: "iPhone 15 Pro Max 256GB Blue" (Score: 2.5) ★
  2. Doc 3: "iPhone 15 Case Black" (Score: 1.2)
  3. Doc 2: "Samsung Galaxy S24 Ultra Blue" (Score: 0.8)
```

---

## 📖 Tokenization ও Analyzers

Analyzer টেক্সটকে সার্চযোগ্য tokens এ রূপান্তর করে:

```
┌─────────────────────────────────────────────────────────┐
│                   Analyzer Pipeline                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Input: "iPhone 15 Pro-Max, 256GB Blue!"                │
│                                                         │
│  Step 1: Character Filters                              │
│  ├── HTML Strip: <b>iPhone</b> → iPhone                 │
│  ├── Mapping: & → and, ৳ → BDT                         │
│  └── Output: "iPhone 15 Pro-Max, 256GB Blue!"           │
│                                                         │
│  Step 2: Tokenizer                                      │
│  ├── Standard: শব্দ boundary তে ভাগ                      │
│  └── Output: ["iPhone", "15", "Pro", "Max", "256GB",    │
│               "Blue"]                                    │
│                                                         │
│  Step 3: Token Filters                                  │
│  ├── Lowercase: iPhone → iphone                         │
│  ├── Stop words: "the", "is", "a" সরানো                 │
│  ├── Stemming: running → run, phones → phone            │
│  ├── Synonyms: cell phone → mobile                      │
│  └── Output: ["iphone", "15", "pro", "max", "256gb",   │
│               "blue"]                                    │
│                                                         │
│  এই tokens গুলো Inverted Index এ যায়                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### বিভিন্ন Analyzer

```
┌────────────────┬──────────────────────────────────────────┐
│   Analyzer     │          কাজ                              │
├────────────────┼──────────────────────────────────────────┤
│ standard       │ Unicode টেক্সট, lowercase, stop words    │
│                │ "The Quick Fox" → [quick, fox]           │
├────────────────┼──────────────────────────────────────────┤
│ simple         │ Letter ছাড়া সব ভাগ করে                   │
│                │ "iPhone-15!" → [iphone, 15]              │
├────────────────┼──────────────────────────────────────────┤
│ whitespace     │ শুধু স্পেসে ভাগ                            │
│                │ "iPhone 15 Pro" → [iPhone, 15, Pro]      │
├────────────────┼──────────────────────────────────────────┤
│ keyword        │ পুরো টেক্সটকে একটি token হিসেবে রাখে     │
│                │ "iPhone 15 Pro" → [iPhone 15 Pro]        │
├────────────────┼──────────────────────────────────────────┤
│ bengali/       │ বাংলা ভাষা সাপোর্ট                       │
│ custom         │ "বাংলাদেশের সেরা ফোন" → [বাংলাদেশ, সের,  │
│                │  ফোন]                                    │
├────────────────┼──────────────────────────────────────────┤
│ edge_ngram     │ Autocomplete এর জন্য                     │
│                │ "iPhone" → [i, ip, iph, ipho, iphon,    │
│                │  iphone]                                 │
└────────────────┴──────────────────────────────────────────┘
```

---

## 🏗️ Elasticsearch Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Elasticsearch Cluster                        │
│                                                          │
│  ┌────────────────┐  ┌────────────────┐                 │
│  │   Node 1       │  │   Node 2       │                 │
│  │  (Master +     │  │  (Data)        │                 │
│  │   Data)        │  │                │                 │
│  │  ┌──────────┐  │  │  ┌──────────┐  │  ┌───────────┐ │
│  │  │ Shard P0 │  │  │  │ Shard P1 │  │  │  Node 3   │ │
│  │  │(Primary) │  │  │  │(Primary) │  │  │  (Data)   │ │
│  │  └──────────┘  │  │  └──────────┘  │  │           │ │
│  │  ┌──────────┐  │  │  ┌──────────┐  │  │┌─────────┐│ │
│  │  │ Shard R1 │  │  │  │ Shard R0 │  │  ││Shard P2 ││ │
│  │  │(Replica) │  │  │  │(Replica) │  │  ││(Primary)││ │
│  │  └──────────┘  │  │  └──────────┘  │  │└─────────┘│ │
│  │                │  │                │  │┌─────────┐│ │
│  └────────────────┘  └────────────────┘  ││Shard R2 ││ │
│                                          ││(Replica)││ │
│                                          │└─────────┘│ │
│                                          └───────────┘ │
│                                                          │
│  Index "products":                                       │
│  ├── Primary Shards: P0, P1, P2 (৩টি)                   │
│  ├── Replica Shards: R0, R1, R2 (প্রতিটির কপি)           │
│  └── মোট: ৬টি Shard (ভিন্ন ভিন্ন Node এ)                 │
│                                                          │
└─────────────────────────────────────────────────────────┘

মূল ধারণা:
──────────
Cluster  = একাধিক Node এর সমষ্টি
Node     = একটি Elasticsearch instance
Index    = ডাটাবেসের টেবিলের মতো
Shard    = Index এর ভাগ (horizontal partitioning)
Document = একটি JSON object (row এর মতো)
Replica  = Shard এর কপি (ফেইলওভার ও read scaling)
```

### ডেটা ফ্লো

```
Write Flow:
──────────
Client ─► Coordinating Node ─► Primary Shard ─► Replica Shard
                                     │                │
                                     ▼                ▼
                              Index Document    Replicate

Read Flow:
──────────
Client ─► Coordinating Node ─► যেকোনো Shard (Primary বা Replica)
                                     │
                                     ▼
                              Return Results

Search Flow:
────────────
Client ─► Coordinating Node
               │
        ┌──────┼──────┐
        ▼      ▼      ▼
     Shard 0 Shard 1 Shard 2  (Query Phase: প্রতিটি shard search)
        │      │      │
        └──────┼──────┘
               ▼
     Coordinating Node          (Fetch Phase: merge + sort)
               │
               ▼
           Client
```

---

## 📖 Mapping ও Data Types

```json
// Daraz প্রোডাক্ট Index Mapping
{
    "mappings": {
        "properties": {
            "name": {
                "type": "text",
                "analyzer": "standard",
                "fields": {
                    "keyword": { "type": "keyword" },
                    "autocomplete": {
                        "type": "text",
                        "analyzer": "edge_ngram_analyzer"
                    }
                }
            },
            "name_bn": {
                "type": "text",
                "analyzer": "bengali_custom"
            },
            "description": {
                "type": "text",
                "analyzer": "standard"
            },
            "category": {
                "type": "keyword"
            },
            "brand": {
                "type": "keyword"
            },
            "price": {
                "type": "float"
            },
            "rating": {
                "type": "float"
            },
            "reviews_count": {
                "type": "integer"
            },
            "in_stock": {
                "type": "boolean"
            },
            "seller": {
                "type": "object",
                "properties": {
                    "name": { "type": "keyword" },
                    "rating": { "type": "float" }
                }
            },
            "tags": {
                "type": "keyword"
            },
            "created_at": {
                "type": "date"
            },
            "location": {
                "type": "geo_point"
            }
        }
    }
}
```

### Data Types

```
┌──────────────┬───────────────────────────────────────────┐
│  Type        │              ব্যবহার                       │
├──────────────┼───────────────────────────────────────────┤
│ text         │ Full-text search (analyzed, tokenized)    │
│              │ প্রোডাক্ট নাম, বিবরণ                       │
├──────────────┼───────────────────────────────────────────┤
│ keyword      │ Exact match, aggregation, sorting        │
│              │ ক্যাটাগরি, ব্র্যান্ড, স্ট্যাটাস              │
├──────────────┼───────────────────────────────────────────┤
│ integer/long │ সংখ্যা (reviews_count, stock_qty)          │
├──────────────┼───────────────────────────────────────────┤
│ float/double │ দশমিক সংখ্যা (price, rating)               │
├──────────────┼───────────────────────────────────────────┤
│ boolean      │ true/false (in_stock, is_active)         │
├──────────────┼───────────────────────────────────────────┤
│ date         │ তারিখ/সময় (created_at, updated_at)       │
├──────────────┼───────────────────────────────────────────┤
│ object       │ JSON object (nested data)                │
├──────────────┼───────────────────────────────────────────┤
│ nested       │ Array of objects (independent querying)  │
├──────────────┼───────────────────────────────────────────┤
│ geo_point    │ GPS coordinates (lat, lon)               │
└──────────────┴───────────────────────────────────────────┘
```

---

## 📖 Query DSL

### Match Query (ফুল-টেক্সট সার্চ)

```json
// "iPhone 15 Pro" সার্চ
{
    "query": {
        "match": {
            "name": {
                "query": "iPhone 15 Pro",
                "operator": "and"
            }
        }
    }
}

// Multi-match (একাধিক ফিল্ডে সার্চ)
{
    "query": {
        "multi_match": {
            "query": "আইফোন ফিফটিন",
            "fields": ["name^3", "name_bn^2", "description"],
            "type": "best_fields",
            "fuzziness": "AUTO"
        }
    }
}
// name^3 = name ফিল্ডের score ৩ গুণ বেশি গুরুত্ব
```

### Bool Query (জটিল কোয়েরি)

```json
// Daraz-style ফিল্টারসহ সার্চ
{
    "query": {
        "bool": {
            "must": [
                { "match": { "name": "iPhone 15" } }
            ],
            "filter": [
                { "term": { "category": "phones" } },
                { "range": { "price": { "gte": 100000, "lte": 200000 } } },
                { "term": { "in_stock": true } }
            ],
            "should": [
                { "term": { "brand": "Apple" } },
                { "range": { "rating": { "gte": 4.0 } } }
            ],
            "must_not": [
                { "term": { "seller.name": "blocked_seller" } }
            ],
            "minimum_should_match": 1
        }
    },
    "sort": [
        { "_score": "desc" },
        { "price": "asc" }
    ],
    "from": 0,
    "size": 20,
    "highlight": {
        "fields": {
            "name": {},
            "description": {}
        }
    }
}

// ব্যাখ্যা:
// must:     অবশ্যই match হতে হবে (AND) + score প্রভাবিত করে
// filter:   অবশ্যই match হতে হবে (AND) + score প্রভাবিত করে না (দ্রুত!)
// should:   match হলে ভালো, না হলেও চলবে (OR) + score বাড়ায়
// must_not: অবশ্যই match করবে না (NOT)
```

### Aggregations (Analytics)

```json
// Daraz ক্যাটাগরি অনুযায়ী প্রোডাক্ট সংখ্যা ও গড় দাম
{
    "size": 0,
    "aggs": {
        "categories": {
            "terms": {
                "field": "category",
                "size": 10
            },
            "aggs": {
                "avg_price": {
                    "avg": { "field": "price" }
                },
                "price_ranges": {
                    "range": {
                        "field": "price",
                        "ranges": [
                            { "key": "budget", "to": 10000 },
                            { "key": "mid", "from": 10000, "to": 50000 },
                            { "key": "premium", "from": 50000 }
                        ]
                    }
                },
                "avg_rating": {
                    "avg": { "field": "rating" }
                },
                "top_brands": {
                    "terms": {
                        "field": "brand",
                        "size": 5
                    }
                }
            }
        }
    }
}

// ফলাফল কেমন দেখায়:
// categories:
//   phones: 1523 products, avg ৳45,000
//     budget: 234, mid: 890, premium: 399
//     top brands: Samsung(450), Xiaomi(380), Apple(250)...
//   accessories: 3456 products, avg ৳2,500
//     ...
```

### Fuzzy ও Autocomplete

```json
// Fuzzy সার্চ (spelling mistake সহ্য করে)
{
    "query": {
        "match": {
            "name": {
                "query": "iphoen",
                "fuzziness": "AUTO"
            }
        }
    }
}
// "iphoen" → "iPhone" পাবে! ✅

// Autocomplete (Search-as-you-type)
// "iph" টাইপ করলেই suggestions আসবে
{
    "query": {
        "match_phrase_prefix": {
            "name.autocomplete": {
                "query": "iph",
                "max_expansions": 10
            }
        }
    },
    "suggest": {
        "product-suggest": {
            "prefix": "iph",
            "completion": {
                "field": "suggest",
                "size": 5,
                "fuzzy": {
                    "fuzziness": "AUTO"
                }
            }
        }
    }
}
```

---

## 📊 LIKE vs Full-Text Search

```
┌────────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য       │  SQL LIKE        │  Elasticsearch   │
├────────────────────┼──────────────────┼──────────────────┤
│ Speed (1M docs)    │ ~5 seconds ❌    │ ~5ms ✅          │
│ Relevance Ranking  │ ❌ নেই            │ ✅ TF-IDF/BM25   │
│ Fuzzy Match        │ ❌ নেই            │ ✅ fuzziness     │
│ Synonyms           │ ❌ নেই            │ ✅ synonym filter│
│ Autocomplete       │ ❌ কঠিন           │ ✅ edge_ngram    │
│ Highlighting       │ ❌ নেই            │ ✅ built-in      │
│ Aggregation        │ ⚠️ GROUP BY      │ ✅ Rich aggs     │
│ Multilingual       │ ❌ কঠিন           │ ✅ Language      │
│                    │                  │   analyzers       │
│ Typo Tolerance     │ ❌ নেই            │ ✅ fuzzy/suggest │
│ Phrase Search      │ ⚠️ LIKE '%a b%' │ ✅ match_phrase  │
│ Weighted Fields    │ ❌ নেই            │ ✅ boost (^)     │
│ Geo Search         │ ⚠️ PostGIS      │ ✅ built-in      │
│ Real-time          │ ❌ After commit  │ ✅ ~1s refresh   │
└────────────────────┴──────────────────┴──────────────────┘
```

---

## 🎯 Use Cases

```
┌───────────────────────┬─────────────────────────────────────┐
│     Use Case          │     বাংলাদেশী উদাহরণ                 │
├───────────────────────┼─────────────────────────────────────┤
│ প্রোডাক্ট সার্চ        │ Daraz: "iPhone 15 blue 256gb"       │
│                       │ ফিল্টার + সার্চ + সর্ট               │
├───────────────────────┼─────────────────────────────────────┤
│ লগ অ্যানালাইসিস        │ bKash: সার্ভার লগ সার্চ ও বিশ্লেষণ  │
│ (ELK Stack)           │ error pattern খুঁজে বের করা         │
├───────────────────────┼─────────────────────────────────────┤
│ নিউজ/কনটেন্ট সার্চ    │ Prothom Alo: আর্টিকেল সার্চ,        │
│                       │ বাংলা টেক্সট সার্চ                    │
├───────────────────────┼─────────────────────────────────────┤
│ Autocomplete          │ Pathao: ঠিকানা টাইপ করলে suggestion │
├───────────────────────┼─────────────────────────────────────┤
│ Geo Search            │ "আমার কাছে রেস্টুরেন্ট" →            │
│                       │ GPS ভিত্তিক সার্চ                     │
├───────────────────────┼─────────────────────────────────────┤
│ Analytics Dashboard   │ Grafana + ES: রিয়েল-টাইম মেট্রিক্স   │
├───────────────────────┼─────────────────────────────────────┤
│ Security (SIEM)       │ সাইবার নিরাপত্তা লগ বিশ্লেষণ         │
└───────────────────────┴─────────────────────────────────────┘
```

---

## 💻 কোড উদাহরণ

### PHP (elasticsearch-php)

```php
<?php
// composer require elasticsearch/elasticsearch

use Elastic\Elasticsearch\ClientBuilder;

// =============================================
// Elasticsearch PHP Client
// বাস্তব উদাহরণ: Daraz প্রোডাক্ট সার্চ
// =============================================

$client = ClientBuilder::create()
    ->setHosts(['localhost:9200'])
    ->build();

// --- Index তৈরি (Settings + Mapping) ---
$client->indices()->create([
    'index' => 'products',
    'body' => [
        'settings' => [
            'number_of_shards' => 3,
            'number_of_replicas' => 1,
            'analysis' => [
                'analyzer' => [
                    'autocomplete_analyzer' => [
                        'type' => 'custom',
                        'tokenizer' => 'autocomplete_tokenizer',
                        'filter' => ['lowercase']
                    ],
                    'bangla_analyzer' => [
                        'type' => 'custom',
                        'tokenizer' => 'standard',
                        'filter' => ['lowercase', 'bengali_stemmer']
                    ]
                ],
                'tokenizer' => [
                    'autocomplete_tokenizer' => [
                        'type' => 'edge_ngram',
                        'min_gram' => 2,
                        'max_gram' => 15,
                        'token_chars' => ['letter', 'digit']
                    ]
                ],
                'filter' => [
                    'bengali_stemmer' => [
                        'type' => 'stemmer',
                        'language' => 'bengali'
                    ]
                ]
            ]
        ],
        'mappings' => [
            'properties' => [
                'name'        => ['type' => 'text', 'analyzer' => 'standard',
                                  'fields' => ['autocomplete' => ['type' => 'text',
                                  'analyzer' => 'autocomplete_analyzer']]],
                'name_bn'     => ['type' => 'text', 'analyzer' => 'bangla_analyzer'],
                'description' => ['type' => 'text'],
                'category'    => ['type' => 'keyword'],
                'brand'       => ['type' => 'keyword'],
                'price'       => ['type' => 'float'],
                'rating'      => ['type' => 'float'],
                'in_stock'    => ['type' => 'boolean'],
                'tags'        => ['type' => 'keyword'],
                'created_at'  => ['type' => 'date'],
            ]
        ]
    ]
]);
echo "✅ Index তৈরি সফল!\n";

// --- ডকুমেন্ট যোগ করা (Bulk) ---
$products = [
    ['name' => 'iPhone 15 Pro Max 256GB', 'name_bn' => 'আইফোন ১৫ প্রো ম্যাক্স', 'category' => 'phones', 'brand' => 'Apple', 'price' => 189999, 'rating' => 4.8, 'in_stock' => true, 'tags' => ['5g', 'flagship']],
    ['name' => 'Samsung Galaxy S24 Ultra', 'name_bn' => 'স্যামসাং গ্যালাক্সি এস২৪ আল্ট্রা', 'category' => 'phones', 'brand' => 'Samsung', 'price' => 159999, 'rating' => 4.7, 'in_stock' => true, 'tags' => ['5g', 'flagship', 'ai']],
    ['name' => 'Xiaomi 14 Pro', 'name_bn' => 'শাওমি ১৪ প্রো', 'category' => 'phones', 'brand' => 'Xiaomi', 'price' => 69999, 'rating' => 4.5, 'in_stock' => true, 'tags' => ['5g', 'value']],
    ['name' => 'AirPods Pro 2nd Gen', 'name_bn' => 'এয়ারপডস প্রো', 'category' => 'accessories', 'brand' => 'Apple', 'price' => 35999, 'rating' => 4.9, 'in_stock' => false, 'tags' => ['wireless', 'anc']],
];

$params = ['body' => []];
foreach ($products as $i => $product) {
    $params['body'][] = ['index' => ['_index' => 'products', '_id' => $i + 1]];
    $params['body'][] = array_merge($product, ['created_at' => date('Y-m-d')]);
}
$client->bulk($params);
echo "✅ " . count($products) . " টি প্রোডাক্ট যোগ হয়েছে!\n";

// --- সার্চ ---

// ১. সিম্পল সার্চ
$result = $client->search([
    'index' => 'products',
    'body' => [
        'query' => [
            'multi_match' => [
                'query' => 'iPhone Pro',
                'fields' => ['name^3', 'name_bn^2', 'description'],
                'fuzziness' => 'AUTO'
            ]
        ]
    ]
]);

echo "\n🔍 'iPhone Pro' সার্চ ফলাফল:\n";
foreach ($result['hits']['hits'] as $hit) {
    echo "  [{$hit['_score']}] {$hit['_source']['name']} - ৳{$hit['_source']['price']}\n";
}

// ২. ফিল্টারসহ সার্চ (Daraz-style)
function searchProducts(string $query, array $filters = [], int $page = 1, int $size = 20): array
{
    global $client;
    
    $must = [
        ['multi_match' => [
            'query' => $query,
            'fields' => ['name^3', 'name_bn^2', 'description', 'tags'],
            'fuzziness' => 'AUTO'
        ]]
    ];
    
    $filter = [['term' => ['in_stock' => true]]];
    
    if (!empty($filters['category'])) {
        $filter[] = ['term' => ['category' => $filters['category']]];
    }
    if (!empty($filters['brand'])) {
        $filter[] = ['terms' => ['brand' => $filters['brand']]];
    }
    if (!empty($filters['price_min']) || !empty($filters['price_max'])) {
        $range = [];
        if (!empty($filters['price_min'])) $range['gte'] = $filters['price_min'];
        if (!empty($filters['price_max'])) $range['lte'] = $filters['price_max'];
        $filter[] = ['range' => ['price' => $range]];
    }
    if (!empty($filters['min_rating'])) {
        $filter[] = ['range' => ['rating' => ['gte' => $filters['min_rating']]]];
    }
    
    return $client->search([
        'index' => 'products',
        'body' => [
            'query' => [
                'bool' => [
                    'must' => $must,
                    'filter' => $filter
                ]
            ],
            'sort' => [
                '_score' => 'desc',
                'rating' => 'desc'
            ],
            'from' => ($page - 1) * $size,
            'size' => $size,
            'highlight' => [
                'fields' => ['name' => new \stdClass(), 'description' => new \stdClass()]
            ],
            'aggs' => [
                'brands' => ['terms' => ['field' => 'brand', 'size' => 10]],
                'categories' => ['terms' => ['field' => 'category', 'size' => 10]],
                'price_stats' => ['stats' => ['field' => 'price']],
                'avg_rating' => ['avg' => ['field' => 'rating']]
            ]
        ]
    ]);
}

// ব্যবহার
$results = searchProducts('phone', [
    'category' => 'phones',
    'price_min' => 50000,
    'price_max' => 200000,
    'min_rating' => 4.0
]);

echo "\n📱 ফিল্টারসহ সার্চ:\n";
echo "মোট: {$results['hits']['total']['value']} প্রোডাক্ট\n";
foreach ($results['hits']['hits'] as $hit) {
    echo "  {$hit['_source']['name']} - ৳{$hit['_source']['price']} ⭐{$hit['_source']['rating']}\n";
}
echo "\nBrands:\n";
foreach ($results['aggregations']['brands']['buckets'] as $bucket) {
    echo "  {$bucket['key']}: {$bucket['doc_count']}\n";
}
```

### JavaScript (Elasticsearch Client)

```javascript
// npm install @elastic/elasticsearch

// =============================================
// Elasticsearch JavaScript Client
// বাস্তব উদাহরণ: Daraz সার্চ API
// =============================================

const { Client } = require('@elastic/elasticsearch');

const client = new Client({ node: 'http://localhost:9200' });

// --- সার্চ API ---
class ProductSearchService {
    
    async search(query, filters = {}, page = 1, size = 20) {
        const must = [];
        const filter = [];
        const should = [];
        
        // টেক্সট সার্চ
        if (query) {
            must.push({
                multi_match: {
                    query,
                    fields: ['name^3', 'name_bn^2', 'description', 'tags^1.5'],
                    type: 'best_fields',
                    fuzziness: 'AUTO',
                    prefix_length: 2
                }
            });
        }
        
        // ফিল্টার
        filter.push({ term: { in_stock: true } });
        
        if (filters.category) {
            filter.push({ term: { category: filters.category } });
        }
        if (filters.brands && filters.brands.length) {
            filter.push({ terms: { brand: filters.brands } });
        }
        if (filters.priceMin || filters.priceMax) {
            const range = {};
            if (filters.priceMin) range.gte = filters.priceMin;
            if (filters.priceMax) range.lte = filters.priceMax;
            filter.push({ range: { price: range } });
        }
        
        // Boosting (জনপ্রিয় ও উচ্চ রেটিং প্রোডাক্ট আগে)
        should.push(
            { range: { rating: { gte: 4.5, boost: 2 } } },
            { range: { reviews_count: { gte: 100, boost: 1.5 } } }
        );
        
        const { body } = await client.search({
            index: 'products',
            body: {
                query: {
                    bool: { must, filter, should }
                },
                sort: [
                    { _score: 'desc' },
                    { rating: 'desc' }
                ],
                from: (page - 1) * size,
                size,
                highlight: {
                    pre_tags: ['<mark>'],
                    post_tags: ['</mark>'],
                    fields: {
                        name: {},
                        description: { fragment_size: 150 }
                    }
                },
                aggs: {
                    categories: {
                        terms: { field: 'category', size: 20 }
                    },
                    brands: {
                        terms: { field: 'brand', size: 20 }
                    },
                    price_ranges: {
                        range: {
                            field: 'price',
                            ranges: [
                                { key: 'budget', to: 10000 },
                                { key: 'mid', from: 10000, to: 50000 },
                                { key: 'premium', from: 50000, to: 100000 },
                                { key: 'flagship', from: 100000 }
                            ]
                        }
                    },
                    avg_price: { avg: { field: 'price' } },
                    avg_rating: { avg: { field: 'rating' } }
                }
            }
        });
        
        return {
            total: body.hits.total.value,
            products: body.hits.hits.map(hit => ({
                id: hit._id,
                score: hit._score,
                ...hit._source,
                highlight: hit.highlight || {}
            })),
            facets: {
                categories: body.aggregations.categories.buckets,
                brands: body.aggregations.brands.buckets,
                priceRanges: body.aggregations.price_ranges.buckets,
                avgPrice: body.aggregations.avg_price.value,
                avgRating: body.aggregations.avg_rating.value
            }
        };
    }
    
    // Autocomplete
    async autocomplete(prefix) {
        const { body } = await client.search({
            index: 'products',
            body: {
                size: 5,
                query: {
                    match_phrase_prefix: {
                        'name.autocomplete': {
                            query: prefix,
                            max_expansions: 10
                        }
                    }
                },
                _source: ['name', 'category', 'price']
            }
        });
        
        return body.hits.hits.map(hit => ({
            name: hit._source.name,
            category: hit._source.category,
            price: hit._source.price
        }));
    }
    
    // "Did you mean?" (স্পেলিং সাজেশন)
    async suggest(text) {
        const { body } = await client.search({
            index: 'products',
            body: {
                suggest: {
                    'spelling-suggestion': {
                        text,
                        term: {
                            field: 'name',
                            suggest_mode: 'popular',
                            sort: 'frequency'
                        }
                    },
                    'phrase-suggestion': {
                        text,
                        phrase: {
                            field: 'name',
                            size: 3,
                            gram_size: 3,
                            highlight: {
                                pre_tag: '<em>',
                                post_tag: '</em>'
                            }
                        }
                    }
                }
            }
        });
        
        return body.suggest;
    }
}

// --- ব্যবহার ---
const searchService = new ProductSearchService();

(async () => {
    // সার্চ
    const results = await searchService.search('iPhone 15', {
        category: 'phones',
        priceMin: 100000,
        brands: ['Apple', 'Samsung']
    });
    
    console.log(`🔍 মোট ${results.total} প্রোডাক্ট পাওয়া গেছে:\n`);
    results.products.forEach(p => {
        console.log(`  [${p.score.toFixed(2)}] ${p.name} - ৳${p.price} ⭐${p.rating}`);
    });
    
    console.log('\n📊 ফেসেট:');
    console.log('  ক্যাটাগরি:', results.facets.categories.map(c => `${c.key}(${c.doc_count})`));
    console.log('  ব্র্যান্ড:', results.facets.brands.map(b => `${b.key}(${b.doc_count})`));
    console.log(`  গড় দাম: ৳${results.facets.avgPrice}`);
    
    // Autocomplete
    const suggestions = await searchService.autocomplete('iph');
    console.log('\n⌨️ Autocomplete "iph":', suggestions.map(s => s.name));
    
    // Did you mean?
    const spellCheck = await searchService.suggest('iphoen');
    console.log('\n❓ Did you mean:', spellCheck);
})();
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: সব ডেটা Elasticsearch এ রাখা (Primary DB হিসেবে)
   → ES ক্র্যাশ হলে ডেটা হারাবেন!
   → ES কে search engine হিসেবে ব্যবহার করুন, DB নয়

✅ সঠিক: RDBMS (source of truth) + ES (search layer)
   MySQL/PG → (Sync via CDC/Queue) → Elasticsearch

──────────────────────────────────────────────

❌ ভুল: mapping ছাড়া ডেটা ইনডেক্স করা (dynamic mapping)
   → ভুল data type, সার্চ কাজ করবে না

✅ সঠিক: সবসময় explicit mapping তৈরি করুন

──────────────────────────────────────────────

❌ ভুল: filter এর বদলে must ব্যবহার করা
   { "must": [{ "term": { "category": "phones" } }] }
   → score calculation হয়, ধীর

✅ সঠিক: exact match এ filter ব্যবহার করুন
   { "filter": [{ "term": { "category": "phones" } }] }
   → score calculation নেই, দ্রুত + cacheable

──────────────────────────────────────────────

❌ ভুল: text field এ keyword query চালানো (বা উল্টো)
   { "term": { "name": "iPhone 15" } }  → match হবে না!
   (কারণ name text হিসেবে tokenized)

✅ সঠিক: text → match, keyword → term
   { "match": { "name": "iPhone 15" } }          → text field
   { "term":  { "name.keyword": "iPhone 15" } }  → keyword field
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### Elasticsearch ব্যবহার করুন:

```
✅ প্রোডাক্ট/কনটেন্ট সার্চ (Daraz, Prothom Alo)
✅ লগ অ্যানালাইসিস (ELK Stack)
✅ Autocomplete / Search-as-you-type
✅ Geo-spatial সার্চ ("আমার কাছে")
✅ Analytics ড্যাশবোর্ড (Kibana)
✅ Security Information (SIEM)
✅ Application Performance Monitoring
```

### Elasticsearch ব্যবহার করবেন না:

```
❌ Primary Database হিসেবে → RDBMS ব্যবহার করুন
❌ ট্রানজ্যাকশনাল ডেটা → RDBMS (ACID)
❌ রিলেশনাল ডেটা (JOIN heavy) → RDBMS
❌ সিম্পল key-value lookup → Redis
❌ Graph relationships → Neo4j
❌ Time-series মেট্রিক্স → InfluxDB/Prometheus
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│          ফুল-টেক্সট সার্চ সারসংক্ষেপ                    │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Elasticsearch = সার্চের রাজা 👑                       │
│                                                       │
│  মূল কনসেপ্ট:                                          │
│  - Inverted Index (শব্দ → ডকুমেন্ট ম্যাপিং)            │
│  - Analyzer (Tokenizer + Filters)                    │
│  - Query DSL (match, bool, range, aggs)              │
│  - Sharding + Replication (স্কেলেবিলিটি)              │
│                                                       │
│  Architecture:                                        │
│  RDBMS → Event/Queue → Elasticsearch → API → Client  │
│  (Source of Truth)   (Search Layer)                   │
│                                                       │
│  Best Practices:                                      │
│  - Explicit mapping ব্যবহার করুন                       │
│  - filter vs must সঠিকভাবে ব্যবহার করুন               │
│  - ES কে primary DB হিসেবে নয়, search layer হিসেবে   │
│  - Monitor: cluster health, shard size, query latency│
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পূর্ববর্তী**: [গ্রাফ ডাটাবেস](./graph-database.md) | **সূচি**: [README](./README.md)
