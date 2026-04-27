# 🚦 রেট লিমিটিং (Rate Limiting) — গভীর বিশ্লেষণ

## 📌 সংজ্ঞা ও প্রয়োজনীয়তা

**রেট লিমিটিং** হলো একটি কৌশল যা নির্দিষ্ট সময়ের মধ্যে কোনো ক্লায়েন্ট বা সার্ভিস থেকে আসা রিকোয়েস্টের সংখ্যা সীমাবদ্ধ করে। এটি সিস্টেম ডিজাইনের সবচেয়ে গুরুত্বপূর্ণ প্রতিরক্ষামূলক প্যাটার্নগুলোর একটি।

### কেন রেট লিমিটিং দরকার?

```
সমস্যা                          সমাধান (রেট লিমিটিং)
─────────────────────────────    ─────────────────────────────
DDoS আক্রমণ                  →  প্রতি IP তে রিকোয়েস্ট সীমা
Brute Force লগইন             →  প্রতি ইউজারে লগইন attempt সীমা
API অপব্যবহার                →  API key ভিত্তিক সীমা
সার্ভার ওভারলোড              →  মোট throughput নিয়ন্ত্রণ
খরচ নিয়ন্ত্রণ                →  3rd-party API কল সীমা
OTP flooding                  →  প্রতি নম্বরে OTP সীমা (bKash/Nagad)
Flash sale abuse              →  প্রতি ইউজারে অর্ডার সীমা
```

### বাস্তব উদাহরণ (বাংলাদেশ Context)

- **bKash API**: প্রতি মার্চেন্টে প্রতি মিনিটে সর্বোচ্চ ৬০টি ট্রানজ্যাকশন রিকোয়েস্ট
- **OTP রেট লিমিটিং**: একই নম্বরে ৫ মিনিটে সর্বোচ্চ ৩টি OTP — OTP flooding প্রতিরোধ
- **ই-কমার্স Flash Sale**: Daraz/Evaly টাইপ সেলে প্রতি ইউজারে প্রতি সেকেন্ডে ১টি অর্ডার attempt
- **Pathao/Uber রাইড**: প্রতি মিনিটে সর্বোচ্চ ৫টি রাইড রিকোয়েস্ট

---

## 📊 Rate Limiter Architecture

### সিস্টেমে রেট লিমিটারের অবস্থান

```
                         ┌─────────────────────────────────────────────┐
                         │            RATE LIMITING LAYERS             │
                         └─────────────────────────────────────────────┘

  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────────┐    ┌──────────┐
  │  Client  │───▶│   CDN/WAF    │───▶│ Load Balancer│───▶│ API Gateway│───▶│  App     │
  │ (Browser/│    │ (Cloudflare) │    │   (Nginx)    │    │  (Kong)    │    │ Server   │
  │  Mobile) │    │              │    │              │    │            │    │          │
  └──────────┘    └──────┬───────┘    └──────┬───────┘    └─────┬──────┘    └────┬─────┘
                         │                   │                  │               │
                    Layer 1:            Layer 2:           Layer 3:         Layer 4:
                    Network             Connection         API-level       Application
                    Level               Level              Rate Limit      Level
                    Rate Limit          Rate Limit                         Rate Limit
                         │                   │                  │               │
                         └───────────────────┴──────────────────┴───────────────┘
                                                    │
                                            ┌───────▼───────┐
                                            │     Redis     │
                                            │  (Centralized │
                                            │   Counter)    │
                                            └───────────────┘
```

### রেট লিমিটার রিকোয়েস্ট ফ্লো

```
  Client Request
       │
       ▼
  ┌─────────────────┐     ┌──────────────┐
  │  Rate Limiter   │────▶│ Redis/Cache  │
  │  Middleware      │     │ (Counter     │
  │                  │◀────│  Storage)    │
  └────────┬────────┘     └──────────────┘
           │
     ┌─────┴─────┐
     │           │
  Allowed?    Denied?
     │           │
     ▼           ▼
  ┌──────┐  ┌──────────────┐
  │ App  │  │ 429 Too Many │
  │Server│  │  Requests    │
  └──────┘  │              │
            │ Headers:     │
            │ Retry-After  │
            │ X-RateLimit- │
            │   Remaining  │
            └──────────────┘
```

---

## 💻 অ্যালগরিদম Deep Dive

### ১. Fixed Window Counter

**ধারণা**: সময়কে নির্দিষ্ট উইন্ডোতে (যেমন ১ মিনিট) ভাগ করে প্রতিটি উইন্ডোতে একটি কাউন্টার রাখা হয়। উইন্ডো শেষ হলে কাউন্টার রিসেট হয়।

```
  Fixed Window (1 minute windows, limit = 5)

  Window 1 (00:00-00:59)     Window 2 (01:00-01:59)     Window 3 (02:00-02:59)
  ┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
  │ ■ ■ ■ ■ ■           │    │ ■ ■ ■               │    │ ■ ■ ■ ■ ■           │
  │ count: 5 (FULL)     │    │ count: 3 (OK)       │    │ count: 5 (FULL)     │
  └─────────────────────┘    └─────────────────────┘    └─────────────────────┘

  সমস্যা — Boundary Burst:
  ┌──────────────┬──────────────┐
  │   Window 1   │   Window 2   │
  │         ■■■■■│■■■■■         │  ← উইন্ডো সীমানায় ১০টি রিকোয়েস্ট
  │   :55  :59   │:00   :05     │    (limit ৫ হলেও ১ সেকেন্ডে ১০!)
  └──────────────┴──────────────┘
```

**PHP (Laravel) Implementation:**

```php
<?php

namespace App\Services\RateLimiter;

use Illuminate\Support\Facades\Redis;

class FixedWindowRateLimiter
{
    /**
     * Fixed Window Counter অ্যালগরিদম
     *
     * @param string $key      ইউনিক আইডেন্টিফায়ার (IP/user_id/api_key)
     * @param int    $limit    উইন্ডো প্রতি সর্বোচ্চ রিকোয়েস্ট
     * @param int    $window   উইন্ডো সাইজ সেকেন্ডে
     */
    public function attempt(string $key, int $limit = 60, int $window = 60): array
    {
        $windowKey = $this->getWindowKey($key, $window);

        // Redis MULTI দিয়ে atomic অপারেশন
        $results = Redis::pipeline(function ($pipe) use ($windowKey, $window) {
            $pipe->incr($windowKey);
            $pipe->ttl($windowKey);
        });

        $count = $results[0];
        $ttl = $results[1];

        // প্রথম রিকোয়েস্ট হলে TTL সেট করো
        if ($ttl === -1) {
            Redis::expire($windowKey, $window);
            $ttl = $window;
        }

        $allowed = $count <= $limit;

        return [
            'allowed'   => $allowed,
            'limit'     => $limit,
            'remaining' => max(0, $limit - $count),
            'reset'     => time() + $ttl,
            'retry_after' => $allowed ? null : $ttl,
        ];
    }

    private function getWindowKey(string $key, int $window): string
    {
        $currentWindow = (int)(time() / $window);
        return "rate_limit:fixed:{$key}:{$currentWindow}";
    }
}
```

**JavaScript (Node.js/Express) Implementation:**

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class FixedWindowRateLimiter {
    async attempt(key, limit = 60, window = 60) {
        const currentWindow = Math.floor(Date.now() / 1000 / window);
        const windowKey = `rate_limit:fixed:${key}:${currentWindow}`;

        const multi = redis.multi();
        multi.incr(windowKey);
        multi.ttl(windowKey);
        const results = await multi.exec();

        const count = results[0][1];
        let ttl = results[1][1];

        if (ttl === -1) {
            await redis.expire(windowKey, window);
            ttl = window;
        }

        const allowed = count <= limit;

        return {
            allowed,
            limit,
            remaining: Math.max(0, limit - count),
            reset: Math.floor(Date.now() / 1000) + ttl,
            retryAfter: allowed ? null : ttl,
        };
    }
}

// Express middleware হিসেবে ব্যবহার
function fixedWindowMiddleware(limit = 60, window = 60) {
    const limiter = new FixedWindowRateLimiter();

    return async (req, res, next) => {
        const key = req.ip;
        const result = await limiter.attempt(key, limit, window);

        res.set('X-RateLimit-Limit', result.limit);
        res.set('X-RateLimit-Remaining', result.remaining);
        res.set('X-RateLimit-Reset', result.reset);

        if (!result.allowed) {
            res.set('Retry-After', result.retryAfter);
            return res.status(429).json({
                error: 'Too Many Requests',
                retryAfter: result.retryAfter,
            });
        }
        next();
    };
}

module.exports = { FixedWindowRateLimiter, fixedWindowMiddleware };
```

---

### ২. Sliding Window Log

**ধারণা**: প্রতিটি রিকোয়েস্টের timestamp একটি sorted set এ লগ করা হয়। নির্দিষ্ট উইন্ডো সময়ের বাইরের পুরনো entry মুছে ফেলা হয় এবং বাকিগুলো গণনা করা হয়। এটি সবচেয়ে accurate কিন্তু মেমরি-ব্যয়বহুল পদ্ধতি।

```
  Sliding Window Log (window = 60s, limit = 5)

  সময়রেখা (Timeline):
  ──────────────────────────────────────────────────▶ time

  t=10   t=20   t=35   t=45   t=55   t=65   t=70
   │      │      │      │      │      │      │
   ■      ■      ■      ■      ■      ■      ■
   │      │      │      │      │      │      │
   └──────┴──────┴──────┴──────┴──────┘      │
          current window (t=10~t=65)          │
          count = 5 (সীমায়)                   │
                                              │
              t=70 এ নতুন রিকোয়েস্ট আসলে:
              window: t=11~t=70
              t=10 মুছে যায় → count=4
              নতুন entry যোগ → count=5 ✓

  Redis Sorted Set:
  ┌─────────────────────────────────────┐
  │ KEY: rate_limit:slide:user_123      │
  │ ┌──────────┬──────────┐            │
  │ │  Score    │  Member  │            │
  │ ├──────────┼──────────┤            │
  │ │ 16900010 │ uuid-1   │            │
  │ │ 16900020 │ uuid-2   │            │
  │ │ 16900035 │ uuid-3   │            │
  │ │ 16900045 │ uuid-4   │            │
  │ │ 16900055 │ uuid-5   │            │
  │ └──────────┴──────────┘            │
  └─────────────────────────────────────┘
```

**PHP Implementation:**

```php
<?php

namespace App\Services\RateLimiter;

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Str;

class SlidingWindowLogRateLimiter
{
    public function attempt(string $key, int $limit = 60, int $window = 60): array
    {
        $now = microtime(true);
        $windowStart = $now - $window;
        $redisKey = "rate_limit:swlog:{$key}";
        $member = Str::uuid()->toString();

        // Lua স্ক্রিপ্ট দিয়ে atomic অপারেশন — race condition এড়ানো
        $luaScript = <<<'LUA'
            local key = KEYS[1]
            local window_start = tonumber(ARGV[1])
            local now = tonumber(ARGV[2])
            local limit = tonumber(ARGV[3])
            local member = ARGV[4]
            local window = tonumber(ARGV[5])

            redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
            local count = redis.call('ZCARD', key)

            if count < limit then
                redis.call('ZADD', key, now, member)
                redis.call('EXPIRE', key, window)
                return {1, limit - count - 1, 0}
            else
                local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
                local retry = oldest[2] and (tonumber(oldest[2]) + window - now) or window
                return {0, 0, math.ceil(retry)}
            end
LUA;

        $result = Redis::eval($luaScript, 1, $redisKey, $windowStart, $now, $limit, $member, $window);

        return [
            'allowed'     => (bool)$result[0],
            'limit'       => $limit,
            'remaining'   => (int)$result[1],
            'retry_after' => $result[2] > 0 ? (int)$result[2] : null,
        ];
    }
}
```

**JavaScript Implementation:**

```javascript
const { v4: uuidv4 } = require('uuid');

class SlidingWindowLogRateLimiter {
    constructor(redis) {
        this.redis = redis;
        this.luaScript = `
            local key = KEYS[1]
            local window_start = tonumber(ARGV[1])
            local now = tonumber(ARGV[2])
            local limit = tonumber(ARGV[3])
            local member = ARGV[4]
            local window = tonumber(ARGV[5])

            redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
            local count = redis.call('ZCARD', key)

            if count < limit then
                redis.call('ZADD', key, now, member)
                redis.call('EXPIRE', key, window)
                return {1, limit - count - 1, 0}
            else
                local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
                local retry = oldest[2]
                    and (tonumber(oldest[2]) + window - now) or window
                return {0, 0, math.ceil(retry)}
            end
        `;
    }

    async attempt(key, limit = 60, window = 60) {
        const now = Date.now() / 1000;
        const windowStart = now - window;
        const redisKey = `rate_limit:swlog:${key}`;
        const member = uuidv4();

        const result = await this.redis.eval(
            this.luaScript, 1, redisKey,
            windowStart, now, limit, member, window
        );

        return {
            allowed: Boolean(result[0]),
            limit,
            remaining: result[1],
            retryAfter: result[2] > 0 ? result[2] : null,
        };
    }
}
```

---

### ৩. Sliding Window Counter

**ধারণা**: Fixed Window ও Sliding Window Log এর সংমিশ্রণ। দুটি পাশাপাশি উইন্ডোর কাউন্টারকে weighted average করে approximate sliding window তৈরি করা হয়। মেমরি সাশ্রয়ী কিন্তু সঠিকতায় কিছুটা আপোষ।

```
  Sliding Window Counter (window = 60s, limit = 100)

  বর্তমান সময়: 01:15 (উইন্ডো ০১:০০ এর ১৫ সেকেন্ড অতিক্রান্ত)

  ┌──────────────────────┬──────────────────────┐
  │   Previous Window    │   Current Window     │
  │   (00:00 - 00:59)   │   (01:00 - 01:59)   │
  │                      │                      │
  │   count_prev = 84    │   count_curr = 36    │
  │                      │  ▲                   │
  └──────────────────────┴──┼───────────────────┘
                            │
                        now = 01:15

  হিসাব:
  ┌─────────────────────────────────────────────────────┐
  │ elapsed_ratio = 15/60 = 0.25 (বর্তমান উইন্ডোর ২৫%)│
  │ prev_weight   = 1 - 0.25 = 0.75                    │
  │                                                     │
  │ estimated = prev_weight * count_prev + count_curr   │
  │           = 0.75 * 84 + 36                          │
  │           = 63 + 36 = 99                             │
  │                                                     │
  │ 99 < 100 → ✅ অনুমতি!                               │
  └─────────────────────────────────────────────────────┘
```

**PHP Implementation:**

```php
<?php

namespace App\Services\RateLimiter;

use Illuminate\Support\Facades\Redis;

class SlidingWindowCounterRateLimiter
{
    public function attempt(string $key, int $limit = 100, int $window = 60): array
    {
        $now = time();
        $currentWindow = (int)($now / $window);
        $prevWindow = $currentWindow - 1;

        $currentKey = "rate_limit:swc:{$key}:{$currentWindow}";
        $prevKey = "rate_limit:swc:{$key}:{$prevWindow}";

        $results = Redis::pipeline(function ($pipe) use ($currentKey, $prevKey) {
            $pipe->get($prevKey);
            $pipe->get($currentKey);
        });

        $prevCount = (int)($results[0] ?? 0);
        $currentCount = (int)($results[1] ?? 0);

        $elapsedInWindow = $now % $window;
        $prevWeight = 1 - ($elapsedInWindow / $window);
        $estimatedCount = ($prevWeight * $prevCount) + $currentCount;

        if ($estimatedCount >= $limit) {
            return [
                'allowed'     => false,
                'limit'       => $limit,
                'remaining'   => 0,
                'retry_after' => $window - $elapsedInWindow,
            ];
        }

        Redis::pipeline(function ($pipe) use ($currentKey, $window) {
            $pipe->incr($currentKey);
            $pipe->expire($currentKey, $window * 2);
        });

        return [
            'allowed'   => true,
            'limit'     => $limit,
            'remaining' => max(0, (int)($limit - $estimatedCount - 1)),
            'reset'     => ($currentWindow + 1) * $window,
        ];
    }
}
```

**JavaScript Implementation:**

```javascript
class SlidingWindowCounterRateLimiter {
    constructor(redis) {
        this.redis = redis;
    }

    async attempt(key, limit = 100, window = 60) {
        const now = Math.floor(Date.now() / 1000);
        const currentWindow = Math.floor(now / window);
        const prevWindow = currentWindow - 1;

        const currentKey = `rate_limit:swc:${key}:${currentWindow}`;
        const prevKey = `rate_limit:swc:${key}:${prevWindow}`;

        const [prevCount, currentCount] = await Promise.all([
            this.redis.get(prevKey).then(v => parseInt(v) || 0),
            this.redis.get(currentKey).then(v => parseInt(v) || 0),
        ]);

        const elapsedInWindow = now % window;
        const prevWeight = 1 - (elapsedInWindow / window);
        const estimatedCount = (prevWeight * prevCount) + currentCount;

        if (estimatedCount >= limit) {
            return { allowed: false, limit, remaining: 0, retryAfter: window - elapsedInWindow };
        }

        const multi = this.redis.multi();
        multi.incr(currentKey);
        multi.expire(currentKey, window * 2);
        await multi.exec();

        return {
            allowed: true, limit,
            remaining: Math.max(0, Math.floor(limit - estimatedCount - 1)),
            reset: (currentWindow + 1) * window,
        };
    }
}
```

---

### ৪. Token Bucket

**ধারণা**: একটি বালতিতে নির্দিষ্ট সংখ্যক টোকেন থাকে। প্রতিটি রিকোয়েস্টে একটি টোকেন খরচ হয়। নির্দিষ্ট হারে নতুন টোকেন যোগ হয়। বালতি পূর্ণ হলে নতুন টোকেন বাতিল হয়। **এটি burst-friendly** — বালতি ভর্তি থাকলে একসাথে অনেক রিকোয়েস্ট পাঠানো যায়।

```
  Token Bucket (capacity = 10, refill_rate = 2 tokens/sec)

  ┌───────────────────────┐
  │    Token Bucket       │
  │  ┌─────────────────┐  │
  │  │ @@@@@@@ (৭টি)   │  │  ← ৭টি টোকেন আছে
  │  │                 │  │
  │  │    capacity=10  │  │
  │  └────────┬────────┘  │
  └───────────┼───────────┘
              │
      ┌───────▼───────┐
      │  Request আসলে │
      │  ১টি টোকেন    │
      │  খরচ হয়       │
      └───────────────┘

  সময়ের সাথে:
  ──────────────────────────────────────────────▶ time
  t=0     t=1     t=2     t=3     t=4     t=5

  tokens:  10  →  8   →  10  →  7   →  9   →  6
  used:     -      2       -      3       -      3
  refill:   -      -       2      -       2      -

  ┌────────────────────────────────────────────────┐
  │ সুবিধা: burst traffic হ্যান্ডল করতে পারে     │
  │ ব্যবহার: AWS API Gateway, Stripe, GitHub API  │
  └────────────────────────────────────────────────┘
```

**PHP Implementation:**

```php
<?php

namespace App\Services\RateLimiter;

use Illuminate\Support\Facades\Redis;

class TokenBucketRateLimiter
{
    public function attempt(
        string $key, int $capacity = 10,
        float $refillRate = 1.0, int $tokensNeeded = 1
    ): array {
        $luaScript = <<<'LUA'
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local tokens_needed = tonumber(ARGV[4])

            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1])
            local last_refill = tonumber(bucket[2])

            if tokens == nil then
                tokens = capacity
                last_refill = now
            end

            local elapsed = now - last_refill
            tokens = math.min(capacity, tokens + elapsed * refill_rate)

            local allowed = 0
            local remaining = tokens
            local retry_after = 0

            if tokens >= tokens_needed then
                tokens = tokens - tokens_needed
                allowed = 1
                remaining = tokens
            else
                retry_after = math.ceil((tokens_needed - tokens) / refill_rate)
            end

            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 10)
            return {allowed, math.floor(remaining), retry_after}
LUA;

        $redisKey = "rate_limit:token:{$key}";
        $now = microtime(true);

        $result = Redis::eval(
            $luaScript, 1, $redisKey,
            $capacity, $refillRate, $now, $tokensNeeded
        );

        return [
            'allowed'     => (bool)$result[0],
            'limit'       => $capacity,
            'remaining'   => (int)$result[1],
            'retry_after' => $result[2] > 0 ? (int)$result[2] : null,
        ];
    }
}
```

**JavaScript Implementation:**

```javascript
class TokenBucketRateLimiter {
    constructor(redis) {
        this.redis = redis;
        this.luaScript = `
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local tokens_needed = tonumber(ARGV[4])

            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1])
            local last_refill = tonumber(bucket[2])

            if tokens == nil then
                tokens = capacity
                last_refill = now
            end

            local elapsed = now - last_refill
            tokens = math.min(capacity, tokens + elapsed * refill_rate)

            local allowed = 0
            local remaining = tokens
            local retry_after = 0

            if tokens >= tokens_needed then
                tokens = tokens - tokens_needed
                allowed = 1
                remaining = tokens
            else
                retry_after = math.ceil((tokens_needed - tokens) / refill_rate)
            end

            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 10)
            return {allowed, math.floor(remaining), retry_after}
        `;
    }

    async attempt(key, capacity = 10, refillRate = 1.0, tokensNeeded = 1) {
        const now = Date.now() / 1000;
        const redisKey = `rate_limit:token:${key}`;

        const result = await this.redis.eval(
            this.luaScript, 1, redisKey,
            capacity, refillRate, now, tokensNeeded
        );

        return {
            allowed: Boolean(result[0]),
            limit: capacity,
            remaining: result[1],
            retryAfter: result[2] > 0 ? result[2] : null,
        };
    }
}
```

---

### ৫. Leaky Bucket

**ধারণা**: একটি বালতিতে পানি (রিকোয়েস্ট) ঢালা হয় এবং নিচ থেকে নির্দিষ্ট হারে পানি বের হয়। বালতি পূর্ণ হলে নতুন রিকোয়েস্ট reject হয়। **Output rate সবসময় স্থির থাকে** — burst smoothing এর জন্য আদর্শ।

```
  Leaky Bucket (capacity = 5, leak_rate = 1 req/sec)

         রিকোয়েস্ট ঢোকে (variable rate)
              │ │ │
              ▼ ▼ ▼
  ┌───────────────────┐
  │ ○ ○ ○ ○           │ ← Queue (capacity = 5)
  │ ○                 │   বর্তমানে ৫টি আছে
  │   ┌───────────┐   │
  │   │  ছিদ্র    │   │ ← নির্দিষ্ট হারে বের হয়
  └───┴─────┬─────┴───┘   (1 req/sec — constant)
            │
            ▼
      প্রসেসিং হয় (constant rate)

  পার্থক্য Token Bucket থেকে:
  ┌────────────────────────────┬────────────────────────────┐
  │      Token Bucket          │      Leaky Bucket          │
  │  ● Burst অনুমতি দেয়       │  ● Burst smooth করে        │
  │  ● Output rate পরিবর্তনশীল│  ● Output rate স্থির       │
  │  ● API rate limiting       │  ● Traffic shaping          │
  └────────────────────────────┴────────────────────────────┘
```

**PHP Implementation:**

```php
<?php

namespace App\Services\RateLimiter;

use Illuminate\Support\Facades\Redis;

class LeakyBucketRateLimiter
{
    public function attempt(string $key, int $capacity = 10, float $leakRate = 1.0): array
    {
        $luaScript = <<<'LUA'
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local leak_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])

            local bucket = redis.call('HMGET', key, 'water', 'last_leak')
            local water = tonumber(bucket[1]) or 0
            local last_leak = tonumber(bucket[2]) or now

            local elapsed = now - last_leak
            water = math.max(0, water - elapsed * leak_rate)

            local allowed = 0
            local retry_after = 0

            if water < capacity then
                water = water + 1
                allowed = 1
            else
                retry_after = math.ceil(1 / leak_rate)
            end

            redis.call('HMSET', key, 'water', water, 'last_leak', now)
            redis.call('EXPIRE', key, math.ceil(capacity / leak_rate) + 10)
            return {allowed, math.floor(capacity - water), retry_after}
LUA;

        $redisKey = "rate_limit:leaky:{$key}";
        $result = Redis::eval($luaScript, 1, $redisKey, $capacity, $leakRate, microtime(true));

        return [
            'allowed'     => (bool)$result[0],
            'limit'       => $capacity,
            'remaining'   => (int)$result[1],
            'retry_after' => $result[2] > 0 ? (int)$result[2] : null,
        ];
    }
}
```

**JavaScript Implementation:**

```javascript
class LeakyBucketRateLimiter {
    constructor(redis) {
        this.redis = redis;
        this.luaScript = `
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local leak_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])

            local bucket = redis.call('HMGET', key, 'water', 'last_leak')
            local water = tonumber(bucket[1]) or 0
            local last_leak = tonumber(bucket[2]) or now

            local elapsed = now - last_leak
            water = math.max(0, water - elapsed * leak_rate)

            local allowed = 0
            local retry_after = 0

            if water < capacity then
                water = water + 1
                allowed = 1
            else
                retry_after = math.ceil(1 / leak_rate)
            end

            redis.call('HMSET', key, 'water', water, 'last_leak', now)
            redis.call('EXPIRE', key, math.ceil(capacity / leak_rate) + 10)
            return {allowed, math.floor(capacity - water), retry_after}
        `;
    }

    async attempt(key, capacity = 10, leakRate = 1.0) {
        const now = Date.now() / 1000;
        const result = await this.redis.eval(
            this.luaScript, 1, `rate_limit:leaky:${key}`,
            capacity, leakRate, now
        );

        return {
            allowed: Boolean(result[0]),
            limit: capacity,
            remaining: result[1],
            retryAfter: result[2] > 0 ? result[2] : null,
        };
    }
}
```

---

### 📊 অ্যালগরিদম তুলনা টেবিল

```
┌──────────────────┬──────────┬──────────┬──────────┬────────────┬───────────┐
│    বৈশিষ্ট্য     │  Fixed   │ Sliding  │ Sliding  │   Token    │  Leaky    │
│                  │  Window  │  Window  │  Window  │  Bucket    │  Bucket   │
│                  │ Counter  │   Log    │ Counter  │            │           │
├──────────────────┼──────────┼──────────┼──────────┼────────────┼───────────┤
│ সঠিকতা           │ ★★       │ ★★★★★   │ ★★★★     │ ★★★★       │ ★★★★     │
│ মেমরি             │ ★★★★★   │ ★        │ ★★★★★   │ ★★★★★     │ ★★★★★   │
│ Burst অনুমতি     │ হ্যাঁ*   │ না       │ কিছুটা   │ হ্যাঁ      │ না        │
│ জটিলতা           │ সহজ     │ মাঝারি    │ মাঝারি   │ মাঝারি     │ মাঝারি    │
│ Boundary সমস্যা  │ আছে     │ নেই      │ কম       │ নেই        │ নেই       │
│ Output rate     │ পরিবর্তন │ পরিবর্তন │ পরিবর্তন │ পরিবর্তন   │ স্থির     │
│ ব্যবহার          │ সাধারণ   │ নিরাপত্তা│ API      │ API GW     │ Traffic   │
│                  │ API     │ Brute    │ Limiter  │(AWS/Stripe)│ Shaping   │
│ BD উদাহরণ        │ সাধারণ   │ OTP/     │ bKash    │ Pathao     │ SMS       │
│                  │ API     │ Login    │ API      │ API        │ Gateway   │
└──────────────────┴──────────┴──────────┴──────────┴────────────┴───────────┘

* Fixed Window এ boundary তে burst হতে পারে (2x limit)
```

---

## 🔧 Implementation

### Redis-based Rate Limiter — Production-Ready

**PHP (Laravel) — Universal Rate Limiter Service:**

```php
<?php

namespace App\Services\RateLimiter;

class RateLimiterService
{
    private array $limiters;

    public function __construct(
        FixedWindowRateLimiter $fixedWindow,
        SlidingWindowCounterRateLimiter $slidingWindow,
        TokenBucketRateLimiter $tokenBucket,
        LeakyBucketRateLimiter $leakyBucket,
    ) {
        $this->limiters = [
            'fixed_window'   => $fixedWindow,
            'sliding_window' => $slidingWindow,
            'token_bucket'   => $tokenBucket,
            'leaky_bucket'   => $leakyBucket,
        ];
    }

    public function check(string $key, string $algorithm = 'token_bucket', array $options = []): RateLimitResult
    {
        $limiter = $this->limiters[$algorithm]
            ?? throw new \InvalidArgumentException("Unknown algorithm: {$algorithm}");

        $result = $limiter->attempt($key, ...array_values($options));

        return new RateLimitResult(
            allowed: $result['allowed'], limit: $result['limit'],
            remaining: $result['remaining'],
            retryAfter: $result['retry_after'] ?? null,
            reset: $result['reset'] ?? null,
        );
    }
}

class RateLimitResult
{
    public function __construct(
        public readonly bool $allowed,
        public readonly int $limit,
        public readonly int $remaining,
        public readonly ?int $retryAfter,
        public readonly ?int $reset,
    ) {}

    public function headers(): array
    {
        $headers = [
            'X-RateLimit-Limit'     => $this->limit,
            'X-RateLimit-Remaining' => $this->remaining,
        ];
        if ($this->reset) $headers['X-RateLimit-Reset'] = $this->reset;
        if ($this->retryAfter) $headers['Retry-After'] = $this->retryAfter;
        return $headers;
    }
}
```

**Laravel Middleware:**

```php
<?php

namespace App\Http\Middleware;

use App\Services\RateLimiter\RateLimiterService;
use Closure;
use Illuminate\Http\Request;

class RateLimitMiddleware
{
    public function __construct(private RateLimiterService $rateLimiter) {}

    public function handle(Request $request, Closure $next, string $limiterName = 'default')
    {
        $config = config("rate_limiting.limiters.{$limiterName}");
        $key = $this->resolveKey($request, $config['key_by'] ?? 'ip');

        $result = $this->rateLimiter->check(
            key: "{$limiterName}:{$key}",
            algorithm: $config['algorithm'] ?? 'token_bucket',
            options: $config['options'] ?? [],
        );

        if (!$result->allowed) {
            return response()->json([
                'error'   => 'Too Many Requests',
                'message' => 'রেট লিমিট অতিক্রম। অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন।',
                'retry_after' => $result->retryAfter,
            ], 429)->withHeaders($result->headers());
        }

        return $next($request)->withHeaders($result->headers());
    }

    private function resolveKey(Request $request, string $keyBy): string
    {
        return match ($keyBy) {
            'ip'      => $request->ip(),
            'user'    => $request->user()?->id ?? $request->ip(),
            'api_key' => $request->header('X-API-Key', $request->ip()),
            default   => $request->ip(),
        };
    }
}
```

**JavaScript (Express) — Universal Rate Limiter:**

```javascript
const Redis = require('ioredis');

class RateLimiterService {
    constructor(redisUrl = 'redis://localhost:6379') {
        this.redis = new Redis(redisUrl);
        this.algorithms = {
            fixed_window: new FixedWindowRateLimiter(this.redis),
            sliding_window: new SlidingWindowCounterRateLimiter(this.redis),
            token_bucket: new TokenBucketRateLimiter(this.redis),
            leaky_bucket: new LeakyBucketRateLimiter(this.redis),
        };
    }

    async check(key, algorithm = 'token_bucket', options = {}) {
        const limiter = this.algorithms[algorithm];
        if (!limiter) throw new Error(`Unknown algorithm: ${algorithm}`);
        const { limit = 60, window = 60, capacity, refillRate, leakRate } = options;

        switch (algorithm) {
            case 'token_bucket':
                return limiter.attempt(key, capacity || limit, refillRate || 1);
            case 'leaky_bucket':
                return limiter.attempt(key, capacity || limit, leakRate || 1);
            default:
                return limiter.attempt(key, limit, window);
        }
    }
}

function createRateLimiter(options = {}) {
    const { algorithm = 'sliding_window', limit = 100, window = 60, keyBy = 'ip' } = options;
    const service = new RateLimiterService(options.redisUrl);

    return async (req, res, next) => {
        const key = keyBy === 'user' ? (req.user?.id || req.ip)
                  : keyBy === 'api_key' ? (req.headers['x-api-key'] || req.ip)
                  : req.ip;
        try {
            const result = await service.check(key, algorithm, { limit, window });
            res.set('X-RateLimit-Limit', result.limit);
            res.set('X-RateLimit-Remaining', result.remaining);

            if (!result.allowed) {
                res.set('Retry-After', result.retryAfter);
                return res.status(429).json({
                    error: 'Too Many Requests',
                    message: 'রেট লিমিট অতিক্রম হয়েছে।',
                });
            }
            next();
        } catch (err) {
            console.error('Rate limiter error:', err.message);
            next(); // fail open — Redis ডাউন হলে রিকোয়েস্ট পাশ করো
        }
    };
}

module.exports = { RateLimiterService, createRateLimiter };
```

---

### Laravel Rate Limiting (Built-in)

```php
<?php
// app/Providers/AppServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // সাধারণ API
    RateLimiter::for('api', function ($request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });

    // bKash Payment — মার্চেন্ট ভিত্তিক
    RateLimiter::for('bkash-payment', function ($request) {
        return Limit::perMinute(30)
            ->by($request->header('X-Merchant-ID'))
            ->response(fn() => response()->json([
                'error' => 'Payment rate limit exceeded',
                'message' => 'প্রতি মিনিটে সর্বোচ্চ ৩০টি পেমেন্ট অনুমোদিত।',
            ], 429));
    });

    // OTP — ফোন নম্বর ভিত্তিক, বহুস্তর
    RateLimiter::for('otp', function ($request) {
        $phone = $request->input('phone');
        return [
            Limit::perMinute(1)->by("otp:min:{$phone}"),
            Limit::perHour(5)->by("otp:hour:{$phone}"),
            Limit::perDay(10)->by("otp:day:{$phone}"),
        ];
    });

    // Flash Sale — tiered
    RateLimiter::for('flash-sale', function ($request) {
        $user = $request->user();
        return match ($user?->subscription_tier) {
            'premium' => Limit::perSecond(5)->by("sale:premium:{$user->id}"),
            'basic'   => Limit::perSecond(2)->by("sale:basic:{$user->id}"),
            default   => Limit::perSecond(1)->by("sale:free:{$user->id}"),
        };
    });

    // Login Brute Force Protection
    RateLimiter::for('login', function ($request) {
        $key = $request->input('email') . '|' . $request->ip();
        return [
            Limit::perMinute(5)->by("login:min:{$key}"),
            Limit::perHour(20)->by("login:hour:{$key}"),
        ];
    });
}
```

```php
// routes/api.php
Route::middleware('throttle:api')->get('/products', [ProductController::class, 'index']);
Route::middleware('throttle:bkash-payment')->post('/payments/bkash', [BkashController::class, 'execute']);
Route::middleware('throttle:otp')->post('/otp/send', [OtpController::class, 'send']);
Route::middleware('throttle:login')->post('/login', [AuthController::class, 'login']);
Route::middleware('throttle:flash-sale')->post('/flash-sale/order', [FlashSaleController::class, 'order']);
```

---

### Express Rate Limiting (Libraries)

```javascript
const express = require('express');
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');
const app = express();
const redis = new Redis();

// সাধারণ API limiter
const apiLimiter = rateLimit({
    store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
    windowMs: 60 * 1000, max: 100,
    standardHeaders: true, legacyHeaders: false,
    message: { error: 'Too Many Requests', message: 'রেট লিমিট অতিক্রম হয়েছে।' },
    keyGenerator: (req) => req.user?.id || req.ip,
});

// OTP limiter
const otpLimiter = rateLimit({
    store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
    windowMs: 5 * 60 * 1000, max: 3,
    keyGenerator: (req) => `otp:${req.body.phone}`,
});

// Login brute force protection
const loginLimiter = rateLimit({
    store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
    windowMs: 15 * 60 * 1000, max: 5,
    keyGenerator: (req) => `login:${req.body.email}:${req.ip}`,
    skipSuccessfulRequests: true,
});

// Tiered limiting
function tieredLimiter() {
    const tiers = {
        free:    rateLimit({ windowMs: 60000, max: 10,   store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }) }),
        basic:   rateLimit({ windowMs: 60000, max: 100,  store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }) }),
        premium: rateLimit({ windowMs: 60000, max: 1000, store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }) }),
    };
    return (req, res, next) => (tiers[req.user?.plan] || tiers.free)(req, res, next);
}

app.use('/api/', apiLimiter);
app.post('/api/otp/send', otpLimiter);
app.post('/api/login', loginLimiter);
app.use('/api/v2/', tieredLimiter());
```

---

### Nginx Rate Limiting

```nginx
http {
    # Zone সংজ্ঞায়ন — 10m = ~১,৬০,০০০ IP ধারণক্ষম
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:5m rate=1r/s;

    map $http_x_api_key $api_key_zone {
        default $binary_remote_addr;
        "~."    $http_x_api_key;
    }
    limit_req_zone $api_key_zone zone=api_key:10m rate=30r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name api.example.com.bd;

        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_conn conn_limit 50;
            proxy_pass http://backend;
        }

        location /api/login {
            limit_req zone=login burst=3 nodelay;
            proxy_pass http://backend;
        }

        error_page 429 = @rate_limited;
        location @rate_limited {
            default_type application/json;
            return 429 '{"error":"Too Many Requests"}';
        }
    }
}
```

---

### API Gateway Rate Limiting

**AWS API Gateway:**

```
  ┌──────────┐    ┌─────────────────────────────────────┐    ┌──────────┐
  │  Client  │───▶│        AWS API Gateway              │───▶│  Lambda  │
  │          │    │  Usage Plan:                         │    └──────────┘
  │          │◀───│  - Throttle: 100 req/sec             │
  │          │    │  - Burst: 200 req                    │
  │          │    │  - Quota: 10,000 req/month           │
  └──────────┘    └─────────────────────────────────────┘
```

```javascript
// AWS CDK
const freePlan = api.addUsagePlan('FreePlan', {
    name: 'Free',
    throttle: { rateLimit: 10, burstLimit: 20 },
    quota: { limit: 1000, period: Period.MONTH },
});

const premiumPlan = api.addUsagePlan('PremiumPlan', {
    name: 'Premium',
    throttle: { rateLimit: 1000, burstLimit: 2000 },
    quota: { limit: 1000000, period: Period.MONTH },
});
```

**Kong Gateway:**

```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 60
      hour: 1000
      policy: redis
      redis_host: redis.local
      fault_tolerant: true
  - name: rate-limiting-advanced
    config:
      limit: [100, 5000]
      window_size: [1, 3600]
      strategy: sliding
```

---

## 🔥 Advanced Topics

### ১. Distributed Rate Limiting

```
  সমস্যা: প্রতিটি সার্ভার আলাদা কাউন্ট রাখলে

  ┌─────────┐  ┌──────────────┐
  │ Server 1│  │ counter: 45  │
  └─────────┘  └──────────────┘
  ┌─────────┐  ┌──────────────┐
  │ Server 2│  │ counter: 38  │
  └─────────┘  └──────────────┘
  ┌─────────┐  ┌──────────────┐
  │ Server 3│  │ counter: 22  │
  └─────────┘  └──────────────┘
  মোট = ১০৫ (limit ১০০ হলেও!)

  সমাধান: Centralized Redis
  ┌─────────┐──┐
  │ Server 1│  │    ┌───────────────┐
  └─────────┘  ├───▶│ Redis Cluster │
  ┌─────────┐  │    │ (Single Truth)│
  │ Server 2│  │    │ counter: 105  │
  └─────────┘  │    └───────────────┘
  ┌─────────┐──┘
  │ Server 3│
  └─────────┘
```

**Race Condition সমাধান — Lua Script:**

```php
<?php
// Race Condition: Thread1 GET->99, Thread2 GET->99, both SET->100 (WRONG!)
// সমাধান: Lua Script atomically execute হয়

class DistributedRateLimiter
{
    private const LUA_SCRIPT = <<<'LUA'
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local current_window = math.floor(now / window)
        local curr_key = key .. ':' .. current_window
        local prev_key = key .. ':' .. (current_window - 1)

        local prev_count = tonumber(redis.call('GET', prev_key) or '0')
        local curr_count = tonumber(redis.call('GET', curr_key) or '0')

        local elapsed = now % window
        local weight = 1 - (elapsed / window)
        local estimated = weight * prev_count + curr_count

        if estimated >= limit then
            return {0, 0, math.ceil(window - elapsed)}
        end

        local new_count = redis.call('INCR', curr_key)
        if new_count == 1 then
            redis.call('EXPIRE', curr_key, window * 2)
        end
        return {1, math.floor(limit - estimated - 1), 0}
LUA;

    public function checkWithCluster(string $key, int $limit, int $window): array
    {
        $hashKey = "{rate_limit:{$key}}"; // hash tag for same Redis slot
        $result = Redis::eval(self::LUA_SCRIPT, 1, $hashKey, $limit, $window, microtime(true));

        return [
            'allowed'     => (bool)$result[0],
            'remaining'   => (int)$result[1],
            'retry_after' => $result[2] > 0 ? (int)$result[2] : null,
        ];
    }
}
```

```javascript
// Node.js — Redis Cluster
const cluster = new Redis.Cluster([
    { host: 'redis-node-1', port: 6379 },
    { host: 'redis-node-2', port: 6379 },
    { host: 'redis-node-3', port: 6379 },
]);

class DistributedRateLimiter {
    constructor(redisCluster) {
        this.redis = redisCluster;
        this.redis.defineCommand('rateLimit', {
            numberOfKeys: 1,
            lua: `
                local key = KEYS[1]
                local limit = tonumber(ARGV[1])
                local window = tonumber(ARGV[2])
                local now = tonumber(ARGV[3])
                local current_window = math.floor(now / window)
                local curr_key = key .. ':' .. current_window
                local prev_key = key .. ':' .. (current_window - 1)

                local prev_count = tonumber(redis.call('GET', prev_key) or '0')
                local curr_count = tonumber(redis.call('GET', curr_key) or '0')

                local elapsed = now % window
                local weight = 1 - (elapsed / window)
                local estimated = weight * prev_count + curr_count

                if estimated >= limit then
                    return {0, 0, math.ceil(window - elapsed)}
                end

                local new_count = redis.call('INCR', curr_key)
                if new_count == 1 then redis.call('EXPIRE', curr_key, window * 2) end
                return {1, math.floor(limit - estimated - 1), 0}
            `,
        });
    }

    async check(key, limit = 100, window = 60) {
        const [allowed, remaining, retryAfter] =
            await this.redis.rateLimit(`{rate_limit:${key}}`, limit, window, Date.now() / 1000);
        return { allowed: Boolean(allowed), remaining, retryAfter: retryAfter || null };
    }
}
```

---

### ২. Per-user vs Per-IP vs Per-API-key Limiting

```
  ┌──────────────┬──────────────────────┬──────────────────────────────────┐
  │    Strategy  │      সুবিধা           │      সমস্যা                      │
  ├──────────────┼──────────────────────┼──────────────────────────────────┤
  │   Per-IP     │ Auth ছাড়াই কাজ করে  │ NAT/Proxy তে সমস্যা, VPN bypass │
  │   Per-User   │ সবচেয়ে ন্যায্য       │ শুধু authenticated request এ    │
  │  Per-API-Key │ Plan-ভিত্তিক সীমা    │ Key management দরকার           │
  └──────────────┴──────────────────────┴──────────────────────────────────┘

  সেরা: Multi-layer
  Request ─▶ Per-IP (DDoS) ─▶ Per-API-Key (Plan) ─▶ Per-User (Fair) ─▶ App
```

---

### ৩. Tiered Rate Limiting (Free/Basic/Premium)

```
  ┌──────────┬──────────────┬──────────────┬───────────────────┐
  │   Plan   │  প্রতি মিনিট  │  প্রতি ঘণ্টা  │  প্রতি দিন        │
  ├──────────┼──────────────┼──────────────┼───────────────────┤
  │  Free    │     10       │     100      │     1,000         │
  │  Basic   │     60       │    1,000     │    10,000         │
  │  Pro     │    300       │   10,000     │   100,000         │
  │Enterprise│   1,000      │   50,000     │  1,000,000        │
  └──────────┴──────────────┴──────────────┴───────────────────┘

  bKash Merchant API: Starter ৫০/min | Growth ২০০/min | Enterprise ১০০০/min
```

```javascript
const TIER_CONFIG = {
    free:    { limits: [{ window: 60, max: 10 }, { window: 3600, max: 100 }] },
    basic:   { limits: [{ window: 60, max: 60 }, { window: 3600, max: 1000 }] },
    premium: { limits: [{ window: 60, max: 300 }, { window: 3600, max: 10000 }] },
};

class TieredRateLimiter {
    constructor(redis) {
        this.slidingWindow = new SlidingWindowCounterRateLimiter(redis);
    }

    async check(userId, tier = 'free') {
        const config = TIER_CONFIG[tier] || TIER_CONFIG.free;
        const results = await Promise.all(
            config.limits.map(({ window, max }) =>
                this.slidingWindow.attempt(`${tier}:${userId}`, max, window)
            )
        );

        const denied = results.find(r => !r.allowed);
        if (denied) return { allowed: false, tier, ...denied,
            upgradeHint: tier !== 'premium' ? `${tier} প্ল্যান আপগ্রেড করুন।` : null };

        return { allowed: true, tier, remaining: Math.min(...results.map(r => r.remaining)) };
    }
}
```

---

### ৪. Rate Limit Headers

```
  ┌─────────────────────────────────────────────────────────────┐
  │ HTTP/1.1 200 OK                                            │
  │ X-RateLimit-Limit: 100          ← মোট সীমা                 │
  │ X-RateLimit-Remaining: 57       ← বাকি আছে                 │
  │ X-RateLimit-Reset: 1719849600   ← রিসেট সময় (epoch)        │
  │                                                             │
  │ রিজেক্ট হলে:                                                │
  │ HTTP/1.1 429 Too Many Requests                              │
  │ Retry-After: 30                 ← কত সেকেন্ড পর retry       │
  │ X-RateLimit-Remaining: 0                                    │
  └─────────────────────────────────────────────────────────────┘
```

---

### ৫. Graceful Degradation Under Rate Limit

```
  Exponential Backoff with Jitter:
  ┌────────────────────────────────────────────────┐
  │ Retry 1: 1s  + random(0, 500ms)               │
  │ Retry 2: 2s  + random(0, 500ms)               │
  │ Retry 3: 4s  + random(0, 500ms)               │
  │ Retry 4: 8s  + random(0, 500ms)               │
  │ Retry 5: 16s + random(0, 500ms) → give up     │
  │                                                │
  │ Jitter কেন? "Thundering herd" এড়াতে          │
  └────────────────────────────────────────────────┘
```

```javascript
class ResilientApiClient {
    constructor(baseURL, maxRetries = 5) {
        this.baseURL = baseURL;
        this.maxRetries = maxRetries;
    }

    async request(path, options = {}, attempt = 0) {
        const response = await fetch(`${this.baseURL}${path}`, options);

        if (response.status === 429 && attempt < this.maxRetries) {
            const retryAfter = parseInt(response.headers.get('Retry-After'));
            const wait = retryAfter ? retryAfter * 1000 : this.backoff(attempt);
            await new Promise(r => setTimeout(r, wait));
            return this.request(path, options, attempt + 1);
        }
        return response;
    }

    backoff(attempt) {
        return Math.min(Math.pow(2, attempt) * 1000 + Math.random() * 500, 30000);
    }
}
```

```php
<?php
// Server-side Graceful Degradation
class GracefulDegradationMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $result = app(RateLimiterService::class)->check(
            "global:{$request->path()}", 'token_bucket',
            ['capacity' => 1000, 'refillRate' => 100]
        );

        // ৮০% সীমায় — non-essential features বন্ধ
        if ($result->remaining < $result->limit * 0.2) {
            config(['app.features.recommendations' => false]);
        }

        // ৯৫% সীমায় — শুধু critical endpoints
        if ($result->remaining < $result->limit * 0.05) {
            if (!in_array($request->path(), ['/api/auth', '/api/payments', '/api/health'])) {
                return response()->json([
                    'message' => 'সিস্টেমে অতিরিক্ত লোড। কিছুক্ষণ পর চেষ্টা করুন।',
                ], 503);
            }
        }
        return $next($request);
    }
}
```

---

### ৬. Rate Limiting in Microservices

```
  ┌──────────┐     ┌────────────────────────────────────────────┐
  │  Client  │────▶│  API Gateway (Global Rate Limiting)       │
  └──────────┘     │  Global: 10,000 req/min | Per-user: 100   │
                   └──────┬──────────────┬──────────────┬──────┘
                          │              │              │
                   ┌──────▼──────┐ ┌─────▼──────┐ ┌────▼───────┐
                   │  Auth       │ │  Payment   │ │  Product   │
                   │ 50 req/min  │ │ 30 req/min │ │200 req/min │
                   └─────────────┘ └────────────┘ └────────────┘
                          │              │              │
                          └──────────────┴──────────────┘
                                  ┌──────▼──────┐
                                  │ Redis Cluster│
                                  └─────────────┘

  Service-to-Service:
  ┌──────────┐  circuit breaker   ┌──────────┐
  │  Order   │────────────────────│ Payment  │
  │ Service  │  rate: 50 req/sec  │ Service  │
  └──────────┘  429 → fallback    └──────────┘
```

---

### ৭. DDoS Mitigation Strategies

```
  Multi-layer Defense:

  Layer 1: CDN/WAF (Cloudflare/AWS Shield)
  ┌──────────────────────────────────────────┐
  │ IP Reputation | Geo-blocking | Bot detect│
  └────────────────────┬─────────────────────┘
                       ▼
  Layer 2: Load Balancer (Nginx/HAProxy)
  ┌──────────────────────────────────────────┐
  │ Connection limiting | Request size limit │
  └────────────────────┬─────────────────────┘
                       ▼
  Layer 3: API Gateway (Kong/AWS)
  ┌──────────────────────────────────────────┐
  │ Per-API-key limiting | Request validation│
  └────────────────────┬─────────────────────┘
                       ▼
  Layer 4: Application (Laravel/Express)
  ┌──────────────────────────────────────────┐
  │ Per-user limiting | Anomaly detection    │
  └──────────────────────────────────────────┘
```

```javascript
// DDoS Anomaly Detection
class AnomalyDetector {
    constructor(redis) { this.redis = redis; }

    async recordAndCheck(ip) {
        const now = Math.floor(Date.now() / 1000);
        const minuteKey = `anomaly:${ip}:${Math.floor(now / 60)}`;

        const count = await this.redis.incr(minuteKey);
        if (count === 1) this.redis.expire(minuteKey, 120);

        if (count > 500) {
            console.warn(`DDoS suspect: ${ip} — ${count} req/min`);
            await this.redis.setex(`banned:${ip}`, 3600, '1');
        }

        return { isBanned: await this.redis.exists(`banned:${ip}`) };
    }
}
```

---

## ✅ সুবিধা ও অসুবিধা

```
┌──────────────────────────────────┬──────────────────────────────────┐
│         ✅ সুবিধা                 │         ❌ অসুবিধা               │
├──────────────────────────────────┼──────────────────────────────────┤
│ DDoS আক্রমণ থেকে সুরক্ষা        │ Legitimate ইউজার block হতে পারে │
│ সার্ভার স্থিতিশীলতা              │ বাস্তবায়ন জটিলতা (distributed)  │
│ ন্যায্য সম্পদ বিতরণ             │ অতিরিক্ত latency যোগ হয়         │
│ API অপব্যবহার রোধ               │ Redis SPOF হতে পারে             │
│ খরচ নিয়ন্ত্রণ                   │ সঠিক limit নির্ধারণ কঠিন        │
│ Brute force প্রতিরোধ            │ IP rotation দিয়ে bypass সম্ভব   │
│ Monetization (tiered plans)      │ User experience খারাপ হতে পারে  │
│ সিস্টেম analytics               │ Testing জটিল                    │
└──────────────────────────────────┴──────────────────────────────────┘
```

---

## ⚠️ সাধারণ ভুল ও সমাধান

### ১. শুধু In-Memory Counter ব্যবহার

```php
// ❌ ভুল — রিস্টার্ট হলে হারিয়ে যায়, distributed এ কাজ করে না
$counter = Cache::get("limit:{$ip}", 0);
if ($counter >= 100) { abort(429); }
Cache::put("limit:{$ip}", $counter + 1, 60);

// ✅ সঠিক — Redis + Lua Script দিয়ে atomic operation
$result = Redis::eval($atomicLuaScript, 1, "limit:{$ip}", 100, 60, time());
```

### ২. Race Condition উপেক্ষা

```javascript
// ❌ GET/CHECK/SET এর মাঝে race condition
const count = await redis.get(key);
if (count >= limit) return deny();
await redis.incr(key); // race!

// ✅ Lua Script দিয়ে atomic operation
const result = await redis.eval(luaScript, 1, key, limit, window, now);
```

### ৩. Fail Open না করা

```javascript
// ❌ Redis ডাউন = পুরো API ডাউন
try { await rateLimiter.check(key); }
catch (err) { return res.status(500).send('Error'); }

// ✅ Redis ডাউন = রিকোয়েস্ট পাশ করো (fail open)
try { const r = await rateLimiter.check(key);
    if (!r.allowed) return res.status(429).send();
} catch (err) { console.error(err.message); next(); }
```

### ৪. এক মাপে সবাইকে Fit করা

```php
// ❌ সব endpoint এ একই limit
Route::middleware('throttle:60,1')->group(fn() => ...);

// ✅ endpoint অনুযায়ী আলাদা limit
Route::middleware('throttle:200,1')->get('/products', ...);
Route::middleware('throttle:30,1')->post('/orders', ...);
Route::middleware('throttle:bkash-payment')->post('/payments', ...);
```

### ৫. Rate Limit Headers না পাঠানো

```
❌ HTTP 429 — কোনো হেডার নেই, client জানে না কী করবে
✅ HTTP 429 + Retry-After + X-RateLimit-Remaining headers
```

### ৬. Clock Synchronization সমস্যা

```
Distributed সিস্টেমে ঘড়ি ভিন্ন হলে:
  Server 1: 12:00:00 → Window 720
  Server 3: 11:59:58 → Window 719 ← ভিন্ন!

সমাধান: NTP sync | Redis TIME ব্যবহার | Sliding window
```

---

## 📋 সারসংক্ষেপ

```
  ┌──────────────────────────────────────────────────────────────┐
  │  রেট লিমিটিং — সিদ্ধান্ত গাইড                                │
  │                                                              │
  │  কোন অ্যালগরিদম?                                             │
  │  সাধারণ API       → Sliding Window Counter                   │
  │  Burst দরকার      → Token Bucket                             │
  │  Smooth output    → Leaky Bucket                             │
  │  Memory সীমিত     → Fixed Window Counter                     │
  │  নিরাপত্তা        → Sliding Window Log                       │
  │                                                              │
  │  কোথায় implement?                                            │
  │  DDoS             → CDN/WAF (Cloudflare)                     │
  │  Connection       → Load Balancer (Nginx)                    │
  │  API management   → API Gateway (Kong/AWS)                   │
  │  Business logic   → Application (Laravel/Express)            │
  │                                                              │
  │  Redis down হলে?                                             │
  │  ● Fail open (রিকোয়েস্ট পাশ করো)                            │
  │  ● In-memory fallback                                        │
  │  ● Circuit breaker pattern                                   │
  └──────────────────────────────────────────────────────────────┘

  মূল নীতি:
  ┌────────────────────────────────────────────────────────────┐
  │ ১. Multi-layer defense — একাধিক স্তরে rate limit          │
  │ ২. Fail gracefully   — Redis ডাউন = রিকোয়েস্ট পাশ        │
  │ ৩. Inform clients    — Headers দিয়ে limit জানাও           │
  │ ৪. Monitor & alert   — সীমা অতিক্রমে alert পাঠাও          │
  │ ৫. Atomic operations — Lua script দিয়ে race condition এড়াও│
  │ ৬. Test thoroughly   — Load test দিয়ে limit যাচাই করো      │
  └────────────────────────────────────────────────────────────┘
```

---

> **মনে রাখুন**: রেট লিমিটিং হলো defense-in-depth এর একটি অংশ। শুধু রেট লিমিটিং দিয়ে সব আক্রমণ ঠেকানো সম্ভব নয়। WAF, CAPTCHA, authentication, input validation — সবকিছু মিলিয়ে একটি সম্পূর্ণ নিরাপত্তা ব্যবস্থা তৈরি করুন।
