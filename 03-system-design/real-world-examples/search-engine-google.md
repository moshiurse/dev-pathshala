# 🔍 সার্চ ইঞ্জিন সিস্টেম ডিজাইন (Google)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Google, Bing, DuckDuckGo, Elasticsearch

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বর্ণনা |
|---|---------|--------|
| 1 | Web Crawling | ইন্টারনেটের ওয়েব পেজগুলো discover ও download করা |
| 2 | Indexing | পেজগুলো parse করে searchable index তৈরি করা |
| 3 | Query Processing | ইউজারের search query বুঝে relevant results বের করা |
| 4 | Ranking | PageRank + relevance score দিয়ে results সাজানো |
| 5 | Autocomplete | টাইপ করার সময় suggestion দেখানো |
| 6 | Spell Check | ভুল বানান সংশোধন করে সঠিক results দেখানো |
| 7 | Snippet Generation | প্রতিটি result-এর জন্য relevant text snippet দেখানো |
| 8 | Image/Video Search | মাল্টিমিডিয়া কনটেন্ট সার্চ করা |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Search Latency | < 200ms (P99) |
| Indexed Pages | 100+ billion pages |
| Freshness | Breaking news < 5 min indexing |
| Availability | 99.99% uptime |
| Throughput | 100K+ queries/second |
| Relevance | Top-3 results should satisfy 70%+ queries |

### 🇧🇩 Bangladesh Context

```
বাংলাদেশে সার্চ ইঞ্জিনের বিশেষ চ্যালেঞ্জ:

১. বাংলা Unicode Support:
   - "ঢাকা" vs "dhaka" — উভয় query-তেই same results দরকার
   - বাংলা stemming: "চলছে", "চলেছিল", "চলা" → root "চল"
   - যুক্তাক্ষর handling: "বিশ্ববিদ্যালয়"

২. Local Business Search:
   - "নিকটতম ফার্মেসি" → Location-based search
   - "মিরপুর ১০ রেস্টুরেন্ট" → Bangla + English mixed query

৩. BD Web Crawling Challenges:
   - অনেক BD ওয়েবসাইট slow hosting ব্যবহার করে
   - .bd domain-এর DNS resolution সমস্যা
   - Mixed encoding (UTF-8, legacy encodings)

৪. Low Bandwidth Optimization:
   - Lite/AMP version prioritize করা
   - Image-heavy result-এ compressed thumbnail
```

---

## 📊 Back-of-envelope Estimation

### Scale Calculation

```
Total Web Pages (Global):        ~100 Billion
Average Page Size:                ~2 MB (with resources)
Average Text Content:             ~50 KB per page
Crawl Rate:                       ~1 Billion pages/day

=== Storage ===
Raw Document Store:     100B × 50KB = 5 PB (text only)
Inverted Index Size:    ~500 TB (compressed)
URL Frontier:           ~50 TB

=== Traffic ===
Search Queries/day:     ~8.5 Billion (global)
QPS (avg):              8.5B / 86400 = ~100K QPS
QPS (peak):             ~300K QPS

=== Bangladesh Specific ===
BD Internet Users:      ~130 Million
BD Search Queries/day:  ~200 Million (estimated)
BD QPS (avg):           ~2,300 QPS
BD QPS (peak):          ~7,000 QPS (news events like BPL, elections)

=== Latency Budget ===
Total Search Time:      < 200ms
├── Query Parsing:      ~10ms
├── Index Lookup:       ~50ms
├── Ranking:            ~80ms
├── Snippet Generation: ~30ms
└── Network:            ~30ms
```

### Server Estimation

```
Assume each server handles 1000 QPS:
Global: 300K / 1000 = 300 search servers (minimum)
With 3x replication = 900 servers

Index Shards:
- 100B pages / 10M pages per shard = 10,000 shards
- Each shard replicated 3x = 30,000 index server instances

Crawlers:
- 1B pages/day = ~12,000 pages/second
- Each crawler: ~10 pages/second
- Crawler instances needed: ~1,200
```

---

## 🏗️ High-Level Design

### System Architecture

```
                            ┌─────────────────┐
                            │   User Query    │
                            │ "ঢাকা রেস্টুরেন্ট" │
                            └────────┬────────┘
                                     │
                                     ▼
                        ┌────────────────────────┐
                        │    Load Balancer        │
                        │  (Regional: BD, US...)  │
                        └────────────┬───────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
            ┌──────────┐    ┌──────────┐    ┌──────────┐
            │  Query   │    │  Auto-   │    │  Spell   │
            │ Service  │    │ Complete │    │  Check   │
            └────┬─────┘    └──────────┘    └──────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Query Parser & │
        │  Tokenizer     │
        │ (Bangla/English)│
        └───────┬────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
┌──────────────┐ ┌──────────────┐
│  Inverted    │ │  Knowledge   │
│  Index       │ │  Graph       │
│  (Sharded)   │ │              │
└──────┬───────┘ └──────┬───────┘
       │                │
       └───────┬────────┘
               ▼
      ┌─────────────────┐
      │   Ranker         │
      │ (PageRank +      │
      │  ML Scoring)     │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │ Snippet Generator│
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │  Search Results  │
      │  (Top 10)        │
      └─────────────────┘
```

### Crawler Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Web Crawler System                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐     ┌──────────────┐     ┌──────────────┐   │
│   │  Seed    │────▶│ URL Frontier │────▶│   Fetcher    │   │
│   │  URLs    │     │ (Priority Q) │     │  (HTTP GET)  │   │
│   └──────────┘     └──────────────┘     └──────┬───────┘   │
│                           ▲                      │           │
│                           │                      ▼           │
│                    ┌──────┴───────┐     ┌──────────────┐   │
│                    │  URL Filter  │     │   Content    │   │
│                    │  & Dedup     │◀────│   Parser     │   │
│                    └──────────────┘     └──────┬───────┘   │
│                                                 │           │
│                                                 ▼           │
│                                         ┌──────────────┐   │
│                                         │  Document    │   │
│                                         │   Store      │   │
│                                         └──────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Politeness Policy:
- robots.txt মেনে চলা
- Same domain-এ consecutive request-এ delay রাখা
- BD sites-এর জন্য: min 2 sec delay (server capacity কম)
```

---

## 💻 Detailed Design

### 1. Web Crawler Design

#### BFS Crawling Strategy

```
BFS (Breadth-First Search) কেন?
─────────────────────────────────
- Important pages (homepage) আগে crawl হয়
- Link depth অনুযায়ী priority assign করা যায়
- "Wide" coverage পাওয়া যায় quickly

URL Frontier Priority Queue:
┌──────────────────────────────────────────┐
│  Priority 1: News sites (prothomalo.com) │
│  Priority 2: Gov sites (.gov.bd)         │
│  Priority 3: Popular BD sites            │
│  Priority 4: General pages               │
│  Priority 5: Deep/archived pages         │
└──────────────────────────────────────────┘
```

#### URL Deduplication

```
URL Fingerprinting:
─────────────────
URL → Normalize → MD5/SHA-256 Hash → Bloom Filter Check

Example:
"https://www.prothomalo.com/bangladesh" 
→ normalize: "prothomalo.com/bangladesh"
→ hash: "a3f2c8..."
→ Bloom Filter: Already crawled? Yes/No

Bloom Filter Size:
- 100B URLs, 0.1% false positive rate
- Size = -n × ln(p) / (ln2)² ≈ 144 GB
- Distributed across crawler nodes
```

### 2. Inverted Index Structure

```
Inverted Index কীভাবে কাজ করে:
═══════════════════════════════

Document Collection:
  Doc1: "ঢাকা বাংলাদেশের রাজধানী"
  Doc2: "ঢাকা বিশ্ববিদ্যালয় বাংলাদেশের সেরা"
  Doc3: "চট্টগ্রাম বাংলাদেশের বন্দর নগরী"

Inverted Index:
┌──────────────────┬────────────────────────────────┐
│  Term            │  Posting List                   │
├──────────────────┼────────────────────────────────┤
│  "ঢাকা"         │  [Doc1(pos:1), Doc2(pos:1)]     │
│  "বাংলাদেশের"   │  [Doc1(pos:2), Doc2(pos:3),    │
│                  │   Doc3(pos:2)]                  │
│  "রাজধানী"      │  [Doc1(pos:3)]                  │
│  "বিশ্ববিদ্যালয়" │  [Doc2(pos:2)]                  │
│  "চট্টগ্রাম"    │  [Doc3(pos:1)]                  │
│  "সেরা"         │  [Doc2(pos:4)]                  │
│  "বন্দর"        │  [Doc3(pos:3)]                  │
└──────────────────┴────────────────────────────────┘

Posting List Entry Structure:
┌─────────┬──────────┬──────────┬───────────┐
│ Doc ID  │ Position │ Frequency│ Field     │
│ (uint32)│ (uint16) │ (uint8)  │ (title/   │
│         │          │          │  body)    │
└─────────┴──────────┴──────────┴───────────┘
```

### 3. PageRank Algorithm (Simplified)

```
PageRank Formula:
PR(A) = (1-d) + d × Σ(PR(Ti) / C(Ti))

Where:
- d = damping factor (0.85)
- Ti = pages that link to A
- C(Ti) = outgoing links count of Ti

Example:
┌─────┐     ┌─────┐     ┌─────┐
│  A  │────▶│  B  │────▶│  C  │
│PR=1 │     │PR=? │     │PR=? │
└──┬──┘     └─────┘     └──┬──┘
   │                        │
   └────────────────────────┘
         (C links to A)

Iteration 1:
PR(B) = 0.15 + 0.85 × (1/1) = 1.0
PR(C) = 0.15 + 0.85 × (1/1) = 1.0
PR(A) = 0.15 + 0.85 × (1/1) = 1.0

(Iterates until convergence)
```

### 4. Query Processing Pipeline

```
User Query: "সেরা ঢাকা রেস্টুরেন্ট 2024"
                    │
                    ▼
┌─────────────────────────────────────┐
│ Step 1: Language Detection           │
│ → Bangla (bn) detected              │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Step 2: Tokenization                 │
│ → ["সেরা", "ঢাকা", "রেস্টুরেন্ট",  │
│    "2024"]                           │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Step 3: Query Expansion              │
│ → "সেরা" + "best"                   │
│ → "রেস্টুরেন্ট" + "restaurant"      │
│ → Synonyms added                     │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Step 4: Intent Classification        │
│ → Local Search (location-based)      │
│ → Add geo-filter: Dhaka, BD         │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Step 5: Index Lookup                 │
│ → Fetch posting lists for each term  │
│ → Intersect/Union based on operator  │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Step 6: Scoring & Ranking            │
│ → TF-IDF + PageRank + Freshness     │
│ → ML Re-ranking (BERT-based)        │
└─────────────────────────────────────┘
```

### 5. PHP (Laravel) - Search API with Elasticsearch

```php
<?php
// app/Services/SearchService.php

namespace App\Services;

use Elasticsearch\ClientBuilder;
use App\DTOs\SearchResult;
use App\Services\SpellChecker;
use App\Services\QueryParser;

class SearchService
{
    private $elasticsearch;
    private SpellChecker $spellChecker;
    private QueryParser $queryParser;

    public function __construct()
    {
        $this->elasticsearch = ClientBuilder::create()
            ->setHosts(config('services.elasticsearch.hosts'))
            ->build();

        $this->spellChecker = new SpellChecker();
        $this->queryParser = new QueryParser();
    }

    /**
     * মূল সার্চ method - query নিয়ে ranked results return করে
     */
    public function search(string $query, array $options = []): array
    {
        $startTime = microtime(true);

        // Step 1: Query parsing ও language detection
        $parsedQuery = $this->queryParser->parse($query);
        $language = $this->detectLanguage($query);

        // Step 2: Spell check
        $correctedQuery = $this->spellChecker->correct($query, $language);
        $didYouMean = ($correctedQuery !== $query) ? $correctedQuery : null;

        // Step 3: Elasticsearch query build
        $searchParams = [
            'index' => 'web_pages',
            'body' => [
                'size' => $options['limit'] ?? 10,
                'from' => $options['offset'] ?? 0,
                'query' => $this->buildElasticQuery($parsedQuery, $language),
                'highlight' => [
                    'fields' => [
                        'content' => ['fragment_size' => 150, 'number_of_fragments' => 3],
                        'title' => new \stdClass(),
                    ],
                ],
                'aggs' => [
                    'domains' => ['terms' => ['field' => 'domain', 'size' => 5]],
                ],
            ],
        ];

        // Bangladesh context: location filter
        if ($parsedQuery['is_local_search'] && isset($options['location'])) {
            $searchParams['body']['query']['bool']['filter'][] = [
                'geo_distance' => [
                    'distance' => '50km',
                    'location' => $options['location'], // e.g., Dhaka coords
                ],
            ];
        }

        // Step 4: Execute search
        $response = $this->elasticsearch->search($searchParams);

        // Step 5: Format results
        $results = $this->formatResults($response);
        $latency = (microtime(true) - $startTime) * 1000;

        return [
            'results' => $results,
            'total' => $response['hits']['total']['value'],
            'latency_ms' => round($latency, 2),
            'did_you_mean' => $didYouMean,
            'language' => $language,
        ];
    }

    /**
     * Elasticsearch query তৈরি — multi_match + boosting
     */
    private function buildElasticQuery(array $parsedQuery, string $language): array
    {
        $analyzer = $language === 'bn' ? 'bangla_analyzer' : 'standard';

        return [
            'bool' => [
                'must' => [
                    'multi_match' => [
                        'query' => $parsedQuery['cleaned_query'],
                        'fields' => ['title^3', 'content^1', 'meta_description^2', 'url^0.5'],
                        'type' => 'best_fields',
                        'analyzer' => $analyzer,
                    ],
                ],
                'should' => [
                    ['term' => ['is_https' => ['value' => true, 'boost' => 1.1]]],
                    ['term' => ['is_mobile_friendly' => ['value' => true, 'boost' => 1.2]]],
                    ['range' => ['page_rank' => ['gte' => 5, 'boost' => 2.0]]],
                    // BD sites boost for Bangla queries
                    ['term' => ['country' => ['value' => 'BD', 'boost' => 1.5]]],
                ],
            ],
        ];
    }

    /**
     * ভাষা detect করা — Bangla Unicode range check
     */
    private function detectLanguage(string $text): string
    {
        // Bangla Unicode range: U+0980 to U+09FF
        if (preg_match('/[\x{0980}-\x{09FF}]/u', $text)) {
            return 'bn';
        }
        return 'en';
    }

    private function formatResults(array $response): array
    {
        return array_map(function ($hit) {
            return new SearchResult(
                title: $hit['_source']['title'],
                url: $hit['_source']['url'],
                snippet: $hit['highlight']['content'][0] ?? $hit['_source']['meta_description'],
                score: $hit['_score'],
                domain: $hit['_source']['domain'],
                cached_at: $hit['_source']['crawled_at'],
            );
        }, $response['hits']['hits']);
    }
}
```

```php
<?php
// app/Http/Controllers/SearchController.php

namespace App\Http\Controllers;

use App\Services\SearchService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\RateLimiter;

class SearchController extends Controller
{
    public function __construct(private SearchService $searchService) {}

    public function search(Request $request)
    {
        $request->validate([
            'q' => 'required|string|max:500',
            'page' => 'integer|min:1|max:100',
        ]);

        $query = $request->input('q');
        $page = $request->input('page', 1);
        $cacheKey = "search:" . md5($query . $page);

        // Popular queries cache (5 min TTL)
        $results = Cache::remember($cacheKey, 300, function () use ($query, $page) {
            return $this->searchService->search($query, [
                'limit' => 10,
                'offset' => ($page - 1) * 10,
                'location' => ['lat' => 23.8103, 'lon' => 90.4125], // Dhaka default
            ]);
        });

        return response()->json($results);
    }
}
```

### 6. Node.js (Express) - Autocomplete Service

```javascript
// services/autocomplete/src/server.js

const express = require('express');
const Redis = require('ioredis');
const compression = require('compression');

const app = express();
app.use(compression());

const redis = new Redis({
    host: process.env.REDIS_HOST || 'localhost',
    port: 6379,
    db: 0,
});

/**
 * Trie-based Autocomplete with Redis Sorted Sets
 * 
 * Redis Structure:
 * - Key: "autocomplete:{prefix}"
 * - Members: query suggestions
 * - Score: query frequency/popularity
 * 
 * Example:
 * autocomplete:ঢা → ["ঢাকা", "ঢাকা বিশ্ববিদ্যালয়", "ঢাকা রেস্টুরেন্ট"]
 */

// Autocomplete endpoint - P99 latency < 50ms required
app.get('/api/autocomplete', async (req, res) => {
    const { q: prefix, lang = 'auto', limit = 7 } = req.query;

    if (!prefix || prefix.length < 1) {
        return res.json({ suggestions: [] });
    }

    try {
        const startTime = Date.now();
        const language = detectLanguage(prefix);
        const normalizedPrefix = normalizeQuery(prefix, language);

        // Strategy 1: Exact prefix match from Redis sorted set
        const cacheKey = `autocomplete:${language}:${normalizedPrefix}`;
        let suggestions = await redis.zrevrange(cacheKey, 0, limit - 1);

        // Strategy 2: Fuzzy match fallback if no exact results
        if (suggestions.length === 0) {
            suggestions = await fuzzyAutocomplete(normalizedPrefix, language, limit);
        }

        // Strategy 3: Trending queries boost
        const trending = await getTrendingSuggestions(normalizedPrefix, language);
        suggestions = mergeSuggestions(suggestions, trending, limit);

        const latency = Date.now() - startTime;

        res.json({
            suggestions: suggestions.map(s => ({
                text: s,
                type: classifySuggestion(s),
            })),
            latency_ms: latency,
            language,
        });
    } catch (error) {
        console.error('Autocomplete error:', error);
        res.status(500).json({ suggestions: [], error: 'Internal error' });
    }
});

/**
 * Bangla/English ভাষা detect করা
 */
function detectLanguage(text) {
    const banglaRegex = /[\u0980-\u09FF]/;
    return banglaRegex.test(text) ? 'bn' : 'en';
}

/**
 * Query normalize — lowercase, trim, Bangla-specific normalization
 */
function normalizeQuery(query, language) {
    let normalized = query.trim().toLowerCase();

    if (language === 'bn') {
        // Bangla-specific: হসন্ত, চন্দ্রবিন্দু normalization
        normalized = normalized.replace(/\u09CD$/g, ''); // trailing hasanta remove
        normalized = normalized.normalize('NFC'); // Unicode NFC normalization
    }

    return normalized;
}

/**
 * Fuzzy autocomplete — edit distance based matching
 * BD context: "resturent" → "restaurant", "ড্যাকা" → "ঢাকা"
 */
async function fuzzyAutocomplete(prefix, language, limit) {
    // Use Redis SCAN with pattern matching as fallback
    const pattern = `autocomplete:${language}:${prefix}*`;
    let cursor = '0';
    const results = [];

    do {
        const [newCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
        cursor = newCursor;

        for (const key of keys.slice(0, limit)) {
            const topSuggestion = await redis.zrevrange(key, 0, 0);
            if (topSuggestion.length) results.push(topSuggestion[0]);
        }
    } while (cursor !== '0' && results.length < limit);

    return results.slice(0, limit);
}

/**
 * Trending suggestions — Bangladesh-এ এখন কী hot
 */
async function getTrendingSuggestions(prefix, language) {
    const trendingKey = `trending:${language}:${new Date().toISOString().slice(0, 13)}`;
    const trending = await redis.zrevrange(trendingKey, 0, 20);
    return trending.filter(t => t.startsWith(prefix));
}

function mergeSuggestions(primary, secondary, limit) {
    const seen = new Set(primary);
    const merged = [...primary];

    for (const item of secondary) {
        if (!seen.has(item) && merged.length < limit) {
            merged.push(item);
            seen.add(item);
        }
    }
    return merged.slice(0, limit);
}

function classifySuggestion(text) {
    if (text.includes('কোথায়') || text.includes('near')) return 'local';
    if (/\d{4}/.test(text)) return 'temporal';
    return 'general';
}

// Suggestion indexing endpoint (called by analytics pipeline)
app.post('/api/autocomplete/index', express.json(), async (req, res) => {
    const { query, frequency, language } = req.body;

    if (!query || !frequency) {
        return res.status(400).json({ error: 'Missing query or frequency' });
    }

    const lang = language || detectLanguage(query);
    const normalized = normalizeQuery(query, lang);

    // Index all prefixes of the query
    const pipeline = redis.pipeline();
    for (let i = 1; i <= normalized.length; i++) {
        const prefix = normalized.substring(0, i);
        pipeline.zadd(`autocomplete:${lang}:${prefix}`, frequency, query);
        // Keep only top 10 per prefix to save memory
        pipeline.zremrangebyrank(`autocomplete:${lang}:${prefix}`, 0, -11);
    }

    await pipeline.exec();
    res.json({ status: 'indexed', prefixes_count: normalized.length });
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
    console.log(`Autocomplete service running on port ${PORT}`);
});
```

### 7. Spell Correction Service (Node.js)

```javascript
// services/spellcheck/src/spellChecker.js

/**
 * Spell Correction — Bangla ও English উভয় ভাষার জন্য
 * 
 * Approach:
 * 1. Edit Distance (Levenshtein) — "googl" → "google"
 * 2. Phonetic Matching — "ড্যাকা" → "ঢাকা"
 * 3. Statistical (Noisy Channel Model) — P(correction|misspelling)
 */

class BanglaSpellChecker {
    constructor(dictionary, bigramModel) {
        this.dictionary = dictionary; // Set of valid Bangla words
        this.bigramModel = bigramModel; // Word pair frequencies
        this.maxEditDistance = 2;
    }

    /**
     * মূল correction method
     * "বাংলাদেস" → "বাংলাদেশ"
     * "resturent" → "restaurant"
     */
    correct(query) {
        const words = query.split(/\s+/);
        const corrections = words.map(word => {
            if (this.dictionary.has(word)) return word;

            const candidates = this.generateCandidates(word);
            if (candidates.length === 0) return word;

            // Rank by: P(word) × P(typo|word)
            return candidates.sort((a, b) => {
                const scoreA = this.scoreCandiate(a, word);
                const scoreB = this.scoreCandiate(b, word);
                return scoreB - scoreA;
            })[0];
        });

        return corrections.join(' ');
    }

    /**
     * Edit distance 1-2 এর মধ্যে সব candidates generate
     */
    generateCandidates(word) {
        const edits1 = this.edits1(word);
        let candidates = [...edits1].filter(w => this.dictionary.has(w));

        if (candidates.length === 0) {
            // Edit distance 2
            for (const e1 of edits1) {
                for (const e2 of this.edits1(e1)) {
                    if (this.dictionary.has(e2)) candidates.push(e2);
                }
            }
        }
        return [...new Set(candidates)];
    }

    edits1(word) {
        const chars = [...word]; // Proper Unicode split for Bangla
        const results = new Set();

        // Deletes
        for (let i = 0; i < chars.length; i++) {
            results.add([...chars.slice(0, i), ...chars.slice(i + 1)].join(''));
        }
        // Transposes
        for (let i = 0; i < chars.length - 1; i++) {
            const swapped = [...chars];
            [swapped[i], swapped[i + 1]] = [swapped[i + 1], swapped[i]];
            results.add(swapped.join(''));
        }
        return results;
    }

    scoreCandiate(candidate, original) {
        const editDist = this.levenshteinDistance(candidate, original);
        const frequency = this.dictionary.getFrequency?.(candidate) || 1;
        return frequency / (editDist + 1);
    }

    levenshteinDistance(a, b) {
        const m = [...a].length, n = [...b].length;
        const dp = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));

        for (let i = 0; i <= m; i++) dp[i][0] = i;
        for (let j = 0; j <= n; j++) dp[0][j] = j;

        const aChars = [...a], bChars = [...b];
        for (let i = 1; i <= m; i++) {
            for (let j = 1; j <= n; j++) {
                dp[i][j] = aChars[i-1] === bChars[j-1]
                    ? dp[i-1][j-1]
                    : 1 + Math.min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]);
            }
        }
        return dp[m][n];
    }
}

module.exports = { BanglaSpellChecker };
```

### 8. Database Design

```
=== Document Store (Distributed — HBase/Cassandra) ===

┌───────────────────────────────────────────────────────────┐
│  Table: web_documents                                      │
├───────────────┬──────────────┬────────────────────────────┤
│  Column       │  Type        │  Description               │
├───────────────┼──────────────┼────────────────────────────┤
│  doc_id       │  UUID        │  Unique document ID        │
│  url          │  TEXT        │  Full URL                  │
│  domain       │  TEXT        │  Domain name               │
│  title        │  TEXT        │  Page title                │
│  content      │  TEXT        │  Raw text content          │
│  html_hash    │  CHAR(64)   │  Content fingerprint       │
│  language     │  CHAR(2)    │  "bn", "en", etc.          │
│  page_rank    │  FLOAT      │  PageRank score            │
│  crawled_at   │  TIMESTAMP  │  Last crawl time           │
│  country      │  CHAR(2)    │  Server country (BD, US)   │
│  encoding     │  VARCHAR(20)│  UTF-8, etc.               │
└───────────────┴──────────────┴────────────────────────────┘

=== URL Frontier (Redis + Kafka) ===

┌───────────────────────────────────────────────────────────┐
│  Priority Queue Structure:                                 │
│                                                            │
│  Queue-P1 (High):   [prothomalo.com/*, bdnews24.com/*]   │
│  Queue-P2 (Medium): [*.edu.bd/*, *.gov.bd/*]             │
│  Queue-P3 (Normal): [general URLs...]                     │
│  Queue-P4 (Low):    [deep pages, archives]               │
│                                                            │
│  Per-Domain Bucket:                                        │
│  ┌─────────────────────────────────────┐                  │
│  │ domain: prothomalo.com              │                  │
│  │ last_crawled: 2024-01-15T10:30:00Z  │                  │
│  │ crawl_delay: 2s                     │                  │
│  │ pending_urls: 15,234                │                  │
│  └─────────────────────────────────────┘                  │
└───────────────────────────────────────────────────────────┘

=== Inverted Index (Custom Sharded Store) ===

Shard Strategy: term_hash % num_shards
Each shard: ~10M documents

┌────────────────────────────────────────────────────────┐
│  Shard #42                                              │
│  ┌────────────┬─────────────────────────────────────┐  │
│  │ Term       │ Posting List (compressed)            │  │
│  ├────────────┼─────────────────────────────────────┤  │
│  │ "ঢাকা"    │ [d1:3:t, d5:1:b, d89:2:t, ...]     │  │
│  │ "cricket"  │ [d2:5:b, d12:1:t, d45:3:b, ...]    │  │
│  │ "বাংলাদেশ" │ [d1:2:t, d3:4:b, d7:1:t, ...]     │  │
│  └────────────┴─────────────────────────────────────┘  │
│                                                         │
│  Format: [doc_id:frequency:field(t=title,b=body)]      │
└────────────────────────────────────────────────────────┘
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. BFS vs DFS Crawling

```
┌─────────────────────────────────────────────────────────────────┐
│  Criteria         │  BFS                  │  DFS                 │
├───────────────────┼───────────────────────┼──────────────────────┤
│  Coverage         │  ✅ Broad, balanced    │  ❌ Deep, narrow     │
│  Important Pages  │  ✅ Found early       │  ❌ May miss         │
│  Memory Usage     │  ❌ High (wide queue) │  ✅ Low (stack)      │
│  URL Priority     │  ✅ Easy to implement │  ❌ Hard             │
│  Freshness        │  ✅ Frequent re-crawl │  ❌ Stale pages      │
│  BD Context       │  ✅ Top BD sites fast │  ❌ Stuck on one site│
├───────────────────┼───────────────────────┼──────────────────────┤
│  সিদ্ধান্ত        │  ✅ BFS preferred     │  DFS for deep archive│
└───────────────────┴───────────────────────┴──────────────────────┘

Google's Approach: Modified BFS with priority scoring
- High PageRank pages: crawled every few hours
- News sites (prothomalo.com): crawled every few minutes
- Low-value pages: crawled monthly
```

### 2. Push vs Pull Indexing

```
┌─────────────────────────────────────────────────────────────────┐
│  Approach    │  Push (Real-time)         │  Pull (Batch)         │
├──────────────┼───────────────────────────┼───────────────────────┤
│  Freshness   │  ✅ Seconds               │  ❌ Hours/Days        │
│  Throughput  │  ❌ Limited by indexer     │  ✅ High batch perf   │
│  Consistency │  ❌ Eventual              │  ✅ Point-in-time     │
│  Complexity  │  ❌ Stream processing     │  ✅ Simple MapReduce  │
│  Cost        │  ❌ Always-on infra       │  ✅ Periodic jobs     │
├──────────────┼───────────────────────────┼───────────────────────┤
│  Best For    │  News, breaking events    │  General web pages    │
└──────────────┴───────────────────────────┴───────────────────────┘

Hybrid Approach (Google uses):
- Breaking news: Push indexing (< 5 min)
- Popular pages: Pull every 4-8 hours
- Long-tail pages: Pull weekly/monthly
```

### 3. Inverted Index: HashMap vs B-Tree

```
┌─────────────────────────────────────────────────────────────────┐
│  Feature     │  HashMap (In-Memory)  │  B-Tree (Disk-based)     │
├──────────────┼───────────────────────┼──────────────────────────┤
│  Lookup Time │  ✅ O(1) average      │  ⚠️ O(log n)            │
│  Range Query │  ❌ Not supported     │  ✅ Efficient            │
│  Memory      │  ❌ All in RAM        │  ✅ Disk + cache         │
│  Scale       │  ❌ Limited by RAM    │  ✅ Scales to PB         │
│  Prefix      │  ❌ No prefix search  │  ✅ Prefix scan          │
├──────────────┼───────────────────────┼──────────────────────────┤
│  সিদ্ধান্ত   │  Hot terms (cache)    │  Full index on disk      │
└──────────────┴───────────────────────┴──────────────────────────┘

Practical Design:
- L1 (Memory): Top 1M terms in HashMap → 90% of queries
- L2 (SSD): Full index in B-Tree/SSTable → remaining 10%
- Compression: Variable-byte encoding on posting lists
```

### 4. Relevance vs Freshness

```
Scoring Formula:
final_score = α × relevance_score + β × freshness_score + γ × pagerank

Where α + β + γ = 1

┌──────────────────────────────────────────────────────────────┐
│  Query Type          │  α (Relevance) │ β (Freshness) │ γ(PR)│
├──────────────────────┼────────────────┼───────────────┼──────┤
│  Navigational        │  0.3           │ 0.1           │ 0.6  │
│  ("facebook login")  │                │               │      │
├──────────────────────┼────────────────┼───────────────┼──────┤
│  Informational       │  0.6           │ 0.2           │ 0.2  │
│  ("ঢাকা ইতিহাস")    │                │               │      │
├──────────────────────┼────────────────┼───────────────┼──────┤
│  News/Temporal       │  0.3           │ 0.6           │ 0.1  │
│  ("BPL score today") │                │               │      │
├──────────────────────┼────────────────┼───────────────┼──────┤
│  Local               │  0.4           │ 0.3           │ 0.3  │
│  ("মিরপুর হাসপাতাল") │                │               │      │
└──────────────────────┴────────────────┴───────────────┴──────┘
```

### 5. Elasticsearch vs Solr vs Custom

```
┌─────────────────────────────────────────────────────────────────────┐
│  Feature          │  Elasticsearch  │  Solr        │  Custom        │
├───────────────────┼─────────────────┼──────────────┼────────────────┤
│  Scalability      │  ✅ Auto-shard  │  ✅ SolrCloud│  ✅ Full ctrl  │
│  Real-time Index  │  ✅ Near RT     │  ⚠️ Soft cmit│  ✅ Tunable    │
│  Bangla Support   │  ⚠️ Plugin need │  ⚠️ Plugin  │  ✅ Built-in   │
│  Operations       │  ✅ Easy        │  ⚠️ Complex │  ❌ Hard        │
│  Cost at Scale    │  ❌ Expensive   │  ⚠️ Medium  │  ✅ Optimized  │
│  Learning Curve   │  ✅ Low         │  ⚠️ Medium  │  ❌ Very high   │
├───────────────────┼─────────────────┼──────────────┼────────────────┤
│  Best For         │  Small-Mid      │  Enterprise  │  Google-scale  │
│  BD Startup Use   │  ✅ Recommended │  ⚠️ OK      │  ❌ Overkill    │
└───────────────────┴─────────────────┴──────────────┴────────────────┘
```

---

## 📈 কেস স্টাডি

### Case Study 1: Indexing 100 Billion Web Pages

```
Challenge: 100B পেজ কীভাবে index করা যায়?

Solution Architecture:
═══════════════════════

Phase 1: Distributed Crawling
┌──────────────────────────────────────────────┐
│  Crawler Fleet (1200+ nodes)                  │
│                                               │
│  Region: US-East  ──→ Americas               │
│  Region: EU-West  ──→ Europe                 │
│  Region: AP-South ──→ Asia (BD included)     │
│                                               │
│  Rate: 12,000 pages/second globally          │
└──────────────────────────────────────────────┘

Phase 2: MapReduce Indexing
┌──────────────────────────────────────────────┐
│  Map Phase:                                   │
│  - Input: Raw HTML documents                 │
│  - Output: (term, doc_id, metadata) pairs    │
│                                               │
│  Shuffle Phase:                               │
│  - Group by term                             │
│  - Sort by doc_id within each term           │
│                                               │
│  Reduce Phase:                                │
│  - Build posting list for each term          │
│  - Compress with variable-byte encoding      │
│  - Write to distributed file system          │
└──────────────────────────────────────────────┘

Phase 3: Index Serving
┌──────────────────────────────────────────────┐
│  10,000 shards × 3 replicas = 30,000 nodes   │
│                                               │
│  Query routing:                               │
│  1. Hash query terms → target shards         │
│  2. Scatter query to relevant shards         │
│  3. Each shard returns top-K results         │
│  4. Gather & merge all shard results         │
│  5. Final global ranking                     │
└──────────────────────────────────────────────┘

Result:
- Full re-index: ~3 days with 10K machines
- Incremental updates: continuous
- Index size: ~500 TB compressed
```

### Case Study 2: Real-time Search During Breaking News

```
Scenario: বাংলাদেশে ভূমিকম্প — লক্ষ লক্ষ মানুষ একসাথে search করছে

Timeline:
─────────
T+0 min:  ভূমিকম্প হলো
T+1 min:  Social media-তে posts শুরু
T+2 min:  News sites publish করতে শুরু করলো
T+3 min:  Search spike: "ভূমিকম্প ঢাকা" +10000% QPS
T+5 min:  Google-এ updated results দেখাচ্ছে

How It Works:
┌─────────────────────────────────────────────────┐
│  1. Real-time Crawl Trigger:                     │
│     - Social signal detection                    │
│     - Sudden query spike alert                   │
│     - Priority crawl queue activated             │
│                                                  │
│  2. Fast Indexing Pipeline:                      │
│     - Skip batch queue → direct index            │
│     - Temporary "fresh" index segment           │
│     - Merged with main index later              │
│                                                  │
│  3. Query Handling:                              │
│     - QID (Query Intent Detection): "breaking"  │
│     - Freshness weight boosted: β = 0.8         │
│     - Cache TTL reduced: 30s → 5s               │
│     - Result diversity: multiple sources        │
│                                                  │
│  4. Capacity Scaling:                            │
│     - Auto-scale search replicas                │
│     - Edge cache activation                     │
│     - Rate limit non-essential features         │
└─────────────────────────────────────────────────┘
```

### Case Study 3: Handling Bangla Unicode Search Queries

```
Problem: বাংলা ভাষায় search করা অনেক complex

Challenge 1: Same word, multiple representations
─────────────────────────────────────────────────
"কী" vs "কি" — context-dependent meaning
"হ্যাঁ" — যুক্তাক্ষর + চন্দ্রবিন্দু
"বিশ্ববিদ্যালয়" — 5টি character cluster

Challenge 2: Mixed script queries
─────────────────────────────────
"iPhone 15 দাম বাংলাদেশ" — English + Bangla + Number

Challenge 3: Transliteration
─────────────────────────────
"dhaka" → "ঢাকা" (should return same results)
"rokomari" → "রকমারি" (book site)

Solution:
┌──────────────────────────────────────────────────┐
│  1. Unicode Normalization (NFC):                  │
│     - All text → NFC form before indexing        │
│     - Consistent storage format                   │
│                                                   │
│  2. Custom Bangla Analyzer:                       │
│     ┌─────────────────────────────────────────┐  │
│     │ Input: "বিশ্ববিদ্যালয়গুলোতে"              │  │
│     │   → Char filter: normalize joiners       │  │
│     │   → Tokenizer: split on spaces          │  │
│     │   → Stemmer: "বিশ্ববিদ্যালয়"             │  │
│     │   → Suffix removal: "গুলোতে" → root     │  │
│     └─────────────────────────────────────────┘  │
│                                                   │
│  3. Cross-language Index:                         │
│     "ঢাকা" → index under ["ঢাকা", "dhaka",     │
│               "Dhaka", "dhaaka"]                 │
│                                                   │
│  4. Query Expansion for BD:                       │
│     "ভার্সিটি" → also search "বিশ্ববিদ্যালয়",   │
│                   "university"                    │
└──────────────────────────────────────────────────┘
```

---

## 🔧 Advanced Topics

### 1. Distributed Crawling Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Distributed Crawler Coordination                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐         ┌──────────────────────┐          │
│  │  Coordinator │────────▶│  URL Assignment       │          │
│  │  (ZooKeeper) │         │  Strategy:            │          │
│  └──────────────┘         │  hash(domain) % nodes │          │
│         │                  └──────────────────────┘          │
│         │                                                     │
│    ┌────┼────┬────────────┐                                  │
│    ▼    ▼    ▼            ▼                                  │
│  ┌───┐┌───┐┌───┐      ┌───┐                                │
│  │C1 ││C2 ││C3 │ ...  │Cn │  (Crawler Nodes)               │
│  └─┬─┘└─┬─┘└─┬─┘      └─┬─┘                                │
│    │     │    │           │                                   │
│    ▼     ▼    ▼           ▼                                  │
│  ┌─────────────────────────────┐                             │
│  │      Kafka Message Queue     │                             │
│  │  (Crawled pages → Indexer)   │                             │
│  └─────────────────────────────┘                             │
│                                                               │
│  Politeness Rules:                                            │
│  - 1 domain → 1 crawler node (no parallel per domain)        │
│  - Crawl delay from robots.txt                               │
│  - BD sites: extra cautious (servers are weaker)             │
│  - Rate limit: max 1 req/2sec per .bd domain                 │
└──────────────────────────────────────────────────────────────┘
```

### 2. Knowledge Graph

```
Knowledge Graph Structure:
══════════════════════════

Entity: "বাংলাদেশ"
┌──────────────────────────────────────────────┐
│  type: Country                                │
│  capital: "ঢাকা"                              │
│  population: 170,000,000                      │
│  language: "বাংলা"                            │
│  currency: "টাকা (BDT)"                      │
│  area: "147,570 km²"                         │
│  relations:                                   │
│    - borders: ["India", "Myanmar"]           │
│    - part_of: ["South Asia"]                 │
│    - famous_for: ["Cricket", "Garments"]     │
└──────────────────────────────────────────────┘

Query: "বাংলাদেশের রাজধানী কি?"
→ Knowledge Graph lookup → Direct answer: "ঢাকা"
→ No need to show 10 blue links!

Implementation (Triple Store):
(বাংলাদেশ) --[capital_of]--> (ঢাকা)
(ঢাকা)     --[country]-----> (বাংলাদেশ)
(বাংলাদেশ) --[language]----> (বাংলা)
```

### 3. Personalized Search

```
Personalization Signals:
────────────────────────
1. Search History: আগে কী search করেছে
2. Location: Dhaka/Chittagong/Sylhet
3. Language Preference: bn/en
4. Click History: কোন results-এ click করে
5. Device: Mobile vs Desktop

Example:
Query: "cricket score"
├── User in BD → Shows BPL/Bangladesh cricket first
├── User in India → Shows IPL/India cricket first
└── User in UK → Shows County Championship first

Privacy Trade-off:
┌──────────────────────────────────────────┐
│  More Personalization ←──→ More Privacy  │
│                                           │
│  Google's Balance:                        │
│  - Anonymized signals after 18 months    │
│  - Option to pause search history        │
│  - Incognito mode = no personalization   │
└──────────────────────────────────────────┘
```

### 4. SEO Gaming Prevention

```
Spam Detection Techniques:
══════════════════════════

1. Keyword Stuffing Detection:
   - TF threshold: if term appears > 5% of content → spam signal
   - "ঢাকা ঢাকা ঢাকা রেস্টুরেন্ট ঢাকা" → flagged

2. Link Farm Detection:
   ┌───┐   ┌───┐   ┌───┐
   │ A │──▶│ B │──▶│ C │──▶ Target
   └───┘   └───┘   └───┘
     ▲                       │
     └───────────────────────┘
   All point to same site → unnatural pattern

3. Cloaking Detection:
   - Googlebot sees different content than users
   - Random user-agent crawling to verify

4. BD-Specific Spam Patterns:
   - Fake news sites with clickbait Bangla titles
   - Auto-generated Bangla content (poor grammar)
   - Copied content from prothomalo/bdnews24
   
5. Countermeasures:
   - Manual quality rater reviews
   - ML classifier trained on spam patterns
   - Sandbox new domains for 3-6 months
   - Penguin/Panda algorithm updates
```

### 5. Image Search Architecture

```
┌────────────────────────────────────────────────────────┐
│  Image Search Pipeline                                  │
├────────────────────────────────────────────────────────┤
│                                                         │
│  Crawl ──→ Download ──→ Feature Extraction ──→ Index   │
│                              │                          │
│                    ┌─────────┼─────────┐               │
│                    ▼         ▼         ▼               │
│              ┌────────┐┌────────┐┌────────┐           │
│              │ Visual ││  OCR   ││  Meta  │           │
│              │Features││ (Text) ││  Data  │           │
│              └────────┘└────────┘└────────┘           │
│                                                         │
│  Query: "সুন্দরবন ছবি"                                 │
│  1. Text match: alt text, surrounding text, filename   │
│  2. Visual similarity: CNN feature vectors             │
│  3. SafeSearch filter: NSFW classifier                 │
│  4. Freshness + quality ranking                        │
└────────────────────────────────────────────────────────┘
```

---

## 🎯 সারসংক্ষেপ

### Key Design Decisions

```
┌───────────────────────────────────────────────────────────────┐
│  Component          │  Technology Choice     │  কেন?           │
├─────────────────────┼────────────────────────┼─────────────────┤
│  Document Store     │  HBase/Cassandra       │  PB-scale data  │
│  Inverted Index     │  Custom + SSD          │  Low latency    │
│  URL Frontier       │  Kafka + Redis         │  High throughput│
│  Autocomplete       │  Redis Sorted Sets     │  < 50ms P99     │
│  Query Processing   │  Elasticsearch         │  Feature-rich   │
│  Spell Check        │  Custom + ML           │  Bangla support │
│  Ranking            │  PageRank + BERT       │  Relevance      │
│  Caching            │  CDN + Redis           │  Latency        │
│  Coordination       │  ZooKeeper             │  Consistency    │
│  Message Queue      │  Kafka                 │  Durability     │
└─────────────────────┴────────────────────────┴─────────────────┘
```

### মূল শিক্ষাসমূহ

```
১. Scale thinking:
   - সবকিছু shard করে ভাবতে হবে
   - 100B documents → 10K+ shards necessary

২. Latency budget:
   - 200ms total → প্রতিটি component-এর budget fixed
   - Caching is essential (90% queries are repeated)

৩. Freshness vs Cost:
   - সব পেজ real-time crawl করা অসম্ভব
   - Priority-based crawling is the answer

৪. Bangladesh-specific:
   - বাংলা NLP pipeline অনেক complex
   - Unicode normalization mandatory
   - Cross-script search (Bangla + English) essential

৫. Trade-offs everywhere:
   - Perfect relevance vs response time
   - Freshness vs crawling cost
   - Personalization vs privacy
   - Completeness vs spam filtering
```

### Interview Tips

```
সার্চ ইঞ্জিন design interview-এ যা মনে রাখতে হবে:

✅ Do:
- Start with requirements ও scale estimation
- Inverted index explain করতে পারা (core concept)
- PageRank basic idea জানা
- Crawling challenges (politeness, dedup) discuss করা
- Trade-offs clearly articulate করা

❌ Don't:
- ML/AI details-এ বেশি ঢুকবেন না (unless asked)
- Single server solution দেবেন না
- Latency budget ভুলবেন না
- Caching strategy skip করবেন না

🎯 Unique Angle (BD Context):
- Multi-language search challenges
- Low-resource language (Bangla) NLP
- Mixed-script query handling
- Regional search with limited infrastructure
```

---

> **পরবর্তী পড়ুন**: [Recommendation System](./recommendation-system.md) | [Chat System (WhatsApp)](./chat-system-whatsapp.md)
