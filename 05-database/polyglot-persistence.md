# 🗄️ Polyglot Persistence Pattern (পলিগ্লট পারসিস্টেন্স)

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [কোন Database কখন ব্যবহার করবেন](#কোন-database-কখন-ব্যবহার-করবেন)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Data Synchronization Challenges](#data-synchronization-challenges)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Migration Strategies](#migration-strategies)
- [Operational Complexity](#operational-complexity)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Polyglot Persistence** হলো একটি architectural pattern যেখানে একটি application বা system এর বিভিন্ন অংশের জন্য **ভিন্ন ভিন্ন database technology** ব্যবহার করা হয়, প্রতিটি তার নির্দিষ্ট কাজের জন্য সবচেয়ে উপযুক্ত।

### 🔑 মূল ধারণা:

```
┌────────────────────────────────────────────────────────────────┐
│              POLYGLOT PERSISTENCE                               │
│                                                                │
│  "একটি database সব কাজের জন্য best হতে পারে না"               │
│                                                                │
│  ┌───────────────────────────────────────────────────────┐     │
│  │                  Application                           │     │
│  └───────────┬──────┬──────┬──────┬──────┬───────────────┘     │
│              │      │      │      │      │                     │
│              ▼      ▼      ▼      ▼      ▼                     │
│  ┌─────┐ ┌──────┐ ┌────┐ ┌────┐ ┌──────┐ ┌───────────┐       │
│  │Postgr│ │Mongo │ │Redis│ │ES  │ │Neo4j │ │TimescaleDB│       │
│  │SQL   │ │DB    │ │    │ │    │ │      │ │           │       │
│  └─────┘ └──────┘ └────┘ └────┘ └──────┘ └───────────┘       │
│  Trans-   Flexible  Cache  Search  Graph   Time-series         │
│  actions  Schema                                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Monoglot vs Polyglot:

```
MONOGLOT (আগের পদ্ধতি):
┌─────────────────────────────────┐
│  সবকিছু একটি MySQL/PostgreSQL  │
│  এ রাখো                         │
│  - Users ✅                      │
│  - Search ⚠️ (slow LIKE query)   │
│  - Cache ❌ (no caching layer)   │
│  - Analytics ⚠️ (blocks OLTP)    │
│  - Graph queries ❌ (complex JOINs) │
└─────────────────────────────────┘

POLYGLOT (আধুনিক পদ্ধতি):
┌─────────────────────────────────┐
│  প্রতিটি কাজের জন্য সেরা DB    │
│  - Users → PostgreSQL ✅         │
│  - Search → Elasticsearch ✅     │
│  - Cache → Redis ✅              │
│  - Analytics → TimescaleDB ✅    │
│  - Relations → Neo4j ✅          │
└─────────────────────────────────┘
```

---

## 🗃️ কোন Database কখন ব্যবহার করবেন

### Database Selection Guide:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DATABASE SELECTION MATRIX                           │
├──────────────┬─────────────────────────┬────────────────────────────┤
│   Database   │    Best For             │    Bangladesh Example       │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ PostgreSQL   │ ACID transactions       │ bKash: wallet balances,    │
│              │ Complex queries         │ payment records            │
│              │ Relational data         │                            │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ MongoDB      │ Flexible schema         │ Prothom Alo: articles,     │
│              │ Document storage        │ comments, user profiles    │
│              │ Rapid prototyping       │                            │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ Redis        │ Caching                 │ Pathao: driver locations,  │
│              │ Session storage         │ surge pricing, sessions    │
│              │ Rate limiting           │                            │
│              │ Real-time leaderboards  │                            │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ Elasticsearch│ Full-text search        │ Daraz: product search,     │
│              │ Log analysis            │ autocomplete, filters      │
│              │ Fuzzy matching          │                            │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ Neo4j        │ Graph relationships     │ LinkedIn BD: connections,  │
│              │ Recommendation engines  │ "people you may know"      │
│              │ Fraud detection         │                            │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ TimescaleDB  │ Time-series data        │ Pathao: ride analytics,    │
│              │ IoT sensor data         │ peak hours, demand         │
│              │ Metrics & monitoring    │ forecasting                │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ ClickHouse   │ Analytics (OLAP)        │ Daraz: sales reports,      │
│              │ Aggregations            │ seller performance         │
│              │ Column-oriented queries │                            │
├──────────────┼─────────────────────────┼────────────────────────────┤
│ Apache Cassandra │ High write throughput│ IoT platforms:             │
│              │ Distributed globally    │ sensor data, event logs    │
│              │ Always-available        │                            │
└──────────────┴─────────────────────────┴────────────────────────────┘
```

### Decision Flowchart:

```
            ┌─────────────────────┐
            │ আপনার data কেমন?    │
            └──────────┬──────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │Structured│  │Semi-      │  │Unstructured│
   │(fixed    │  │Structured │  │(text/media)│
   │schema)   │  │(JSON-like)│  │           │
   └────┬─────┘  └────┬─────┘  └────┬──────┘
        │              │              │
        ▼              ▼              ▼
   ACID দরকার?   Schema কি     Search দরকার?
        │          পালটায়?          │
   ┌────┴────┐        │         ┌───┴────┐
   │Yes │ No │        ▼         │Yes │No │
   ▼    ▼    ▼   ┌────────┐    ▼    ▼
PostgreSQL  Redis │MongoDB │  Elastic  S3/
MySQL       Cassandra│       │  search   MinIO
                └────────┘
```

---

## 🏗️ আর্কিটেকচার ডায়াগ্রাম

### Pathao এর Polyglot Architecture:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PATHAO - Polyglot Architecture                     │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                      API Gateway                            │     │
│  └──────┬──────────┬──────────┬──────────┬────────────────────┘     │
│         │          │          │          │                          │
│         ▼          ▼          ▼          ▼                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│  │  Ride    │ │ Driver   │ │  Search  │ │Analytics │               │
│  │ Service  │ │ Location │ │ Service  │ │ Service  │               │
│  │          │ │ Service  │ │          │ │          │               │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘               │
│       │             │            │            │                      │
│       ▼             ▼            ▼            ▼                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐           │
│  │PostgreSQL│ │  Redis   │ │Elastic-  │ │ TimescaleDB  │           │
│  │          │ │  Cluster │ │search    │ │              │           │
│  │• rides   │ │          │ │          │ │• ride_metrics│           │
│  │• payments│ │• driver  │ │• places  │ │• peak_hours  │           │
│  │• users   │ │  lat/lng │ │• areas   │ │• demand_data │           │
│  │• wallets │ │• surge   │ │• drivers │ │• fare_history│           │
│  │          │ │  pricing │ │          │ │              │           │
│  └──────────┘ │• sessions│ └──────────┘ └──────────────┘           │
│               │• ride    │                                          │
│               │  matching│                                          │
│               └──────────┘                                          │
│                                                                     │
│  কেন এই combination:                                               │
│  • PostgreSQL: Ride booking ACID transaction দরকার                  │
│  • Redis: Driver location প্রতি সেকেন্ডে update, ultra-fast read    │
│  • Elasticsearch: "মিরপুর ১০" লিখলে place suggestion দরকার          │
│  • TimescaleDB: "শুক্রবার সন্ধ্যায় গুলশানে demand কত?"             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🇧🇩 বাস্তব জীবনের উদাহরণ

### উদাহরণ ১: Pathao Complete Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Pathao - Database Per Service                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ RIDE SERVICE (PostgreSQL)                            │    │
│  │ • ride_requests (id, rider_id, pickup, dropoff)     │    │
│  │ • ride_assignments (ride_id, driver_id, status)     │    │
│  │ • payments (ride_id, amount, method, status)        │    │
│  │ • কেন PostgreSQL: ACID, complex JOIN, foreign key   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ DRIVER LOCATION SERVICE (Redis)                      │    │
│  │ • driver:location:{id} → {lat, lng, timestamp}      │    │
│  │ • GEO: GEOADD drivers {lng} {lat} {driver_id}      │    │
│  │ • GEORADIUS: ৩ কিমি এর মধ্যে সব driver খোঁজো        │    │
│  │ • কেন Redis: O(1) read/write, GEO commands          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ SEARCH SERVICE (Elasticsearch)                       │    │
│  │ • places index: নাম, ঠিকানা, coordinates             │    │
│  │ • "ধানমন্ডি" টাইপ করলে suggestions আসবে              │    │
│  │ • Fuzzy matching: ভুল বানানেও কাজ করে                │    │
│  │ • কেন ES: full-text search, auto-complete            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ ANALYTICS SERVICE (TimescaleDB)                      │    │
│  │ • ride_metrics: প্রতি মিনিটে rides, avg fare         │    │
│  │ • demand_zones: area-wise demand heatmap             │    │
│  │ • surge_history: কখন surge pricing ছিল              │    │
│  │ • কেন TimescaleDB: time-series query optimized       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### উদাহরণ ২: Daraz E-commerce

```
┌─────────────────────────────────────────────────────────────┐
│              Daraz - Polyglot Persistence                     │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ PostgreSQL   │  │   MongoDB    │  │    Redis     │      │
│  │              │  │              │  │              │      │
│  │ • orders     │  │ • product    │  │ • cart       │      │
│  │ • payments   │  │   catalog    │  │ • sessions   │      │
│  │ • inventory  │  │ • reviews    │  │ • rate limit │      │
│  │ • users      │  │ • seller     │  │ • flash sale │      │
│  │              │  │   profiles   │  │   counters   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Elasticsearch │  │  ClickHouse  │  │    Neo4j     │      │
│  │              │  │              │  │              │      │
│  │ • product    │  │ • sales      │  │ • "also      │      │
│  │   search     │  │   analytics  │  │   bought"    │      │
│  │ • filters    │  │ • seller     │  │ • recommend- │      │
│  │ • auto-      │  │   reports    │  │   ations     │      │
│  │   complete   │  │ • trends     │  │ • categories │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔄 Data Synchronization Challenges

### Eventual Consistency Problem:

```
┌────────────────────────────────────────────────────────────────┐
│          DATA SYNC - EVENTUAL CONSISTENCY                       │
│                                                                │
│  সমস্যা: একই data বিভিন্ন DB তে থাকলে sync কিভাবে?           │
│                                                                │
│  Product update হলো PostgreSQL এ:                               │
│                                                                │
│  সময়  ────────────────────────────────────────────►            │
│                                                                │
│  T0: PostgreSQL updated ✅                                      │
│  T1: Elasticsearch ← sync event... (processing)               │
│  T2: Redis cache ← invalidation... (processing)               │
│  T3: Elasticsearch updated ✅ (200ms delay)                     │
│  T4: Redis cleared ✅ (50ms delay)                              │
│                                                                │
│  T1-T3 সময়ে: Search পুরানো data দেখাবে! 😱                    │
│  এটাই "Eventual Consistency"                                   │
│                                                                │
│  ┌────────────────────────────────────────────┐                │
│  │       CONSISTENCY WINDOW                    │                │
│  │  PostgreSQL  ████████████████████████████   │                │
│  │  Elastic     ░░░░████████████████████████   │                │
│  │  Redis       ░░████████████████████████████ │                │
│  │              ↑                              │                │
│  │         inconsistent                        │                │
│  │         period                              │                │
│  └────────────────────────────────────────────┘                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Sync Strategies:

```
┌────────────────────────────────────────────────────────────────┐
│              SYNCHRONIZATION STRATEGIES                         │
│                                                                │
│  ১. Event-Driven Sync (Recommended):                          │
│  ┌──────────┐    Event     ┌────────┐    ┌──────────────┐     │
│  │PostgreSQL│───────────►│ Kafka  │───►│ ES Consumer  │     │
│  └──────────┘  (CDC/Outbox)└────────┘    └──────────────┘     │
│                                                                │
│  ২. Dual Write (Risky):                                       │
│  ┌──────────┐                                                  │
│  │ Service  │───► PostgreSQL                                   │
│  │          │───► Elasticsearch  ← 💥 Partial failure!         │
│  └──────────┘                                                  │
│                                                                │
│  ৩. Scheduled Sync (Simple but delayed):                      │
│  ┌──────────┐  Cron job    ┌──────────────────┐               │
│  │PostgreSQL│─── every ───►│ Sync to ES/Redis │               │
│  └──────────┘   5 min      └──────────────────┘               │
│                                                                │
│  ৪. CDC (Best for real-time):                                 │
│  ┌──────────┐  binlog/WAL  ┌──────────┐  ┌──────────┐        │
│  │PostgreSQL│─────────────►│ Debezium │─►│ ES/Redis │        │
│  └──────────┘              └──────────┘  └──────────┘        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Multi-Database Repository Pattern:

```php
<?php

namespace App\Repositories;

use App\Models\Product;
use Elasticsearch\Client as ESClient;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

/**
 * Daraz Product Repository - Polyglot Persistence
 * PostgreSQL (source of truth) + Redis (cache) + Elasticsearch (search)
 */
class ProductRepository
{
    private ESClient $elasticsearch;
    private const CACHE_TTL = 3600; // 1 hour

    public function __construct(ESClient $elasticsearch)
    {
        $this->elasticsearch = $elasticsearch;
    }

    /**
     * Product খোঁজো - প্রথমে Redis, তারপর PostgreSQL
     */
    public function findById(int $id): ?array
    {
        // ১. Redis cache চেক করো (fastest)
        $cached = Redis::get("product:{$id}");
        if ($cached) {
            Log::debug("Cache HIT for product:{$id}");
            return json_decode($cached, true);
        }

        // ২. PostgreSQL থেকে নাও (source of truth)
        $product = Product::with(['category', 'brand'])->find($id);
        if (!$product) {
            return null;
        }

        $data = $product->toArray();

        // ৩. Redis এ cache করো
        Redis::setex("product:{$id}", self::CACHE_TTL, json_encode($data));

        return $data;
    }

    /**
     * Product search করো - Elasticsearch ব্যবহার করো
     * "Samsung Galaxy" বা "স্যামসাং" লিখলে কাজ করবে
     */
    public function search(string $query, array $filters = [], int $page = 1, int $perPage = 20): array
    {
        $body = [
            'from' => ($page - 1) * $perPage,
            'size' => $perPage,
            'query' => [
                'bool' => [
                    'must' => [
                        [
                            'multi_match' => [
                                'query'  => $query,
                                'fields' => ['name^3', 'name_bn^3', 'description', 'brand^2', 'tags'],
                                'type'   => 'best_fields',
                                'fuzziness' => 'AUTO',
                            ],
                        ],
                    ],
                    'filter' => $this->buildFilters($filters),
                ],
            ],
            'sort' => $this->buildSort($filters['sort'] ?? 'relevance'),
            'aggs' => [
                'categories' => ['terms' => ['field' => 'category_id', 'size' => 20]],
                'brands'     => ['terms' => ['field' => 'brand.keyword', 'size' => 20]],
                'price_ranges' => [
                    'range' => [
                        'field' => 'price',
                        'ranges' => [
                            ['to' => 500],
                            ['from' => 500, 'to' => 2000],
                            ['from' => 2000, 'to' => 5000],
                            ['from' => 5000, 'to' => 20000],
                            ['from' => 20000],
                        ],
                    ],
                ],
            ],
        ];

        $result = $this->elasticsearch->search([
            'index' => 'daraz_products',
            'body'  => $body,
        ]);

        return [
            'products'     => array_map(fn($hit) => $hit['_source'], $result['hits']['hits']),
            'total'        => $result['hits']['total']['value'],
            'aggregations' => $result['aggregations'],
        ];
    }

    /**
     * Product তৈরি করো - সব DB তে write করো (via events)
     */
    public function create(array $data): Product
    {
        return DB::transaction(function () use ($data) {
            // ১. PostgreSQL এ সেভ (source of truth)
            $product = Product::create($data);

            // ২. Outbox event রাখো (Elasticsearch ও Redis async sync হবে)
            DB::table('outbox_messages')->insert([
                'id'             => \Str::uuid(),
                'aggregate_type' => 'Product',
                'aggregate_id'   => $product->id,
                'event_type'     => 'ProductCreated',
                'payload'        => json_encode($product->toArray()),
                'status'         => 'pending',
                'created_at'     => now(),
            ]);

            return $product;
        });
    }

    /**
     * Product update করো
     */
    public function update(int $id, array $data): Product
    {
        return DB::transaction(function () use ($id, $data) {
            $product = Product::findOrFail($id);
            $product->update($data);

            // Cache invalidate (immediate)
            Redis::del("product:{$id}");

            // Outbox event (ES async update)
            DB::table('outbox_messages')->insert([
                'id'             => \Str::uuid(),
                'aggregate_type' => 'Product',
                'aggregate_id'   => $product->id,
                'event_type'     => 'ProductUpdated',
                'payload'        => json_encode($product->fresh()->toArray()),
                'status'         => 'pending',
                'created_at'     => now(),
            ]);

            return $product->fresh();
        });
    }

    private function buildFilters(array $filters): array
    {
        $esFilters = [];

        if (isset($filters['category_id'])) {
            $esFilters[] = ['term' => ['category_id' => $filters['category_id']]];
        }
        if (isset($filters['min_price'])) {
            $esFilters[] = ['range' => ['price' => ['gte' => $filters['min_price']]]];
        }
        if (isset($filters['max_price'])) {
            $esFilters[] = ['range' => ['price' => ['lte' => $filters['max_price']]]];
        }
        if (isset($filters['in_stock']) && $filters['in_stock']) {
            $esFilters[] = ['range' => ['stock' => ['gt' => 0]]];
        }

        return $esFilters;
    }

    private function buildSort(string $sortType): array
    {
        return match ($sortType) {
            'price_asc'  => [['price' => 'asc']],
            'price_desc' => [['price' => 'desc']],
            'newest'     => [['updated_at' => 'desc']],
            'popular'    => [['sold_count' => 'desc']],
            default      => ['_score'],
        };
    }
}
```

### Redis Geo Service (Pathao Driver Location):

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Redis;

/**
 * Pathao: Driver location management with Redis GEO
 * প্রতি ৫ সেকেন্ডে driver এর location update হয়
 */
class DriverLocationService
{
    private const GEO_KEY = 'drivers:active:locations';
    private const LOCATION_TTL = 60; // 60 sec না আসলে offline ধরো

    /**
     * Driver এর location update করো
     * App থেকে প্রতি ৫ সেকেন্ডে আসে
     */
    public function updateLocation(string $driverId, float $lat, float $lng): void
    {
        // GEO index এ রাখো
        Redis::geoadd(self::GEO_KEY, $lng, $lat, $driverId);

        // Driver details hash এ রাখো
        Redis::hmset("driver:{$driverId}:location", [
            'lat'        => $lat,
            'lng'        => $lng,
            'updated_at' => time(),
            'status'     => 'active',
        ]);

        // TTL set করো (expire হলে offline)
        Redis::expire("driver:{$driverId}:location", self::LOCATION_TTL);
    }

    /**
     * নির্দিষ্ট location এর কাছে available drivers খোঁজো
     * Customer ride request করলে এটা call হয়
     */
    public function findNearbyDrivers(float $lat, float $lng, float $radiusKm = 3, int $limit = 10): array
    {
        // Redis GEORADIUS - O(N+log(M)) complexity
        $nearbyDriverIds = Redis::georadius(
            self::GEO_KEY,
            $lng,
            $lat,
            $radiusKm,
            'km',
            [
                'WITHCOORD',
                'WITHDIST',
                'ASC', // কাছের driver আগে
                'COUNT', $limit,
            ]
        );

        $drivers = [];
        foreach ($nearbyDriverIds as $item) {
            $driverId = $item[0];
            $distance = $item[1];
            $coordinates = $item[2];

            // Driver details নাও
            $details = Redis::hgetall("driver:{$driverId}:location");

            if (!empty($details) && $details['status'] === 'active') {
                $drivers[] = [
                    'driver_id'  => $driverId,
                    'distance_km'=> round((float)$distance, 2),
                    'lat'        => (float)$coordinates[1],
                    'lng'        => (float)$coordinates[0],
                    'last_update'=> (int)$details['updated_at'],
                ];
            }
        }

        return $drivers;
    }

    /**
     * Surge pricing calculate করো
     * নির্দিষ্ট area তে demand vs supply
     */
    public function calculateSurgeMultiplier(float $lat, float $lng): float
    {
        // ২ কিমি radius এ active drivers কত?
        $driversNearby = count(Redis::georadius(
            self::GEO_KEY, $lng, $lat, 2, 'km'
        ));

        // Pending ride requests কত? (sorted set)
        $pendingRequests = Redis::zcount(
            "pending_rides:area:" . $this->getAreaCode($lat, $lng),
            '-inf', '+inf'
        );

        if ($driversNearby === 0) return 3.0; // Maximum surge
        
        $ratio = $pendingRequests / $driversNearby;
        
        return match (true) {
            $ratio > 3   => 2.5,
            $ratio > 2   => 2.0,
            $ratio > 1.5 => 1.5,
            $ratio > 1   => 1.2,
            default      => 1.0,
        };
    }

    private function getAreaCode(float $lat, float $lng): string
    {
        // Geohash precision 5 (≈5km area)
        return substr(md5("{$lat},{$lng}"), 0, 8);
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Polyglot Data Access Layer:

```javascript
// services/DataAccessLayer.js
const { Pool } = require('pg');
const { Client: ESClient } = require('@elastic/elasticsearch');
const Redis = require('ioredis');
const mongoose = require('mongoose');

/**
 * Pathao: Multi-database access layer
 * প্রতিটি service তার নিজের DB ব্যবহার করে
 */
class PathaoDataLayer {
    constructor(config) {
        // PostgreSQL - rides, payments, users
        this.pg = new Pool({
            host: config.postgres.host,
            database: 'pathao_rides',
            user: config.postgres.user,
            password: config.postgres.password,
        });

        // Redis - locations, sessions, cache
        this.redis = new Redis({
            host: config.redis.host,
            port: 6379,
            keyPrefix: 'pathao:',
        });

        // Elasticsearch - search
        this.es = new ESClient({
            node: config.elasticsearch.url,
        });

        // TimescaleDB - analytics
        this.timescale = new Pool({
            host: config.timescale.host,
            database: 'pathao_analytics',
        });
    }

    // === RIDE SERVICE (PostgreSQL) ===

    async createRide(rideData) {
        const client = await this.pg.connect();
        try {
            await client.query('BEGIN');

            const result = await client.query(
                `INSERT INTO rides (rider_id, pickup_lat, pickup_lng,
                 dropoff_lat, dropoff_lng, status, estimated_fare)
                 VALUES ($1, $2, $3, $4, $5, $6, $7)
                 RETURNING *`,
                [
                    rideData.riderId,
                    rideData.pickupLat,
                    rideData.pickupLng,
                    rideData.dropoffLat,
                    rideData.dropoffLng,
                    'requested',
                    rideData.estimatedFare,
                ]
            );

            await client.query('COMMIT');
            return result.rows[0];
        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }

    async assignDriver(rideId, driverId) {
        return this.pg.query(
            `UPDATE rides SET driver_id = $1, status = 'assigned',
             assigned_at = NOW() WHERE id = $2 RETURNING *`,
            [driverId, rideId]
        );
    }

    // === DRIVER LOCATION SERVICE (Redis) ===

    async updateDriverLocation(driverId, lat, lng) {
        const pipeline = this.redis.pipeline();

        // GEO index update
        pipeline.geoadd('drivers:geo', lng, lat, driverId);

        // Driver metadata
        pipeline.hmset(`driver:${driverId}`, {
            lat: lat.toString(),
            lng: lng.toString(),
            updated_at: Date.now().toString(),
            status: 'active',
        });

        // TTL - 60 sec না আসলে offline
        pipeline.expire(`driver:${driverId}`, 60);

        await pipeline.exec();
    }

    async findNearbyDrivers(lat, lng, radiusKm = 3) {
        const results = await this.redis.georadius(
            'drivers:geo',
            lng, lat, radiusKm, 'km',
            'WITHCOORD', 'WITHDIST', 'ASC', 'COUNT', 15
        );

        const drivers = [];
        for (const [driverId, distance, coords] of results) {
            const meta = await this.redis.hgetall(`driver:${driverId}`);
            if (meta && meta.status === 'active') {
                drivers.push({
                    driverId,
                    distance: parseFloat(distance),
                    lat: parseFloat(coords[1]),
                    lng: parseFloat(coords[0]),
                    lastUpdate: parseInt(meta.updated_at),
                });
            }
        }

        return drivers;
    }

    // === SEARCH SERVICE (Elasticsearch) ===

    async searchPlaces(query, lat, lng) {
        const result = await this.es.search({
            index: 'pathao_places',
            body: {
                query: {
                    bool: {
                        must: [
                            {
                                multi_match: {
                                    query,
                                    fields: ['name^3', 'name_bn^3', 'address', 'area'],
                                    fuzziness: 'AUTO',
                                },
                            },
                        ],
                        should: [
                            {
                                // কাছের places কে priority দাও
                                geo_distance: {
                                    distance: '10km',
                                    location: { lat, lon: lng },
                                    boost: 2,
                                },
                            },
                        ],
                    },
                },
                sort: [
                    '_score',
                    {
                        _geo_distance: {
                            location: { lat, lon: lng },
                            order: 'asc',
                            unit: 'km',
                        },
                    },
                ],
                size: 10,
            },
        });

        return result.hits.hits.map(hit => ({
            ...hit._source,
            score: hit._score,
            distance: hit.sort?.[1],
        }));
    }

    // === ANALYTICS SERVICE (TimescaleDB) ===

    async recordRideMetric(rideData) {
        await this.timescale.query(
            `INSERT INTO ride_metrics
             (time, ride_id, area, fare, distance_km, duration_min, surge_multiplier)
             VALUES (NOW(), $1, $2, $3, $4, $5, $6)`,
            [
                rideData.rideId,
                rideData.area,
                rideData.fare,
                rideData.distanceKm,
                rideData.durationMin,
                rideData.surgeMultiplier,
            ]
        );
    }

    async getAreaDemand(area, hours = 24) {
        const result = await this.timescale.query(
            `SELECT
                time_bucket('1 hour', time) AS hour,
                COUNT(*) AS ride_count,
                AVG(fare) AS avg_fare,
                AVG(surge_multiplier) AS avg_surge
             FROM ride_metrics
             WHERE area = $1
               AND time > NOW() - INTERVAL '${hours} hours'
             GROUP BY hour
             ORDER BY hour DESC`,
            [area]
        );
        return result.rows;
    }

    async getPeakHours(area) {
        const result = await this.timescale.query(
            `SELECT
                EXTRACT(HOUR FROM time) AS hour_of_day,
                EXTRACT(DOW FROM time) AS day_of_week,
                AVG(ride_count) AS avg_rides
             FROM (
                SELECT time_bucket('1 hour', time) AS time, COUNT(*) AS ride_count
                FROM ride_metrics
                WHERE area = $1 AND time > NOW() - INTERVAL '30 days'
                GROUP BY time_bucket('1 hour', time)
             ) hourly
             GROUP BY hour_of_day, day_of_week
             ORDER BY avg_rides DESC
             LIMIT 10`,
            [area]
        );
        return result.rows;
    }
}

module.exports = PathaoDataLayer;
```

### Data Sync Service:

```javascript
// services/DataSyncService.js
const { Kafka } = require('kafkajs');

/**
 * Multi-database sync service
 * PostgreSQL → Redis/Elasticsearch/TimescaleDB sync
 * CDC events অথবা application events ব্যবহার করে
 */
class DataSyncService {
    constructor(dataLayer) {
        this.dataLayer = dataLayer;
        this.kafka = new Kafka({
            clientId: 'pathao-sync-service',
            brokers: ['kafka:9092'],
        });
    }

    async startSync() {
        const consumer = this.kafka.consumer({ groupId: 'data-sync' });
        await consumer.connect();

        await consumer.subscribe({
            topics: [
                'pathao.rides.events',
                'pathao.drivers.events',
                'pathao.places.events',
            ],
        });

        await consumer.run({
            eachMessage: async ({ topic, message }) => {
                const event = JSON.parse(message.value.toString());
                await this.handleSyncEvent(topic, event);
            },
        });

        console.log('🔄 Data sync service started');
    }

    async handleSyncEvent(topic, event) {
        switch (event.type) {
            case 'RideCompleted':
                // PostgreSQL → TimescaleDB analytics sync
                await this.syncRideToAnalytics(event.data);
                // Cache invalidate
                await this.dataLayer.redis.del(`rider:${event.data.rider_id}:history`);
                break;

            case 'PlaceCreated':
            case 'PlaceUpdated':
                // PostgreSQL → Elasticsearch sync
                await this.syncPlaceToSearch(event.data);
                break;

            case 'DriverOnline':
                // Redis GEO update
                await this.dataLayer.updateDriverLocation(
                    event.data.driver_id,
                    event.data.lat,
                    event.data.lng
                );
                break;

            case 'DriverOffline':
                // Redis থেকে remove
                await this.dataLayer.redis.zrem('drivers:geo', event.data.driver_id);
                await this.dataLayer.redis.del(`driver:${event.data.driver_id}`);
                break;
        }
    }

    async syncRideToAnalytics(rideData) {
        await this.dataLayer.recordRideMetric({
            rideId: rideData.id,
            area: rideData.area,
            fare: rideData.final_fare,
            distanceKm: rideData.distance_km,
            durationMin: rideData.duration_min,
            surgeMultiplier: rideData.surge_multiplier,
        });
    }

    async syncPlaceToSearch(placeData) {
        await this.dataLayer.es.index({
            index: 'pathao_places',
            id: placeData.id.toString(),
            body: {
                name: placeData.name,
                name_bn: placeData.name_bn,
                address: placeData.address,
                area: placeData.area,
                location: {
                    lat: placeData.lat,
                    lon: placeData.lng,
                },
                type: placeData.type,
                popularity: placeData.popularity_score || 0,
            },
        });
    }
}

module.exports = DataSyncService;
```

---

## 🔀 Migration Strategies

### Monolith থেকে Polyglot এ যাওয়ার কৌশল:

```
┌────────────────────────────────────────────────────────────────┐
│           MIGRATION STRATEGY: STRANGLER FIG PATTERN            │
│                                                                │
│  Phase 1: Identify (চিহ্নিত করো)                              │
│  ┌──────────────────────────────────────────┐                  │
│  │ MySQL (Monolith DB)                       │                  │
│  │ ├── users         → PostgreSQL রাখো      │                  │
│  │ ├── products      → PostgreSQL + ES sync  │                  │
│  │ ├── orders        → PostgreSQL রাখো       │                  │
│  │ ├── sessions      → Redis এ নাও          │                  │
│  │ ├── search_index  → Elasticsearch এ নাও  │                  │
│  │ └── analytics     → TimescaleDB এ নাও    │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                │
│  Phase 2: Shadow Write (পাশাপাশি লিখো)                        │
│  ┌──────────┐     ┌──────────┐                                 │
│  │  MySQL   │     │  Redis   │  ← session data copy করো        │
│  │(primary) │     │ (shadow) │  ← read from Redis, verify     │
│  └──────────┘     └──────────┘                                 │
│                                                                │
│  Phase 3: Switch (পাল্টাও)                                    │
│  ┌──────────┐     ┌──────────┐                                 │
│  │  MySQL   │     │  Redis   │  ← Redis primary হলো            │
│  │(fallback)│     │(primary) │  ← MySQL fallback               │
│  └──────────┘     └──────────┘                                 │
│                                                                │
│  Phase 4: Cleanup (পুরানো সরাও)                               │
│  ┌──────────┐                                                  │
│  │  Redis   │  ← শুধু Redis, MySQL থেকে sessions table drop    │
│  │(primary) │                                                  │
│  └──────────┘                                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Migration Example (PHP):

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

/**
 * Shadow Write Pattern - Migration helper
 * Phase 2: দুটি DB তে write করো, একটি থেকে read করো
 */
class SessionMigrationService
{
    private bool $useRedisAsSource = false; // Phase 3 তে true করো

    public function getSession(string $sessionId): ?array
    {
        if ($this->useRedisAsSource) {
            // Phase 3: Redis primary
            $data = Redis::hgetall("session:{$sessionId}");
            if (empty($data)) {
                // Fallback to MySQL
                return $this->getFromMySQL($sessionId);
            }
            return $data;
        }

        // Phase 2: MySQL primary, Redis verify করো
        $mysqlData = $this->getFromMySQL($sessionId);
        $redisData = Redis::hgetall("session:{$sessionId}");

        // Verify consistency
        if ($mysqlData && $redisData && $mysqlData !== $redisData) {
            Log::warning("Session data mismatch!", [
                'session_id' => $sessionId,
                'mysql'      => $mysqlData,
                'redis'      => $redisData,
            ]);
        }

        return $mysqlData;
    }

    public function setSession(string $sessionId, array $data): void
    {
        // Phase 2 & 3: দুটিতেই write করো
        $this->writeToMySQL($sessionId, $data);
        $this->writeToRedis($sessionId, $data);
    }

    private function getFromMySQL(string $sessionId): ?array
    {
        $row = DB::table('sessions')->where('id', $sessionId)->first();
        return $row ? json_decode($row->data, true) : null;
    }

    private function writeToMySQL(string $sessionId, array $data): void
    {
        DB::table('sessions')->updateOrInsert(
            ['id' => $sessionId],
            ['data' => json_encode($data), 'updated_at' => now()]
        );
    }

    private function writeToRedis(string $sessionId, array $data): void
    {
        Redis::hmset("session:{$sessionId}", $data);
        Redis::expire("session:{$sessionId}", 86400); // 24 hours
    }
}
```

---

## ⚙️ Operational Complexity

### Challenge Matrix:

```
┌────────────────────────────────────────────────────────────────┐
│          OPERATIONAL COMPLEXITY MATRIX                          │
│                                                                │
│  Database Count    Complexity    Team Size Needed              │
│  ─────────────    ──────────    ─────────────────              │
│  1 (Monoglot)     Low           2-3 DBAs                      │
│  2-3              Medium        3-5 + DevOps                  │
│  4-6              High          5-8 + SRE team                │
│  7+               Very High     Dedicated platform team       │
│                                                                │
│  প্রতিটি নতুন DB যোগ করলে:                                     │
│  ┌──────────────────────────────────────────┐                  │
│  │ + Backup strategy                        │                  │
│  │ + Monitoring & alerting                  │                  │
│  │ + Scaling strategy                       │                  │
│  │ + Security (access control)              │                  │
│  │ + Disaster recovery                      │                  │
│  │ + Performance tuning expertise           │                  │
│  │ + Data sync/consistency management       │                  │
│  │ + Developer training                     │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Monitoring Setup:

```
┌────────────────────────────────────────────────────────────────┐
│              MONITORING - Polyglot System                       │
│                                                                │
│  Grafana Dashboard:                                            │
│  ┌──────────────────────────────────────────────────────┐      │
│  │                                                      │      │
│  │  PostgreSQL          Redis            Elasticsearch  │      │
│  │  ┌──────────┐       ┌──────────┐    ┌──────────┐   │      │
│  │  │CPU: 45%  │       │Mem: 72%  │    │Heap: 68% │   │      │
│  │  │Conn: 120 │       │Keys: 5M  │    │Docs: 50M │   │      │
│  │  │QPS: 5000 │       │Hit: 98%  │    │QPS: 2000 │   │      │
│  │  │Repl lag: │       │Evict: 0  │    │Lag: 200ms│   │      │
│  │  │ 200ms    │       │           │    │          │   │      │
│  │  └──────────┘       └──────────┘    └──────────┘   │      │
│  │                                                      │      │
│  │  TimescaleDB         Data Sync        Alerts        │      │
│  │  ┌──────────┐       ┌──────────┐    ┌──────────┐   │      │
│  │  │Chunks: 48│       │Lag: 1.2s │    │🔴 ES lag  │   │      │
│  │  │Compr: 8:1│       │Failed: 0 │    │🟡 Redis   │   │      │
│  │  │QPS: 500  │       │Queue: 42 │    │   memory  │   │      │
│  │  └──────────┘       └──────────┘    └──────────┘   │      │
│  │                                                      │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                │
│  Key Metrics to Monitor:                                       │
│  • Cross-DB sync lag (data কত দেরিতে sync হচ্ছে)              │
│  • Cache hit ratio (Redis miss হলে PostgreSQL এ load)          │
│  • ES indexing lag (search এ latest data আছে কি?)             │
│  • Connection pool utilization                                 │
│  • Replication lag (replica/follower databases)                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Polyglot Persistence ব্যবহার করবেন:

| পরিস্থিতি | উদাহরণ |
|-----------|---------|
| ভিন্ন ভিন্ন data access pattern | Read-heavy vs Write-heavy |
| Search + CRUD দুটোই দরকার | Product CRUD + full-text search |
| Real-time + historical data | Live location + analytics |
| High scale specific areas | Millions of cache reads/sec |
| Microservices architecture | প্রতি service নিজের DB |
| Cost optimization | Hot data Redis, cold data S3 |

### ❌ কখন ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|-------|
| Small application | Overkill, complexity বাড়বে |
| Small team (1-3 developers) | Operational burden সামলাতে পারবে না |
| Strong consistency সর্বত্র দরকার | Eventual consistency acceptable না হলে |
| Budget constraint | প্রতিটি DB এর hosting cost |
| Simple CRUD app | PostgreSQL alone যথেষ্ট |
| No clear performance bottleneck | Premature optimization |

### 💡 Best Practices:

1. **Start Simple** — প্রথমে একটি DB, সমস্যা হলে আরেকটি যোগ করুন
2. **Source of Truth define করুন** — প্রতিটি data এর একটিই authoritative source
3. **Sync strategy আগেই ঠিক করুন** — CDC/Outbox/Events কোনটা ব্যবহার করবেন
4. **Accept eventual consistency** — সব জায়গায় instant consistency সম্ভব না
5. **Monitor sync lag** — Data কত দেরিতে sync হচ্ছে track করুন
6. **Abstract the complexity** — Repository pattern দিয়ে DB details hide করুন
7. **Team skill assess করুন** — নতুন DB যোগ করার আগে team ready কিনা দেখুন

---

## 📊 Database Selection Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│              QUICK SELECTION GUIDE                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  "টাকা-পয়সার হিসাব?" → PostgreSQL (ACID)                   │
│  "বিভিন্ন format এর data?" → MongoDB (flexible)             │
│  "দ্রুত পড়তে হবে বারবার?" → Redis (cache)                  │
│  "text search করতে হবে?" → Elasticsearch                     │
│  "কে কার সাথে connected?" → Neo4j (graph)                   │
│  "সময়ের সাথে data?" → TimescaleDB (time-series)             │
│  "অনেক বেশি write?" → Cassandra (distributed)               │
│  "Analytics/Reports?" → ClickHouse (OLAP)                   │
│  "File/Image storage?" → S3/MinIO (object store)            │
│  "Message queue?" → Redis Streams / Kafka                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎓 সারসংক্ষেপ

```
┌────────────────────────────────────────────────────────┐
│         Polyglot Persistence - মূল কথা                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│  কি: বিভিন্ন কাজের জন্য বিভিন্ন database              │
│  কেন: প্রতিটি DB তার কাজে best                        │
│  চ্যালেঞ্জ: Data sync, eventual consistency            │
│  সমাধান: CDC, Outbox, Events                           │
│                                                        │
│  মনে রাখুন:                                           │
│  ┌──────────────────────────────────────────────┐      │
│  │ "Right tool for the right job"               │      │
│  │ "কিন্তু unnecessary complexity avoid করুন"    │      │
│  │ "Start simple, grow as needed"               │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  Pathao Example:                                       │
│  • PostgreSQL → Rides (ACID)                           │
│  • Redis → Driver locations (speed)                    │
│  • Elasticsearch → Place search (text)                 │
│  • TimescaleDB → Analytics (time-series)               │
│                                                        │
└────────────────────────────────────────────────────────┘
```
