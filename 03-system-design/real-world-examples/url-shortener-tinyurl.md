# 🔗 URL শর্টনার সিস্টেম ডিজাইন (TinyURL)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: TinyURL, Bitly, Short.io, Rebrandly

---

## 📑 সূচিপত্র

- [📌 Requirements](#-requirements)
- [📊 Back-of-envelope Estimation](#-back-of-envelope-estimation)
- [🏗️ High-Level Design](#️-high-level-design)
- [💻 Detailed Design](#-detailed-design)
- [⚖️ ট্রেড-অফ বিশ্লেষণ](#️-ট্রেড-অফ-বিশ্লেষণ-tradeoffs)
- [📈 কেস স্টাডি](#-কেস-স্টাডি-case-study)
- [🔧 Advanced Topics](#-advanced-topics)
- [🎯 সারসংক্ষেপ](#-সারসংক্ষেপ-summary)

---

## 📌 Requirements

### Functional Requirements

| # | Requirement | বিবরণ |
|---|-------------|--------|
| FR-1 | URL Shortening | Long URL থেকে short URL তৈরি করা |
| FR-2 | URL Redirection | Short URL hit করলে original URL-এ redirect করা |
| FR-3 | Custom Alias | User নিজের পছন্দমতো short key দিতে পারবে |
| FR-4 | Link Expiration | নির্দিষ্ট সময় পর link expire হবে |
| FR-5 | Analytics | Click count, location, device tracking |
| FR-6 | API Access | REST API দিয়ে programmatic access |

### Non-Functional Requirements

| # | Requirement | Target | বিবরণ |
|---|-------------|--------|--------|
| NFR-1 | Availability | 99.99% | সবসময় redirect কাজ করতে হবে |
| NFR-2 | Latency | < 50ms | Redirect অত্যন্ত দ্রুত হতে হবে |
| NFR-3 | Scalability | 100M URLs/month | বিশাল ট্রাফিক handle করা |
| NFR-4 | Durability | 0% data loss | একবার create হলে কখনো হারাবে না |
| NFR-5 | Security | Rate limited | Spam/abuse prevention |

### 🇧🇩 Bangladesh Context

```
বাংলাদেশের কোম্পানিগুলোর URL Shortener প্রয়োজনীয়তা:

• bKash: Payment link share করতে → bkas.h/pay123
• Daraz: Product link share → drz.bd/deal99
• Pathao: Ride share link → ptho.co/ride456
• Grameenphone: Offer links → gp.link/offer22
• Nagad: Transaction links → ngd.co/send789
• eSewa: QR/Payment short links

সমস্যা: বাংলাদেশে SMS-এ character limit (160),
তাই short URL অপরিহার্য।
```

---

## 📊 Back-of-envelope Estimation

### Traffic Estimation

```
Assumptions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• নতুন URL creation: 100M / month
• Read:Write ratio = 100:1 (Reads অনেক বেশি)
• Redirections: 10B / month

Calculations:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Write QPS = 100M / (30 × 24 × 3600) ≈ 40 URLs/sec
• Read QPS  = 40 × 100 = 4,000 redirections/sec
• Peak QPS  = 4,000 × 3 = 12,000 redirections/sec (3x peak factor)
```

### Storage Estimation

```
Assumptions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• প্রতিটি URL record = ~500 bytes
  - short_url: 7 bytes
  - long_url: 200 bytes (avg)
  - created_at: 8 bytes
  - expires_at: 8 bytes
  - user_id: 8 bytes
  - metadata: ~270 bytes

• Retention period: 5 years

Calculations:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Total URLs in 5 years = 100M × 12 × 5 = 6 Billion
• Total storage = 6B × 500 bytes = 3 TB
• With replication (3x) = 9 TB
```

### Bandwidth Estimation

```
Write bandwidth:
• 40 URLs/sec × 500 bytes = 20 KB/s (negligible)

Read bandwidth:
• 4,000 req/sec × 500 bytes = 2 MB/s

Peak bandwidth:
• 12,000 req/sec × 500 bytes = 6 MB/s
```

### Cache Estimation (80-20 Rule)

```
80-20 Rule: 20% URL গুলো 80% traffic generate করে

Daily redirections = 4,000 × 86,400 = ~350M/day
Cache করতে হবে = 350M × 20% = 70M URLs/day
Memory needed = 70M × 500 bytes = 35 GB

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Redis cluster: 3 nodes × 16GB = 48GB (sufficient)
```

---

## 🏗️ High-Level Design

### System Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              LOAD BALANCER                   │
                        │         (Nginx / AWS ALB)                    │
                        └──────────────┬──────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────────┐
                    │                  │                       │
            ┌───────▼───────┐  ┌───────▼───────┐  ┌──────────▼──────┐
            │  App Server 1 │  │  App Server 2 │  │  App Server N   │
            │  (Laravel)    │  │  (Express.js) │  │  (Laravel)      │
            └───────┬───────┘  └───────┬───────┘  └──────────┬──────┘
                    │                  │                       │
                    └──────────────────┼───────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────────┐
                    │                  │                       │
            ┌───────▼───────┐  ┌───────▼───────┐  ┌──────────▼──────┐
            │  Redis Cache  │  │  Redis Cache  │  │  Redis Cache    │
            │  (Primary)    │  │  (Replica 1)  │  │  (Replica 2)    │
            └───────┬───────┘  └───────────────┘  └─────────────────┘
                    │
            ┌───────▼───────────────────────────────────────────────┐
            │                    DATABASE LAYER                       │
            │                                                        │
            │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
            │  │  Shard 1 │  │  Shard 2 │  │  Shard N │            │
            │  │(MySQL/PG)│  │(MySQL/PG)│  │(MySQL/PG)│            │
            │  └──────────┘  └──────────┘  └──────────┘            │
            └───────────────────────────────────────────────────────┘
```

### Request Flow — URL Creation

```
Client                 API Gateway          App Server           DB
  │                        │                    │                 │
  │── POST /shorten ──────▶│                    │                 │
  │                        │── Route Request ──▶│                 │
  │                        │                    │── Check Cache ──▶│ (Redis)
  │                        │                    │◀── Miss ─────────│
  │                        │                    │                  │
  │                        │                    │── Generate Key ─▶│ (Base62)
  │                        │                    │── Store URL ────▶│ (MySQL)
  │                        │                    │── Set Cache ────▶│ (Redis)
  │                        │                    │                  │
  │◀── 201 {short_url} ───│◀── Response ───────│                  │
  │                        │                    │                  │
```

### Request Flow — URL Redirect

```
Client                 Load Balancer       App Server         Cache/DB
  │                        │                    │                 │
  │── GET /abc123 ────────▶│                    │                 │
  │                        │── Route ──────────▶│                 │
  │                        │                    │── Check Cache ──▶│
  │                        │                    │◀── HIT! ─────────│
  │                        │                    │                  │
  │◀── 301 Redirect ──────│◀── Response ───────│                  │
  │                        │                    │                  │
  │── GET original_url ───▶│ (destination)      │                  │
```

### Component Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│ Component          │ Responsibility                              │
├─────────────────────────────────────────────────────────────────┤
│ Load Balancer      │ Traffic distribution, SSL termination       │
│ API Gateway        │ Rate limiting, auth, routing                │
│ App Server         │ Business logic, URL generation              │
│ Redis Cache        │ Hot URL caching, rate limiting counters     │
│ MySQL/PostgreSQL   │ Persistent URL storage                      │
│ Analytics Service  │ Async click tracking, stats                 │
│ Key Generation Svc │ Pre-generate unique keys (optional)         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💻 Detailed Design

### Data Model / Schema

```sql
-- Main URL table
CREATE TABLE urls (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    short_key VARCHAR(10) NOT NULL UNIQUE,
    long_url TEXT NOT NULL,
    user_id BIGINT UNSIGNED NULL,
    custom_alias BOOLEAN DEFAULT FALSE,
    expires_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    click_count BIGINT UNSIGNED DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_short_key (short_key),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
) ENGINE=InnoDB;

-- Analytics table
CREATE TABLE url_analytics (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    url_id BIGINT UNSIGNED NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    referer TEXT,
    country_code CHAR(2),
    city VARCHAR(100),
    device_type ENUM('desktop', 'mobile', 'tablet'),
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_url_id (url_id),
    INDEX idx_clicked_at (clicked_at),
    FOREIGN KEY (url_id) REFERENCES urls(id)
) ENGINE=InnoDB;

-- Pre-generated keys table (Key Generation Service)
CREATE TABLE pre_generated_keys (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    short_key VARCHAR(10) NOT NULL UNIQUE,
    is_used BOOLEAN DEFAULT FALSE,
    used_at TIMESTAMP NULL,
    INDEX idx_unused (is_used, id)
) ENGINE=InnoDB;
```

### Key Generation Algorithms

#### Algorithm 1: Base62 Encoding

```
Base62 Characters: [0-9][a-z][A-Z] = 62 characters

Key Length Calculation:
━━━━━━━━━━━━━━━━━━━━━
• 6 characters: 62^6 = ~56.8 Billion combinations
• 7 characters: 62^7 = ~3.5 Trillion combinations

আমাদের 6B URLs store করতে হবে (5 বছরে)
→ 7 characters যথেষ্ট (3.5T >> 6B)
```

#### Algorithm 2: Counter Range + Base62

```
┌─────────────────────────────────────────────────┐
│          COUNTER RANGE APPROACH                  │
├─────────────────────────────────────────────────┤
│                                                  │
│  Zookeeper/etcd                                 │
│  ┌─────────────────────────────┐                │
│  │ Server 1: Range [1, 1M]     │                │
│  │ Server 2: Range [1M+1, 2M]  │                │
│  │ Server 3: Range [2M+1, 3M]  │                │
│  └─────────────────────────────┘                │
│                                                  │
│  প্রতিটি server নিজের range থেকে               │
│  sequential counter বাড়ায়                       │
│  → Base62 encode করে short key পায়             │
│                                                  │
│  Benefits:                                       │
│  ✓ No collision                                 │
│  ✓ No coordination needed between servers       │
│  ✓ Highly scalable                              │
└─────────────────────────────────────────────────┘
```

### PHP (Laravel) Implementation

```php
<?php
// app/Services/UrlShortenerService.php

namespace App\Services;

use App\Models\Url;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;

class UrlShortenerService
{
    private const BASE62_CHARS = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    private const KEY_LENGTH = 7;

    /**
     * Long URL থেকে short URL তৈরি করে
     */
    public function shorten(string $longUrl, ?string $customAlias = null, ?int $userId = null): string
    {
        // Custom alias check
        if ($customAlias) {
            if (Url::where('short_key', $customAlias)->exists()) {
                throw new \Exception('Custom alias already taken');
            }
            $shortKey = $customAlias;
        } else {
            $shortKey = $this->generateUniqueKey();
        }

        // Database-এ store
        $url = Url::create([
            'short_key'    => $shortKey,
            'long_url'     => $longUrl,
            'user_id'      => $userId,
            'custom_alias' => (bool) $customAlias,
            'expires_at'   => now()->addYears(2),
        ]);

        // Cache-এ রাখো (hot data)
        Cache::put("url:{$shortKey}", $longUrl, now()->addDays(30));

        return config('app.short_domain') . '/' . $shortKey;
    }

    /**
     * Short key থেকে original URL খুঁজে বের করে
     */
    public function resolve(string $shortKey): ?string
    {
        // প্রথমে Cache check
        $cached = Cache::get("url:{$shortKey}");
        if ($cached) {
            return $cached;
        }

        // Cache miss → Database query
        $url = Url::where('short_key', $shortKey)
            ->where('is_active', true)
            ->where(function ($query) {
                $query->whereNull('expires_at')
                      ->orWhere('expires_at', '>', now());
            })
            ->first();

        if (!$url) {
            return null;
        }

        // Cache-এ set করো future requests-এর জন্য
        Cache::put("url:{$shortKey}", $url->long_url, now()->addDays(30));

        // Click count increment (async)
        dispatch(new \App\Jobs\IncrementClickCount($url->id));

        return $url->long_url;
    }

    /**
     * Base62 encoding দিয়ে unique key generate করে
     */
    private function generateUniqueKey(): string
    {
        // Counter-based approach with DB auto-increment
        $counter = DB::table('pre_generated_keys')
            ->where('is_used', false)
            ->orderBy('id')
            ->lockForUpdate()
            ->first();

        if ($counter) {
            DB::table('pre_generated_keys')
                ->where('id', $counter->id)
                ->update(['is_used' => true, 'used_at' => now()]);
            return $counter->short_key;
        }

        // Fallback: random generation with collision check
        do {
            $key = $this->randomBase62(self::KEY_LENGTH);
        } while (Url::where('short_key', $key)->exists());

        return $key;
    }

    /**
     * Base62 encode করে
     */
    public function base62Encode(int $number): string
    {
        if ($number === 0) return '0';

        $result = '';
        while ($number > 0) {
            $result = self::BASE62_CHARS[$number % 62] . $result;
            $number = intdiv($number, 62);
        }

        return str_pad($result, self::KEY_LENGTH, '0', STR_PAD_LEFT);
    }

    private function randomBase62(int $length): string
    {
        $result = '';
        for ($i = 0; $i < $length; $i++) {
            $result .= self::BASE62_CHARS[random_int(0, 61)];
        }
        return $result;
    }
}
```

```php
<?php
// app/Http/Controllers/UrlController.php

namespace App\Http\Controllers;

use App\Services\UrlShortenerService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\RedirectResponse;

class UrlController extends Controller
{
    public function __construct(
        private UrlShortenerService $shortener
    ) {}

    /**
     * POST /api/shorten — নতুন short URL তৈরি
     */
    public function shorten(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'url'          => 'required|url|max:2048',
            'custom_alias' => 'nullable|alpha_dash|min:4|max:20',
        ]);

        try {
            $shortUrl = $this->shortener->shorten(
                longUrl: $validated['url'],
                customAlias: $validated['custom_alias'] ?? null,
                userId: $request->user()?->id
            );

            return response()->json([
                'success'   => true,
                'short_url' => $shortUrl,
                'expires_at' => now()->addYears(2)->toIso8601String(),
            ], 201);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'error'   => $e->getMessage(),
            ], 409);
        }
    }

    /**
     * GET /{shortKey} — Redirect to original URL
     */
    public function redirect(string $shortKey): RedirectResponse
    {
        $longUrl = $this->shortener->resolve($shortKey);

        if (!$longUrl) {
            abort(404, 'Short URL not found or expired');
        }

        // 301 = permanent redirect (browser caches)
        // 302 = temporary redirect (better for analytics)
        return redirect($longUrl, 302);
    }
}
```

### Node.js (Express) Implementation

```javascript
// src/services/urlShortener.js

const crypto = require('crypto');
const Redis = require('ioredis');
const { Pool } = require('pg');

const BASE62_CHARS = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
const KEY_LENGTH = 7;

const redis = new Redis({
    host: process.env.REDIS_HOST || 'localhost',
    port: 6379,
    maxRetriesPerRequest: 3,
});

const db = new Pool({
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    max: 20, // connection pool size
});

/**
 * Base62 encode — number কে short string-এ convert করে
 */
function base62Encode(number) {
    if (number === 0n) return '0'.padStart(KEY_LENGTH, '0');

    let result = '';
    let num = BigInt(number);
    while (num > 0n) {
        result = BASE62_CHARS[Number(num % 62n)] + result;
        num = num / 62n;
    }
    return result.padStart(KEY_LENGTH, '0');
}

/**
 * Unique short key generate করে (Counter-based)
 */
async function generateShortKey() {
    // Atomic counter from Redis
    const counter = await redis.incr('url:global_counter');
    return base62Encode(BigInt(counter));
}

/**
 * URL shorten করে — main business logic
 */
async function shortenUrl(longUrl, customAlias = null, userId = null) {
    // Duplicate check — একই URL আবার shorten করলে আগেরটা return
    const existing = await db.query(
        'SELECT short_key FROM urls WHERE long_url = $1 AND user_id IS NOT DISTINCT FROM $2 AND is_active = true',
        [longUrl, userId]
    );

    if (existing.rows.length > 0) {
        return `${process.env.SHORT_DOMAIN}/${existing.rows[0].short_key}`;
    }

    let shortKey;
    if (customAlias) {
        // Custom alias available কিনা check
        const aliasCheck = await db.query(
            'SELECT id FROM urls WHERE short_key = $1', [customAlias]
        );
        if (aliasCheck.rows.length > 0) {
            throw new Error('Custom alias already taken');
        }
        shortKey = customAlias;
    } else {
        shortKey = await generateShortKey();
    }

    // Database insert
    await db.query(
        `INSERT INTO urls (short_key, long_url, user_id, custom_alias, expires_at, created_at)
         VALUES ($1, $2, $3, $4, NOW() + INTERVAL '2 years', NOW())`,
        [shortKey, longUrl, userId, !!customAlias]
    );

    // Cache-এ store (30 দিনের জন্য)
    await redis.setex(`url:${shortKey}`, 86400 * 30, longUrl);

    return `${process.env.SHORT_DOMAIN}/${shortKey}`;
}

/**
 * Short URL resolve করে — redirect-এর জন্য original URL খোঁজে
 */
async function resolveUrl(shortKey) {
    // Step 1: Cache check (fastest)
    const cached = await redis.get(`url:${shortKey}`);
    if (cached) {
        // Async analytics tracking
        trackClick(shortKey).catch(console.error);
        return cached;
    }

    // Step 2: Database query
    const result = await db.query(
        `SELECT id, long_url FROM urls
         WHERE short_key = $1
         AND is_active = true
         AND (expires_at IS NULL OR expires_at > NOW())`,
        [shortKey]
    );

    if (result.rows.length === 0) {
        return null;
    }

    const { id, long_url } = result.rows[0];

    // Cache populate
    await redis.setex(`url:${shortKey}`, 86400 * 30, long_url);

    // Async analytics
    trackClick(shortKey, id).catch(console.error);

    return long_url;
}

/**
 * Click analytics track করে (async, non-blocking)
 */
async function trackClick(shortKey, urlId = null) {
    // Redis-এ real-time counter
    await redis.incr(`clicks:${shortKey}`);
    await redis.incr(`clicks:daily:${new Date().toISOString().split('T')[0]}:${shortKey}`);
}

module.exports = { shortenUrl, resolveUrl, base62Encode };
```

```javascript
// src/routes/urlRoutes.js

const express = require('express');
const router = express.Router();
const rateLimit = require('express-rate-limit');
const { shortenUrl, resolveUrl } = require('../services/urlShortener');

// Rate limiting — bKash/Daraz-এর মতো বড় client-দের জন্য আলাদা tier
const createLimiter = rateLimit({
    windowMs: 60 * 1000,  // 1 minute window
    max: 10,              // 10 creates per minute
    message: { error: 'Too many requests. Please try again later.' },
});

const redirectLimiter = rateLimit({
    windowMs: 60 * 1000,
    max: 1000,           // redirects বেশি allow করা হয়
});

/**
 * POST /api/v1/shorten — Short URL তৈরি
 */
router.post('/api/v1/shorten', createLimiter, async (req, res) => {
    try {
        const { url, custom_alias } = req.body;

        if (!url || !isValidUrl(url)) {
            return res.status(400).json({
                success: false,
                error: 'Invalid URL provided',
            });
        }

        const shortUrl = await shortenUrl(
            url,
            custom_alias || null,
            req.user?.id || null
        );

        return res.status(201).json({
            success: true,
            data: {
                short_url: shortUrl,
                original_url: url,
                created_at: new Date().toISOString(),
            },
        });
    } catch (error) {
        const status = error.message.includes('already taken') ? 409 : 500;
        return res.status(status).json({
            success: false,
            error: error.message,
        });
    }
});

/**
 * GET /:shortKey — Redirect to original URL
 */
router.get('/:shortKey', redirectLimiter, async (req, res) => {
    const { shortKey } = req.params;

    // Validation: short key must be alphanumeric, 4-20 chars
    if (!/^[a-zA-Z0-9]{4,20}$/.test(shortKey)) {
        return res.status(400).json({ error: 'Invalid short URL format' });
    }

    const longUrl = await resolveUrl(shortKey);

    if (!longUrl) {
        return res.status(404).json({ error: 'URL not found or expired' });
    }

    // 302 redirect (temporary — allows analytics tracking)
    return res.redirect(302, longUrl);
});

function isValidUrl(string) {
    try {
        new URL(string);
        return true;
    } catch {
        return false;
    }
}

module.exports = router;
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ (Tradeoffs)

### Key Generation Strategy তুলনা

| Approach | Pros | Cons | Use When |
|----------|------|------|----------|
| **MD5/SHA256 Hash** | Simple, deterministic | Collision possible, long output | Same URL → same short key চাইলে |
| **Base62 + Counter** | No collision, sequential | Predictable, single point (counter) | High throughput দরকার |
| **Pre-generated Keys** | No runtime computation | Storage overhead, key exhaustion | Max performance চাই |
| **Random + Retry** | Simple, unpredictable | Collision retry overhead | Low traffic systems |
| **Snowflake ID** | Distributed, time-sorted | Complex setup | Multi-datacenter |

### 301 vs 302 Redirect

```
┌─────────────────────────────────────────────────────────────────┐
│                   REDIRECT STATUS CODE                           │
├──────────────┬──────────────────────┬───────────────────────────┤
│              │  301 (Permanent)     │  302 (Temporary)          │
├──────────────┼──────────────────────┼───────────────────────────┤
│ Caching      │ Browser caches       │ Browser always asks       │
│ Analytics    │ ❌ Miss clicks       │ ✅ Track every click      │
│ SEO          │ Passes link juice    │ Doesn't pass link juice   │
│ Performance  │ ✅ Faster (cached)   │ ❌ Extra hop every time   │
├──────────────┼──────────────────────┼───────────────────────────┤
│ সিদ্ধান্ত    │ SEO tools           │ Analytics tools (Bitly)   │
└──────────────┴──────────────────────┴───────────────────────────┘

আমাদের পছন্দ: 302 — কারণ bKash/Daraz-এর জন্য analytics গুরুত্বপূর্ণ
```

### SQL vs NoSQL Decision

```
┌─────────────────────────────────────────────────────────────────┐
│                     SQL vs NoSQL                                  │
├──────────────────────┬──────────────────────────────────────────┤
│ Factor               │ Decision                                  │
├──────────────────────┼──────────────────────────────────────────┤
│ Data Structure       │ Simple key-value → NoSQL friendly         │
│ Consistency Need     │ Strong (no duplicate keys) → SQL          │
│ Read Pattern         │ Point lookup by key → Both OK             │
│ Scale                │ Billions of records → Need sharding       │
│ Relationships        │ Minimal → NoSQL OK                        │
│ Transactions         │ Not critical for reads → NoSQL OK         │
├──────────────────────┼──────────────────────────────────────────┤
│ ✅ FINAL DECISION   │ MySQL/PostgreSQL with sharding             │
│                      │ (Strong consistency + familiar tools)      │
└──────────────────────┴──────────────────────────────────────────┘

বিকল্প: DynamoDB / Cassandra — যদি global distribution দরকার হয়
```

### CAP Theorem Implications

```
CAP Theorem-এ আমাদের choice:

    C ─────────── A
    │ \         / │
    │   \     /   │
    │     \ /     │
    │      X      │
    │     / \     │
    │   /     \   │
    │ /         \ │
    P ─────────── ?

URL Shortener = CP (Consistency + Partition tolerance)
কারণ: একই short key দুটো different URL-এ point করতে পারবে না!

কিন্তু Availability-ও চাই (99.99%)...
→ Solution: Multiple replicas with leader election
→ Read replicas দিয়ে availability বাড়ানো
```

### Caching Strategy Comparison

| Strategy | Hit Rate | Complexity | Best For |
|----------|----------|------------|----------|
| **Write-through** | High | Medium | Consistent reads |
| **Write-behind** | High | High | High write throughput |
| **Cache-aside** | Medium-High | Low | Simple implementation |
| **Read-through** | High | Medium | Uniform access |

```
আমাদের পছন্দ: Cache-Aside (Lazy Loading)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Read Path:
  1. App → Redis check
  2. Miss? → DB query → Redis-এ set

Write Path:
  1. App → DB write
  2. App → Redis set (or invalidate)

কেন?
• Simple implementation
• Only hot data cached (memory efficient)
• Cache failure doesn't break system
```

---

## 📈 কেস স্টাডি (Case Study)

### 🇧🇩 বাংলাদেশ কনটেক্সট: bKash Payment Link System

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCENARIO: bKash "Send Money" Feature-এ URL Shortener
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Problem:
  bKash payment request link: 
  https://www.bkash.com/payment/request?merchant=12345&amount=500&ref=TXN98765432&callback=https://shop.example.com/confirm

  SMS-এ পাঠাতে গেলে 160 character limit exceed করে!

Solution:
  bkas.h/p/7Kx9mN → Shortened to 20 characters

Traffic Pattern:
  • সকাল 9-10 AM: Office payment links (spike)
  • দুপুর 1-2 PM: Lunch payment (medium)
  • সন্ধ্যা 6-9 PM: Online shopping peak (highest)
  • ঈদ/পূজা: 10x normal traffic

Scale Requirements:
  • Daily active users: 40M+ (bKash)
  • Payment links/day: 5M+
  • Redirects/day: 50M+
  • Peak QPS: 2,000 creates, 20,000 redirects
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Daraz Flash Sale Scenario

```
Scenario: Daraz 11.11 Sale — URL Shortener Under Pressure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Normal Day:
  • Product link shares: 500K/day
  • Redirects: 5M/day

11.11 Flash Sale Day:
  • Product link shares: 10M/day (20x spike!)
  • Redirects: 100M/day
  • Peak window: 12:00 AM - 12:05 AM (sale start)
  • Peak QPS: 50,000 redirects/sec

How to handle:
  1. Pre-warm cache (popular product links cached before sale)
  2. Scale horizontally (auto-scaling group triggers at 70% CPU)
  3. Rate limit non-critical analytics writes
  4. Serve from edge (CDN-level 301 caching)
  5. Circuit breaker on DB writes (queue overflow → reject gracefully)
```

### Failure Scenarios & Handling

```
┌─────────────────────────────────────────────────────────────────┐
│ Failure Scenario        │ Impact          │ Mitigation           │
├─────────────────────────┼─────────────────┼──────────────────────┤
│ Redis cache down        │ Increased DB    │ Fallback to DB,      │
│                         │ load            │ multiple replicas    │
├─────────────────────────┼─────────────────┼──────────────────────┤
│ Primary DB down         │ No new URLs     │ Failover to replica, │
│                         │                 │ read still works     │
├─────────────────────────┼─────────────────┼──────────────────────┤
│ Counter service down    │ No key gen      │ Fallback to random   │
│                         │                 │ key generation       │
├─────────────────────────┼─────────────────┼──────────────────────┤
│ Full DB disk            │ System halt     │ Alert + cleanup      │
│                         │                 │ expired URLs         │
├─────────────────────────┼─────────────────┼──────────────────────┤
│ DDoS attack             │ Service down    │ Rate limit + WAF +   │
│                         │                 │ Cloudflare           │
└─────────────────────────┴─────────────────┴──────────────────────┘
```

---

## 🔧 Advanced Topics

### Sharding Strategy

```
Database Sharding — URL billions হলে single DB-তে হবে না
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Strategy: Hash-based sharding on short_key

  shard_id = hash(short_key) % num_shards

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Shard 0  │    │ Shard 1  │    │ Shard 2  │    │ Shard 3  │
  │ keys:    │    │ keys:    │    │ keys:    │    │ keys:    │
  │ 0-9,a-f  │    │ g-m      │    │ n-t      │    │ u-z,A-Z  │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘

Consistent Hashing:
  • নতুন shard add করলে minimal data migration
  • Virtual nodes দিয়ে even distribution ensure

  Ring: [0 ─── Shard1 ─── Shard2 ─── Shard3 ─── Shard4 ─── 0]
         ↑                                                    ↑
       hash("abc123") → nearest shard clockwise
```

### Analytics Pipeline

```
Real-time + Batch Analytics Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Click Event
      │
      ▼
  ┌─────────┐     ┌─────────────┐     ┌──────────────┐
  │  Kafka  │────▶│  Consumer   │────▶│  ClickHouse  │
  │  Queue  │     │  (Node.js)  │     │  (Analytics) │
  └─────────┘     └─────────────┘     └──────────────┘
      │                                       │
      │           ┌─────────────┐             │
      └──────────▶│  Spark Job  │             │
                  │  (Daily)    │             ▼
                  └──────┬──────┘     ┌──────────────┐
                         │            │  Dashboard   │
                         ▼            │  (Grafana)   │
                  ┌─────────────┐     └──────────────┘
                  │  Data Lake  │
                  │  (S3/HDFS)  │
                  └─────────────┘

Tracked Metrics:
  • Total clicks per URL
  • Clicks by country (BD, IN, US, etc.)
  • Clicks by device (Mobile 78%, Desktop 20%, Tablet 2%)
  • Clicks by time (hourly distribution)
  • Referrer analysis
  • Unique vs repeat visitors
```

### Rate Limiting Implementation

```php
<?php
// app/Http/Middleware/RateLimiter.php (Laravel)

namespace App\Http\Middleware;

use Illuminate\Support\Facades\Redis;
use Closure;

class UrlRateLimiter
{
    /**
     * Sliding Window Rate Limiter
     * bKash/Daraz API key ভিত্তিক আলাদা limit
     */
    public function handle($request, Closure $next)
    {
        $key = $this->resolveKey($request);
        $limit = $this->getLimit($request);
        $window = 60; // 1 minute window

        $current = Redis::incr("rate:{$key}");

        if ($current === 1) {
            Redis::expire("rate:{$key}", $window);
        }

        if ($current > $limit) {
            return response()->json([
                'error' => 'Rate limit exceeded',
                'retry_after' => Redis::ttl("rate:{$key}"),
            ], 429);
        }

        return $next($request);
    }

    private function getLimit($request): int
    {
        // Enterprise clients (bKash, Daraz) get higher limits
        $apiKey = $request->header('X-API-Key');
        $tiers = [
            'enterprise' => 1000,  // bKash, Daraz
            'business'   => 100,   // Small businesses
            'free'       => 10,    // Free users
        ];

        $tier = $this->getTier($apiKey);
        return $tiers[$tier] ?? $tiers['free'];
    }

    private function resolveKey($request): string
    {
        return $request->header('X-API-Key')
            ?? $request->ip();
    }

    private function getTier(?string $apiKey): string
    {
        if (!$apiKey) return 'free';
        return Redis::get("tier:{$apiKey}") ?? 'free';
    }
}
```

### Custom Domain Support

```
Custom Domain Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━

bKash → bkas.h/p/abc123
Daraz → drz.bd/deal/xyz
Pathao → ptho.co/ride/123

Implementation:
  1. Client adds CNAME: bkas.h → our-service.example.com
  2. SSL cert via Let's Encrypt (wildcard or per-domain)
  3. Nginx server_name matching

  ┌────────────┐     ┌──────────────┐     ┌───────────┐
  │  bkas.h    │────▶│   Nginx      │────▶│  App      │
  │  drz.bd    │     │  (SNI-based  │     │  Server   │
  │  ptho.co   │     │   routing)   │     │           │
  └────────────┘     └──────────────┘     └───────────┘

Database schema addition:
  CREATE TABLE custom_domains (
      id BIGINT PRIMARY KEY,
      domain VARCHAR(255) UNIQUE NOT NULL,
      user_id BIGINT NOT NULL,
      ssl_status ENUM('pending','active','expired'),
      verified_at TIMESTAMP,
      created_at TIMESTAMP DEFAULT NOW()
  );
```

### Link Expiration Strategy

```
Expiration Approaches:
━━━━━━━━━━━━━━━━━━━━━

1. Passive Expiration (আমাদের পছন্দ):
   • Read-time check: expires_at > NOW()
   • Expired link → 410 Gone response
   • কোনো background job দরকার নেই

2. Active Cleanup (storage reclaim):
   • Daily cron job deletes expired URLs
   • Frees up short keys for reuse

3. TTL-based (Redis):
   • Redis EXPIREAT set করা
   • Automatic eviction

Laravel Scheduler Implementation:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// app/Console/Kernel.php
$schedule->command('urls:cleanup-expired')
         ->daily()
         ->at('03:00')  // রাত ৩টায় (low traffic time in BD)
         ->withoutOverlapping();
```

---

## 🎯 সারসংক্ষেপ (Summary)

### Key Decisions Summary

| Decision Area | Choice | Reasoning |
|---------------|--------|-----------|
| Key Generation | Counter + Base62 | No collision, scalable |
| Key Length | 7 characters | 3.5T combinations (5+ years sufficient) |
| Database | MySQL/PostgreSQL | Strong consistency, familiar |
| Cache | Redis (Cache-aside) | Simple, effective 80%+ hit rate |
| Redirect Code | 302 (Temporary) | Analytics tracking capability |
| Sharding | Hash-based | Even distribution, simple lookup |
| Rate Limiting | Sliding Window (Redis) | Accurate, memory efficient |
| Analytics | Async (Kafka → ClickHouse) | Non-blocking, scalable |

### Architecture Decision Records

```
ADR-1: Base62 Counter over MD5 Hash
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Context: Unique short key generation method প্রয়োজন
Decision: Counter-based Base62 encoding
Consequences: 
  ✅ Zero collision guarantee
  ✅ Sequential, predictable
  ❌ Counter = single coordination point
  ❌ Predictable URLs (security concern → mitigated by random offset)

ADR-2: PostgreSQL over DynamoDB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Context: Primary datastore selection
Decision: PostgreSQL with read replicas
Consequences:
  ✅ ACID compliance
  ✅ Familiar to BD developer ecosystem
  ✅ Cost effective for our scale
  ❌ Sharding complexity at extreme scale
  ❌ No built-in global distribution

ADR-3: 302 over 301 Redirect
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Context: HTTP redirect status code selection
Decision: 302 Temporary Redirect
Consequences:
  ✅ Every click tracked (analytics)
  ✅ Can change destination later
  ❌ Slightly higher latency (no browser cache)
  ❌ More server load
```

### Interview Tips 💡

```
System Design Interview-এ যা মনে রাখবেন:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Requirements আগে clarify করুন (5 min)
   • Read-heavy vs Write-heavy?
   • Analytics দরকার?
   • Custom aliases?
   • Expiration?

2. Back-of-envelope calculation দেখান (5 min)
   • QPS, storage, bandwidth
   • Interviewer impressed হবে

3. High-level design আঁকুন (10 min)
   • Components identify করুন
   • Data flow দেখান

4. Deep dive (15 min)
   • Key generation algorithm
   • Database choice + sharding
   • Caching strategy
   • Trade-offs explain করুন

5. Scale & Edge cases (5 min)
   • Hot URLs (viral links)
   • Thundering herd
   • Data center failover

Common Follow-up Questions:
  Q: "How to handle hot URLs?" → Cache + CDN
  Q: "How to prevent abuse?" → Rate limiting + CAPTCHA
  Q: "How to scale writes?" → Sharding + async processing
  Q: "301 vs 302?" → Trade-off between caching & analytics
```

---

> 📝 **শেষ কথা**: URL Shortener একটি classic system design problem যেটা
> read-heavy workload, caching strategy, এবং distributed key generation
> concepts শেখার জন্য অসাধারণ। বাংলাদেশের context-এ bKash, Daraz, Pathao-র
> মতো কোম্পানিগুলো প্রতিদিন millions of short links handle করে।
> এই design production-ready এবং interview-ready উভয়ই।
