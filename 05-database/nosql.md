# 🗄️ NoSQL ডাটাবেস — সম্পূর্ণ গভীর গাইড

> **"Not Only SQL"** — যখন relational database আপনার data model, scale, বা performance চাহিদা মেটাতে পারে না।

---

## 📌 সংজ্ঞা ও মূল ধারণা

NoSQL হলো এমন database paradigm যেখানে data কে traditional tabular format এ store করা হয় না। এটি **flexible schema** সমর্থন করে। মূল দর্শন — **application-driven data modeling**: আগে ভাবুন "application কী query করবে?", তারপর data model ডিজাইন করুন।

### CAP Theorem গভীর বিশ্লেষণ

Distributed data store একই সাথে তিনটি গ্যারান্টি দিতে পারে না:

- **Consistency (C):** প্রতিটি read সাম্প্রতিক write এর data ফেরত দেবে
- **Availability (A):** প্রতিটি request response পাবে, node down থাকলেও
- **Partition Tolerance (P):** Network partition হলেও system কাজ চালিয়ে যাবে

বাস্তবে network partition অনিবার্য, তাই আসল সিদ্ধান্ত **CP vs AP**:

| সিদ্ধান্ত | মানে | উদাহরণ |
|-----------|-------|---------|
| **CP** | Consistency রাখবে, availability ত্যাগ | MongoDB, HBase, Redis |
| **AP** | Availability রাখবে, consistency ত্যাগ | Cassandra, DynamoDB, CouchDB |
| **CA** | Partition tolerance নেই (single node) | Traditional RDBMS |

### ACID vs BASE

| বৈশিষ্ট্য | ACID (SQL) | BASE (NoSQL) |
|-----------|------------|--------------|
| **পূর্ণরূপ** | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| **দর্শন** | Strong consistency সর্বোচ্চ | Availability ও performance সর্বোচ্চ |
| **Use Case** | bKash transaction ledger | bKash session cache, notification feed |

### SQL vs NoSQL তুলনা

| মানদণ্ড | SQL | NoSQL |
|---------|-----|-------|
| Data Model | Fixed schema, tables | Flexible, documents/key-value/graph |
| Scaling | Vertical (bigger server) | Horizontal (more servers) |
| Relationships | JOINs | Embedding বা app-level JOIN |
| Transactions | ACID গ্যারান্টি | BASE (কিছুতে multi-doc ACID) |
| Query | SQL (standardized) | Database-specific (MQL, CQL, Cypher) |

---

## 📊 CAP Theorem ডায়াগ্রাম

```
                     Consistency (C)
                         /\
                        /  \
                       / CP  \
                      / MongoDB\
                     / HBase    \
                    /   Redis    \
                   /    ⚠️ CA     \
                  / (Single Node)  \
                 / PostgreSQL,MySQL \
                /____________________\
     Availability (A) ————— Partition Tolerance (P)
                       AP
                 Cassandra, CouchDB
                 DynamoDB, Riak
```

---

## 💻 NoSQL Categories Deep Dive

### ১. Document Store (MongoDB)

Data BSON (Binary JSON) format এ store হয়। `Database → Collection → Document → Field`

**Embedding vs Referencing:** Embed করুন যদি data একসাথে read হয়, child ছোট। Reference করুন যদি data independently access হয়, many:many relationship।

**PHP (Laravel + jenssegers/mongodb):**

```php
use Jenssegers\Mongodb\Eloquent\Model;

class Product extends Model
{
    protected $connection = 'mongodb';
    protected $collection = 'products';
    protected $fillable = ['name', 'slug', 'sku', 'price', 'category', 'specifications', 'variants', 'tags'];

    public function addReview(array $review): self
    {
        $review['created_at'] = now();
        $review['id'] = (string) new \MongoDB\BSON\ObjectId();
        $this->push('reviews', $review);
        // Average rating update
        $reviews = $this->reviews ?? [];
        $avg = array_sum(array_column($reviews, 'rating')) / count($reviews);
        $this->update(['average_rating' => round($avg, 2)]);
        return $this;
    }

    // Aggregation Pipeline — seller analytics
    public static function salesAnalytics(): array
    {
        return self::raw(fn($col) => $col->aggregate([
            ['$group' => [
                '_id' => '$seller_id',
                'total_products' => ['$sum' => 1],
                'avg_price' => ['$avg' => '$price'],
                'categories' => ['$addToSet' => '$category'],
            ]],
            ['$sort' => ['total_products' => -1]],
        ]));
    }
}

// Indexing
Schema::connection('mongodb')->table('products', function ($col) {
    $col->index('slug', null, null, ['unique' => true]);
    $col->index(['category' => 1, 'price' => -1]);          // Compound
    $col->index(['name' => 'text', 'tags' => 'text']);       // Text
    $col->index(['seller_location' => '2dsphere']);           // Geospatial
});
```

**JavaScript (Mongoose):**

```javascript
const productSchema = new mongoose.Schema({
  name: { type: String, required: true },
  slug: { type: String, unique: true },
  price: { type: Number, required: true, min: 0 },
  category: { type: String, index: true },
  specifications: mongoose.Schema.Types.Mixed,
  variants: [{ color: String, size: String, stock: Number }],
  reviews: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    rating: { type: Number, min: 1, max: 5 },
    comment: String,
  }],
  average_rating: { type: Number, default: 0 },
  seller_location: { type: { type: String, default: 'Point' }, coordinates: [Number] },
}, { timestamps: true, toJSON: { virtuals: true } });

productSchema.virtual('discount_percentage').get(function () {
  if (!this.discount_price) return 0;
  return Math.round(((this.price - this.discount_price) / this.price) * 100);
});

productSchema.pre('save', function (next) {
  if (this.isModified('reviews') && this.reviews.length) {
    this.average_rating = +(this.reviews.reduce((s, r) => s + r.rating, 0) / this.reviews.length).toFixed(2);
  }
  next();
});

productSchema.index({ name: 'text', tags: 'text' });
productSchema.index({ seller_location: '2dsphere' });

// Aggregation
productSchema.statics.getCategoryAnalytics = function () {
  return this.aggregate([
    { $match: { stock: { $gt: 0 } } },
    { $group: { _id: '$category', count: { $sum: 1 }, avg_price: { $avg: '$price' } } },
    { $sort: { count: -1 } },
  ]);
};
```

---

### ২. Key-Value Store (Redis)

| Structure | ব্যবহার | উদাহরণ |
|-----------|---------|---------|
| **String** | Counter, cache | Page views, OTP |
| **Hash** | Object storage | User session |
| **List** | Queue, recent items | Notifications |
| **Set** | Unique collections | Online users |
| **Sorted Set** | Rankings | Top sellers, rate limiting |
| **Stream** | Event sourcing | Activity feed |

#### Caching Patterns
- **Cache-Aside:** Cache miss → DB → Cache এ রাখো (সবচেয়ে জনপ্রিয়)
- **Write-Through:** Cache ও DB তে একসাথে লেখো (always updated, write ধীর)
- **Write-Behind:** Cache এ লেখো → Async DB (দ্রুত, data loss ঝুঁকি)

**PHP (Laravel + phpredis):**

```php
// Cache-Aside — bKash user profile caching
class UserProfileCache
{
    public function __construct(private \Redis $redis, private UserRepository $repo) {}

    public function getProfile(string $userId): array
    {
        $cached = $this->redis->get("user:profile:{$userId}");
        if ($cached !== false) return json_decode($cached, true);

        $profile = $this->repo->findById($userId);
        $this->redis->setex("user:profile:{$userId}", 3600, json_encode($profile));
        return $profile;
    }
}

// Rate Limiting — Sliding Window (Lua script for atomicity)
class RateLimiter
{
    public function isAllowed(string $id, int $max = 60, int $window = 60): bool
    {
        $key = "rate:{$id}";
        $now = microtime(true);
        $script = <<<'LUA'
            redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', ARGV[2])
            local count = redis.call('ZCARD', KEYS[1])
            if count < tonumber(ARGV[3]) then
                redis.call('ZADD', KEYS[1], ARGV[1], ARGV[1]..math.random(1000000))
                redis.call('EXPIRE', KEYS[1], ARGV[4])
                return 1
            end
            return 0
        LUA;
        return (bool) $this->redis->eval($script, [$key, $now, $now - $window, $max, $window], 1);
    }
}

// Session — bKash login
$this->redis->hMSet("session:user:12345", [
    'phone' => '01712345678', 'name' => 'রহিম', 'device' => 'Android',
]);
$this->redis->expire("session:user:12345", 1800); // ৩০ মিনিট

// Pub/Sub — Pathao real-time ride updates
$this->redis->publish('ride:dhaka', json_encode([
    'ride_id' => 'RIDE_789', 'status' => 'arriving', 'eta' => 3,
]));
```

**JavaScript (ioredis):**

```javascript
import Redis from 'ioredis';
const redis = new Redis({ host: '127.0.0.1', port: 6379 });

// Cache with stampede protection
class CacheManager {
  async getOrSet(key, fetchFn, ttl = 3600) {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);

    const lockKey = `lock:${key}`;
    const locked = await redis.set(lockKey, '1', 'EX', 10, 'NX');
    if (!locked) { // অন্য কেউ fetch করছে, wait
      await new Promise(r => setTimeout(r, 100));
      return this.getOrSet(key, fetchFn, ttl);
    }
    try {
      const data = await fetchFn();
      await redis.setex(key, ttl, JSON.stringify(data));
      return data;
    } finally { await redis.del(lockKey); }
  }
}

// Leaderboard — Sorted Set (Daraz top sellers)
class Leaderboard {
  constructor(name) { this.key = `leaderboard:${name}`; }
  async addScore(userId, score) { await redis.zincrby(this.key, score, userId); }
  async getTopN(n = 10) {
    const res = await redis.zrevrange(this.key, 0, n - 1, 'WITHSCORES');
    return res.reduce((acc, v, i) => i % 2 === 0 ? [...acc, { userId: v }] : (acc[acc.length - 1].score = +v, acc), []);
  }
}

// Redis Stream — event-driven
await redis.xadd('stream:activity', '*', 'user_id', '123', 'action', 'purchase');
```

#### Eviction Policies

| Policy | বিবরণ | কখন |
|--------|--------|------|
| **allkeys-lru** | সব key থেকে LRU মুছে | General cache |
| **volatile-lru** | TTL সেট করা key থেকে LRU | Mixed data |
| **allkeys-lfu** | LFU মুছে | Frequency গুরুত্বপূর্ণ |
| **volatile-ttl** | কম TTL বাকি সেটি মুছে | Short-lived priority |
| **noeviction** | Error দেয় | Data loss অগ্রহণযোগ্য |

---

### ৩. Wide-Column Store (Cassandra)

**Write-optimized** database (LSM Tree)। **Query-driven design** — আগে query, তারপর table।

- **Partition Key:** Data কোন node এ যাবে (hash)
- **Clustering Key:** Partition এর মধ্যে sort order
- **Primary Key = Partition Key + Clustering Key(s)**

```sql
CREATE KEYSPACE ride_sharing WITH replication = {
  'class': 'NetworkTopologyStrategy', 'dc_dhaka': 3, 'dc_chittagong': 2
};

-- Ride history — "এই তারিখে সব rides দেখাও"
CREATE TABLE rides_by_date (
  ride_date DATE, ride_time TIMESTAMP, ride_id UUID,
  rider_id UUID, driver_id UUID, pickup TEXT, dropoff TEXT, fare DECIMAL,
  PRIMARY KEY ((ride_date), ride_time, ride_id)
) WITH CLUSTERING ORDER BY (ride_time DESC);

-- IoT sensor data (ঢাকার বায়ু মান)
CREATE TABLE sensor_readings (
  sensor_id TEXT, reading_date DATE, reading_time TIMESTAMP,
  temperature FLOAT, humidity FLOAT, aqi INT,
  PRIMARY KEY ((sensor_id, reading_date), reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC)
AND default_time_to_live = 7776000;  -- ৯০ দিন
```

**PHP:**

```php
$cluster = Cassandra::cluster()
    ->withContactPoints('node1', 'node2')
    ->withDefaultConsistency(Cassandra::CONSISTENCY_LOCAL_QUORUM)
    ->build();
$session = $cluster->connect('ride_sharing');

$stmt = $session->prepare(
    'INSERT INTO rides_by_date (ride_date, ride_time, ride_id, pickup, dropoff, fare) VALUES (?, ?, ?, ?, ?, ?)'
);
$session->execute($stmt, ['arguments' => [
    new Cassandra\Date(time()), new Cassandra\Timestamp(time() * 1000),
    new Cassandra\Uuid(), 'মিরপুর-১০', 'গুলশান-২', new Cassandra\Decimal('250.50'),
]]);
```

**JavaScript:**

```javascript
import { Client, types } from 'cassandra-driver';
const client = new Client({ contactPoints: ['node1', 'node2'], localDataCenter: 'dc_dhaka', keyspace: 'ride_sharing' });
await client.connect();

await client.execute(
  'INSERT INTO rides_by_date (ride_date, ride_time, ride_id, pickup, dropoff, fare) VALUES (?, ?, ?, ?, ?, ?)',
  [types.LocalDate.now(), new Date(), types.Uuid.random(), 'মিরপুর-১০', 'গুলশান-২', 250.50],
  { prepare: true, consistency: types.consistencies.localQuorum },
);
```

---

### ৪. Graph Database (Neo4j)

যখন data এর **সম্পর্ক** data এর চেয়ে বেশি গুরুত্বপূর্ণ। SQL এ ৩-৪ level deep JOIN অত্যন্ত ধীর, graph traversal constant-time।

```cypher
// Social Network (বাংলাদেশ)
CREATE (rahim:Person {name: 'রহিম', city: 'ঢাকা'})
CREATE (karim:Person {name: 'করিম', city: 'চট্টগ্রাম'})
CREATE (rahim)-[:FRIEND {since: 2020}]->(karim)
CREATE (karim)-[:WORKS_AT {role: 'Dev'}]->(:Company {name: 'Pathao'})

// Friend-of-Friend recommendation
MATCH (me:Person {name: 'রহিম'})-[:FRIEND]->(f)-[:FRIEND]->(suggestion)
WHERE NOT (me)-[:FRIEND]->(suggestion) AND me <> suggestion
RETURN suggestion.name, COUNT(f) AS mutual_friends ORDER BY mutual_friends DESC

// Fraud Detection
MATCH (a:Account)-[t:TRANSFER]->(b:Account)-[t2:TRANSFER]->(c:Account)
WHERE t.amount > 100000 AND a.id = c.id  -- টাকা ঘুরে ফিরে এসেছে
RETURN a.id, collect(DISTINCT b.id) AS intermediaries
```

**PHP:**

```php
use Laudis\Neo4j\ClientBuilder;
$client = ClientBuilder::create()
    ->withDriver('neo4j', 'neo4j://localhost:7687', authenticate: Authenticate::basic('neo4j', 'pass'))
    ->build();

$result = $client->run(
    'MATCH (me:Person {name: $name})-[:FRIEND]->(f)-[:FRIEND]->(sug)
     WHERE NOT (me)-[:FRIEND]->(sug) AND me <> sug
     RETURN sug.name AS name, COUNT(f) AS mutual ORDER BY mutual DESC LIMIT 10',
    ['name' => 'রহিম']
);
```

**JavaScript:**

```javascript
import neo4j from 'neo4j-driver';
const driver = neo4j.driver('neo4j://localhost:7687', neo4j.auth.basic('neo4j', 'pass'));

async function findFraud() {
  const session = driver.session();
  try {
    const result = await session.run(`
      MATCH (a:Account)-[t:TRANSFER]->(b)-[t2:TRANSFER]->(c)
      WHERE t.amount > 50000 AND t2.amount > 50000 AND a.id = c.id
      RETURN a.id AS account, collect(DISTINCT b.id) AS intermediaries, sum(t.amount) AS total
    `);
    return result.records.map(r => ({
      account: r.get('account'), total: r.get('total').toNumber(),
    }));
  } finally { await session.close(); }
}
```

---

### ৫. Search Engine (Elasticsearch)

#### Inverted Index কাজের পদ্ধতি

```
Doc 1: "ঢাকায় সেরা মোবাইল দোকান"     Doc 2: "চট্টগ্রামের মোবাইল সার্ভিস"

Inverted Index:  "ঢাকায়" → [1]  |  "মোবাইল" → [1, 2]  |  "চট্টগ্রাম" → [2]
Search "ঢাকায় মোবাইল" → Doc 1 (2 matches, higher score)
```

**PHP (Elasticsearch + Laravel Scout):**

```php
$client = ClientBuilder::create()->setHosts(['localhost:9200'])->build();

// Bangla analyzer সহ mapping
$client->indices()->create(['index' => 'products_bd', 'body' => [
    'settings' => ['analysis' => ['analyzer' => ['bangla' => [
        'type' => 'custom', 'tokenizer' => 'standard',
        'filter' => ['lowercase', 'bengali_stop', 'bengali_stemmer'],
    ]]]],
    'mappings' => ['properties' => [
        'name' => ['type' => 'text', 'analyzer' => 'bangla'],
        'price' => ['type' => 'float'],
        'location' => ['type' => 'geo_point'],
        'category' => ['type' => 'keyword'],
    ]],
]]);

// Complex search — Daraz style
$response = $client->search(['index' => 'products_bd', 'body' => [
    'query' => ['bool' => [
        'must' => [['multi_match' => [
            'query' => 'Samsung মোবাইল', 'fields' => ['name^3', 'description', 'tags^2'], 'fuzziness' => 'AUTO',
        ]]],
        'filter' => [
            ['range' => ['price' => ['gte' => 10000, 'lte' => 100000]]],
            ['term' => ['in_stock' => true]],
            ['geo_distance' => ['distance' => '10km', 'location' => ['lat' => 23.81, 'lon' => 90.41]]],
        ],
    ]],
    'aggs' => [
        'categories' => ['terms' => ['field' => 'category', 'size' => 20]],
        'price_stats' => ['stats' => ['field' => 'price']],
    ],
    'sort' => ['_score', ['rating' => 'desc']],
]]);
```

**JavaScript (@elastic/elasticsearch):**

```javascript
import { Client } from '@elastic/elasticsearch';
const es = new Client({ node: 'http://localhost:9200' });

class ProductSearch {
  async search(query, filters = {}, page = 1, perPage = 20) {
    const must = query ? [{ multi_match: {
      query, fields: ['name^3', 'description', 'tags^2'], fuzziness: 'AUTO',
    }}] : [];
    const filter = [];
    if (filters.category) filter.push({ term: { category: filters.category } });
    if (filters.priceMax) filter.push({ range: { price: { lte: filters.priceMax } } });

    const { hits, aggregations } = await es.search({
      index: 'products_bd',
      body: {
        query: { bool: { must, filter } },
        aggs: { categories: { terms: { field: 'category' } }, avg_price: { avg: { field: 'price' } } },
        highlight: { fields: { name: {}, description: { fragment_size: 150 } } },
        from: (page - 1) * perPage, size: perPage,
      },
    });
    return {
      products: hits.hits.map(h => ({ ...h._source, score: h._score, highlights: h.highlight })),
      total: hits.total.value,
      facets: aggregations,
    };
  }

  async suggest(prefix) {
    const { suggest } = await es.search({ index: 'products_bd', body: {
      suggest: { product: { prefix, completion: { field: 'name_suggest', fuzzy: { fuzziness: 'AUTO' } } } },
    }});
    return suggest.product[0].options.map(o => o.text);
  }
}
```

---

## 🔥 Advanced Scenarios

### ১. Polyglot Persistence

```
┌─────────────────────────────────────────────────┐
│          E-Commerce Architecture                │
│                                                 │
│  MySQL ──── Orders, Payments, Users (ACID)      │
│  Redis ──── Sessions, Cache, Rate Limiting      │
│  Elastic ── Product Search, Analytics           │
│  MongoDB ── Product Catalog, Reviews, CMS       │
│       ↓            ↓           ↓         ↓      │
│       └────── Application Service Layer ────┘   │
└─────────────────────────────────────────────────┘
```

```php
// PHP — Polyglot orchestration
class ProductService
{
    public function __construct(
        private ProductMySQLRepo $mysql,    // transactional
        private ProductMongoRepo $mongo,    // catalog
        private CacheManager $cache,        // Redis
        private SearchService $search,      // Elasticsearch
    ) {}

    public function create(CreateProductDTO $dto): Product
    {
        $product = DB::transaction(fn() => $this->mysql->create($dto)); // ১. MySQL
        $this->mongo->upsert($product->id, $dto->catalogData());       // ২. MongoDB
        $this->search->index($product->id, $dto->searchData());        // ৩. Elasticsearch
        $this->cache->warmUp("product:{$product->id}");                // ৪. Redis
        return $product;
    }

    public function get(string $id): array
    {
        return $this->cache->getOrSet("product:{$id}", fn() => array_merge(
            $this->mysql->find($id)->toArray(),
            $this->mongo->find($id),
        ), ttl: 1800);
    }
}
```

```javascript
// JS — Polyglot
class ProductService {
  constructor({ mysql, mongo, redis, es }) {
    Object.assign(this, { mysql, mongo, redis, es });
  }

  async create(data) {
    const conn = await this.mysql.getConnection();
    try {
      await conn.beginTransaction();
      const [r] = await conn.execute('INSERT INTO products (sku,price,stock) VALUES (?,?,?)', [data.sku, data.price, data.stock]);
      await conn.commit();
      const id = r.insertId;
      await Promise.all([
        this.mongo.collection('products').updateOne({ product_id: id }, { $set: data.catalog }, { upsert: true }),
        this.es.index({ index: 'products', id: String(id), body: data.search }),
        this.redis.setex(`product:${id}`, 1800, JSON.stringify({ id, ...data })),
      ]);
      return { id, ...data };
    } catch (e) { await conn.rollback(); throw e; }
    finally { conn.release(); }
  }
}
```

### ২. Data Modeling Patterns

**Bucket Pattern (Time-Series):**
```javascript
// ❌ প্রতি reading এ document → অদক্ষ
{ sensor_id: 'temp_01', timestamp: '10:00', value: 32.5 }

// ✅ Bucket: ১ ঘণ্টার data একটি document এ
{
  sensor_id: 'temp_01', bucket: '2024-01-15T10:00',
  readings: [{ t: '10:00', v: 32.5 }, { t: '10:01', v: 32.7 }],
  summary: { min: 31.2, max: 33.8, avg: 32.4, count: 60 }
}
```

**Outlier Pattern:** বেশিরভাগ document ছোট, কিছু অস্বাভাবিকভাবে বড় হলে `has_overflow: true` flag দিয়ে আলাদা collection এ overflow data রাখুন।

**Tree Structures (Materialized Path):**
```javascript
// Category hierarchy — Daraz style
{ _id: 'phones', name: 'মোবাইল', path: ',electronics,phones,' }
{ _id: 'samsung', name: 'Samsung', path: ',electronics,phones,samsung,' }
// Samsung এর সব sub-category: db.categories.find({ path: /,samsung,/ })
```

### ৩. Scaling NoSQL

**Consistent Hashing:** Ring এ node বসানো হয়, key hash করে নিকটতম node এ data যায়। Node যোগ/বাদ হলে শুধু পার্শ্ববর্তী data move হয়।

**MongoDB Sharding:**
```javascript
// ✅ ভালো shard key: high cardinality, even distribution
sh.shardCollection('ecommerce.orders', { customer_region: 1, order_date: 1 });
// ❌ খারাপ: monotonically increasing → hotspot
sh.shardCollection('ecommerce.orders', { _id: 1 });
```

**Redis Cluster:**
```javascript
const cluster = new Redis.Cluster([
  { host: 'node-1', port: 6379 }, { host: 'node-2', port: 6379 },
], { scaleReads: 'slave' });
// Hash tags — সম্পর্কিত keys একই slot এ
await cluster.set('{user:123}:profile', data);
await cluster.set('{user:123}:cart', cart);
```

### ৪. SQL → NoSQL Migration

**Dual-Write Pattern (ক্রমশ migration):**

```php
class DualWriteRepo
{
    private string $readFrom = 'sql'; // Phase toggle

    public function create(array $data): Product
    {
        $product = DB::transaction(fn() => SQLProduct::create($data)); // Phase 1: SQL primary
        dispatch(new SyncToMongo('create', $product->id, $data));      // Async MongoDB sync
        return $product;
    }

    public function find(string $id): array
    {
        // Phase 2: Shadow Read — দুটো থেকে পড়ো, mismatch log করো
        $sql = SQLProduct::find($id)->toArray();
        $mongo = MongoProduct::find($id)->toArray();
        if ($sql != $mongo) Log::warning('Data mismatch', compact('id', 'sql', 'mongo'));
        return $this->readFrom === 'sql' ? $sql : $mongo;
        // Phase 3: MongoDB primary → Phase 4: SQL বন্ধ
    }
}
```

---

## 🆚 NoSQL Database Comparison Table

| মানদণ্ড | MongoDB | Redis | Cassandra | Neo4j | Elasticsearch |
|---------|---------|-------|-----------|-------|--------------|
| **Type** | Document | Key-Value | Wide-Column | Graph | Search |
| **CAP** | CP | CP | AP | CA/CP | AP |
| **Query** | MQL | Commands | CQL | Cypher | Query DSL |
| **Schema** | Flexible | Schema-less | Flexible | Schema-less | Mapping |
| **Scaling** | Sharding | Cluster | Linear | Causal cluster | Sharding |
| **Write Speed** | Medium | Extreme | Extreme | Medium | Medium |
| **Read Speed** | Fast | Extreme | Fast (by key) | Graph traversal | Full-text |
| **Data Size** | TB-PB | GB-TB (RAM) | PB | GB-TB | TB |
| **Transactions** | Multi-doc ACID | MULTI/Lua | Lightweight | ACID | না |
| **License** | SSPL | BSD | Apache 2.0 | GPL/Comm. | SSPL |
| **Learning** | সহজ | সহজ | মাঝারি | মাঝারি | কঠিন |

---

## ✅ সুবিধা ও ❌ অসুবিধা

| Database | ✅ সুবিধা | ❌ অসুবিধা |
|----------|-----------|-----------|
| **MongoDB** | Flexible schema, rich queries, Atlas managed | JOIN নেই, memory বেশি, SSPL license |
| **Redis** | Sub-ms latency, বৈচিত্র্যময় structures | RAM-bound ব্যয়বহুল, persistence সীমিত |
| **Cassandra** | Linear scale, multi-DC, write-optimized | Query flexibility কম, modeling কঠিন |
| **Neo4j** | Relationship queries blazing fast, ACID | Horizontal scaling কঠিন, RAM নির্ভর |
| **Elasticsearch** | Full-text search অতুলনীয়, analytics | Primary DB অনুপযুক্ত, resource hungry |

---

## ⚠️ সাধারণ ভুল

**১. "NoSQL মানেই ভালো"** — Financial transactions এ PostgreSQL আজও সেরা। সঠিক tool সঠিক কাজে।

**২. MongoDB তে SQL mindset** — ৪টি collection JOIN করা ভুল। Query pattern অনুযায়ী embed করুন।

**৩. Redis কে primary database ভাবা** — Power failure তে data হারানোর ঝুঁকি। Persistent DB backup রাখুন।

**৪. Cassandra তে অতিরিক্ত secondary index** — Query-first design করুন, প্রতি query pattern এ আলাদা table।

**৫. Elasticsearch কে source of truth ভাবা** — Search/analytics এর জন্য, primary DB আলাদা রাখুন।

**৬. ভুল shard/partition key** — Monotonically increasing key → hotspot। High cardinality নিশ্চিত করুন।

---

## 📏 কোন NoSQL কখন — Decision Tree

```
আপনার প্রধান চাহিদা কী?
├── Full-text search / Log analytics? → Elasticsearch
│   (Chaldal product search, Prothom Alo articles)
├── Cache / Real-time / Session / Queue? → Redis
│   (bKash session, Pathao location, API rate limit)
├── Complex relationships / Graph traversal? → Neo4j
│   (Friend suggestions, fraud detection)
├── Massive writes / Time-series / Multi-DC? → Cassandra
│   (IoT sensor, ride history, messaging)
├── Flexible schema / Documents / General purpose? → MongoDB
│   (E-commerce catalog, CMS, user profiles)
└── Strong consistency / Complex transactions? → PostgreSQL/MySQL!
    (bKash ledger, Banking, Inventory)
```

### বাংলাদেশ Context

| সেবা | ব্যবহার | Database |
|------|---------|----------|
| **bKash** | Transaction ledger | PostgreSQL (ACID) |
| **bKash** | Session/OTP cache | Redis (TTL) |
| **Pathao** | Real-time location | Redis (Geo) |
| **Pathao** | Ride history | Cassandra (time-series) |
| **Daraz** | Product catalog | MongoDB |
| **Daraz** | Product search | Elasticsearch |
| **Social App** | Connections | Neo4j |
| **IoT** | ঢাকার বায়ু মান | Cassandra |

---

## 📋 সারসংক্ষেপ

1. **NoSQL একটি tool, solution নয়।** ভুল database বাছাইয়ের মূল্য অনেক বেশি।
2. **CAP Theorem বুঝুন।** CP নাকি AP — business requirement অনুযায়ী সিদ্ধান্ত।
3. **Data modeling সবচেয়ে গুরুত্বপূর্ণ।** Flexible schema মানে "যেকোনো schema" নয়।
4. **Polyglot Persistence গ্রহণ করুন।** প্রতি কাজে উপযুক্ত tool।
5. **Scaling plan আগেই করুন।** Shard key পরে পরিবর্তন অত্যন্ত কষ্টকর।

```
MongoDB  → Documents, flexible, general purpose
Redis    → In-memory, cache, real-time, queues
Cassandra→ Write-heavy, time-series, multi-DC
Neo4j    → Graphs, relationships, traversals
Elastic  → Search, full-text, analytics, logs
```

> **"প্রতিটি database একটি বিশেষ সমস্যা সমাধানের জন্য তৈরি — আপনার toolbox বড় করুন।"**
