# 🗑️ Cache Invalidation Strategies

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [Caching Strategies](#caching-strategies)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII Diagram](#ascii-diagram)
- [Thundering Herd Problem](#thundering-herd-problem)
- [Multi-Level Caching](#multi-level-caching)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Cache Warming & Tag-based Invalidation](#cache-warming--tag-based-invalidation)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Cache Invalidation** হলো cache-এ থাকা পুরনো (stale) data সরিয়ে ফেলা বা update করার প্রক্রিয়া। এটি কঠিন কারণ:
- **কখন** invalidate করবেন? (Too early = cache miss, Too late = stale data)
- **কী** invalidate করবেন? (Dependencies complex হতে পারে)
- **কীভাবে** distribute করবেন? (Multiple cache layers)

### 🔑 মূল Caching Strategies:

| Strategy | কীভাবে কাজ করে | Write Path | Read Path |
|---|---|---|---|
| **Cache-Aside** | App নিজে cache manage করে | DB-তে write, cache invalidate | Cache miss → DB read → Cache set |
| **Read-Through** | Cache নিজে DB থেকে load করে | DB-তে write | Cache miss → Cache loads from DB |
| **Write-Through** | Write cache ও DB দুটোতেই | Cache + DB simultaneously | Always from cache |
| **Write-Behind** | Write cache-এ, DB-তে async | Cache only, async DB write | Always from cache |
| **Write-Around** | Write শুধু DB-তে | DB only | Cache miss → DB read |

### 📊 Invalidation Methods:

1. **TTL-based**: নির্দিষ্ট সময় পর expire
2. **Event-based**: Data change event-এ invalidate
3. **Version-based**: Version mismatch-এ invalidate
4. **Tag-based**: Related data group invalidate

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 📰 Prothom Alo News Article Caching:

```
সমস্যা:
- প্রতিদিন ৫০+ নতুন article publish হয়
- Breaking news immediately দেখাতে হবে
- ৫০ লক্ষ+ daily visitors
- CDN edge servers বাংলাদেশ জুড়ে (Dhaka, Chittagong, Sylhet)

Challenge:
- Editor নতুন article publish করলো
- CDN-এ পুরনো homepage cached আছে
- Redis-এ পুরনো article list cached
- Browser-এ পুরনো page cached
- কীভাবে instantly সব জায়গায় update করবো?
```

### 🛒 Daraz Flash Sale Product Price:

```
সমস্যা:
- Flash sale শুরু: Product price ৫০০০ → ২৫০০ টাকা
- ১০ লক্ষ user একসাথে product page দেখছে
- Cache-এ পুরনো price (৫০০০) আছে
- কিছু user পুরনো price দেখছে, কিছু নতুন
- Order place করলে কোন price apply হবে?

সমাধান: Event-based invalidation + version check at checkout
```

### 📱 Grameenphone MyGP App Balance:

```
- User balance: ১৫০ টাকা (cached in app)
- User call করলো: ৫ টাকা কেটে গেলো
- Backend balance: ১৪৫ টাকা
- App এখনো ১৫০ দেখাচ্ছে (stale cache)
- TTL expire না হওয়া পর্যন্ত পুরনো value দেখাবে

সমাধান: Event-driven invalidation via WebSocket/Push notification
```

---

## 📊 ASCII Diagram

### Cache-Aside (Lazy Loading) Pattern:

```
┌─────────────────────────────────────────────────────────┐
│              CACHE-ASIDE PATTERN                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  READ PATH:                                              │
│  ┌──────┐   1.Get   ┌───────┐                           │
│  │Client│──────────►│ Cache │                            │
│  │      │◄──────────│(Redis)│                            │
│  │      │  2a.Hit!  └───────┘                            │
│  │      │                                                │
│  │      │  2b.Miss  ┌───────┐                            │
│  │      │──────────►│  DB   │                            │
│  │      │◄──────────│       │                            │
│  │      │  3.Data   └───────┘                            │
│  │      │                                                │
│  │      │──Set────►┌───────┐                             │
│  └──────┘  4.Cache │ Cache │                             │
│                     └───────┘                             │
│                                                          │
│  WRITE PATH:                                             │
│  ┌──────┐  1.Write  ┌───────┐                            │
│  │Client│──────────►│  DB   │                            │
│  │      │           └───────┘                            │
│  │      │                                                │
│  │      │  2.Delete ┌───────┐                            │
│  │      │──────────►│ Cache │  (Invalidate!)             │
│  └──────┘           └───────┘                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Write-Through Pattern:

```
┌─────────────────────────────────────────────────────────┐
│              WRITE-THROUGH PATTERN                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────┐  1.Write  ┌───────┐  2.Write  ┌───────┐      │
│  │Client│──────────►│ Cache │──────────►│  DB   │      │
│  │      │           │(Redis)│           │       │      │
│  │      │◄──────────│       │◄──────────│       │      │
│  └──────┘  4.ACK    └───────┘  3.ACK    └───────┘      │
│                                                          │
│  ✅ Cache always fresh                                   │
│  ❌ Write latency higher (cache + DB)                    │
│  ❌ Cache may have data never read                       │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Write-Behind (Write-Back) Pattern:

```
┌─────────────────────────────────────────────────────────┐
│              WRITE-BEHIND PATTERN                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────┐  1.Write  ┌───────┐                           │
│  │Client│──────────►│ Cache │  ◄── Immediate ACK!       │
│  │      │◄──────────│(Redis)│                            │
│  └──────┘  2.ACK    └───────┘                            │
│                          │                               │
│                          │ 3. Async (batch/delayed)      │
│                          ▼                               │
│                     ┌───────┐                            │
│                     │  DB   │                            │
│                     └───────┘                            │
│                                                          │
│  ✅ Ultra-fast writes                                    │
│  ❌ Data loss risk (cache crash before DB write)         │
│  Use case: Grameenphone call log aggregation             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Multi-Level Cache Architecture:

```
┌─────────────────────────────────────────────────────────────┐
│           MULTI-LEVEL CACHING (Prothom Alo)                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ L1:      │   │ L2:      │   │ L3:      │   │ L4:     │ │
│  │ Browser  │──►│ CDN      │──►│ App      │──►│ DB      │ │
│  │ Cache    │   │ (Edge)   │   │ (Redis)  │   │ Query   │ │
│  │          │   │          │   │          │   │ Cache   │ │
│  ├──────────┤   ├──────────┤   ├──────────┤   ├─────────┤ │
│  │TTL: 60s  │   │TTL: 300s │   │TTL: 3600s│   │TTL: ∞  │ │
│  │Size: 50MB│   │Size: 1TB │   │Size: 64GB│   │Size:100G│ │
│  │Hit: 50%  │   │Hit: 85%  │   │Hit: 95%  │   │Hit: 99% │ │
│  └──────────┘   └──────────┘   └──────────┘   └─────────┘ │
│                                                              │
│  Invalidation Flow (Article Published):                      │
│  Editor ──► API ──► Invalidate L3 (Redis)                    │
│                  ──► Purge L2 (CDN API call)                  │
│                  ──► Push notification to L1 (Cache-Control)  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ Thundering Herd Problem

### সমস্যা (Cache Stampede):

```
┌──────────────────────────────────────────────────────┐
│         THUNDERING HERD / CACHE STAMPEDE              │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Popular cache key expires (TTL=0)                   │
│                                                      │
│  Request-1 ──► Cache MISS ──► DB Query              │
│  Request-2 ──► Cache MISS ──► DB Query  ◄── SAME!  │
│  Request-3 ──► Cache MISS ──► DB Query  ◄── SAME!  │
│  Request-4 ──► Cache MISS ──► DB Query  ◄── SAME!  │
│  ...                                                 │
│  Request-10000 ──► Cache MISS ──► DB Query           │
│                                                      │
│  ⚠️ ১০,০০০টি identical DB query একসাথে!             │
│  💀 Database overloaded / crashed!                   │
│                                                      │
│  Prothom Alo homepage article list cache expire:     │
│  ৫০ লক্ষ user সবাই DB hit করছে!                     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### সমাধান ১: Distributed Lock:

```
┌──────────────────────────────────────────────────────┐
│         SOLUTION 1: DISTRIBUTED LOCK                 │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Request-1 ──► Cache MISS ──► Acquire Lock ✅        │
│                                   │                  │
│  Request-2 ──► Cache MISS ──► Lock BUSY → Wait      │
│  Request-3 ──► Cache MISS ──► Lock BUSY → Wait      │
│                                   │                  │
│  Request-1:  DB Query ──► Set Cache ──► Release Lock │
│                                                      │
│  Request-2 ──► Cache HIT ✅ (from Request-1)         │
│  Request-3 ──► Cache HIT ✅                          │
│                                                      │
│  Result: শুধু ১টি DB query!                          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### সমাধান ২: Probabilistic Early Expiration:

```
┌──────────────────────────────────────────────────────┐
│     SOLUTION 2: PROBABILISTIC EARLY RECOMPUTATION    │
├──────────────────────────────────────────────────────┤
│                                                      │
│  TTL = 3600s (1 hour)                                │
│  Current time remaining = 120s                       │
│                                                      │
│  Should recompute?                                   │
│  probability = exp(-time_remaining / beta)           │
│                                                      │
│  time=3600: prob=0.001 (almost never recompute)      │
│  time=300:  prob=0.05  (5% chance)                   │
│  time=60:   prob=0.30  (30% chance)                  │
│  time=10:   prob=0.90  (90% chance - almost certain) │
│                                                      │
│  ✅ Cache refreshed before expiry (no stampede)      │
│  ✅ No external lock needed                          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Cache Invalidation Strategies
 * Prothom Alo News Caching System
 */

class CacheInvalidationService
{
    private $redis;
    private $db;
    private $cdnClient;
    
    public function __construct($redis, $db, $cdnClient)
    {
        $this->redis = $redis;
        $this->db = $db;
        $this->cdnClient = $cdnClient;
    }
    
    // ===========================
    // CACHE-ASIDE PATTERN
    // ===========================
    
    /**
     * Cache-Aside Read (Lazy Loading)
     * Prothom Alo article fetch
     */
    public function getArticle(string $articleId): ?array
    {
        $cacheKey = "article:{$articleId}";
        
        // Step 1: Cache থেকে চেষ্টা
        $cached = $this->redis->get($cacheKey);
        if ($cached !== null) {
            return json_decode($cached, true); // Cache HIT
        }
        
        // Step 2: Cache MISS → Database থেকে পড়ো
        $article = $this->db->query(
            "SELECT * FROM articles WHERE id = ? AND status = 'published'",
            [$articleId]
        );
        
        if ($article === null) {
            // Cache penetration prevention: null value cache করো
            $this->redis->setex($cacheKey, 60, json_encode(null));
            return null;
        }
        
        // Step 3: Cache-এ set করো with TTL
        $this->redis->setex($cacheKey, 3600, json_encode($article));
        
        return $article;
    }
    
    /**
     * Cache-Aside Write with Invalidation
     * Article publish/update
     */
    public function publishArticle(array $articleData): bool
    {
        // Step 1: Database-এ write
        $this->db->insert('articles', $articleData);
        
        // Step 2: Related cache invalidate
        $this->invalidateArticleCache($articleData['id']);
        
        // Step 3: CDN purge
        $this->purgeCDN($articleData['id']);
        
        // Step 4: Event publish (for other services)
        $this->publishEvent('article.published', $articleData);
        
        return true;
    }
    
    /**
     * Tag-based Cache Invalidation
     * একটি category-র সব article invalidate
     */
    public function invalidateByTag(string $tag): int
    {
        // Tag-এ associated সব keys খুঁজে বের করো
        $tagKey = "cache_tag:{$tag}";
        $keys = $this->redis->sMembers($tagKey);
        
        $invalidatedCount = 0;
        foreach ($keys as $key) {
            $this->redis->del($key);
            $invalidatedCount++;
        }
        
        // Tag set নিজেও clear
        $this->redis->del($tagKey);
        
        return $invalidatedCount;
    }
    
    /**
     * Version-based Cache Invalidation
     * Cache key-তে version number রাখা
     */
    public function getArticleVersioned(string $articleId): ?array
    {
        $version = $this->redis->get("article_version:{$articleId}") ?? 1;
        $cacheKey = "article:{$articleId}:v{$version}";
        
        $cached = $this->redis->get($cacheKey);
        if ($cached !== null) {
            return json_decode($cached, true);
        }
        
        $article = $this->db->query("SELECT * FROM articles WHERE id = ?", [$articleId]);
        $this->redis->setex($cacheKey, 3600, json_encode($article));
        
        return $article;
    }
    
    public function invalidateByVersion(string $articleId): void
    {
        // Version increment করলে পুরনো cache key আর match করবে না
        $this->redis->incr("article_version:{$articleId}");
    }
    
    // ===========================
    // WRITE-THROUGH PATTERN
    // ===========================
    
    /**
     * Write-Through: Cache ও DB একসাথে write
     */
    public function updateArticleWriteThrough(string $id, array $data): bool
    {
        // Step 1: Database update
        $success = $this->db->update('articles', $data, ['id' => $id]);
        
        if (!$success) {
            return false;
        }
        
        // Step 2: Cache update (not delete — write through!)
        $article = $this->db->query("SELECT * FROM articles WHERE id = ?", [$id]);
        $this->redis->setex("article:{$id}", 3600, json_encode($article));
        
        return true;
    }
    
    // ===========================
    // THUNDERING HERD SOLUTIONS
    // ===========================
    
    /**
     * Solution 1: Distributed Lock (Mutex)
     * শুধু একজন DB query করবে, বাকিরা wait করবে
     */
    public function getArticleWithLock(string $articleId): ?array
    {
        $cacheKey = "article:{$articleId}";
        $lockKey = "lock:article:{$articleId}";
        
        $cached = $this->redis->get($cacheKey);
        if ($cached !== null) {
            return json_decode($cached, true);
        }
        
        // Lock acquire করার চেষ্টা (NX = only if not exists, EX = 5s timeout)
        $lockAcquired = $this->redis->set($lockKey, '1', ['NX', 'EX' => 5]);
        
        if ($lockAcquired) {
            try {
                // Lock পেয়েছি — DB query করি
                $article = $this->db->query("SELECT * FROM articles WHERE id = ?", [$articleId]);
                $this->redis->setex($cacheKey, 3600, json_encode($article));
                return $article;
            } finally {
                $this->redis->del($lockKey); // Lock release
            }
        }
        
        // Lock পাইনি — wait করে আবার cache check
        usleep(100000); // 100ms wait
        $cached = $this->redis->get($cacheKey);
        if ($cached !== null) {
            return json_decode($cached, true);
        }
        
        // Fallback: direct DB query (rare case)
        return $this->db->query("SELECT * FROM articles WHERE id = ?", [$articleId]);
    }
    
    /**
     * Solution 2: Probabilistic Early Expiration
     * TTL expire হওয়ার আগেই refresh করা
     */
    public function getArticleWithEarlyExpiry(string $articleId): ?array
    {
        $cacheKey = "article:{$articleId}";
        
        $cached = $this->redis->get($cacheKey);
        $ttl = $this->redis->ttl($cacheKey);
        
        if ($cached !== null) {
            // Early recomputation check
            $beta = 60; // Tuning parameter
            $shouldRecompute = (mt_rand() / mt_getrandmax()) < exp(-$ttl / $beta);
            
            if (!$shouldRecompute) {
                return json_decode($cached, true); // Normal return
            }
            // Fall through to recompute (probabilistically)
        }
        
        // Recompute (either cache miss or early refresh)
        $article = $this->db->query("SELECT * FROM articles WHERE id = ?", [$articleId]);
        $this->redis->setex($cacheKey, 3600, json_encode($article));
        
        return $article;
    }
    
    /**
     * Solution 3: Pre-computation (Background refresh)
     * Expire হওয়ার আগে background-এ refresh
     */
    public function schedulePreComputation(): void
    {
        // জনপ্রিয় articles cache যেগুলো শীঘ্রই expire হবে
        $keys = $this->redis->keys("article:*");
        
        foreach ($keys as $key) {
            $ttl = $this->redis->ttl($key);
            
            // TTL ৫ মিনিটের কম হলে background refresh
            if ($ttl > 0 && $ttl < 300) {
                $articleId = str_replace('article:', '', $key);
                $this->dispatchRefreshJob($articleId);
            }
        }
    }
    
    // ===========================
    // MULTI-LEVEL INVALIDATION
    // ===========================
    
    /**
     * Prothom Alo-style multi-level cache invalidation
     * Browser → CDN → Redis → DB Query Cache
     */
    public function invalidateAllLevels(string $articleId): array
    {
        $results = [];
        
        // L3: Application cache (Redis)
        $this->redis->del("article:{$articleId}");
        $this->redis->del("article_list:homepage");
        $this->redis->del("article_list:category:*");
        $results['redis'] = 'invalidated';
        
        // L2: CDN purge
        try {
            $this->cdnClient->purge([
                "/articles/{$articleId}",
                "/api/articles/{$articleId}",
                "/",  // Homepage
            ]);
            $results['cdn'] = 'purged';
        } catch (\Exception $e) {
            $results['cdn'] = 'failed: ' . $e->getMessage();
        }
        
        // L1: Browser cache (via headers in next response)
        // Cache-Control: no-cache, must-revalidate
        // অথবা WebSocket/Server-Sent Events দিয়ে push
        $results['browser'] = 'will expire on next request (Cache-Control)';
        
        // L4: DB query cache
        $this->db->query("RESET QUERY CACHE");
        $results['db_query_cache'] = 'reset';
        
        return $results;
    }
    
    // Helper methods
    private function invalidateArticleCache(string $id): void
    {
        $this->redis->del("article:{$id}");
        $this->redis->del("article_list:homepage");
    }
    
    private function purgeCDN(string $articleId): void
    {
        $this->cdnClient->purge(["/articles/{$articleId}"]);
    }
    
    private function publishEvent(string $event, array $data): void { }
    private function dispatchRefreshJob(string $articleId): void { }
}

/**
 * Cache Warming Service
 * সিস্টেম start/deploy-এর পর cache warm করা
 */
class CacheWarmingService
{
    private $redis;
    private $db;
    
    public function __construct($redis, $db)
    {
        $this->redis = $redis;
        $this->db = $db;
    }
    
    /**
     * Deploy-এর পর popular content cache warm করা
     * Prothom Alo: Top 100 articles + homepage data
     */
    public function warmAfterDeploy(): array
    {
        $warmed = [];
        
        // Top articles by views (last 24h)
        $topArticles = $this->db->query(
            "SELECT * FROM articles ORDER BY views_24h DESC LIMIT 100"
        );
        
        foreach ($topArticles as $article) {
            $key = "article:{$article['id']}";
            $this->redis->setex($key, 3600, json_encode($article));
            $warmed[] = $key;
        }
        
        // Homepage data
        $homepage = $this->db->query(
            "SELECT * FROM articles WHERE featured = 1 ORDER BY published_at DESC LIMIT 20"
        );
        $this->redis->setex('article_list:homepage', 300, json_encode($homepage));
        $warmed[] = 'article_list:homepage';
        
        // Category pages
        $categories = ['politics', 'sports', 'entertainment', 'technology'];
        foreach ($categories as $cat) {
            $articles = $this->db->query(
                "SELECT * FROM articles WHERE category = ? ORDER BY published_at DESC LIMIT 20",
                [$cat]
            );
            $key = "article_list:category:{$cat}";
            $this->redis->setex($key, 600, json_encode($articles));
            $warmed[] = $key;
        }
        
        return $warmed;
    }
    
    /**
     * Staggered TTL - সব cache একসাথে expire যেন না হয়
     */
    public function setWithJitter(string $key, string $value, int $baseTtl): void
    {
        // ±20% jitter add করা
        $jitter = (int) ($baseTtl * 0.2 * (mt_rand() / mt_getrandmax() * 2 - 1));
        $finalTtl = $baseTtl + $jitter;
        
        $this->redis->setex($key, $finalTtl, $value);
    }
}

// ব্যবহার উদাহরণ:
$cache = new CacheInvalidationService($redis, $db, $cdnClient);

// Article পড়া (cache-aside)
$article = $cache->getArticle('breaking-news-123');

// Article publish (multi-level invalidation)
$cache->publishArticle([
    'id' => 'new-article-456',
    'title' => 'বাংলাদেশ ক্রিকেট জিতলো!',
    'category' => 'sports'
]);

// Thundering herd protection
$article = $cache->getArticleWithLock('viral-article-789');
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Cache Invalidation Service
 * Event-driven cache invalidation with Pub/Sub
 * Example: Prothom Alo real-time article updates
 */

const Redis = require('ioredis');

class CacheInvalidationService {
    constructor(redisConfig) {
        this.cache = new Redis(redisConfig);
        this.subscriber = new Redis(redisConfig);
        this.publisher = new Redis(redisConfig);
        this.localCache = new Map(); // L1 in-process cache
        this.locks = new Map();
        
        this.setupSubscriptions();
    }

    // ===========================
    // EVENT-BASED INVALIDATION
    // ===========================

    /**
     * Pub/Sub based cache invalidation
     * যেকোনো server-এ write হলে সব server-এ invalidate
     */
    setupSubscriptions() {
        this.subscriber.subscribe('cache:invalidate', (err) => {
            if (err) console.error('Subscribe error:', err);
        });

        this.subscriber.on('message', (channel, message) => {
            if (channel === 'cache:invalidate') {
                const { keys, tags, pattern } = JSON.parse(message);
                
                // Local cache invalidate
                if (keys) {
                    keys.forEach(key => this.localCache.delete(key));
                }
                if (tags) {
                    this.invalidateLocalByTags(tags);
                }
                if (pattern) {
                    this.invalidateLocalByPattern(pattern);
                }
            }
        });
    }

    /**
     * Publish invalidation event
     * সব server instances কে জানানো
     */
    async publishInvalidation(options) {
        await this.publisher.publish(
            'cache:invalidate',
            JSON.stringify(options)
        );
    }

    // ===========================
    // CACHE-ASIDE WITH STAMPEDE PROTECTION
    // ===========================

    /**
     * Get with distributed lock (stampede protection)
     * Prothom Alo homepage articles
     */
    async getWithLock(key, fetchFn, ttl = 3600) {
        // L1: In-process cache check
        if (this.localCache.has(key)) {
            const local = this.localCache.get(key);
            if (local.expiresAt > Date.now()) {
                return local.value;
            }
            this.localCache.delete(key);
        }

        // L2: Redis cache check
        const cached = await this.cache.get(key);
        if (cached !== null) {
            const parsed = JSON.parse(cached);
            // L1 cache set (short TTL)
            this.localCache.set(key, {
                value: parsed,
                expiresAt: Date.now() + 10000 // 10s local cache
            });
            return parsed;
        }

        // Cache MISS - acquire lock
        const lockKey = `lock:${key}`;
        const lockId = `${process.pid}-${Date.now()}`;
        
        const acquired = await this.cache.set(lockKey, lockId, 'NX', 'EX', 5);
        
        if (acquired) {
            try {
                // আমি lock পেয়েছি - data fetch করি
                const data = await fetchFn();
                
                // Redis-এ set
                await this.cache.setex(key, ttl, JSON.stringify(data));
                
                // Local cache-এও set
                this.localCache.set(key, {
                    value: data,
                    expiresAt: Date.now() + 10000
                });
                
                return data;
            } finally {
                // Lock release (only if we still own it)
                const currentLock = await this.cache.get(lockKey);
                if (currentLock === lockId) {
                    await this.cache.del(lockKey);
                }
            }
        }

        // Lock পাইনি - retry with backoff
        await new Promise(resolve => setTimeout(resolve, 100));
        const retryData = await this.cache.get(key);
        if (retryData) return JSON.parse(retryData);
        
        // Last resort: direct fetch
        return await fetchFn();
    }

    // ===========================
    // WRITE-BEHIND (ASYNC WRITE)
    // ===========================

    /**
     * Write-Behind Pattern
     * Grameenphone-এর মতো high-write scenarios
     * Cache-এ immediately write, DB-তে batch async
     */
    async writeWithBehind(key, value, ttl = 3600) {
        // Immediate cache write
        await this.cache.setex(key, ttl, JSON.stringify(value));
        
        // Queue for async DB write
        await this.cache.rpush('write_behind_queue', JSON.stringify({
            key,
            value,
            timestamp: Date.now()
        }));
        
        return { status: 'cached', persistence: 'queued' };
    }

    /**
     * Background worker: Queue থেকে DB-তে batch write
     */
    async processWriteBehindQueue(batchSize = 100) {
        const items = [];
        
        for (let i = 0; i < batchSize; i++) {
            const item = await this.cache.lpop('write_behind_queue');
            if (!item) break;
            items.push(JSON.parse(item));
        }

        if (items.length === 0) return;

        // Batch database write
        try {
            await this.batchDatabaseWrite(items);
            console.log(`Written ${items.length} items to DB`);
        } catch (error) {
            // Re-queue failed items
            for (const item of items) {
                await this.cache.rpush('write_behind_queue', JSON.stringify(item));
            }
            console.error('Write-behind failed, re-queued:', error.message);
        }
    }

    // ===========================
    // PROBABILISTIC EARLY EXPIRATION
    // ===========================

    /**
     * XFetch algorithm - probabilistic early recomputation
     * Cache stampede prevention without locks
     */
    async xFetch(key, fetchFn, ttl = 3600, beta = 1.0) {
        const cacheData = await this.cache.get(`xfetch:${key}`);
        
        if (cacheData) {
            const { value, delta, expiry } = JSON.parse(cacheData);
            const now = Date.now() / 1000;
            
            // XFetch probability calculation
            const timeRemaining = expiry - now;
            const random = -delta * beta * Math.log(Math.random());
            
            if (timeRemaining > random) {
                // Cache still fresh enough
                return value;
            }
            // Fall through to recompute (early refresh)
        }

        // Compute new value
        const startTime = Date.now();
        const value = await fetchFn();
        const delta = (Date.now() - startTime) / 1000; // computation time

        // Store with metadata
        const expiry = (Date.now() / 1000) + ttl;
        await this.cache.setex(
            `xfetch:${key}`,
            ttl + 60, // Extra buffer TTL
            JSON.stringify({ value, delta, expiry })
        );

        return value;
    }

    // ===========================
    // TAG-BASED INVALIDATION
    // ===========================

    /**
     * Tag-based cache management
     * Prothom Alo: "sports" tag invalidate করলে সব sports article cache clear
     */
    async setWithTags(key, value, ttl, tags = []) {
        const pipeline = this.cache.pipeline();
        
        // Set the value
        pipeline.setex(key, ttl, JSON.stringify(value));
        
        // Associate key with each tag
        for (const tag of tags) {
            pipeline.sadd(`tag:${tag}`, key);
            pipeline.expire(`tag:${tag}`, ttl + 3600); // Tag survives longer
        }
        
        await pipeline.exec();
    }

    async invalidateByTags(tags) {
        const pipeline = this.cache.pipeline();
        const allKeys = new Set();
        
        for (const tag of tags) {
            const keys = await this.cache.smembers(`tag:${tag}`);
            keys.forEach(k => allKeys.add(k));
            pipeline.del(`tag:${tag}`);
        }

        for (const key of allKeys) {
            pipeline.del(key);
        }

        await pipeline.exec();

        // Notify other servers
        await this.publishInvalidation({ tags });

        return allKeys.size;
    }

    // ===========================
    // CACHE WARMING
    // ===========================

    /**
     * Intelligent Cache Warming
     * Deploy-এর পর বা cold start-এ popular data pre-load
     */
    async warmCache(warmingStrategy) {
        const strategies = {
            // Top articles by popularity
            popular: async () => {
                const articles = await db.query(
                    'SELECT * FROM articles ORDER BY views DESC LIMIT 200'
                );
                for (const article of articles) {
                    const jitter = Math.floor(Math.random() * 600); // 0-10min jitter
                    await this.cache.setex(
                        `article:${article.id}`,
                        3600 + jitter, // Staggered TTL!
                        JSON.stringify(article)
                    );
                }
                return articles.length;
            },
            
            // Recently published
            recent: async () => {
                const articles = await db.query(
                    'SELECT * FROM articles WHERE published_at > NOW() - INTERVAL 24 HOUR'
                );
                for (const article of articles) {
                    await this.cache.setex(
                        `article:${article.id}`,
                        1800,
                        JSON.stringify(article)
                    );
                }
                return articles.length;
            },
            
            // Based on access patterns (predictive)
            predictive: async () => {
                const hour = new Date().getHours();
                // সকালে sports, দুপুরে politics, রাতে entertainment
                const category = hour < 12 ? 'sports' 
                    : hour < 18 ? 'politics' 
                    : 'entertainment';
                    
                const articles = await db.query(
                    'SELECT * FROM articles WHERE category = ? LIMIT 50',
                    [category]
                );
                for (const article of articles) {
                    await this.cache.setex(
                        `article:${article.id}`,
                        1800,
                        JSON.stringify(article)
                    );
                }
                return articles.length;
            }
        };

        const strategy = strategies[warmingStrategy];
        if (!strategy) throw new Error(`Unknown strategy: ${warmingStrategy}`);
        
        return await strategy();
    }

    // Helper methods
    invalidateLocalByTags(tags) {
        for (const [key, entry] of this.localCache) {
            if (entry.tags && entry.tags.some(t => tags.includes(t))) {
                this.localCache.delete(key);
            }
        }
    }
    
    invalidateLocalByPattern(pattern) {
        const regex = new RegExp(pattern.replace('*', '.*'));
        for (const key of this.localCache.keys()) {
            if (regex.test(key)) this.localCache.delete(key);
        }
    }
    
    async batchDatabaseWrite(items) { /* DB batch insert */ }
}

// ====== ব্যবহার উদাহরণ ======

const cacheService = new CacheInvalidationService({
    host: 'redis-cluster.prothomalo.internal',
    port: 6379
});

// Article read with stampede protection
const article = await cacheService.getWithLock(
    'article:breaking-news-123',
    async () => {
        return await db.query('SELECT * FROM articles WHERE id = ?', ['breaking-news-123']);
    },
    3600
);

// Article publish → multi-level invalidation
await cacheService.publishInvalidation({
    keys: ['article:123', 'article_list:homepage'],
    tags: ['sports', 'cricket'],
    pattern: 'article_list:*'
});

// Tag-based caching
await cacheService.setWithTags(
    'article:world-cup-final',
    articleData,
    3600,
    ['sports', 'cricket', 'featured']
);

// Invalidate all cricket content
const count = await cacheService.invalidateByTags(['cricket']);
console.log(`Invalidated ${count} cached items`);

// Cache warming after deploy
const warmed = await cacheService.warmCache('popular');
console.log(`Warmed ${warmed} articles into cache`);
```

---

## 🔄 Cache Coherence in Distributed Systems

```
┌──────────────────────────────────────────────────────────────┐
│        DISTRIBUTED CACHE COHERENCE                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Server-1 (Dhaka)          Server-2 (Chittagong)             │
│  ┌──────────────┐          ┌──────────────┐                  │
│  │ Local Cache  │          │ Local Cache  │                  │
│  │ article:123  │          │ article:123  │                  │
│  │ = "v1"       │          │ = "v1"       │                  │
│  └──────┬───────┘          └──────┬───────┘                  │
│         │                         │                          │
│         └─────────┬───────────────┘                          │
│                   │                                          │
│            ┌──────┴──────┐                                   │
│            │ Redis Pub/Sub│  ◄── Invalidation Channel        │
│            └──────┬──────┘                                   │
│                   │                                          │
│  Article Update:  │                                          │
│  1. Writer updates DB                                        │
│  2. Writer publishes "invalidate article:123"                │
│  3. ALL servers receive message                              │
│  4. ALL servers delete local cache                           │
│  5. Next read → fresh data from DB/Redis                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## ✅❌ কখন ব্যবহার করবেন / করবেন না

### Strategy Selection Guide:

| Strategy | ব্যবহার করবেন | ব্যবহার করবেন না |
|---|---|---|
| **TTL-based** | Simple data, acceptable staleness | Real-time critical data |
| **Event-based** | Real-time updates needed | Simple static content |
| **Write-Through** | Read-heavy, must be fresh | Write-heavy workloads |
| **Write-Behind** | Write-heavy, can afford lag | Critical consistency |
| **Cache-Aside** | General purpose, flexible | When cache must always be warm |
| **Tag-based** | Complex relationships | Simple key-value data |

### ✅ Cache Invalidation দরকার:

| Scenario | Strategy | উদাহরণ |
|---|---|---|
| Breaking news publish | Event-based + CDN purge | Prothom Alo |
| Price change (sale) | Event-based + version | Daraz flash sale |
| User profile update | Write-through | Pathao driver info |
| Balance change | Immediate invalidation | bKash balance |
| Trending content | TTL (short) + early refresh | Grameenphone trending |

### ❌ Cache Invalidation এড়িয়ে চলুন:

| Scenario | Alternative | কারণ |
|---|---|---|
| Static assets (images) | Fingerprinted URLs | Never invalidate, new URL |
| Immutable data (logs) | No cache needed | Data never changes |
| Highly personalized | No shared cache | Too many variations |
| Sub-second freshness | Don't cache at all | Invalidation too complex |

### 🎯 TTL Selection Guide:

```
┌──────────────────────────────────────────────┐
│  CONTENT TYPE          │  RECOMMENDED TTL     │
├────────────────────────┼──────────────────────┤
│  Static assets (CSS)   │  1 year (immutable)  │
│  User profile photo    │  24 hours            │
│  News article content  │  1 hour              │
│  Homepage article list │  5 minutes           │
│  Stock price           │  10 seconds          │
│  Live cricket score    │  0 (no cache)        │
│  bKash balance         │  0 (no cache)        │
│  Product catalog       │  30 minutes          │
│  Search results        │  15 minutes          │
└────────────────────────┴──────────────────────┘
```

---

## 📝 সারসংক্ষেপ

> **মনে রাখবেন**: Cache invalidation কঠিন কারণ আপনাকে একসাথে performance (fast reads) এবং freshness (correct data) balance করতে হয়।

- Cache-Aside সবচেয়ে common এবং flexible pattern
- Thundering herd সমস্যা serious — lock বা probabilistic expiry ব্যবহার করুন
- Multi-level caching-এ invalidation cascade দরকার
- TTL-এ jitter যোগ করুন (সব cache একসাথে expire হওয়া আটকান)
- Event-driven invalidation সবচেয়ে responsive কিন্তু complex
- Cache warming deploy-এর পর cold-start latency কমায়
