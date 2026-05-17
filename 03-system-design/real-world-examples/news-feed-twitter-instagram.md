# 📰 নিউজ ফিড সিস্টেম ডিজাইন (Twitter/Instagram)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Twitter, Instagram, Facebook News Feed

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#️-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#️-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বিবরণ |
|---|---------|--------|
| 1 | Post Creation | ইউজার টেক্সট, ইমেজ, ভিডিও পোস্ট করতে পারবে |
| 2 | Feed Generation | ফলো করা ইউজারদের পোস্ট টাইমলাইনে দেখাবে |
| 3 | Follow/Unfollow | অন্য ইউজারকে ফলো/আনফলো করা |
| 4 | Like/Comment | পোস্টে লাইক এবং কমেন্ট করা |
| 5 | Media Upload | ছবি/ভিডিও আপলোড ও প্রসেসিং |
| 6 | Share/Retweet | পোস্ট শেয়ার বা রিটুইট করা |

### Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────┐
│  Performance:  Feed latency < 200ms                     │
│  Availability: 99.99% uptime                            │
│  Consistency:  Eventual consistency (acceptable)        │
│  Scalability:  500M+ DAU support                        │
│  Durability:   কোনো পোস্ট হারানো যাবে না               │
└─────────────────────────────────────────────────────────┘
```

### 🇧🇩 Bangladesh Context

- **প্রথম আলো** অনলাইনে প্রতিদিন ১০০+ নিউজ পোস্ট করে — তাদের ফলোয়ার ১ কোটি+
- **বাংলাদেশে Facebook ইউজার** ৫ কোটি+ — নিউজ ফিড সবচেয়ে বেশি ব্যবহৃত ফিচার
- **Grameenphone** এর ডাটা প্ল্যান অনুযায়ী ইউজাররা সাধারণত 1-2GB/মাস ব্যবহার করে
- **বাংলাদেশী ইনফ্লুয়েন্সার** (যেমন: Salman Muqtadir, Tawhid Afridi) — ১০M+ ফলোয়ার
- **ক্রিকেট ম্যাচ** চলাকালীন ট্রাফিক ৫-১০x বেড়ে যায়
- **Low bandwidth optimization** দরকার — গ্রামীণ এলাকায় 2G/3G কানেকশন

---

## 📊 Back-of-envelope Estimation

### ইউজার ও ট্রাফিক

```
DAU (Daily Active Users):     500M (globally), 50M (Bangladesh)
Posts/day:                     200M new posts
Feed reads/day:                10B (avg 20 reads/user/day)
Peak QPS (feed reads):         10B / 86400 = ~115K QPS
Peak burst (cricket match):    ~500K QPS

Avg followers per user:        200
Celebrity followers:           10M+ (e.g., Shakib Al Hasan)
```

### স্টোরেজ Estimation

```
প্রতিটি পোস্ট (text):        ~1 KB
প্রতিটি ইমেজ:                ~200 KB (compressed)
প্রতিটি ভিডিও:               ~5 MB (compressed, 30s)

Daily storage (posts):         200M * 1KB = 200 GB/day
Daily storage (media):         50M images * 200KB = 10 TB/day
Yearly total:                  ~4 PB/year

Feed cache per user:           500 post IDs * 8 bytes = 4 KB
Total feed cache:              500M * 4KB = 2 TB (Redis)
```

### Bandwidth

```
Incoming (writes):             200M * 1KB = 200 GB/day = ~2.3 MB/s
Outgoing (reads):              10B * 5KB = 50 TB/day = ~580 GB/s

Bangladesh context:
- Grameenphone users: avg 50 feed refreshes/day
- Per refresh: ~10 posts * 50KB (with thumbnails) = 500KB
- Monthly per user: 50 * 500KB * 30 = 750MB (fits 1GB plan!)
```

---

## 🏗️ High-Level Design

### সিস্টেম আর্কিটেকচার

```
                         ┌──────────────────────┐
                         │    Load Balancer      │
                         │    (Nginx/HAProxy)    │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
              │  Post API  │  │  Feed API  │  │ User API  │
              │  (Laravel) │  │ (Node.js)  │  │ (Laravel) │
              └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
                    │               │               │
         ┌──────────┘        ┌──────┘               │
         │                   │                      │
    ┌────▼────┐       ┌──────▼──────┐        ┌─────▼─────┐
    │  Post   │       │  Feed Cache │        │  Social   │
    │  Store  │       │   (Redis)   │        │  Graph    │
    │ (MySQL) │       └──────┬──────┘        │  (Neo4j)  │
    └────┬────┘              │               └───────────┘
         │                   │
    ┌────▼────────────────────▼────┐
    │       Fan-out Service        │
    │    (Background Workers)      │
    │       Queue: RabbitMQ        │
    └──────────────────────────────┘
```

### Push vs Pull Model

```
┌─────────────── PUSH MODEL (Fan-out on Write) ────────────────┐
│                                                               │
│  User A posts ──► Fan-out Service ──► Write to all           │
│                                        followers' caches      │
│                                                               │
│  장점: Feed read = O(1), খুব fast                             │
│  단점: Celebrity post = millions of writes                    │
│                                                               │
│  Example: Shakib posts → 10M cache updates! 😱               │
└───────────────────────────────────────────────────────────────┘

┌─────────────── PULL MODEL (Fan-out on Read) ─────────────────┐
│                                                               │
│  User B opens feed ──► Query all followed users' posts       │
│                         ──► Merge & sort ──► Return           │
│                                                               │
│  장점: Write = O(1), no wasted fan-out                        │
│  단점: Feed read = slow (merge N users' posts)               │
│                                                               │
│  Example: User follows 500 people → 500 queries! 😱          │
└───────────────────────────────────────────────────────────────┘

┌─────────────── HYBRID MODEL (Best of Both) ──────────────────┐
│                                                               │
│  Regular user (< 10K followers): PUSH model                  │
│  Celebrity (> 10K followers):    PULL model                  │
│                                                               │
│  Feed = cached_posts (push) + celebrity_posts (pull at read) │
│                                                               │
│  ✅ Twitter/Instagram এই approach ব্যবহার করে                │
└───────────────────────────────────────────────────────────────┘
```

---

## 💻 Detailed Design

### ১. Database Schema

```sql
-- Posts Table (MySQL / PostgreSQL)
CREATE TABLE posts (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id         BIGINT NOT NULL,
    content         TEXT,
    media_urls      JSON,
    post_type       ENUM('text', 'image', 'video', 'retweet'),
    like_count      INT DEFAULT 0,
    comment_count   INT DEFAULT 0,
    share_count     INT DEFAULT 0,
    is_deleted      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_created (user_id, created_at DESC),
    INDEX idx_created (created_at DESC)
);

-- Follow Graph (MySQL or Neo4j)
CREATE TABLE follows (
    follower_id     BIGINT NOT NULL,
    followee_id     BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);

-- Feed Cache Structure (Redis)
-- Key: feed:{user_id}
-- Value: Sorted Set (score = timestamp, member = post_id)
-- TTL: 7 days
```

### ২. Post Creation with Fan-out (PHP/Laravel)

```php
<?php
// app/Services/PostService.php

namespace App\Services;

use App\Models\Post;
use App\Jobs\FanOutPost;
use App\Services\MediaService;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Cache;

class PostService
{
    private const CELEBRITY_THRESHOLD = 10000;
    private const FEED_MAX_SIZE = 500;

    public function createPost(int $userId, array $data): Post
    {
        // পোস্ট তৈরি করা
        $post = Post::create([
            'user_id'    => $userId,
            'content'    => $data['content'],
            'media_urls' => $this->processMedia($data['media'] ?? []),
            'post_type'  => $data['type'] ?? 'text',
        ]);

        // Fan-out decision: celebrity নাকি regular user?
        $followerCount = $this->getFollowerCount($userId);

        if ($followerCount < self::CELEBRITY_THRESHOLD) {
            // Regular user: Push model — fan-out on write
            FanOutPost::dispatch($post)->onQueue('fan-out');
        } else {
            // Celebrity: Pull model — শুধু post store এ রাখো
            // Followers রা read time এ fetch করবে
            Cache::tags(['celebrity_posts'])->put(
                "celebrity:{$userId}:latest",
                $post->id,
                now()->addDays(7)
            );
        }

        // Post cache update
        Redis::zadd(
            "user_posts:{$userId}",
            $post->created_at->timestamp,
            $post->id
        );

        return $post;
    }

    private function getFollowerCount(int $userId): int
    {
        return Cache::remember(
            "follower_count:{$userId}",
            3600,
            fn() => \DB::table('follows')
                ->where('followee_id', $userId)
                ->count()
        );
    }

    private function processMedia(array $media): array
    {
        // Bangladesh optimization: multiple resolutions
        // Low quality for 2G/3G users
        return array_map(function ($file) {
            return [
                'original' => MediaService::upload($file),
                'thumb'    => MediaService::resize($file, 150, 150),
                'medium'   => MediaService::resize($file, 600, 600),
                'low_quality' => MediaService::compress($file, quality: 30),
            ];
        }, $media);
    }
}
```

```php
<?php
// app/Jobs/FanOutPost.php

namespace App\Jobs;

use App\Models\Post;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Redis;

class FanOutPost implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $timeout = 300;

    public function __construct(private Post $post) {}

    public function handle(): void
    {
        // Follower দের feed cache এ পোস্ট push করা
        $followers = \DB::table('follows')
            ->where('followee_id', $this->post->user_id)
            ->pluck('follower_id');

        // Batch processing — ১০০০ জন করে
        $followers->chunk(1000)->each(function ($chunk) {
            $pipeline = Redis::pipeline();

            foreach ($chunk as $followerId) {
                $key = "feed:{$followerId}";

                // Sorted set এ add (score = timestamp)
                $pipeline->zadd(
                    $key,
                    $this->post->created_at->timestamp,
                    $this->post->id
                );

                // Feed size limit maintain (পুরনো পোস্ট remove)
                $pipeline->zremrangebyrank($key, 0, -501);
            }

            $pipeline->exec();
        });
    }
}
```

### ৩. Feed Retrieval (Node.js/Express)

```javascript
// src/services/feedService.js

const Redis = require('ioredis');
const { Pool } = require('pg');

const redis = new Redis.Cluster([
  { host: 'redis-1.bd-dc.internal', port: 6379 },
  { host: 'redis-2.bd-dc.internal', port: 6379 },
  { host: 'redis-3.bd-dc.internal', port: 6379 },
]);

const pgPool = new Pool({ connectionString: process.env.DATABASE_URL });

const FEED_PAGE_SIZE = 20;
const CELEBRITY_THRESHOLD = 10000;

class FeedService {
  /**
   * হাইব্রিড ফিড জেনারেশন
   * Step 1: Redis cache থেকে pre-computed feed নাও (push model)
   * Step 2: Celebrity দের latest posts merge করো (pull model)
   * Step 3: Rank & paginate
   */
  async getFeed(userId, cursor = null, options = {}) {
    const { quality = 'high' } = options; // Bangladesh: low quality for slow connections

    // Step 1: Cached feed from fan-out (regular users' posts)
    const cachedPostIds = await this.getCachedFeed(userId, cursor);

    // Step 2: Celebrity posts (pull at read time)
    const celebrityPostIds = await this.getCelebrityPosts(userId);

    // Step 3: Merge, deduplicate, sort
    const allPostIds = [...new Set([...cachedPostIds, ...celebrityPostIds])];

    // Step 4: Fetch full post data
    const posts = await this.hydratePostIds(allPostIds);

    // Step 5: Rank posts (engagement score + recency)
    const rankedPosts = this.rankPosts(posts, userId);

    // Step 6: Paginate
    const page = rankedPosts.slice(0, FEED_PAGE_SIZE);
    const nextCursor = page.length > 0
      ? page[page.length - 1].created_at
      : null;

    // Bangladesh optimization: image quality based on connection
    if (quality === 'low') {
      page.forEach(post => {
        if (post.media_urls) {
          post.media_urls = post.media_urls.map(m => ({
            ...m, url: m.low_quality || m.thumb
          }));
        }
      });
    }

    return { posts: page, nextCursor, hasMore: rankedPosts.length > FEED_PAGE_SIZE };
  }

  async getCachedFeed(userId, cursor) {
    const maxScore = cursor || '+inf';
    const minScore = '-inf';

    // Redis Sorted Set: score = timestamp, member = post_id
    const postIds = await redis.zrevrangebyscore(
      `feed:${userId}`,
      maxScore,
      minScore,
      'LIMIT', 0, FEED_PAGE_SIZE * 3 // Extra for merging
    );

    return postIds.map(id => parseInt(id));
  }

  async getCelebrityPosts(userId) {
    // ইউজার যেসব celebrity কে follow করে তাদের latest posts
    const celebrities = await pgPool.query(`
      SELECT f.followee_id
      FROM follows f
      JOIN user_stats us ON us.user_id = f.followee_id
      WHERE f.follower_id = $1 AND us.follower_count >= $2
    `, [userId, CELEBRITY_THRESHOLD]);

    if (celebrities.rows.length === 0) return [];

    // Celebrity দের recent posts fetch
    const celebrityIds = celebrities.rows.map(r => r.followee_id);
    const posts = await pgPool.query(`
      SELECT id FROM posts
      WHERE user_id = ANY($1)
        AND created_at > NOW() - INTERVAL '24 hours'
        AND is_deleted = false
      ORDER BY created_at DESC
      LIMIT 50
    `, [celebrityIds]);

    return posts.rows.map(r => r.id);
  }

  async hydratePostIds(postIds) {
    if (postIds.length === 0) return [];

    // Multi-get from Redis cache first
    const cacheKeys = postIds.map(id => `post:${id}`);
    const cached = await redis.mget(cacheKeys);

    const result = [];
    const missedIds = [];

    cached.forEach((data, index) => {
      if (data) {
        result.push(JSON.parse(data));
      } else {
        missedIds.push(postIds[index]);
      }
    });

    // Cache miss: DB থেকে fetch করো
    if (missedIds.length > 0) {
      const dbPosts = await pgPool.query(
        'SELECT * FROM posts WHERE id = ANY($1) AND is_deleted = false',
        [missedIds]
      );

      // Cache এ store করো (24 hour TTL)
      const pipeline = redis.pipeline();
      dbPosts.rows.forEach(post => {
        pipeline.setex(`post:${post.id}`, 86400, JSON.stringify(post));
        result.push(post);
      });
      await pipeline.exec();
    }

    return result;
  }

  rankPosts(posts, userId) {
    // Ranking formula: engagement + recency + affinity
    return posts
      .map(post => ({
        ...post,
        score: this.calculateScore(post, userId)
      }))
      .sort((a, b) => b.score - a.score);
  }

  calculateScore(post, userId) {
    const now = Date.now();
    const ageHours = (now - new Date(post.created_at).getTime()) / 3600000;

    // Engagement score
    const engagement = (post.like_count * 1)
      + (post.comment_count * 3)
      + (post.share_count * 5);

    // Recency decay (half-life = 6 hours)
    const recencyScore = Math.pow(0.5, ageHours / 6);

    // Affinity (simplified — real system uses ML)
    const affinity = 1.0; // TODO: interaction history based

    return (engagement * 0.3) + (recencyScore * 100 * 0.5) + (affinity * 0.2);
  }
}

module.exports = new FeedService();
```

```javascript
// src/routes/feed.js — Express Router

const express = require('express');
const router = express.Router();
const feedService = require('../services/feedService');
const { authenticate } = require('../middleware/auth');
const rateLimit = require('express-rate-limit');

// Rate limiting: Bangladesh context — Grameenphone fair usage
const feedLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 30, // 30 requests per minute
  message: { error: 'Too many requests. অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন।' }
});

router.get('/feed', authenticate, feedLimiter, async (req, res) => {
  try {
    const { cursor, quality } = req.query;
    const userId = req.user.id;

    // Connection quality detection (Bangladesh 2G/3G/4G)
    const connectionQuality = req.headers['x-connection-quality'] || 'high';

    const feed = await feedService.getFeed(userId, cursor, {
      quality: connectionQuality
    });

    res.json({
      success: true,
      data: feed.posts,
      pagination: {
        nextCursor: feed.nextCursor,
        hasMore: feed.hasMore
      }
    });
  } catch (error) {
    console.error('Feed generation error:', error);
    res.status(500).json({
      success: false,
      error: 'ফিড লোড করতে সমস্যা হয়েছে। আবার চেষ্টা করুন।'
    });
  }
});

module.exports = router;
```

### ৪. Feed Generation Flow (ASCII Diagram)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    FEED GENERATION FLOW                               │
└──────────────────────────────────────────────────────────────────────┘

  User A posts "BPL match update! 🏏"
       │
       ▼
  ┌─────────────┐     ┌─────────────────┐
  │  Post API   │────►│  MySQL (posts)   │
  │  (Laravel)  │     │  Store post      │
  └──────┬──────┘     └─────────────────┘
         │
         │  Is User A a celebrity?
         │
    ┌────┴────┐
    │ NO      │ YES (followers > 10K)
    ▼         ▼
┌────────┐  ┌────────────────────┐
│ Queue  │  │ Mark as celebrity  │
│ Job    │  │ post in cache      │
└───┬────┘  └────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│       Fan-out Workers            │
│                                  │
│  for each follower:              │
│    ZADD feed:{follower_id}       │
│         timestamp post_id        │
│                                  │
│  Batch size: 1000 followers      │
│  Workers: 100 concurrent         │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│  Redis Cluster (Feed Cache)      │
│                                  │
│  feed:user_101 = [post_999, ..]  │
│  feed:user_102 = [post_999, ..]  │
│  feed:user_103 = [post_999, ..]  │
│  ...                             │
└──────────────────────────────────┘

  ═══════════════════════════════════

  User B opens app (reads feed):
       │
       ▼
  ┌─────────────┐
  │  Feed API   │
  │  (Node.js)  │
  └──────┬──────┘
         │
    ┌────┴────────────────────┐
    │                         │
    ▼                         ▼
┌────────────┐    ┌───────────────────┐
│ Redis GET  │    │ Celebrity posts   │
│ feed:userB │    │ (pull at read)    │
└─────┬──────┘    └────────┬──────────┘
      │                    │
      └────────┬───────────┘
               │
               ▼
       ┌───────────────┐
       │  Merge + Rank │
       │  + Paginate   │
       └───────┬───────┘
               │
               ▼
       ┌───────────────┐
       │  Response to  │
       │  User B       │
       └───────────────┘
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### Push vs Pull vs Hybrid

```
┌─────────────┬──────────────────┬──────────────────┬──────────────────┐
│  Criteria   │      PUSH        │      PULL        │     HYBRID       │
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Write cost  │ O(N followers)   │ O(1)             │ O(N) regular     │
│             │ ❌ Expensive     │ ✅ Cheap         │    O(1) celebrity│
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Read cost   │ O(1)             │ O(N followees)   │ O(1) + O(K)     │
│             │ ✅ Very fast     │ ❌ Slow          │ K = celebrities  │
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Latency     │ < 50ms read      │ 200-500ms read   │ < 100ms read     │
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Celebrity   │ 10M writes/post  │ No problem       │ ✅ Handled       │
│ problem     │ ❌ Terrible      │ ✅ Fine          │                  │
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Storage     │ ❌ Huge (copies) │ ✅ Minimal       │ Moderate         │
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Freshness   │ ✅ Instant       │ Always fresh     │ ✅ Near-instant  │
├─────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Used by     │ Early Twitter    │ N/A (too slow)   │ Twitter/Instagram│
└─────────────┴──────────────────┴──────────────────┴──────────────────┘
```

### Chronological vs Ranked Feed

```
┌─────────────────┬────────────────────────┬────────────────────────┐
│  Aspect         │    Chronological       │    Ranked (ML)         │
├─────────────────┼────────────────────────┼────────────────────────┤
│ User experience │ Simple, predictable    │ Higher engagement      │
│ Implementation  │ Easy (sort by time)    │ Complex (ML pipeline)  │
│ Fairness        │ All posts equal        │ Popular posts favored  │
│ Bangladesh use  │ প্রথম আলো news feed   │ Facebook/Instagram     │
│ Controversy     │ None                   │ "Algorithm bias" 😤    │
│ Best for        │ Breaking news          │ Casual browsing        │
└─────────────────┴────────────────────────┴────────────────────────┘
```

### Storage: Redis vs Cassandra

```
┌───────────────┬─────────────────────────┬─────────────────────────┐
│  Feature      │      Redis              │      Cassandra          │
├───────────────┼─────────────────────────┼─────────────────────────┤
│ Speed         │ ✅ In-memory, < 1ms     │ 5-10ms SSD             │
│ Cost          │ ❌ Expensive (RAM)      │ ✅ Cheap (disk)        │
│ Persistence   │ RDB/AOF (risky)         │ ✅ Durable by default  │
│ Scalability   │ Cluster (limited)       │ ✅ Linear scale        │
│ Best for      │ Hot feed cache          │ Historical feed store   │
│ BD context    │ Cache recent 500 posts  │ Store all posts forever │
└───────────────┴─────────────────────────┴─────────────────────────┘

✅ Recommendation: Redis for active feed + Cassandra for archive
```

### Social Graph: SQL vs Graph DB

```
┌───────────────┬─────────────────────────┬─────────────────────────┐
│  Query Type   │      MySQL              │      Neo4j              │
├───────────────┼─────────────────────────┼─────────────────────────┤
│ Direct follow │ ✅ Fast (indexed)       │ ✅ Fast                │
│ 2nd degree    │ ❌ JOIN expensive       │ ✅ Natural traversal   │
│ Mutual friends│ ❌ Complex query        │ ✅ Simple pattern      │
│ "Who to       │ ❌ Very complex         │ ✅ Graph algorithms    │
│  follow"      │                         │                         │
│ Cost          │ ✅ Cheap, familiar      │ ❌ Expensive, learning │
│ Scale         │ Sharding complex        │ Good for social graphs  │
└───────────────┴─────────────────────────┴─────────────────────────┘

BD Context: ৫০M ইউজারের জন্য MySQL + Redis combination
           যথেষ্ট। Neo4j শুধু "People you may know" feature এর জন্য।
```

---

## 📈 কেস স্টাডি

### কেস ১: Celebrity Post — Shakib Al Hasan (10M followers)

```
Scenario: শাকিব IPL/BPL ম্যাচ শেষে পোস্ট করলেন "Great win today! 🏏"

Without Hybrid Model (Pure Push):
┌─────────────────────────────────────────────┐
│ 10M followers × 1 write = 10M Redis writes  │
│ Time: 10M / 100K writes/sec = 100 seconds!  │
│ Result: 100 সেকেন্ড ধরে fan-out চলবে       │
│         অনেক ফলোয়ার দেরিতে পোস্ট দেখবে    │
│ ❌ UNACCEPTABLE                              │
└─────────────────────────────────────────────┘

With Hybrid Model:
┌─────────────────────────────────────────────┐
│ Shakib marked as celebrity (followers > 10K)│
│ Post stored in posts table only             │
│ When follower opens feed:                   │
│   1. Get cached feed (regular users' posts) │
│   2. Pull Shakib's latest posts             │
│   3. Merge & rank                           │
│ Time: < 100ms per feed read                 │
│ ✅ PERFECT                                  │
└─────────────────────────────────────────────┘
```

### কেস ২: Viral Content — বাংলাদেশ vs ভারত ক্রিকেট ম্যাচ

```
Scenario: বাংলাদেশ T20 World Cup এ ভারতকে হারালো!
          ৫ মিনিটে ২ কোটি পোস্ট + ১০০ কোটি ফিড রিফ্রেশ

Traffic Pattern:
                     ▲ QPS
            500K ────┤         ╭──╮
                     │        ╭╯  ╰╮
            300K ────┤       ╭╯    ╰╮
                     │      ╭╯      ╰──╮
            100K ────┤─────╯            ╰────────
                     │
              0 ─────┼──┬──┬──┬──┬──┬──┬──┬──► Time
                     Match  Win  Post-match

Handling Strategy:
┌─────────────────────────────────────────────────────┐
│ 1. Auto-scaling: 3x → 15x instances within 2 min   │
│ 2. Write batching: Queue posts, process in batches  │
│ 3. Fan-out delay: Acceptable 30s delay for non-live │
│ 4. CDN caching: Trending posts cached at edge       │
│ 5. Rate limiting: 5 posts/min per user              │
│ 6. Graceful degradation:                            │
│    - Disable ranking, show chronological            │
│    - Skip celebrity pull, show cached only          │
│    - Reduce feed size from 20 to 10 posts           │
└─────────────────────────────────────────────────────┘

Bangladesh-specific:
- Grameenphone/Robi network congestion management
- ISP level CDN (BDIX peering) for media
- SMS notification fallback for 2G users
```

### কেস ৩: Content Moderation — ফেক নিউজ সমস্যা

```
Scenario: নির্বাচনের সময় ফেক নিউজ ভাইরাল হচ্ছে
          ৫ মিনিটে ১ লাখ শেয়ার!

Detection Pipeline:
┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌─────────┐
│  Post    │───►│  AI/ML   │───►│  Human       │───►│ Action  │
│ Created  │    │  Filter  │    │  Review (BD) │    │ Remove/ │
│          │    │  Score   │    │  Bangla NLP  │    │ Label   │
└──────────┘    └──────────┘    └──────────────┘    └─────────┘

Challenges in Bangladesh:
- বাংলা ভাষায় NLP model কম accurate
- Banglish (Roman Bangla) detection কঠিন
- Cultural context বোঝা AI এর জন্য কঠিন
- Real-time moderation team ২৪/৭ দরকার

Action Matrix:
┌───────────────┬────────────┬─────────────────────┐
│ Risk Level    │ Action     │ Timeline            │
├───────────────┼────────────┼─────────────────────┤
│ High (>0.9)   │ Auto-block │ Instant             │
│ Medium (0.7)  │ Limit reach│ < 5 min review      │
│ Low (0.5)     │ Flag       │ < 1 hour review     │
│ Safe (<0.5)   │ Allow      │ Periodic sampling   │
└───────────────┴────────────┴─────────────────────┘
```

---

## 🔧 Advanced Topics

### ১. Content Ranking ML Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│              FEED RANKING PIPELINE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Features:                                                  │
│  ┌──────────────────────────────────────────┐              │
│  │ • User-Post affinity (interaction history)│              │
│  │ • Post engagement rate (likes/impressions)│              │
│  │ • Content freshness (age in hours)        │              │
│  │ • Author credibility score                │              │
│  │ • Content type preference                 │              │
│  │ • Time of day (BD timezone: UTC+6)        │              │
│  │ • Network quality (2G/3G/4G/WiFi)        │              │
│  └──────────────────────────────────────────┘              │
│                                                             │
│  Model: Gradient Boosted Trees (LightGBM)                  │
│  Training: Daily retrain on last 7 days data               │
│  Serving: < 50ms inference for 500 candidates              │
│                                                             │
│  Score = 0.3 × engagement + 0.4 × affinity                │
│        + 0.2 × freshness + 0.1 × diversity                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### ২. Real-time Notifications (Node.js WebSocket)

```javascript
// src/services/notificationService.js

const WebSocket = require('ws');
const Redis = require('ioredis');

const subscriber = new Redis(process.env.REDIS_URL);

class NotificationService {
  constructor() {
    this.connections = new Map(); // userId -> WebSocket
  }

  initWebSocket(server) {
    const wss = new WebSocket.Server({ server, path: '/ws/notifications' });

    wss.on('connection', (ws, req) => {
      const userId = this.authenticateWS(req);
      if (!userId) return ws.close();

      this.connections.set(userId, ws);

      ws.on('close', () => this.connections.delete(userId));
    });

    // Redis pub/sub for cross-server notifications
    subscriber.subscribe('notifications');
    subscriber.on('message', (channel, message) => {
      const { userId, payload } = JSON.parse(message);
      const ws = this.connections.get(userId);
      if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(payload));
      }
    });
  }

  async notifyNewPost(posterId, postId) {
    // Notify online followers immediately
    const followers = await this.getOnlineFollowers(posterId);

    followers.forEach(followerId => {
      const payload = {
        type: 'NEW_POST',
        posterId,
        postId,
        message: 'নতুন পোস্ট!',
        timestamp: Date.now()
      };

      // Publish to Redis (distributed across servers)
      const redis = new Redis(process.env.REDIS_URL);
      redis.publish('notifications', JSON.stringify({
        userId: followerId, payload
      }));
    });
  }
}

module.exports = new NotificationService();
```

### ৩. Media CDN Architecture (Bangladesh Optimized)

```
┌───────────────────────────────────────────────────────────────┐
│                 MEDIA DELIVERY (BD OPTIMIZED)                  │
└───────────────────────────────────────────────────────────────┘

  User uploads image
       │
       ▼
  ┌─────────────┐     ┌─────────────────────────────┐
  │ Upload API  │────►│  Object Storage (S3/MinIO)  │
  └──────┬──────┘     │  Singapore Region           │
         │            └──────────────┬──────────────┘
         │                           │
         ▼                           ▼
  ┌──────────────┐           ┌──────────────────┐
  │ Image Worker │           │  Transcoding     │
  │ (Laravel Job)│           │  (Video: FFmpeg) │
  └──────┬───────┘           └────────┬─────────┘
         │                            │
         ▼                            ▼
  ┌─────────────────────────────────────────────┐
  │           CDN (CloudFront / Cloudflare)      │
  │                                             │
  │  Edge locations:                            │
  │  • Dhaka (BDIX) ← Primary for BD users     │
  │  • Singapore    ← Fallback                 │
  │  • Mumbai       ← For BD diaspora in India │
  └──────────────────────────┬──────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼───┐   ┌─────▼───┐   ┌─────▼───┐
        │ 4G User │   │ 3G User │   │ 2G User │
        │ 1080px  │   │  600px  │   │  150px  │
        │ WebP    │   │  JPEG   │   │  JPEG   │
        │ ~100KB  │   │  ~50KB  │   │  ~15KB  │
        └─────────┘   └─────────┘   └─────────┘

  Adaptive Quality:
  - Client sends: X-Connection-Quality header
  - CDN returns appropriate resolution
  - Progressive JPEG for slow connections
  - Lazy loading for below-the-fold images
```

### ৪. Spam/Abuse Detection

```php
<?php
// app/Services/SpamDetectionService.php

namespace App\Services;

use App\Models\Post;
use Illuminate\Support\Facades\Redis;

class SpamDetectionService
{
    // Bangladesh-specific spam patterns
    private const SPAM_INDICATORS = [
        'rate_limit'     => 10,    // Max 10 posts/hour
        'duplicate_text' => 0.9,   // 90% similarity = spam
        'url_ratio'      => 0.5,   // 50%+ URLs = suspicious
        'new_account'    => 24,    // Hours before full access
    ];

    public function isSpam(int $userId, string $content): array
    {
        $score = 0;
        $reasons = [];

        // Rate check
        $recentPosts = Redis::get("post_rate:{$userId}") ?? 0;
        if ($recentPosts > self::SPAM_INDICATORS['rate_limit']) {
            $score += 0.4;
            $reasons[] = 'excessive_posting';
        }

        // Duplicate content check
        $lastPosts = Redis::lrange("recent_content:{$userId}", 0, 9);
        foreach ($lastPosts as $prev) {
            if ($this->similarity($content, $prev) > self::SPAM_INDICATORS['duplicate_text']) {
                $score += 0.5;
                $reasons[] = 'duplicate_content';
                break;
            }
        }

        // URL spam check (common in BD: fake lottery/job links)
        $urlCount = preg_match_all('/https?:\/\//', $content);
        $wordCount = str_word_count($content);
        if ($wordCount > 0 && ($urlCount / $wordCount) > self::SPAM_INDICATORS['url_ratio']) {
            $score += 0.3;
            $reasons[] = 'url_spam';
        }

        // Bangla keyword blacklist
        $blacklist = ['লটারি জিতেছেন', 'ফ্রি রিচার্জ', 'বিকাশে টাকা পাঠান'];
        foreach ($blacklist as $keyword) {
            if (str_contains($content, $keyword)) {
                $score += 0.6;
                $reasons[] = 'blacklisted_keyword';
                break;
            }
        }

        return [
            'is_spam'  => $score >= 0.7,
            'score'    => min($score, 1.0),
            'reasons'  => $reasons,
            'action'   => $score >= 0.9 ? 'block' : ($score >= 0.7 ? 'review' : 'allow'),
        ];
    }

    private function similarity(string $a, string $b): float
    {
        similar_text($a, $b, $percent);
        return $percent / 100;
    }
}
```

### ৫. Trending Topics (Bangladesh Context)

```javascript
// src/services/trendingService.js

const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

class TrendingService {
  /**
   * Trending topics calculation
   * Window: Last 1 hour, updated every 5 min
   * Scope: Bangladesh (country-specific trends)
   */
  async calculateTrends() {
    const window = 3600; // 1 hour
    const now = Math.floor(Date.now() / 1000);
    const windowStart = now - window;

    // Get hashtag counts from sliding window
    const hashtags = await redis.zrevrangebyscore(
      'hashtags:bd:hourly',
      now,
      windowStart,
      'WITHSCORES',
      'LIMIT', 0, 50
    );

    // Calculate velocity (rate of increase)
    const trends = [];
    for (let i = 0; i < hashtags.length; i += 2) {
      const tag = hashtags[i];
      const currentCount = parseInt(hashtags[i + 1]);
      const prevCount = await redis.zscore('hashtags:bd:prev_hour', tag) || 0;

      const velocity = prevCount > 0
        ? (currentCount - prevCount) / prevCount
        : currentCount; // New trend

      trends.push({
        hashtag: tag,
        count: currentCount,
        velocity,
        score: currentCount * 0.4 + velocity * 100 * 0.6
      });
    }

    // Sort by score and store
    trends.sort((a, b) => b.score - a.score);
    const topTrends = trends.slice(0, 20);

    await redis.setex(
      'trending:bd',
      300, // 5 min cache
      JSON.stringify(topTrends)
    );

    return topTrends;
  }

  /**
   * Extract hashtags from post (supports Bangla + English)
   */
  extractHashtags(content) {
    // Match both English and Bangla hashtags
    const regex = /#([\w\u0980-\u09FF]+)/g;
    const matches = content.match(regex) || [];
    return matches.map(tag => tag.toLowerCase());
  }
}

// Example trending output for Bangladesh:
// 1. #বাংলাদেশ_জিতবে (velocity: 500%)
// 2. #BPL2024 (velocity: 300%)
// 3. #ঢাকা_ট্রাফিক (velocity: 150%)
// 4. #প্রথম_আলো (steady)
// 5. #GrameenphoneOffer (velocity: 200%)

module.exports = new TrendingService();
```

---

## 🎯 সারসংক্ষেপ

### মূল শিক্ষাসমূহ

```
┌─────────────────────────────────────────────────────────────┐
│              NEWS FEED SYSTEM - KEY TAKEAWAYS               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Hybrid fan-out সবচেয়ে ভালো approach                   │
│     • Regular users → Push (fan-out on write)              │
│     • Celebrities → Pull (fan-out on read)                 │
│                                                             │
│  2. Caching is king                                        │
│     • Redis sorted sets for feed cache                     │
│     • Multi-layer: L1 (local) → L2 (Redis) → L3 (DB)    │
│                                                             │
│  3. Bangladesh-specific optimizations                      │
│     • Adaptive image quality (2G/3G/4G)                   │
│     • BDIX CDN for low latency                            │
│     • Bangla NLP for content moderation                   │
│                                                             │
│  4. Scale considerations                                   │
│     • 50M BD users: Redis cluster যথেষ্ট                  │
│     • Cricket match surge: Auto-scale + degrade gracefully│
│     • Celebrity problem: Hybrid model solves it           │
│                                                             │
│  5. Trade-offs সবসময় context-dependent                     │
│     • Consistency vs Availability → AP (feed is not bank) │
│     • Cost vs Performance → Redis expensive but necessary │
│     • Simplicity vs Features → Start simple, add ranking  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### ইন্টারভিউ চেকলিস্ট

```
✅ Requirements clarify করেছেন? (DAU, read/write ratio)
✅ Back-of-envelope calculation দেখিয়েছেন?
✅ High-level architecture diagram এঁকেছেন?
✅ Fan-out strategy explain করেছেন? (Push/Pull/Hybrid)
✅ Celebrity problem address করেছেন?
✅ Database schema design করেছেন?
✅ Caching strategy বলেছেন? (Redis sorted sets)
✅ Ranking algorithm explain করেছেন?
✅ Scale numbers দিয়েছেন? (QPS, storage, bandwidth)
✅ Trade-offs discuss করেছেন?
✅ Edge cases handle করেছেন? (viral content, spam)
✅ Monitoring & alerting mention করেছেন?
```

### Technology Stack Summary

```
┌────────────────────┬────────────────────────────────────┐
│  Component         │  Technology                        │
├────────────────────┼────────────────────────────────────┤
│  Post API          │  PHP (Laravel) — CRUD operations   │
│  Feed API          │  Node.js (Express) — real-time     │
│  Database          │  MySQL (posts) + PostgreSQL (users)│
│  Feed Cache        │  Redis Cluster (sorted sets)       │
│  Message Queue     │  RabbitMQ / Redis Streams          │
│  Social Graph      │  MySQL + Neo4j (recommendations)   │
│  Object Storage    │  S3 / MinIO (media files)          │
│  CDN               │  CloudFront (BDIX edge)            │
│  Search            │  Elasticsearch (posts/hashtags)    │
│  Monitoring        │  Prometheus + Grafana              │
│  ML Ranking        │  LightGBM (Python microservice)    │
│  WebSocket         │  Socket.io (notifications)         │
└────────────────────┴────────────────────────────────────┘
```

---

> 💡 **টিপ**: ইন্টারভিউতে প্রথমে requirements ক্লিয়ার করুন, তারপর high-level design আঁকুন,
> এবং সবশেষে interviewer যেদিকে deep dive করতে চান সেদিকে যান। সব কিছু একসাথে
> বলার চেষ্টা করবেন না — structured approach follow করুন! 🚀
