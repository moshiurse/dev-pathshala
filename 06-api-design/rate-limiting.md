# 🚦 Rate Limiting — API Rate Limiting বিস্তারিত গাইড

## 📖 সংজ্ঞা ও মূল ধারণা

Rate Limiting হলো এমন একটি technique যেটি নির্দিষ্ট সময়ের মধ্যে কোনো client কতবার API request পাঠাতে পারবে তা সীমাবদ্ধ করে। এটি API-কে abuse, DDoS attack, এবং resource exhaustion থেকে রক্ষা করে।

### মূল উদ্দেশ্য:
- **Service Protection**: সার্ভারকে overload থেকে রক্ষা করা
- **Fair Usage**: সকল ব্যবহারকারীর জন্য সমান সুযোগ নিশ্চিত করা
- **Cost Control**: Infrastructure খরচ নিয়ন্ত্রণে রাখা
- **Security**: Brute-force ও DDoS attack প্রতিরোধ করা

---

## 🏗️ Rate Limiting Algorithms

### 1. Fixed Window Algorithm

Fixed Window-এ সময়কে নির্দিষ্ট আকারের window-তে ভাগ করা হয় (যেমন প্রতি ১ মিনিট)। প্রতিটি window-তে একটি counter থাকে যা request আসলে বাড়ে।

```
Fixed Window (প্রতি মিনিটে ৫টি request):

সময়রেখা:
|-------- Window 1 --------|-------- Window 2 --------|
0:00                  1:00                        2:00

Window 1: ■ ■ ■ ■ ■ ✗ ✗    (৫টি সফল, ২টি প্রত্যাখ্যাত)
Window 2: ■ ■ ■ □ □          (৩টি সফল, ২টি বাকি)

■ = সফল request    ✗ = প্রত্যাখ্যাত    □ = ব্যবহার হয়নি
```

**সমস্যা — Boundary Burst**: দুটি window-এর সীমানায় burst হতে পারে। Window 1-এর শেষে ৫টি + Window 2-এর শুরুতে ৫টি = ১ সেকেন্ডে ১০টি request।

### 2. Sliding Window Log Algorithm

প্রতিটি request-এর timestamp সংরক্ষণ করা হয়। নতুন request আসলে পুরাতন (window-এর বাইরের) entry মুছে ফেলা হয় এবং বাকিগুলো গণনা করা হয়।

```
Sliding Window Log (১ মিনিটে ৫টি):

সময়রেখা (সেকেন্ড):
  10s   20s   30s   45s   55s   65s   70s
   ■     ■     ■     ■     ■     ?     ?

65s-এ request আসলে → [10s, 20s, 30s, 45s, 55s] → 5:05 থেকে 1:05 window
→ 10s বাদ → [20s, 30s, 45s, 55s] = ৪টি → ✅ অনুমোদিত

70s-এ request আসলে → [20s, 30s, 45s, 55s, 65s] = ৫টি → ✗ প্রত্যাখ্যাত
```

### 3. Token Bucket Algorithm (সবচেয়ে জনপ্রিয়)

একটি bucket-এ নির্দিষ্ট সংখ্যক token থাকে। প্রতিটি request-এ ১টি token খরচ হয়। নির্দিষ্ট rate-এ bucket-এ নতুন token যোগ হয়। Bucket পূর্ণ থাকলে নতুন token বাতিল হয়।

```
Token Bucket (সর্বোচ্চ ১০ token, প্রতি সেকেন্ডে ২ token refill):

সময়  | Bucket | Request | ফলাফল
------|--------|---------|--------
0s    | ●●●●●●●●●● (10) | —       | —
1s    | ●●●●●●●●●  (9)  | 1টি     | ✅ token ব্যবহার
2s    | ●●●●●●●●   (8)  | 3টি     | ✅ 3 token ব্যবহার → 7 (+ 2 refill = 9)
3s    | ●●●●●●●●●  (9)  | 0       | + 2 refill = 10 (max)
4s    | ●●●●●●●●●● (10) | 10টি    | ✅ সব token ব্যবহার → 0
5s    | ●●         (2)   | 3টি     | ✅ 2টি, ✗ 1টি (token নেই)

● = token    ✅ = সফল    ✗ = প্রত্যাখ্যাত
```

---

## 🎯 বাস্তব উদাহরণ (Real-life Analogy)

**Token Bucket → পানির ট্যাংক:**
কল্পনা করুন আপনার বাড়ির ছাদে একটি পানির ট্যাংক আছে যেটি সর্বোচ্চ ১০০০ লিটার ধরে। পাম্প প্রতি ঘণ্টায় ১০০ লিটার পানি ওঠায়। আপনি যখন খুশি পানি ব্যবহার করতে পারেন — একবারে ৫০০ লিটারও খরচ করতে পারেন (burst)। কিন্তু ট্যাংক খালি হলে পাম্পের rate-এ অপেক্ষা করতে হবে।

**Fixed Window → সিনেমা হলের শো:**
প্রতিটি শো-তে ২০০ সিট আছে। ১টি শো শেষ হলে আবার নতুন ২০০ সিট পাওয়া যায়। দুটি শো-র মাঝে কতজন এসেছিল সেটা matter করে না।

---

## 💻 PHP Code Example — Rate Limiting Middleware

```php
<?php

class TokenBucketRateLimiter
{
    private Redis $redis;
    private int $maxTokens;
    private int $refillRate; // tokens per second
    
    public function __construct(Redis $redis, int $maxTokens = 60, int $refillRate = 1)
    {
        $this->redis = $redis;
        $this->maxTokens = $maxTokens;
        $this->refillRate = $refillRate;
    }
    
    public function attempt(string $identifier): array
    {
        $key = "rate_limit:{$identifier}";
        $now = microtime(true);
        
        // Lua script দিয়ে atomic operation নিশ্চিত করা
        $luaScript = <<<'LUA'
            local key = KEYS[1]
            local max_tokens = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1])
            local last_refill = tonumber(bucket[2])
            
            if tokens == nil then
                tokens = max_tokens
                last_refill = now
            end
            
            -- Token refill গণনা
            local elapsed = now - last_refill
            local new_tokens = elapsed * refill_rate
            tokens = math.min(max_tokens, tokens + new_tokens)
            
            local allowed = 0
            local remaining = tokens
            
            if tokens >= 1 then
                tokens = tokens - 1
                allowed = 1
                remaining = tokens
            end
            
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, max_tokens / refill_rate * 2)
            
            return {allowed, math.floor(remaining), math.floor((1 - tokens) / refill_rate)}
        LUA;
        
        $result = $this->redis->eval(
            $luaScript, [$key, $this->maxTokens, $this->refillRate, $now], 1
        );
        
        return [
            'allowed'   => (bool) $result[0],
            'remaining' => $result[1],
            'retry_after' => $result[2],
        ];
    }
}

// Laravel Middleware উদাহরণ
class RateLimitMiddleware
{
    private TokenBucketRateLimiter $limiter;
    
    public function __construct(TokenBucketRateLimiter $limiter)
    {
        $this->limiter = $limiter;
    }
    
    public function handle($request, Closure $next)
    {
        $identifier = $request->user()?->id ?? $request->ip();
        $result = $this->limiter->attempt($identifier);
        
        if (!$result['allowed']) {
            return response()->json([
                'error' => 'Too Many Requests',
                'retry_after' => $result['retry_after'],
            ], 429)->withHeaders([
                'X-RateLimit-Remaining' => $result['remaining'],
                'Retry-After' => $result['retry_after'],
            ]);
        }
        
        $response = $next($request);
        
        return $response->withHeaders([
            'X-RateLimit-Remaining' => $result['remaining'],
        ]);
    }
}
```

---

## 🟨 JavaScript Code Example — Express Middleware

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class SlidingWindowRateLimiter {
    constructor({ windowMs = 60000, maxRequests = 100 } = {}) {
        this.windowMs = windowMs;
        this.maxRequests = maxRequests;
    }

    async isAllowed(identifier) {
        const key = `rate_limit:${identifier}`;
        const now = Date.now();
        const windowStart = now - this.windowMs;

        // Redis sorted set ব্যবহার করে sliding window log
        const pipeline = redis.pipeline();
        pipeline.zremrangebyscore(key, 0, windowStart);   // পুরানো entry মুছে ফেলা
        pipeline.zadd(key, now, `${now}-${Math.random()}`); // নতুন entry যোগ
        pipeline.zcard(key);                                // মোট entry গণনা
        pipeline.expire(key, Math.ceil(this.windowMs / 1000));

        const results = await pipeline.exec();
        const requestCount = results[2][1];

        return {
            allowed: requestCount <= this.maxRequests,
            remaining: Math.max(0, this.maxRequests - requestCount),
            resetAt: new Date(now + this.windowMs).toISOString(),
        };
    }
}

// Express Middleware
function rateLimitMiddleware(options = {}) {
    const limiter = new SlidingWindowRateLimiter(options);

    return async (req, res, next) => {
        const identifier = req.user?.id || req.ip;
        const result = await limiter.isAllowed(identifier);

        // Standard rate limit headers সেট করা
        res.set({
            'X-RateLimit-Limit': options.maxRequests || 100,
            'X-RateLimit-Remaining': result.remaining,
            'X-RateLimit-Reset': result.resetAt,
        });

        if (!result.allowed) {
            return res.status(429).json({
                error: 'অনুরোধ সীমা অতিক্রম করেছে',
                message: 'Too Many Requests',
                retryAfter: result.resetAt,
            });
        }

        next();
    };
}

// ব্যবহার
const express = require('express');
const app = express();

// সকল route-এ প্রতি মিনিটে ৬০টি request
app.use(rateLimitMiddleware({ windowMs: 60000, maxRequests: 60 }));

// নির্দিষ্ট route-এ আলাদা limit
app.post('/api/login', rateLimitMiddleware({ windowMs: 900000, maxRequests: 5 }), (req, res) => {
    res.json({ message: 'Login endpoint' });
});
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | Algorithm | কারণ |
|-----------|-----------|------|
| সাধারণ API protection | Token Bucket | Burst allow করে, সহজ implementation |
| Login/OTP endpoint | Fixed Window | কঠোর সীমা দরকার |
| Paid API (per-request billing) | Sliding Window Log | সুনির্দিষ্ট গণনা প্রয়োজন |
| Microservice-to-microservice | Token Bucket | Backpressure সমর্থন করে |
| Public API (GitHub, Stripe style) | Token Bucket + Fixed Window | Flexibility ও সুরক্ষা দুটোই |

## ❌ কখন ব্যবহার করবেন না

- **Internal-only service** যেখানে trust boundary নেই — অপ্রয়োজনীয় complexity বাড়ে
- **Event-driven/async system** যেখানে request-response model নেই — queue-based throttling ভালো কাজ করে
- **Batch processing pipeline** — এখানে backpressure mechanism ব্যবহার করুন, rate limiting নয়
- শুধুমাত্র rate limiting দিয়ে **security** নিশ্চিত করার চেষ্টা করবেন না — এটি defense-in-depth এর একটি layer মাত্র

---

## 📊 Algorithm তুলনা

```
                 | Fixed Window | Sliding Log | Token Bucket |
-----------------+--------------+-------------+--------------+
Memory           | কম (O(1))    | বেশি (O(n)) | কম (O(1))    |
সুনির্দিষ্টতা    | মাঝারি       | সর্বোচ্চ     | ভালো          |
Burst সমর্থন     | না           | না          | হ্যাঁ          |
Implementation   | সহজ         | মাঝারি       | মাঝারি        |
Boundary সমস্যা  | হ্যাঁ          | না          | না            |
```

---

## 🔗 Response Headers Best Practice

```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100          ← মোট অনুমোদিত request
X-RateLimit-Remaining: 0        ← বাকি request
X-RateLimit-Reset: 1672531200   ← কখন reset হবে (Unix timestamp)
Retry-After: 30                 ← কত সেকেন্ড পর আবার চেষ্টা করবেন
```

> 💡 **Senior Engineer Tip:** Distributed system-এ rate limiting করতে হলে centralized store (Redis) ব্যবহার করুন। প্রতিটি application instance-এ local rate limiting করলে সামগ্রিক limit সঠিক থাকবে না। তবে Redis down থাকলে fail-open নাকি fail-closed হবে — সেই সিদ্ধান্ত আপনার system-এর criticality-র উপর নির্ভর করে।
