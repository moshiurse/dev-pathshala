# 🗄️ ক্যাশিং (Caching)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**ক্যাশিং** হলো ডেটা বা কম্পিউটেশনের ফলাফল একটি দ্রুত-অ্যাক্সেসযোগ্য স্টোরেজ লেয়ারে সাময়িকভাবে সংরক্ষণ করার প্রক্রিয়া, যাতে ভবিষ্যতের অনুরোধগুলো মূল ডেটা সোর্সে না গিয়েই দ্রুত পূরণ করা যায়। এটি সিস্টেম ডিজাইনের সবচেয়ে গুরুত্বপূর্ণ পারফরম্যান্স অপটিমাইজেশন কৌশল।

### কেন ক্যাশিং দরকার?

```
বাস্তব উদাহরণ — Daraz Bangladesh:

ক্যাশ ছাড়া:
  ইউজার → প্রোডাক্ট পেজ → DB Query (50ms) → Response
  প্রতি সেকেন্ডে 10,000 রিকোয়েস্ট = 10,000 DB queries/sec 💀

ক্যাশ সহ:
  ইউজার → প্রোডাক্ট পেজ → Redis (0.5ms) → Response
  DB load 95% কমে গেল ✅
```

### Cache Hit vs Cache Miss

```
┌─────────────────────────────────────────────────────────┐
│                    CACHE HIT (দ্রুত)                      │
│                                                         │
│  Client ──→ Cache ──→ ডেটা পাওয়া গেছে! ──→ Response    │
│              ✅         (0.5ms latency)                  │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                    CACHE MISS (ধীর)                      │
│                                                         │
│  Client ──→ Cache ──→ ডেটা নেই! ──→ DB Query ──→ Cache  │
│              ❌        (50-200ms)       │       Update   │
│                                        ↓                │
│                                    Response             │
└─────────────────────────────────────────────────────────┘
```

### Cache Hit Ratio (ক্যাশ হিট অনুপাত)

```
Cache Hit Ratio = (Cache Hits) / (Cache Hits + Cache Misses) × 100%

উদাহরণ:
  1000 রিকোয়েস্ট → 950 হিট + 50 মিস
  Hit Ratio = 950/1000 = 95% ← খুবই ভালো!

লক্ষ্যমাত্রা:
  > 95%  → চমৎকার (Excellent)
  > 90%  → ভালো (Good)
  > 80%  → গ্রহণযোগ্য (Acceptable)
  < 80%  → ক্যাশিং কৌশল পুনর্বিবেচনা দরকার
```

---

## 📊 Caching Layers Architecture

একটি প্রোডাকশন সিস্টেমে ক্যাশিং একাধিক স্তরে কাজ করে। প্রতিটি স্তর নির্দিষ্ট ধরনের ডেটা ক্যাশ করে এবং সামগ্রিক পারফরম্যান্স উন্নত করে:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    সম্পূর্ণ ক্যাশিং আর্কিটেকচার                       │
│                                                                      │
│  ┌──────────┐   ┌───────────┐   ┌──────┐   ┌─────────┐   ┌───────┐ │
│  │ Browser  │──→│    CDN    │──→│  LB  │──→│   App   │──→│ Cache │ │
│  │  Cache   │   │ (CF/CFN)  │   │      │   │ Server  │   │(Redis)│ │
│  │  L1      │   │  L2       │   │      │   │  L3     │   │  L4   │ │
│  └──────────┘   └───────────┘   └──────┘   └─────────┘   └───┬───┘ │
│       │               │                         │             │     │
│   Service          Edge         Nginx/      In-Memory      Remote   │
│   Worker          Server        HAProxy     (OPcache/      Cache    │
│   Cache-          Static +                  node-cache)             │
│   Control         Dynamic                                    │     │
│   ETag            Content                                    ↓     │
│                                                         ┌───────┐  │
│                                                         │  DB   │  │
│                                                         │ Query │  │
│                                                         │ Cache │  │
│                                                         │  L5   │  │
│                                                         └───┬───┘  │
│                                                             ↓      │
│                                                         ┌───────┐  │
│                                                         │  DB   │  │
│                                                         │(MySQL/│  │
│                                                         │Postgres│ │
│                                                         └───────┘  │
└──────────────────────────────────────────────────────────────────────┘

প্রতিটি লেয়ারে Latency:
  L1 Browser     : ~0ms (লোকাল মেমরি)
  L2 CDN         : ~5-20ms (এজ সার্ভার, সিঙ্গাপুর/মুম্বাই)
  L3 App Memory  : ~0.1ms (প্রসেস মেমরি)
  L4 Redis       : ~0.5-2ms (নেটওয়ার্ক হপ)
  L5 DB Cache    : ~5-10ms (কোয়েরি ক্যাশ)
  DB Direct      : ~20-200ms (ডিস্ক I/O)
```

---

## 💻 Caching Strategies Deep Dive

### ১. Cache-Aside (Lazy Loading)

সবচেয়ে বহুল ব্যবহৃত কৌশল। অ্যাপ্লিকেশন নিজেই ক্যাশ ম্যানেজ করে — প্রথমে ক্যাশ চেক করে, না পেলে DB থেকে এনে ক্যাশে রাখে।

```
┌──────────────────────────────────────────────────────┐
│              Cache-Aside Flow                        │
│                                                      │
│  App ──[1. GET]──→ Cache                             │
│   │                  │                               │
│   │            ┌─────┴─────┐                         │
│   │           HIT         MISS                       │
│   │            │            │                        │
│   │       [2. Return]  [3. Query DB]                 │
│   │         Data           │                         │
│   │                   [4. SET Cache]                  │
│   │                        │                         │
│   ←────────────────── [5. Return Data]               │
└──────────────────────────────────────────────────────┘
```

**PHP (Laravel):**

```php
<?php

namespace App\Services;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class ProductCacheService
{
    private const CACHE_TTL = 3600; // ১ ঘণ্টা
    private const CACHE_PREFIX = 'product:';

    /**
     * Cache-Aside প্যাটার্ন — Daraz-এর মতো ই-কমার্স প্রোডাক্ট ক্যাশিং
     */
    public function getProduct(int $productId): ?Product
    {
        $cacheKey = self::CACHE_PREFIX . $productId;

        // ধাপ ১: ক্যাশ চেক
        $cached = Cache::get($cacheKey);

        if ($cached !== null) {
            Log::debug("Cache HIT: {$cacheKey}");
            return $cached;
        }

        Log::debug("Cache MISS: {$cacheKey}");

        // ধাপ ২: DB থেকে ডেটা আনা
        $product = Product::with(['category', 'variants', 'reviews'])
            ->find($productId);

        if ($product === null) {
            // নেগেটিভ ক্যাশিং — DB-তে না থাকলে ছোট TTL দিয়ে ক্যাশ করো
            Cache::put($cacheKey, null, 300);
            return null;
        }

        // ধাপ ৩: ক্যাশে সংরক্ষণ
        Cache::put($cacheKey, $product, self::CACHE_TTL);

        return $product;
    }

    /**
     * ক্যাশ ইনভ্যালিডেশন — প্রোডাক্ট আপডেট হলে
     */
    public function invalidateProduct(int $productId): void
    {
        $cacheKey = self::CACHE_PREFIX . $productId;
        Cache::forget($cacheKey);

        // সম্পর্কিত ক্যাশও ইনভ্যালিডেট করো
        Cache::tags(['product-listings'])->flush();
    }

    /**
     * Laravel Cache::remember ব্যবহার করে সংক্ষিপ্ত রূপ
     */
    public function getProductSimple(int $productId): ?Product
    {
        return Cache::remember(
            self::CACHE_PREFIX . $productId,
            self::CACHE_TTL,
            fn () => Product::with('category')->find($productId)
        );
    }
}
```

**JavaScript (Node.js):**

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: '127.0.0.1', port: 6379 });

class ProductCacheService {
    static CACHE_TTL = 3600;
    static CACHE_PREFIX = 'product:';

    /**
     * Cache-Aside — Node.js ভার্সন
     * Daraz/Chaldal-এর প্রোডাক্ট ক্যাশিং সিমুলেশন
     */
    async getProduct(productId) {
        const cacheKey = `${ProductCacheService.CACHE_PREFIX}${productId}`;

        // ধাপ ১: Redis থেকে চেক
        const cached = await redis.get(cacheKey);

        if (cached) {
            console.log(`Cache HIT: ${cacheKey}`);
            return JSON.parse(cached);
        }

        console.log(`Cache MISS: ${cacheKey}`);

        // ধাপ ২: ডেটাবেজ কোয়েরি
        const product = await db.query(
            `SELECT p.*, c.name as category_name
             FROM products p
             JOIN categories c ON p.category_id = c.id
             WHERE p.id = $1`,
            [productId]
        );

        if (!product) {
            // নেগেটিভ ক্যাশিং — cache penetration প্রতিরোধ
            await redis.setex(cacheKey, 300, JSON.stringify(null));
            return null;
        }

        // ধাপ ৩: ক্যাশে সেট
        await redis.setex(
            cacheKey,
            ProductCacheService.CACHE_TTL,
            JSON.stringify(product)
        );

        return product;
    }

    async invalidateProduct(productId) {
        const cacheKey = `${ProductCacheService.CACHE_PREFIX}${productId}`;
        await redis.del(cacheKey);

        // প্যাটার্ন ম্যাচিং দিয়ে সম্পর্কিত ক্যাশ মুছো
        const relatedKeys = await redis.keys(`product-list:*`);
        if (relatedKeys.length > 0) {
            await redis.del(...relatedKeys);
        }
    }
}
```

**সুবিধা:** সহজ বাস্তবায়ন, শুধু প্রয়োজনীয় ডেটা ক্যাশ হয়, ক্যাশ ফেইলুরে সিস্টেম চলতে থাকে।
**অসুবিধা:** প্রথম রিকোয়েস্ট সবসময় ধীর (cold miss), ক্যাশ ও DB-র মধ্যে stale data হতে পারে।

---

### ২. Read-Through Cache

Cache-Aside-এর মতো, কিন্তু এখানে ক্যাশ লাইব্রেরি/প্রভাইডার নিজেই DB থেকে ডেটা লোড করে। অ্যাপ্লিকেশন শুধু ক্যাশকে জিজ্ঞেস করে।

```
┌──────────────────────────────────────────────────────┐
│             Read-Through Flow                        │
│                                                      │
│  App ──[1. GET]──→ Cache Provider                    │
│   ↑                    │                             │
│   │              ┌─────┴─────┐                       │
│   │             HIT         MISS                     │
│   │              │            │                      │
│   │         Return Data   [2. Cache নিজেই            │
│   │                        DB Query করে]             │
│   │                           │                      │
│   │                     [3. Cache-এ                   │
│   │                      স্টোর করে]                   │
│   │                           │                      │
│   ←───────────────── [4. Return Data]                │
│                                                      │
│  পার্থক্য: App জানে না ডেটা কোথা থেকে এলো!          │
└──────────────────────────────────────────────────────┘
```

**PHP (Laravel):**

```php
<?php

namespace App\Cache;

use Illuminate\Support\Facades\Redis;

class ReadThroughCache
{
    private array $loaders = [];

    /**
     * লোডার রেজিস্টার — নির্দিষ্ট key prefix-এর জন্য DB লোডার ফাংশন সেট
     */
    public function registerLoader(string $prefix, callable $loader): void
    {
        $this->loaders[$prefix] = $loader;
    }

    /**
     * Read-Through — ক্যাশ নিজেই সোর্স থেকে ডেটা আনবে
     */
    public function get(string $key, int $ttl = 3600): mixed
    {
        // ক্যাশ চেক
        $cached = Redis::get($key);
        if ($cached !== null) {
            return unserialize($cached);
        }

        // কোন লোডার ম্যাচ করে বের করো
        $loader = $this->findLoader($key);
        if ($loader === null) {
            throw new \RuntimeException("No loader registered for key: {$key}");
        }

        // ক্যাশ প্রোভাইডার নিজেই ডেটা লোড করে
        $data = $loader($key);
        Redis::setex($key, $ttl, serialize($data));

        return $data;
    }

    private function findLoader(string $key): ?callable
    {
        foreach ($this->loaders as $prefix => $loader) {
            if (str_starts_with($key, $prefix)) {
                return $loader;
            }
        }
        return null;
    }
}

// ব্যবহার:
$cache = new ReadThroughCache();

$cache->registerLoader('user:', function (string $key) {
    $userId = (int) str_replace('user:', '', $key);
    return User::find($userId);
});

$user = $cache->get('user:123'); // ক্যাশে থাকলে ক্যাশ থেকে, না থাকলে DB
```

**JavaScript (Node.js):**

```javascript
class ReadThroughCache {
    constructor(redisClient) {
        this.redis = redisClient;
        this.loaders = new Map();
    }

    registerLoader(prefix, loaderFn) {
        this.loaders.set(prefix, loaderFn);
    }

    async get(key, ttl = 3600) {
        const cached = await this.redis.get(key);
        if (cached) {
            return JSON.parse(cached);
        }

        const loader = this.findLoader(key);
        if (!loader) {
            throw new Error(`No loader for key: ${key}`);
        }

        const data = await loader(key);
        await this.redis.setex(key, ttl, JSON.stringify(data));
        return data;
    }

    findLoader(key) {
        for (const [prefix, loader] of this.loaders) {
            if (key.startsWith(prefix)) return loader;
        }
        return null;
    }
}

// ব্যবহার — bKash ইউজার ব্যালেন্স ক্যাশ
const cache = new ReadThroughCache(redis);

cache.registerLoader('balance:', async (key) => {
    const userId = key.replace('balance:', '');
    const result = await db.query(
        'SELECT balance FROM wallets WHERE user_id = $1',
        [userId]
    );
    return result.rows[0];
});

const balance = await cache.get('balance:user_42'); // স্বয়ংক্রিয়ভাবে DB থেকে লোড
```

**সুবিধা:** অ্যাপ্লিকেশন কোড সরল, ক্যাশ লজিক centralized।
**অসুবিধা:** প্রথম লোড ধীর, কাস্টম ক্যাশ প্রোভাইডার বানাতে হয়।

---

### ৩. Write-Through Cache

ডেটা লেখার সময় একই সাথে ক্যাশ এবং DB — দুটোতেই লেখা হয়। ক্যাশ সবসময় DB-র সাথে sync থাকে।

```
┌──────────────────────────────────────────────────────┐
│            Write-Through Flow                        │
│                                                      │
│  App ──[1. WRITE]──→ Cache Provider                  │
│   ↑                       │                          │
│   │                  [2. Cache-এ                      │
│   │                   স্টোর করো]                      │
│   │                       │                          │
│   │                  [3. DB-তেও                       │
│   │                   লেখো]                           │
│   │                       │                          │
│   ←──────────────── [4. ACK]                         │
│                                                      │
│  ⚠️ দুটোতেই সফল না হলে রোলব্যাক!                     │
└──────────────────────────────────────────────────────┘
```

**PHP (Laravel):**

```php
<?php

namespace App\Cache;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;

class WriteThroughCache
{
    /**
     * Write-Through — ক্যাশ ও DB সমসাময়িকভাবে আপডেট
     * bKash ব্যালেন্স আপডেটের মতো critical ডেটার জন্য উপযুক্ত
     */
    public function put(string $key, mixed $data, string $table, array $conditions): bool
    {
        return DB::transaction(function () use ($key, $data, $table, $conditions) {
            // ধাপ ১: DB-তে লেখো
            DB::table($table)
                ->where($conditions)
                ->update(is_array($data) ? $data : ['value' => $data]);

            // ধাপ ২: ক্যাশে লেখো
            Cache::put($key, $data, now()->addHour());

            return true;
        });
    }

    /**
     * bKash-এর মতো ওয়ালেট ব্যালেন্স আপডেট
     */
    public function updateWalletBalance(int $userId, float $newBalance): bool
    {
        $cacheKey = "wallet:balance:{$userId}";

        return DB::transaction(function () use ($userId, $newBalance, $cacheKey) {
            DB::table('wallets')
                ->where('user_id', $userId)
                ->update([
                    'balance'    => $newBalance,
                    'updated_at' => now(),
                ]);

            Cache::put($cacheKey, $newBalance, now()->addMinutes(30));

            return true;
        });
    }
}
```

**JavaScript (Node.js):**

```javascript
class WriteThroughCache {
    constructor(redis, db) {
        this.redis = redis;
        this.db = db;
    }

    /**
     * Write-Through — ট্রানজেকশনাল আপডেট
     */
    async put(key, data, tableName, conditions, ttl = 3600) {
        const client = await this.db.connect();

        try {
            await client.query('BEGIN');

            // DB আপডেট
            const setClauses = Object.keys(data)
                .map((k, i) => `${k} = $${i + 1}`)
                .join(', ');
            const values = [...Object.values(data), ...Object.values(conditions)];

            await client.query(
                `UPDATE ${tableName} SET ${setClauses}
                 WHERE ${Object.keys(conditions).map((k, i) =>
                    `${k} = $${Object.keys(data).length + i + 1}`
                 ).join(' AND ')}`,
                values
            );

            // ক্যাশ আপডেট
            await this.redis.setex(key, ttl, JSON.stringify(data));

            await client.query('COMMIT');
            return true;
        } catch (error) {
            await client.query('ROLLBACK');
            // ক্যাশ ইনভ্যালিডেট — consistency নিশ্চিত করো
            await this.redis.del(key);
            throw error;
        } finally {
            client.release();
        }
    }
}

// ব্যবহার — প্রোডাক্ট স্টক আপডেট (Daraz flash sale)
const wtCache = new WriteThroughCache(redis, pool);
await wtCache.put(
    'product:stock:456',
    { stock_count: 99, updated_at: new Date().toISOString() },
    'products',
    { id: 456 }
);
```

**সুবিধা:** ক্যাশ সবসময় fresh, read পারফরম্যান্স চমৎকার।
**অসুবিধা:** write latency বেশি (দুবার লেখা), অপ্রয়োজনীয় ডেটাও ক্যাশ হতে পারে।

---

### ৪. Write-Behind (Write-Back)

ডেটা প্রথমে শুধু ক্যাশে লেখা হয় এবং পরবর্তীতে asynchronously DB-তে লেখা হয়। অত্যন্ত দ্রুত write পারফরম্যান্স, কিন্তু ডেটা হারানোর ঝুঁকি আছে।

```
┌──────────────────────────────────────────────────────┐
│           Write-Behind (Write-Back) Flow             │
│                                                      │
│  App ──[1. WRITE]──→ Cache                           │
│   ↑                    │                             │
│   │              [2. তাৎক্ষণিক ACK]                   │
│   ←────────────────────┘                             │
│                                                      │
│              ... সময় পরে (async) ...                 │
│                                                      │
│                  Cache ──[3. Batch Write]──→ DB       │
│                                                      │
│  ⚠️ ক্যাশ ক্র্যাশ হলে uncommitted ডেটা হারাবে!       │
└──────────────────────────────────────────────────────┘
```

**PHP (Laravel):**

```php
<?php

namespace App\Cache;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Queue;
use App\Jobs\PersistToDatabaseJob;

class WriteBehindCache
{
    /**
     * Write-Behind — তাৎক্ষণিক ক্যাশ আপডেট, পরে async DB write
     * নিউজ সাইটের view count-এর জন্য আদর্শ (Prothom Alo, BD News 24)
     */
    public function put(string $key, mixed $data, int $ttl = 3600): void
    {
        // ধাপ ১: শুধু ক্যাশে লেখো (অতি দ্রুত)
        Cache::put($key, $data, $ttl);

        // ধাপ ২: কিউতে DB write জব পাঠাও
        Queue::push(new PersistToDatabaseJob($key, $data));
    }

    /**
     * ব্যাচ write — একসাথে অনেক আপডেট
     * যেমন: আর্টিকেল ভিউ কাউন্ট ব্যাচে DB-তে সেভ
     */
    public function incrementViewCount(int $articleId): void
    {
        $cacheKey = "article:views:{$articleId}";

        // Redis INCR — atomic ও দ্রুত
        $newCount = Cache::increment($cacheKey);

        // প্রতি ১০০ ভিউতে DB-তে persist করো
        if ($newCount % 100 === 0) {
            Queue::push(new PersistToDatabaseJob($cacheKey, $newCount));
        }
    }
}
```

**JavaScript (Node.js):**

```javascript
const Bull = require('bull');
const writeQueue = new Bull('db-write-queue', { redis: { port: 6379 } });

class WriteBehindCache {
    constructor(redis) {
        this.redis = redis;
        this.buffer = new Map();
        this.flushInterval = setInterval(() => this.flush(), 5000);
    }

    async put(key, data, ttl = 3600) {
        // তাৎক্ষণিক ক্যাশ আপডেট
        await this.redis.setex(key, ttl, JSON.stringify(data));

        // বাফারে জমা করো (ব্যাচ write-এর জন্য)
        this.buffer.set(key, { data, timestamp: Date.now() });
    }

    /**
     * বাফার ফ্লাশ — জমানো ডেটা DB-তে পাঠাও
     * নিউজ সাইটের page view count batching
     */
    async flush() {
        if (this.buffer.size === 0) return;

        const entries = Array.from(this.buffer.entries());
        this.buffer.clear();

        // কিউতে ব্যাচ জব পাঠাও
        await writeQueue.add('batch-persist', {
            entries: entries.map(([key, val]) => ({
                key, data: val.data, timestamp: val.timestamp
            }))
        });
    }

    // ভিউ কাউন্ট ইনক্রিমেন্ট — BD News 24 এর মতো
    async incrementViews(articleId) {
        const key = `article:views:${articleId}`;
        const newCount = await this.redis.incr(key);

        if (newCount % 100 === 0) {
            await writeQueue.add('persist-views', {
                articleId, count: newCount
            });
        }
        return newCount;
    }
}

// কিউ প্রসেসর
writeQueue.process('batch-persist', async (job) => {
    const { entries } = job.data;
    const client = await pool.connect();
    try {
        await client.query('BEGIN');
        for (const entry of entries) {
            await client.query(
                'INSERT INTO cache_writes (key, data) VALUES ($1, $2) ON CONFLICT (key) DO UPDATE SET data = $2',
                [entry.key, JSON.stringify(entry.data)]
            );
        }
        await client.query('COMMIT');
    } catch (e) {
        await client.query('ROLLBACK');
        throw e;
    } finally {
        client.release();
    }
});
```

**সুবিধা:** অত্যন্ত দ্রুত write, DB load কমায়, batch write দিয়ে throughput বাড়ায়।
**অসুবিধা:** ডেটা হারানোর ঝুঁকি, শেষ পর্যন্ত consistent (eventual consistency), জটিল বাস্তবায়ন।

---

### ৫. Refresh-Ahead

ক্যাশ expire হওয়ার আগেই background-এ নতুন ডেটা লোড করে রাখে। ইউজার কখনো cache miss দেখে না।

```
┌──────────────────────────────────────────────────────┐
│            Refresh-Ahead Flow                        │
│                                                      │
│  TTL = 60 সেকেন্ড, Refresh Threshold = 50 সেকেন্ড   │
│                                                      │
│  সময়:  0s        50s           60s                   │
│   ├──────────┼───────────────┤                        │
│   ↑          ↑               ↑                       │
│   SET     Background      Expire                     │
│           Refresh         (কিন্তু আগেই                │
│           শুরু            refresh হয়ে                  │
│                           গেছে!)                      │
│                                                      │
│  ইউজার সবসময় ক্যাশ থেকে ডেটা পায় ✅                 │
└──────────────────────────────────────────────────────┘
```

**PHP (Laravel) — Scheduled Refresh:**

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

class RefreshAheadCache extends Command
{
    protected $signature = 'cache:refresh-ahead';
    protected $description = 'ক্যাশ expire হওয়ার আগে রিফ্রেশ করো';

    public function handle(): void
    {
        // ট্র্যাক করা কী-গুলোর TTL চেক করো
        $trackedKeys = Redis::smembers('cache:tracked-keys');

        foreach ($trackedKeys as $key) {
            $ttl = Redis::ttl($key);
            $originalTtl = Redis::hget('cache:meta:' . $key, 'original_ttl') ?? 3600;
            $threshold = $originalTtl * 0.2; // ২০% TTL বাকি থাকলে refresh

            if ($ttl > 0 && $ttl < $threshold) {
                $this->refreshKey($key, $originalTtl);
            }
        }
    }

    private function refreshKey(string $key, int $ttl): void
    {
        $loader = $this->resolveLoader($key);
        if ($loader) {
            $freshData = $loader();
            Cache::put($key, $freshData, $ttl);
            $this->info("Refreshed: {$key}");
        }
    }
}
```

**সুবিধা:** ইউজার কখনো cache miss-এর latency পায় না।
**অসুবিধা:** কোন ডেটা refresh করতে হবে সেটা predict করা কঠিন, অপ্রয়োজনীয় refresh হতে পারে।

---

## 🔧 Cache Technologies

### Redis Deep Dive

Redis হলো ক্যাশিংয়ের জন্য সবচেয়ে জনপ্রিয় in-memory data store। এটি শুধু key-value নয়, একটি সম্পূর্ণ ডেটা স্ট্রাকচার সার্ভার।

#### ডেটা টাইপ এবং ব্যবহার

```
┌─────────────────────────────────────────────────────────────┐
│                  Redis Data Types                           │
│                                                             │
│  String   → সাধারণ ক্যাশিং, কাউন্টার, সেশন                  │
│  Hash     → অবজেক্ট/ইউজার প্রোফাইল ক্যাশিং                  │
│  List     → মেসেজ কিউ, রিসেন্ট আইটেম                        │
│  Set      → ইউনিক ভিজিটর ট্র্যাকিং, ট্যাগিং                  │
│  Sorted   → লিডারবোর্ড, রেট লিমিটিং,                        │
│  Set        সময় ভিত্তিক ইভেন্ট                               │
│  Stream   → ইভেন্ট সোর্সিং, রিয়েল-টাইম ফিড                  │
│  Bitmap   → ইউজার অ্যাক্টিভিটি ট্র্যাকিং                     │
│  HyperLog → ইউনিক কাউন্ট (approximate)                      │
└─────────────────────────────────────────────────────────────┘
```

**PHP — Redis Advanced Usage:**

```php
<?php

use Illuminate\Support\Facades\Redis;

class RedisAdvancedService
{
    // Hash — ইউজার প্রোফাইল ক্যাশিং
    public function cacheUserProfile(int $userId, array $profile): void
    {
        Redis::hmset("user:{$userId}", $profile);
        Redis::expire("user:{$userId}", 3600);
    }

    // Sorted Set — Daraz প্রোডাক্ট ট্রেন্ডিং লিডারবোর্ড
    public function addToTrending(int $productId, float $score): void
    {
        Redis::zadd('trending:products', $score, $productId);
        Redis::zremrangebyrank('trending:products', 0, -101); // শীর্ষ ১০০ রাখো
    }

    public function getTrendingProducts(int $limit = 10): array
    {
        return Redis::zrevrange('trending:products', 0, $limit - 1, 'WITHSCORES');
    }

    // Pub/Sub — রিয়েল-টাইম ক্যাশ ইনভ্যালিডেশন
    public function publishInvalidation(string $key): void
    {
        Redis::publish('cache:invalidation', json_encode([
            'key'       => $key,
            'timestamp' => now()->timestamp,
            'server'    => gethostname(),
        ]));
    }

    // Lua Script — Atomic অপারেশন (race condition এড়ানো)
    public function deductStockAtomic(int $productId, int $quantity): bool
    {
        $script = <<<'LUA'
            local stock = tonumber(redis.call('GET', KEYS[1]))
            if stock == nil then return -1 end
            if stock >= tonumber(ARGV[1]) then
                redis.call('DECRBY', KEYS[1], ARGV[1])
                return 1
            end
            return 0
        LUA;

        $result = Redis::eval($script, 1, "stock:{$productId}", $quantity);
        return $result === 1;
    }
}
```

**Node.js — Redis Advanced:**

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class RedisAdvanced {
    // Sorted Set — ট্রেন্ডিং নিউজ (Prothom Alo)
    async addTrendingArticle(articleId, score) {
        const pipeline = redis.pipeline();
        pipeline.zadd('trending:articles', score, articleId);
        pipeline.zremrangebyrank('trending:articles', 0, -51); // শীর্ষ ৫০
        pipeline.expire('trending:articles', 86400);
        await pipeline.exec();
    }

    // Pipeline — একসাথে অনেক কমান্ড (network round-trip কমায়)
    async batchGetProducts(productIds) {
        const pipeline = redis.pipeline();
        productIds.forEach(id => pipeline.get(`product:${id}`));
        const results = await pipeline.exec();
        return results.map(([err, val]) => val ? JSON.parse(val) : null);
    }

    // Lua Script — bKash ব্যালেন্স ট্রান্সফার (atomic)
    async transferBalance(fromId, toId, amount) {
        const script = `
            local fromBal = tonumber(redis.call('GET', KEYS[1]))
            local toBal = tonumber(redis.call('GET', KEYS[2]))
            if fromBal == nil or toBal == nil then return -1 end
            if fromBal < tonumber(ARGV[1]) then return 0 end
            redis.call('DECRBY', KEYS[1], ARGV[1])
            redis.call('INCRBY', KEYS[2], ARGV[1])
            return 1
        `;
        return redis.eval(script, 2,
            `balance:${fromId}`, `balance:${toId}`, amount
        );
    }

    // Pub/Sub — ক্যাশ ইনভ্যালিডেশন ইভেন্ট
    async subscribeInvalidation(callback) {
        const subscriber = redis.duplicate();
        await subscriber.subscribe('cache:invalidation');
        subscriber.on('message', (channel, message) => {
            const { key } = JSON.parse(message);
            callback(key);
        });
    }
}
```

#### Redis Cluster ও Sentinel

```
┌──────────────────────────────────────────────────────────┐
│                Redis Sentinel (HA)                       │
│                                                          │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│     │ Sentinel │  │ Sentinel │  │ Sentinel │            │
│     │    #1    │  │    #2    │  │    #3    │            │
│     └────┬─────┘  └────┬─────┘  └────┬─────┘            │
│          │              │              │                  │
│          └──────────┬───┴──────────────┘                  │
│                     │ মনিটরিং                             │
│          ┌──────────▼──────────┐                          │
│          │   Master (R/W)     │                          │
│          │   Port: 6379       │                          │
│          └────────┬───────────┘                          │
│                   │ রেপ্লিকেশন                             │
│          ┌────────┴────────┐                              │
│    ┌─────▼─────┐   ┌──────▼────┐                         │
│    │ Replica 1 │   │ Replica 2 │                         │
│    │   (Read)  │   │   (Read)  │                         │
│    └───────────┘   └───────────┘                         │
│                                                          │
│  Master ডাউন হলে Sentinel স্বয়ংক্রিয়ভাবে                 │
│  একটি Replica-কে Master বানিয়ে দেয় (Failover)            │
├──────────────────────────────────────────────────────────┤
│                Redis Cluster                             │
│                                                          │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│    │ Node A  │  │ Node B  │  │ Node C  │                │
│    │ Slot    │  │ Slot    │  │ Slot    │                │
│    │ 0-5460  │  │ 5461-   │  │ 10923-  │                │
│    │         │  │ 10922   │  │ 16383   │                │
│    └────┬────┘  └────┬────┘  └────┬────┘                │
│         │            │            │                      │
│    ┌────▼────┐  ┌────▼────┐  ┌────▼────┐                │
│    │Replica A│  │Replica B│  │Replica C│                │
│    └─────────┘  └─────────┘  └─────────┘                │
│                                                          │
│  ১৬,৩৮৪টি হ্যাশ স্লট ৩টি নোডে ভাগ                       │
│  key → CRC16(key) % 16384 → সঠিক নোড                    │
└──────────────────────────────────────────────────────────┘
```

#### Redis Persistence

```
RDB (Snapshotting):
  - নির্দিষ্ট সময়ে পুরো ডেটাসেটের স্ন্যাপশট
  - save 900 1     → ৯০০ সেকেন্ডে ১টি পরিবর্তন হলে
  - save 300 10    → ৩০০ সেকেন্ডে ১০টি পরিবর্তন হলে
  - দ্রুত restart, কিন্তু সর্বশেষ স্ন্যাপশটের পর ডেটা হারাতে পারে

AOF (Append Only File):
  - প্রতিটি write অপারেশন লগ করে
  - appendfsync always    → সবচেয়ে safe, ধীর
  - appendfsync everysec  → প্রতি সেকেন্ডে (ভালো ব্যালেন্স)
  - appendfsync no        → OS-এর উপর নির্ভর
  - ডেটা হারানোর ঝুঁকি কম, কিন্তু ফাইল বড় হয়

Hybrid (RDB + AOF): প্রোডাকশনে সবচেয়ে ভালো
```

---

### Memcached vs Redis তুলনা

```
┌───────────────────┬──────────────────┬──────────────────┐
│    বৈশিষ্ট্য       │     Redis        │   Memcached      │
├───────────────────┼──────────────────┼──────────────────┤
│ ডেটা স্ট্রাকচার   │ ৮+ ধরন           │ শুধু String      │
│ Persistence       │ RDB + AOF        │ নেই              │
│ Replication       │ আছে (Master-     │ নেই              │
│                   │ Replica)         │                  │
│ Cluster           │ Native Cluster   │ Client-side      │
│                   │                  │ sharding         │
│ Pub/Sub           │ আছে             │ নেই              │
│ Lua Scripting     │ আছে             │ নেই              │
│ Max Value Size    │ 512MB            │ 1MB              │
│ Threading         │ Single-threaded  │ Multi-threaded   │
│                   │ (6.0+ I/O থ্রেড) │                  │
│ Memory Efficiency │ বেশি overhead     │ কম overhead      │
│ Use Case          │ Complex caching, │ সরল key-value    │
│                   │ sessions, queues │ ক্যাশিং           │
└───────────────────┴──────────────────┴──────────────────┘

কখন কোনটা ব্যবহার করবেন:
  Redis     → বেশিরভাগ ক্ষেত্রে (ডিফল্ট চয়েস)
  Memcached → শুধু সরল ক্যাশিং, multi-threaded পারফরম্যান্স দরকার হলে
```

---

### CDN Caching (Content Delivery Network)

বাংলাদেশের ইউজারদের জন্য CDN অত্যন্ত গুরুত্বপূর্ণ — সিঙ্গাপুর/মুম্বাই এজ সার্ভার থেকে কন্টেন্ট সরবরাহ করলে latency অনেক কমে।

```
CDN ক্যাশিং ফ্লো:
┌──────────┐    ┌──────────────┐    ┌──────────────┐
│ ঢাকার    │───→│ সিঙ্গাপুর    │───→│  অরিজিন      │
│ ইউজার    │←───│ এজ সার্ভার   │←───│  সার্ভার      │
│          │    │  (CDN)       │    │  (US/EU)     │
│ ~20ms    │    │  ক্যাশ HIT   │    │  ~200ms      │
│ response │    │  হলে অরিজিনে │    │              │
│          │    │  যাবে না      │    │              │
└──────────┘    └──────────────┘    └──────────────┘
```

**Cache-Control Headers:**

```php
<?php
// Laravel — CDN-friendly response headers

// স্ট্যাটিক অ্যাসেট — দীর্ঘ ক্যাশ
Route::get('/api/products/{id}', function ($id) {
    $product = Cache::remember("product:{$id}", 3600, fn() => Product::find($id));

    return response()->json($product)
        ->header('Cache-Control', 'public, max-age=300, s-maxage=600, stale-while-revalidate=60')
        ->header('Vary', 'Accept-Encoding')
        ->header('ETag', md5(json_encode($product)));
});

// ব্যক্তিগত ডেটা — CDN ক্যাশ করবে না
Route::get('/api/user/profile', function () {
    return response()->json(auth()->user())
        ->header('Cache-Control', 'private, no-store, max-age=0');
});
```

```javascript
// Node.js/Express — CDN Caching Headers

// স্ট্যাটিক কন্টেন্ট
app.get('/api/articles/:slug', async (req, res) => {
    const article = await getArticle(req.params.slug);

    res.set({
        'Cache-Control': 'public, max-age=300, s-maxage=3600',
        'Surrogate-Key': `article-${article.id} category-${article.categoryId}`,
        'ETag': `"${article.updatedAt.getTime()}"`,
    });

    res.json(article);
});

// CloudFront/Cloudflare Invalidation
const AWS = require('aws-sdk');
const cloudfront = new AWS.CloudFront();

async function invalidateCDN(paths) {
    await cloudfront.createInvalidation({
        DistributionId: process.env.CF_DISTRIBUTION_ID,
        InvalidationBatch: {
            CallerReference: Date.now().toString(),
            Paths: {
                Quantity: paths.length,
                Items: paths, // ['/api/products/*', '/images/*']
            }
        }
    }).promise();
}
```

---

### Browser/Client Caching

```
Cache-Control হেডার বিশ্লেষণ:
┌────────────────────────────────────────────────────────┐
│ Cache-Control: public, max-age=31536000, immutable     │
│                                                        │
│ public       → CDN ও ব্রাউজার উভয়ই ক্যাশ করতে পারবে  │
│ max-age      → ৩১,৫৩৬,০০০ সেকেন্ড (১ বছর) ক্যাশ       │
│ immutable    → এই সময়ের মধ্যে revalidation দরকার নেই   │
│                                                        │
│ Cache-Control: private, no-cache, must-revalidate      │
│                                                        │
│ private      → শুধু ব্রাউজার ক্যাশ করবে (CDN নয়)       │
│ no-cache     → প্রতিবার সার্ভারে validate করতে হবে      │
│ must-revali. → stale কন্টেন্ট ব্যবহার করা যাবে না       │
│                                                        │
│ Cache-Control: no-store                                │
│ কোথাও কোনো ক্যাশ হবে না (সংবেদনশীল ডেটা)               │
└────────────────────────────────────────────────────────┘

ETag ভ্যালিডেশন ফ্লো:
  ১ম রিকোয়েস্ট: Server → ETag: "abc123" → Client ক্যাশ করে
  ২য় রিকোয়েস্ট: Client → If-None-Match: "abc123"
                  Server → 304 Not Modified (বডি পাঠায় না)
                  অথবা → 200 OK + নতুন ডেটা + নতুন ETag
```

**Service Worker Cache (Offline-first):**

```javascript
// service-worker.js — Progressive Web App ক্যাশিং

const CACHE_NAME = 'bdnews24-v3';
const STATIC_ASSETS = [
    '/', '/offline.html', '/css/app.css', '/js/app.js'
];

// ইনস্টল — স্ট্যাটিক অ্যাসেট ক্যাশ
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => cache.addAll(STATIC_ASSETS))
    );
});

// ফেচ — Network First with Cache Fallback
self.addEventListener('fetch', (event) => {
    const { request } = event;

    if (request.url.includes('/api/')) {
        // API রিকোয়েস্ট — Network First
        event.respondWith(networkFirstStrategy(request));
    } else {
        // স্ট্যাটিক — Cache First
        event.respondWith(cacheFirstStrategy(request));
    }
});

async function networkFirstStrategy(request) {
    try {
        const response = await fetch(request);
        const cache = await caches.open(CACHE_NAME);
        cache.put(request, response.clone());
        return response;
    } catch (error) {
        // অফলাইনে ক্যাশ থেকে দেখাও
        return caches.match(request) || caches.match('/offline.html');
    }
}

async function cacheFirstStrategy(request) {
    const cached = await caches.match(request);
    return cached || fetch(request);
}
```

---

### Application-Level Cache

**PHP OPcache:**

```php
<?php
// php.ini কনফিগারেশন
// opcache.enable=1
// opcache.memory_consumption=256
// opcache.max_accelerated_files=20000
// opcache.revalidate_freq=60

// In-Memory Cache (single-request lifecycle)
class InMemoryCache
{
    private static array $store = [];

    public static function remember(string $key, callable $callback): mixed
    {
        if (!isset(self::$store[$key])) {
            self::$store[$key] = $callback();
        }
        return self::$store[$key];
    }
}

// ব্যবহার — একই রিকোয়েস্টে বারবার কল এড়ানো
$user = InMemoryCache::remember("user:{$id}", fn() => User::find($id));
```

**Node.js In-Memory Cache:**

```javascript
const NodeCache = require('node-cache');

// stdTTL: ডিফল্ট TTL (সেকেন্ড), checkperiod: ক্লিনআপ চেক ইন্টারভাল
const appCache = new NodeCache({ stdTTL: 600, checkperiod: 120 });

class AppCacheService {
    async getConfig(key) {
        const cached = appCache.get(key);
        if (cached !== undefined) return cached;

        const config = await db.query(
            'SELECT value FROM configs WHERE key = $1', [key]
        );
        appCache.set(key, config.rows[0]?.value);
        return config.rows[0]?.value;
    }

    // LRU Cache — সীমিত মেমরি ব্যবহার
    createLRUCache(maxSize = 1000) {
        const map = new Map();
        return {
            get(key) {
                if (!map.has(key)) return undefined;
                const value = map.get(key);
                map.delete(key);
                map.set(key, value); // সবচেয়ে সাম্প্রতিক হিসেবে সরাও
                return value;
            },
            set(key, value) {
                map.delete(key);
                map.set(key, value);
                if (map.size > maxSize) {
                    const firstKey = map.keys().next().value;
                    map.delete(firstKey); // সবচেয়ে পুরনো মুছে ফেলো
                }
            }
        };
    }
}
```

---

### Database Query Cache

```php
<?php
// Laravel Query Cache — ভারী কোয়েরির জন্য

class ReportService
{
    // ভারী aggregation কোয়েরি — ক্যাশ করো
    public function getDailySalesReport(string $date): array
    {
        return Cache::remember(
            "report:daily-sales:{$date}",
            now()->addHours(6),
            function () use ($date) {
                return DB::table('orders')
                    ->selectRaw('
                        COUNT(*) as total_orders,
                        SUM(amount) as total_revenue,
                        AVG(amount) as avg_order_value
                    ')
                    ->whereDate('created_at', $date)
                    ->where('status', 'completed')
                    ->first();
            }
        );
    }
}

// MySQL Query Cache (8.0-তে deprecated, কিন্তু ধারণা গুরুত্বপূর্ণ):
// query_cache_type = ON
// query_cache_size = 64M

// PostgreSQL — prepared statement cache
// আপনার ORM (Eloquent/Sequelize) স্বয়ংক্রিয়ভাবে prepared statements ব্যবহার করে
```

---

## 🔥 Advanced Scenarios

### ১. Cache Invalidation Strategies

> "কম্পিউটার সায়েন্সে দুটি কঠিন জিনিস আছে: ক্যাশ ইনভ্যালিডেশন এবং নামকরণ।" — Phil Karlton

```
┌──────────────────────────────────────────────────────────┐
│        তিনটি মূল ইনভ্যালিডেশন কৌশল                       │
│                                                          │
│  ১. TTL-based (সময় ভিত্তিক):                             │
│     Cache::put('key', $data, 3600);  // ১ ঘণ্টা পর      │
│     ✅ সরল | ❌ TTL শেষ হওয়া পর্যন্ত stale                │
│                                                          │
│  ২. Event-based (ইভেন্ট ভিত্তিক):                         │
│     প্রোডাক্ট আপডেট → ইভেন্ট ফায়ার → ক্যাশ মুছো          │
│     ✅ তাৎক্ষণিক | ❌ সব ইভেন্ট track করতে হবে            │
│                                                          │
│  ৩. Version-based (সংস্করণ ভিত্তিক):                      │
│     key: "product:123:v5" → v6 হলে নতুন কী               │
│     ✅ zero-downtime | ❌ পুরনো কী মেমরি দখল করে          │
└──────────────────────────────────────────────────────────┘
```

**PHP (Laravel) — Event-based Invalidation:**

```php
<?php

namespace App\Observers;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;

class ProductObserver
{
    public function updated(Product $product): void
    {
        // সরাসরি ক্যাশ মুছো
        Cache::forget("product:{$product->id}");

        // সম্পর্কিত লিস্ট ক্যাশও মুছো (tag-based)
        Cache::tags([
            "category:{$product->category_id}",
            'product-listings'
        ])->flush();

        // অন্যান্য সার্ভারেও ইনভ্যালিডেশন (distributed)
        Redis::publish('cache:invalidation', json_encode([
            'type'   => 'product',
            'id'     => $product->id,
            'action' => 'updated',
        ]));
    }
}
```

**JavaScript — Version-based Invalidation:**

```javascript
class VersionedCache {
    constructor(redis) {
        this.redis = redis;
    }

    async get(entity, id) {
        const version = await this.redis.get(`${entity}:${id}:version`) || '1';
        return JSON.parse(
            await this.redis.get(`${entity}:${id}:v${version}`)
        );
    }

    async set(entity, id, data, ttl = 3600) {
        const version = await this.redis.incr(`${entity}:${id}:version`);
        await this.redis.setex(
            `${entity}:${id}:v${version}`, ttl, JSON.stringify(data)
        );
    }

    // সংস্করণ বাড়ালেই পুরনো ক্যাশ স্বয়ংক্রিয়ভাবে অকেজো
    async invalidate(entity, id) {
        await this.redis.incr(`${entity}:${id}:version`);
    }
}
```

---

### ২. Cache Stampede / Thundering Herd

একটি জনপ্রিয় ক্যাশ কী expire হলে শত শত রিকোয়েস্ট একসাথে DB-তে যায় — সিস্টেম ক্র্যাশ করতে পারে।

```
সমস্যা:
┌──────────────────────────────────────────────────────┐
│  সময়: TTL expire হলো                                │
│                                                      │
│  Req 1 ──→ Cache MISS ──→ DB Query ──┐               │
│  Req 2 ──→ Cache MISS ──→ DB Query ──┤               │
│  Req 3 ──→ Cache MISS ──→ DB Query ──┤  DB 💀        │
│  ...                                 │               │
│  Req N ──→ Cache MISS ──→ DB Query ──┘               │
│                                                      │
│  Daraz flash sale-এ ১ লাখ ইউজার একই প্রোডাক্ট        │
│  দেখছে — ক্যাশ expire = বিপর্যয়!                      │
└──────────────────────────────────────────────────────┘

সমাধান ১: Mutex Lock
┌──────────────────────────────────────────────────────┐
│  Req 1 ──→ MISS ──→ Lock অর্জন ──→ DB Query ──→ SET  │
│  Req 2 ──→ MISS ──→ Lock ব্যস্ত ──→ অপেক্ষা করো...   │
│  Req 3 ──→ MISS ──→ Lock ব্যস্ত ──→ অপেক্ষা করো...   │
│                                     ↓                │
│            Req 1 ক্যাশ সেট করলো → Req 2,3 ক্যাশ HIT  │
└──────────────────────────────────────────────────────┘
```

**PHP (Laravel) — Mutex Lock Solution:**

```php
<?php

use Illuminate\Support\Facades\Cache;

class StampedeProtectedCache
{
    /**
     * Mutex Lock দিয়ে Cache Stampede প্রতিরোধ
     */
    public function getWithLock(string $key, int $ttl, callable $callback): mixed
    {
        $cached = Cache::get($key);
        if ($cached !== null) {
            return $cached;
        }

        // Laravel Atomic Lock
        $lock = Cache::lock("lock:{$key}", 10); // ১০ সেকেন্ড lock

        if ($lock->get()) {
            try {
                // ডাবল-চেক — অন্য কেউ ইতিমধ্যে সেট করে থাকতে পারে
                $cached = Cache::get($key);
                if ($cached !== null) {
                    return $cached;
                }

                $data = $callback();
                Cache::put($key, $data, $ttl);
                return $data;
            } finally {
                $lock->release();
            }
        }

        // Lock পাইনি — সামান্য অপেক্ষা করে আবার ক্যাশ চেক
        usleep(100_000); // 100ms
        return Cache::get($key) ?? $callback();
    }
}
```

**Node.js — Probabilistic Early Expiry (XFetch):**

```javascript
class StampedeProtection {
    constructor(redis) {
        this.redis = redis;
    }

    /**
     * Probabilistic Early Expiration
     * TTL শেষ হওয়ার আগেই কিছু রিকোয়েস্ট refresh করে —
     * stampede এড়ানো যায় কোনো lock ছাড়াই
     */
    async getWithEarlyExpiry(key, ttl, fetchFn, beta = 1.0) {
        const cached = await this.redis.get(key);

        if (cached) {
            const { data, expiry, delta } = JSON.parse(cached);
            const now = Date.now();

            // XFetch অ্যালগরিদম
            const shouldRefresh =
                (now - delta * beta * Math.log(Math.random())) >= expiry;

            if (!shouldRefresh) {
                return data;
            }
            // refresh দরকার — পুরনো ডেটা ফেরত দেওয়ার আগে background-এ refresh
            this.backgroundRefresh(key, ttl, fetchFn);
            return data;
        }

        return this.fetchAndCache(key, ttl, fetchFn);
    }

    async fetchAndCache(key, ttl, fetchFn) {
        const start = Date.now();
        const data = await fetchFn();
        const delta = Date.now() - start; // fetch-এ কত সময় লাগলো

        const cacheValue = {
            data,
            delta,
            expiry: Date.now() + ttl * 1000
        };

        await this.redis.setex(key, ttl * 2, JSON.stringify(cacheValue));
        return data;
    }

    async backgroundRefresh(key, ttl, fetchFn) {
        // নন-ব্লকিং রিফ্রেশ
        setImmediate(async () => {
            try {
                await this.fetchAndCache(key, ttl, fetchFn);
            } catch (err) {
                console.error(`Background refresh failed: ${key}`, err);
            }
        });
    }

    // Mutex Lock সমাধান (Redlock প্যাটার্ন)
    async getWithMutex(key, ttl, fetchFn, lockTtl = 10) {
        const cached = await this.redis.get(key);
        if (cached) return JSON.parse(cached);

        const lockKey = `lock:${key}`;
        const lockAcquired = await this.redis.set(
            lockKey, '1', 'EX', lockTtl, 'NX'
        );

        if (lockAcquired) {
            try {
                // ডাবল-চেক
                const rechecked = await this.redis.get(key);
                if (rechecked) return JSON.parse(rechecked);

                const data = await fetchFn();
                await this.redis.setex(key, ttl, JSON.stringify(data));
                return data;
            } finally {
                await this.redis.del(lockKey);
            }
        }

        // lock পাইনি — স্বল্প অপেক্ষা করে retry
        await new Promise(r => setTimeout(r, 100));
        const retried = await this.redis.get(key);
        return retried ? JSON.parse(retried) : fetchFn();
    }
}
```

---

### ৩. Distributed Caching ও Consistent Hashing

```
সমস্যা: ৩টি Redis নোড থাকলে কোন key কোন নোডে যাবে?

সাধারণ Hashing (hash(key) % N):
  - নোড যোগ/বাদ হলে প্রায় সব key remap হয় 💀

Consistent Hashing:
┌──────────────────────────────────────────────────────┐
│              হ্যাশ রিং (0 থেকে 2^32)                 │
│                                                      │
│                    0/2^32                             │
│                   ╱      ╲                            │
│               Node A    Node B                       │
│              ╱                ╲                       │
│          key1, key4         key2, key5               │
│              ╲                ╱                       │
│               Node C    Node D                       │
│                   ╲      ╱                            │
│                   key3, key6                         │
│                                                      │
│  নোড যোগ হলে শুধু পাশের কী-গুলো remap হয়             │
│  বেশিরভাগ key অপরিবর্তিত থাকে ✅                     │
└──────────────────────────────────────────────────────┘
```

**Node.js — Consistent Hashing Implementation:**

```javascript
const crypto = require('crypto');

class ConsistentHash {
    constructor(replicas = 150) {
        this.replicas = replicas; // ভার্চুয়াল নোড সংখ্যা
        this.ring = new Map();
        this.sortedKeys = [];
        this.nodes = new Set();
    }

    hash(key) {
        return crypto.createHash('md5').update(key).digest('hex');
    }

    hashToInt(hash) {
        return parseInt(hash.substring(0, 8), 16);
    }

    addNode(node) {
        this.nodes.add(node);
        for (let i = 0; i < this.replicas; i++) {
            const virtualKey = this.hashToInt(this.hash(`${node}:${i}`));
            this.ring.set(virtualKey, node);
            this.sortedKeys.push(virtualKey);
        }
        this.sortedKeys.sort((a, b) => a - b);
    }

    removeNode(node) {
        this.nodes.delete(node);
        for (let i = 0; i < this.replicas; i++) {
            const virtualKey = this.hashToInt(this.hash(`${node}:${i}`));
            this.ring.delete(virtualKey);
            this.sortedKeys = this.sortedKeys.filter(k => k !== virtualKey);
        }
    }

    getNode(key) {
        if (this.ring.size === 0) return null;

        const hash = this.hashToInt(this.hash(key));
        // ঘড়ির কাঁটার দিকে প্রথম নোড খুঁজো
        for (const k of this.sortedKeys) {
            if (k >= hash) return this.ring.get(k);
        }
        return this.ring.get(this.sortedKeys[0]); // wrap around
    }
}

// ব্যবহার
const ch = new ConsistentHash();
ch.addNode('redis-1:6379');
ch.addNode('redis-2:6379');
ch.addNode('redis-3:6379');

const targetNode = ch.getNode('product:12345');
// → 'redis-2:6379' — সবসময় একই নোডে যাবে
```

---

### ৪. Cache Warming

সিস্টেম ডিপ্লয়/রিস্টার্টের পর ক্যাশ খালি থাকে — Cold Start সমস্যা। Cache Warming দিয়ে আগে থেকে ক্যাশ পূরণ করা হয়।

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Cache;
use App\Models\Product;

class WarmCache extends Command
{
    protected $signature = 'cache:warm';
    protected $description = 'ডিপ্লয়ের পর ক্যাশ ওয়ার্ম করো';

    public function handle(): void
    {
        $this->info('ক্যাশ ওয়ার্মিং শুরু...');

        // শীর্ষ ১০০০ জনপ্রিয় প্রোডাক্ট ক্যাশ করো
        Product::query()
            ->orderByDesc('view_count')
            ->take(1000)
            ->chunk(100, function ($products) {
                foreach ($products as $product) {
                    Cache::put(
                        "product:{$product->id}",
                        $product->load('category'),
                        now()->addHour()
                    );
                }
                $this->info('১০০টি প্রোডাক্ট ক্যাশ হয়েছে');
            });

        // ক্যাটাগরি ট্রি
        $categories = Category::with('children')->whereNull('parent_id')->get();
        Cache::put('category:tree', $categories, now()->addDay());

        // সাইট কনফিগারেশন
        $configs = Config::all()->pluck('value', 'key');
        Cache::put('site:config', $configs, now()->addDay());

        $this->info('✅ ক্যাশ ওয়ার্মিং সম্পন্ন!');
    }
}

// ডিপ্লয় স্ক্রিপ্টে যোগ করো:
// php artisan cache:warm
```

---

### ৫. Multi-tier Caching (L1 + L2)

```
┌──────────────────────────────────────────────────────┐
│             Multi-tier Cache Architecture             │
│                                                      │
│   App Server 1          App Server 2                 │
│  ┌───────────┐         ┌───────────┐                 │
│  │ L1: Local │         │ L1: Local │                 │
│  │ (Memory)  │         │ (Memory)  │                 │
│  │ ~0.1ms    │         │ ~0.1ms    │                 │
│  └─────┬─────┘         └─────┬─────┘                 │
│        │                     │                       │
│        └──────────┬──────────┘                       │
│                   │                                  │
│           ┌───────▼───────┐                          │
│           │ L2: Redis     │                          │
│           │ (Distributed) │                          │
│           │ ~1-2ms        │                          │
│           └───────┬───────┘                          │
│                   │                                  │
│           ┌───────▼───────┐                          │
│           │  Database     │                          │
│           │  ~20-200ms    │                          │
│           └───────────────┘                          │
│                                                      │
│  L1 MISS → L2 চেক → L2 MISS → DB → L2 SET → L1 SET │
└──────────────────────────────────────────────────────┘
```

**Node.js — Multi-tier Implementation:**

```javascript
const NodeCache = require('node-cache');
const Redis = require('ioredis');

class MultiTierCache {
    constructor() {
        this.l1 = new NodeCache({ stdTTL: 60, maxKeys: 5000 }); // ছোট TTL
        this.l2 = new Redis();
    }

    async get(key) {
        // L1 চেক (প্রসেস মেমরি — অতি দ্রুত)
        const l1Value = this.l1.get(key);
        if (l1Value !== undefined) {
            return l1Value;
        }

        // L2 চেক (Redis — দ্রুত)
        const l2Value = await this.l2.get(key);
        if (l2Value) {
            const parsed = JSON.parse(l2Value);
            this.l1.set(key, parsed); // L1-এ প্রমোট করো
            return parsed;
        }

        return null; // উভয় ক্যাশেই নেই
    }

    async set(key, value, l1Ttl = 60, l2Ttl = 3600) {
        this.l1.set(key, value, l1Ttl);
        await this.l2.setex(key, l2Ttl, JSON.stringify(value));
    }

    async invalidate(key) {
        this.l1.del(key);
        await this.l2.del(key);

        // অন্যান্য App Server-এর L1 ক্যাশও মুছতে হবে
        await this.l2.publish('cache:invalidation', key);
    }

    // Pub/Sub দিয়ে সব সার্ভারের L1 sync
    subscribeInvalidation() {
        const subscriber = this.l2.duplicate();
        subscriber.subscribe('cache:invalidation');
        subscriber.on('message', (channel, key) => {
            this.l1.del(key);
        });
    }
}
```

---

### ৬. Cache Eviction Policies

ক্যাশ মেমরি পূর্ণ হলে কোন ডেটা মুছবে সেটা নির্ধারণ করার নীতি:

```
┌────────────────────────────────────────────────────────────┐
│                  Eviction Policies                         │
│                                                            │
│  LRU (Least Recently Used):                                │
│    সবচেয়ে পুরনো অ্যাক্সেস হওয়া আইটেম মুছে ফেলো            │
│    [A(3s), B(1s), C(5s)] → C মুছবে (৫ সেকেন্ড আগে ব্যবহৃত)│
│    ✅ সবচেয়ে জনপ্রিয় | Redis ডিফল্ট                        │
│                                                            │
│  LFU (Least Frequently Used):                              │
│    সবচেয়ে কম ব্যবহৃত আইটেম মুছে ফেলো                      │
│    [A(10x), B(2x), C(50x)] → B মুছবে (মাত্র ২ বার ব্যবহৃত)│
│    ✅ জনপ্রিয়তা ভিত্তিক | ❌ নতুন আইটেমের ক্ষতি             │
│                                                            │
│  FIFO (First In, First Out):                               │
│    প্রথমে ঢোকা আইটেম প্রথমে বের হবে                        │
│    ✅ সরল | ❌ জনপ্রিয়তা বিবেচনা করে না                     │
│                                                            │
│  TTL-based:                                                │
│    নির্দিষ্ট সময় পর স্বয়ংক্রিয়ভাবে মুছে যায়               │
│    ✅ predictable | ❌ জনপ্রিয় আইটেমও মুছে যায়             │
│                                                            │
│  Random:                                                   │
│    যেকোনো আইটেম random-ভাবে মুছে ফেলো                     │
│    ✅ অতি দ্রুত O(1) | ❌ অনুমান অসম্ভব                     │
└────────────────────────────────────────────────────────────┘

Redis Eviction কনফিগারেশন:
  maxmemory 2gb
  maxmemory-policy allkeys-lru    ← সবচেয়ে বেশি ব্যবহৃত
  // অন্যান্য: volatile-lru, allkeys-lfu, volatile-lfu,
  //          allkeys-random, volatile-random, noeviction
```

---

### ৭. Laravel Caching (গভীর পর্যালোচনা)

```php
<?php

use Illuminate\Support\Facades\Cache;

// ==============================
// Cache::remember — সবচেয়ে ব্যবহৃত
// ==============================
$products = Cache::remember('products:featured', 3600, function () {
    return Product::where('is_featured', true)
        ->with('category')
        ->orderByDesc('created_at')
        ->limit(20)
        ->get();
});

// ==============================
// Cache Tags — গ্রুপ ইনভ্যালিডেশন (Redis/Memcached only)
// ==============================
// ক্যাটাগরি অনুযায়ী প্রোডাক্ট ক্যাশ
Cache::tags(['products', "category:{$categoryId}"])
    ->remember("products:cat:{$categoryId}:page:{$page}", 3600, function () use ($categoryId, $page) {
        return Product::where('category_id', $categoryId)
            ->paginate(20, ['*'], 'page', $page);
    });

// নির্দিষ্ট ক্যাটাগরির সব ক্যাশ মুছো
Cache::tags(["category:{$categoryId}"])->flush();

// সব প্রোডাক্ট ক্যাশ মুছো
Cache::tags(['products'])->flush();

// ==============================
// Atomic Lock — Race condition প্রতিরোধ
// ==============================
$lock = Cache::lock('processing-order:' . $orderId, 30);

if ($lock->get()) {
    try {
        // অর্ডার প্রসেস করো — একই সময়ে একটাই
        $this->processOrder($orderId);
    } finally {
        $lock->release();
    }
} else {
    // অন্য কেউ ইতিমধ্যে প্রসেস করছে
    return response()->json(['message' => 'অর্ডার প্রসেস হচ্ছে'], 409);
}

// Block এবং অপেক্ষা করো (সর্বোচ্চ ৫ সেকেন্ড)
$lock = Cache::lock('processing:' . $orderId, 30);
$lock->block(5, function () use ($orderId) {
    $this->processOrder($orderId);
});

// ==============================
// Cache Drivers — config/cache.php
// ==============================
// file   → ডেভেলপমেন্ট (ডিফল্ট)
// redis  → প্রোডাকশন (সবচেয়ে ভালো)
// memcached → বিকল্প
// array  → টেস্টিং
// database → শেষ উপায়

// ==============================
// HTTP Cache — Response Middleware
// ==============================
Route::middleware('cache.headers:public;max_age=600;etag')
    ->get('/api/products', [ProductController::class, 'index']);
```

---

### ৮. Node.js Caching Ecosystem

```javascript
// ==============================
// node-cache — In-process caching
// ==============================
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600, checkperiod: 120 });

app.get('/api/products/:id', async (req, res) => {
    const key = `product:${req.params.id}`;
    let product = cache.get(key);

    if (!product) {
        product = await Product.findById(req.params.id);
        cache.set(key, product);
    }

    res.json(product);
});

// ==============================
// ioredis — Redis client (production)
// ==============================
const Redis = require('ioredis');

// Cluster মোড
const cluster = new Redis.Cluster([
    { host: 'redis-1', port: 6379 },
    { host: 'redis-2', port: 6379 },
    { host: 'redis-3', port: 6379 },
]);

// ==============================
// Express Middleware — Response Caching
// ==============================
function cacheMiddleware(ttl = 300) {
    return async (req, res, next) => {
        if (req.method !== 'GET') return next();

        const key = `route:${req.originalUrl}`;
        const cached = await redis.get(key);

        if (cached) {
            const { body, headers } = JSON.parse(cached);
            Object.entries(headers).forEach(([k, v]) => res.set(k, v));
            return res.json(body);
        }

        const originalJson = res.json.bind(res);
        res.json = (body) => {
            redis.setex(key, ttl, JSON.stringify({
                body,
                headers: { 'X-Cache': 'HIT' }
            }));
            originalJson(body);
        };

        next();
    };
}

app.get('/api/articles', cacheMiddleware(600), articleController.list);

// ==============================
// CDN Integration — Fastly/CloudFront Headers
// ==============================
app.use('/api/public', (req, res, next) => {
    res.set({
        'Cache-Control': 'public, s-maxage=3600, stale-while-revalidate=600',
        'Surrogate-Control': 'max-age=86400',
        'CDN-Cache-Control': 'max-age=7200',
    });
    next();
});
```

---

## ✅ Pros (সুবিধাসমূহ) / Cons (অসুবিধাসমূহ)

```
┌──────────────────────────────┬──────────────────────────────┐
│        ✅ সুবিধা              │        ❌ অসুবিধা             │
├──────────────────────────────┼──────────────────────────────┤
│ Latency ব্যাপকভাবে কমায়     │ Stale data সমস্যা            │
│ (50ms → 0.5ms)              │ (ক্যাশ ও DB-র মধ্যে mismatch)│
│                              │                              │
│ DB load কমায় (90%+)         │ অতিরিক্ত জটিলতা ও রক্ষণাবেক্ষণ│
│                              │ (ক্যাশ ইনভ্যালিডেশন)          │
│                              │                              │
│ Throughput বাড়ায়             │ মেমরি খরচ বাড়ে               │
│ (প্রতি সেকেন্ডে বেশি req)    │ (Redis সার্ভার খরচ)           │
│                              │                              │
│ ইউজার এক্সপেরিয়েন্স উন্নত   │ ডিবাগিং কঠিন হয়              │
│                              │ ("কেন পুরনো ডেটা দেখাচ্ছে?")   │
│                              │                              │
│ Cost efficient               │ Cold start সমস্যা             │
│ (কম সার্ভার দিয়ে বেশি ট্রাফিক)│ (ডিপ্লয়ের পর ক্যাশ খালি)     │
│                              │                              │
│ Scalability বাড়ায়            │ Consistency চ্যালেঞ্জ          │
│                              │ (distributed ক্যাশে)           │
└──────────────────────────────┴──────────────────────────────┘
```

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ১. Stale Data (পুরনো ডেটা)

```
সমস্যা: প্রোডাক্টের দাম DB-তে পরিবর্তন হয়েছে কিন্তু ক্যাশে পুরনো দাম!

ভুল:
  Cache::put('product:123', $product, 86400); // ২৪ ঘণ্টা TTL 😱

সমাধান:
  ১. TTL কমাও (৫-৩০ মিনিট)
  ২. Event-based invalidation ব্যবহার করো
  ৩. Write-Through ক্যাশিং ব্যবহার করো
```

### ২. Over-Caching (অতিরিক্ত ক্যাশিং)

```
ভুল:
  // এটা দরকার নেই — প্রতিটা micro কোয়েরি ক্যাশ করা
  Cache::remember("user:count", 3600, fn() => User::count());
  Cache::remember("today:date", 3600, fn() => now()->format('Y-m-d'));

সমাধান:
  শুধু ভারী/ঘনঘন কোয়েরি ক্যাশ করো
  ক্যাশ করার আগে জিজ্ঞেস করো:
  - এই কোয়েরি কি ধীর? (>10ms)
  - এটা কি ঘনঘন অনুরোধ হয়? (>10 req/sec)
  - ডেটা কি কিছুক্ষণ পুরনো থাকলে সমস্যা নেই?
```

### ৩. Cache Penetration (ক্যাশ ভেদন)

```
সমস্যা: অস্তিত্বহীন key-র জন্য বারবার DB query হচ্ছে

আক্রমণকারী: GET /api/products/-1, GET /api/products/999999999
  → প্রতিটি রিকোয়েস্টে ক্যাশ MISS → DB query → DB overload!

সমাধান ১: Negative Caching (null ক্যাশ করো)
  if ($product === null) {
      Cache::put("product:{$id}", null, 300); // ৫ মিনিট
  }

সমাধান ২: Bloom Filter (সম্ভাব্য চেক)
  if (!bloomFilter.mightContain(productId)) {
      return null; // DB query-ই করো না
  }

সমাধান ৩: Input Validation
  if ($id < 1 || $id > 10_000_000) abort(404);
```

### ৪. Hot Key সমস্যা

```
সমস্যা: Flash sale-এ ১ লাখ ইউজার একই প্রোডাক্ট দেখছে
  → একটি Redis key-তে অত্যধিক read → নেটওয়ার্ক bottleneck

সমাধান:
  ১. Local cache (L1) দিয়ে Redis call কমাও
  ২. Key replicate করো: product:123:shard1, product:123:shard2
  ৩. Redis Cluster-এ read replica ব্যবহার করো
```

---

## 🧪 Testing Cached Systems

ক্যাশড সিস্টেম টেস্ট করা চ্যালেঞ্জিং — ক্যাশ state টেস্টের ফলাফল প্রভাবিত করে।

**PHP (Laravel) — Cache Testing:**

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\Product;
use Illuminate\Support\Facades\Cache;

class ProductCacheTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        Cache::flush(); // প্রতিটি টেস্টের আগে ক্যাশ পরিষ্কার
    }

    public function test_product_is_cached_after_first_fetch(): void
    {
        $product = Product::factory()->create(['name' => 'Test']);

        // প্রথম কল — DB query হবে
        $this->getJson("/api/products/{$product->id}")
            ->assertOk();

        // ক্যাশে আছে কিনা যাচাই
        $this->assertTrue(Cache::has("product:{$product->id}"));
    }

    public function test_cached_product_returns_without_db_query(): void
    {
        $product = Product::factory()->create();

        // ম্যানুয়ালি ক্যাশে সেট
        Cache::put("product:{$product->id}", $product, 3600);

        // DB থেকে প্রোডাক্ট মুছে ফেলো
        $product->delete();

        // তবুও ক্যাশ থেকে পাওয়া উচিত
        $response = $this->getJson("/api/products/{$product->id}");
        $response->assertOk();
    }

    public function test_cache_invalidated_on_product_update(): void
    {
        $product = Product::factory()->create(['price' => 100]);
        Cache::put("product:{$product->id}", $product, 3600);

        $product->update(['price' => 200]);

        // ক্যাশ মুছে যাওয়া উচিত
        $this->assertFalse(Cache::has("product:{$product->id}"));
    }

    public function test_cache_stampede_protection(): void
    {
        $callCount = 0;
        $fetcher = function () use (&$callCount) {
            $callCount++;
            usleep(100_000);
            return ['data' => 'value'];
        };

        // ১০টি concurrent রিকোয়েস্ট সিমুলেট
        $promises = [];
        for ($i = 0; $i < 10; $i++) {
            $promises[] = async(fn() =>
                $this->stampedeCache->getWithLock('test:key', 3600, $fetcher)
            );
        }

        await($promises);

        // DB query শুধু ১ বার হওয়া উচিত (বাকি ৯টি ক্যাশ থেকে)
        $this->assertLessThanOrEqual(2, $callCount);
    }
}
```

**JavaScript (Node.js) — Cache Testing:**

```javascript
const { expect } = require('chai');
const sinon = require('sinon');
const Redis = require('ioredis-mock');

describe('ProductCacheService', () => {
    let cacheService;
    let redis;

    beforeEach(() => {
        redis = new Redis();
        cacheService = new ProductCacheService(redis);
    });

    afterEach(async () => {
        await redis.flushall();
        sinon.restore();
    });

    it('ক্যাশ MISS হলে DB থেকে ডেটা এনে ক্যাশ করা উচিত', async () => {
        const dbSpy = sinon.spy(db, 'query');
        const product = await cacheService.getProduct(123);

        expect(product).to.exist;
        expect(dbSpy.calledOnce).to.be.true;

        // দ্বিতীয় কলে DB query হওয়া উচিত না
        await cacheService.getProduct(123);
        expect(dbSpy.calledOnce).to.be.true; // এখনো ১ বার
    });

    it('TTL expire হলে নতুন ডেটা আনা উচিত', async () => {
        await redis.setex('product:123', 1, JSON.stringify({ id: 123, name: 'Old' }));

        // TTL expire হতে দাও
        await new Promise(r => setTimeout(r, 1100));

        const dbStub = sinon.stub(db, 'query').resolves({
            rows: [{ id: 123, name: 'New' }]
        });

        const product = await cacheService.getProduct(123);
        expect(product.name).to.equal('New');
        expect(dbStub.calledOnce).to.be.true;
    });

    it('null ডেটা নেগেটিভ ক্যাশিং করা উচিত', async () => {
        sinon.stub(db, 'query').resolves({ rows: [] });

        const result = await cacheService.getProduct(999);
        expect(result).to.be.null;

        const cached = await redis.get('product:999');
        expect(cached).to.equal(JSON.stringify(null));
    });
});
```

---

## 📋 সারসংক্ষেপ (Summary)

### বাংলাদেশ কনটেক্সটে ক্যাশিং প্রয়োগ

```
┌─────────────────────────────────────────────────────────────┐
│             বাংলাদেশি সিস্টেমে ক্যাশিং প্রয়োগ               │
│                                                             │
│  🛒 Daraz Bangladesh:                                       │
│     - প্রোডাক্ট ক্যাটালগ → Redis Cache-Aside                │
│     - Flash Sale স্টক → Lua Script (atomic decrement)       │
│     - সার্চ রেজাল্ট → Elasticsearch + Redis                 │
│     - স্ট্যাটিক ইমেজ → CDN (CloudFront, সিঙ্গাপুর এজ)      │
│     - সেশন ডেটা → Redis (Cluster mode)                     │
│                                                             │
│  💰 bKash:                                                  │
│     - ব্যালেন্স ক্যাশ → Write-Through (consistency critical) │
│     - ট্রানজাকশন হিস্টোরি → Read-Through                    │
│     - Rate Limiting → Redis Sorted Set                      │
│     - OTP ভেরিফিকেশন → Redis TTL (৫ মিনিট)                 │
│     - ⚠️ ব্যালেন্স ক্যাশে কখনো eventual consistency নয়!     │
│                                                             │
│  📰 Prothom Alo / BD News 24:                               │
│     - ব্রেকিং নিউজ → CDN (short TTL: ৩০ সেকেন্ড)           │
│     - আর্কাইভ আর্টিকেল → CDN (long TTL: ২৪ ঘণ্টা)          │
│     - ভিউ কাউন্ট → Write-Behind (Redis INCR + batch DB)    │
│     - ট্রেন্ডিং নিউজ → Redis Sorted Set                     │
│     - ইমেজ/ভিডিও → CDN + Browser Cache (immutable)         │
│     - Service Worker → অফলাইন রিডিং                         │
└─────────────────────────────────────────────────────────────┘
```

### দ্রুত সিদ্ধান্ত গাইড

```
কোন কৌশল কখন ব্যবহার করবেন?

  Cache-Aside     → ডিফল্ট চয়েস, বেশিরভাগ read-heavy সিস্টেমে
  Read-Through    → ক্যাশ লজিক centralize করতে চাইলে
  Write-Through   → Strong consistency দরকার হলে (bKash)
  Write-Behind    → Write-heavy, eventual consistency চলবে (view count)
  Refresh-Ahead   → জনপ্রিয় কন্টেন্ট, cache miss সহ্য হবে না

কোন টেকনোলজি কখন?

  Redis           → ৯৫% ক্ষেত্রে ডিফল্ট চয়েস
  Memcached       → শুধু সরল key-value, multi-threaded দরকার
  CDN             → স্ট্যাটিক কন্টেন্ট, গ্লোবাল ইউজার বেস
  Browser Cache   → স্ট্যাটিক অ্যাসেট (JS, CSS, ইমেজ)
  In-Memory       → অতি দ্রুত, ছোট ডেটাসেট, single-server
  DB Query Cache  → ভারী aggregation, রিপোর্টিং কোয়েরি
```

### মনে রাখার চেকলিস্ট

```
☑ ক্যাশ করার আগে: "এই ডেটা কি সত্যিই ক্যাশ করার উপযুক্ত?"
☑ TTL সবসময় সেট করো — কখনো infinite ক্যাশ নয়
☑ ক্যাশ ইনভ্যালিডেশন পরিকল্পনা আগেই করো
☑ Cache penetration প্রতিরোধে নেগেটিভ ক্যাশিং
☑ Cache stampede প্রতিরোধে mutex lock বা early expiry
☑ মনিটরিং: hit ratio, memory usage, eviction rate ট্র্যাক করো
☑ টেস্টিং: ক্যাশ সহ ও ক্যাশ ছাড়া উভয় পরিস্থিতি টেস্ট করো
☑ সংবেদনশীল ডেটা (পাসওয়ার্ড, টোকেন) ক্যাশ করো না
☑ Graceful degradation: ক্যাশ ডাউন হলেও সিস্টেম যেন চলে
☑ ডিপ্লয়ের পর cache warming বিবেচনা করো
```

---

> **"ক্যাশিং হলো সিস্টেম ডিজাইনের সবচেয়ে শক্তিশালী অস্ত্র — কিন্তু ভুলভাবে ব্যবহার করলে এটি সবচেয়ে বিপজ্জনক বাগের উৎসও হতে পারে। সঠিক কৌশল, যথাযথ TTL, এবং দৃঢ় ইনভ্যালিডেশন পরিকল্পনা — এই তিনটি মিলিয়েই একটি কার্যকর ক্যাশিং সিস্টেম তৈরি হয়।"**
