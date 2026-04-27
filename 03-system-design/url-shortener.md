# 🔗 URL শর্টনার সিস্টেম ডিজাইন (কেস স্টাডি)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **প্রেক্ষাপট**: বাংলাদেশে bKash প্রমোশনাল লিংক, .bd ডোমেইন, ই-কমার্স ক্যাম্পেইন ট্র্যাকিং

---

## 📑 সূচিপত্র

- [Requirements](#-requirements)
- [Back-of-envelope Estimation](#-back-of-envelope-estimation)
- [High-Level Design](#️-high-level-design)
- [Detailed Design](#-detailed-design)
- [Advanced Topics](#-advanced-topics)
- [Full Implementation](#-full-implementation)
- [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### ফাংশনাল রিকোয়ারমেন্টস (Functional Requirements)

URL শর্টনার সিস্টেমের মূল কাজগুলো কী কী হবে — সেটা আগে ঠিক করতে হবে।

| # | রিকোয়ারমেন্ট | বিস্তারিত |
|---|---------------|-----------|
| F1 | **URL শর্ট করা** | লম্বা URL ইনপুট দিলে ছোট URL তৈরি হবে (যেমন `sho.rt/abc123`) |
| F2 | **রিডাইরেক্ট** | শর্ট URL-এ ক্লিক করলে অরিজিনাল URL-এ নিয়ে যাবে |
| F3 | **কাস্টম অ্যালিয়াস** | ইউজার নিজের পছন্দের শর্ট কোড দিতে পারবে (যেমন `sho.rt/bkash-offer`) |
| F4 | **এক্সপিরেশন** | URL-এর মেয়াদ সেট করা যাবে (যেমন bKash ক্যাশব্যাক অফার ৭ দিনের জন্য) |
| F5 | **অ্যানালিটিক্স** | কতবার ক্লিক হয়েছে, কোন দেশ থেকে, কোন ডিভাইস — সব ট্র্যাক হবে |
| F6 | **ইউজার ম্যানেজমেন্ট** | রেজিস্ট্রেশন করে লিংক ম্যানেজ করা যাবে |

### নন-ফাংশনাল রিকোয়ারমেন্টস (Non-Functional Requirements)

এগুলো সিস্টেমের "গুণগত মান" — ফাংশনালিটি নয়, পারফরম্যান্স ও রিলায়েবিলিটি সম্পর্কিত।

| # | রিকোয়ারমেন্ট | টার্গেট |
|---|---------------|---------|
| NF1 | **লো লেটেন্সি** | রিডাইরেক্ট < 100ms (ক্যাশ হিট হলে < 10ms) |
| NF2 | **হাই অ্যাভেইলেবিলিটি** | 99.99% আপটাইম (বছরে মাত্র ~52 মিনিট ডাউনটাইম) |
| NF3 | **রিড হেভি** | প্রতিদিন 100M URL রিড (রিডাইরেক্ট), 1M URL রাইট (শর্টেন) |
| NF4 | **স্কেলেবিলিটি** | বিলিয়ন URL সাপোর্ট করতে পারবে |
| NF5 | **ডিউরেবিলিটি** | একবার তৈরি হওয়া শর্ট URL কখনো হারাবে না |
| NF6 | **সিকিউরিটি** | ম্যালিশাস URL ডিটেকশন, রেট লিমিটিং |

> **বাংলাদেশ কনটেক্সট**: bKash/Nagad প্রমোশনাল ক্যাম্পেইনে একই লিংকে লক্ষ লক্ষ ক্লিক আসতে পারে। ঈদের সময় ট্রাফিক স্পাইক হ্যান্ডেল করা জরুরি। .com.bd ডোমেইনে শর্ট URL সার্ভিস দিলে লোকাল ট্রাস্ট বাড়ে।

---

## 📊 Back-of-envelope Estimation

সিস্টেম ডিজাইনের আগে "মোটামুটি হিসাব" করা অত্যন্ত গুরুত্বপূর্ণ। এটা আমাদের বলে দেয় কত বড় ইনফ্রাস্ট্রাকচার লাগবে।

### ট্রাফিক এস্টিমেশন

```
রাইট (URL শর্টেন):     1M/day = ~12 URLs/sec
রিড (রিডাইরেক্ট):      100M/day = ~1,160 URLs/sec
রিড:রাইট রেশিও:        100:1 (অত্যন্ত রিড-হেভি)

পিক ট্রাফিক (ঈদ/bKash অফার):
  রিড পিক:             ~5,000 URLs/sec (4-5x স্বাভাবিকের চেয়ে বেশি)
  রাইট পিক:            ~50 URLs/sec
```

### স্টোরেজ এস্টিমেশন

```
প্রতিটি URL রেকর্ডের গড় সাইজ:
  - short_code:         7 bytes
  - original_url:       ~200 bytes (গড়ে)
  - user_id:            8 bytes
  - created_at:         8 bytes
  - expires_at:         8 bytes
  - click_count:        8 bytes
  - metadata:           ~100 bytes
  ─────────────────────────────
  মোট:                  ~340 bytes/record

৫ বছরে মোট URL:
  1M/day × 365 × 5 = 1.825 billion URLs
  1.825B × 340 bytes ≈ 620 GB (শুধু URL ডাটা)

অ্যানালিটিক্স ডাটা (প্রতি ক্লিক):
  - short_code_ref:     7 bytes
  - timestamp:          8 bytes
  - ip_hash:            16 bytes
  - country:            2 bytes
  - device_type:        1 byte
  - referrer:           ~100 bytes
  ─────────────────────────────
  মোট:                  ~134 bytes/click

৫ বছরে অ্যানালিটিক্স:
  100M/day × 365 × 5 = 182.5 billion clicks
  182.5B × 134 bytes ≈ 24.5 TB (এটা অনেক বড়!)
```

### ব্যান্ডউইথ এস্টিমেশন

```
ইনকামিং (রাইট):  12/sec × 340 bytes ≈ 4 KB/s (নগণ্য)
আউটগোয়িং (রিড): 1,160/sec × 340 bytes ≈ 400 KB/s (নগণ্য)

কিন্তু HTTP হেডার সহ (redirect response ~500 bytes):
আউটগোয়িং:       1,160/sec × 500 bytes ≈ 580 KB/s
পিক আউটগোয়িং:   5,000/sec × 500 bytes ≈ 2.5 MB/s
```

### ক্যাশ এস্টিমেশন (80-20 রুল)

```
80% ট্রাফিক আসে 20% URL থেকে (পেরেটো প্রিন্সিপল)

দৈনিক ইউনিক URL রিড:    100M × 20% = 20M ইউনিক URL
ক্যাশ সাইজ:              20M × 340 bytes ≈ 6.8 GB

Redis-এ 8 GB RAM রাখলে যথেষ্ট।
ক্যাশ হিট রেট:            ~80-90% (প্রায় সব জনপ্রিয় URL ক্যাশে থাকবে)
```

---

## 🏗️ High-Level Design

### আর্কিটেকচার ওভারভিউ

```
                          ┌─────────────────────────────────┐
                          │         DNS (.com.bd)            │
                          │    sho.com.bd → Load Balancer    │
                          └──────────────┬──────────────────┘
                                         │
                          ┌──────────────▼──────────────────┐
                          │        Load Balancer             │
                          │     (Nginx / AWS ALB)            │
                          │   - SSL Termination              │
                          │   - Rate Limiting (L7)           │
                          └──────┬───────────┬──────────────┘
                                 │           │
                    ┌────────────▼──┐   ┌────▼────────────┐
                    │  Write Service │   │  Read Service    │
                    │  (Laravel API) │   │  (Node.js)       │
                    │                │   │                  │
                    │  POST /shorten │   │  GET /:code      │
                    │  CRUD APIs     │   │  (Redirect)      │
                    │                │   │  Ultra-fast      │
                    └───┬────┬──────┘   └──┬───────┬──────┘
                        │    │             │       │
               ┌────────▼┐   │    ┌────────▼┐     │
               │ Postgres │   │    │  Redis   │     │
               │ (Write)  │   │    │ Cache    │     │
               │          │   │    │ Cluster  │     │
               └────┬─────┘   │    └────┬─────┘     │
                    │         │         │           │
               ┌────▼─────────▼─────────▼───────────▼──┐
               │        PostgreSQL Read Replicas        │
               │     (Async Replication, 2-3 nodes)     │
               └────────────────┬──────────────────────┘
                                │
               ┌────────────────▼──────────────────────┐
               │     Analytics Pipeline                 │
               │  Redis Counter → Kafka → ClickHouse   │
               └────────────────────────────────────────┘
```

### কেন দুইটা আলাদা সার্ভিস?

**Write Service (Laravel)**: URL শর্টেন করা তুলনামূলক কম ঘটে (1M/day) কিন্তু জটিল — ভ্যালিডেশন, কলিশন চেক, ডাটাবেজ রাইট। Laravel-এর Eloquent ORM, Validation, Middleware এখানে শক্তিশালী।

**Read Service (Node.js)**: রিডাইরেক্ট অনেক বেশি ঘটে (100M/day) এবং সিম্পল — ক্যাশ/DB থেকে URL খুঁজে redirect। Node.js-এর non-blocking I/O এই ধরনের হাই-থ্রুপুট, লো-লেটেন্সি কাজে দারুণ।

### রিকোয়েস্ট ফ্লো (সংক্ষেপে)

```
[শর্টেন করা]
User → POST /api/shorten → Laravel → Validate → Generate Code
     → Check Collision → Store in DB → Return Short URL

[রিডাইরেক্ট]
User → GET /abc123 → Node.js → Redis Cache Hit? → Yes → 301 Redirect
                                                → No  → DB Lookup
                                                      → Cache Store
                                                      → 301 Redirect
                                                      → Async: Analytics Event
```

---

## 💻 Detailed Design

### ১. URL এনকোডিং স্ট্র্যাটেজি

শর্ট কোড জেনারেট করা হলো পুরো সিস্টেমের সবচেয়ে গুরুত্বপূর্ণ অংশ। কারণ কোড ইউনিক হতে হবে, ছোট হতে হবে, আর দ্রুত জেনারেট হতে হবে।

#### Base62 এনকোডিং

Base62 মানে হলো `[a-zA-Z0-9]` — মোট 62টি ক্যারেক্টার। কোনো special character নেই, তাই URL-safe।

```
৭ ক্যারেক্টারের Base62 কোডে কতগুলো ইউনিক URL সম্ভব?
62^7 = 3,521,614,606,208 (3.5 ট্রিলিয়ন!)

এটা আমাদের ৫ বছরের 1.825B URL-এর চেয়ে ~1,900 গুণ বেশি।
তাই ৭ ক্যারেক্টারই যথেষ্ট।
```

#### বিভিন্ন এনকোডিং পদ্ধতির তুলনা

| পদ্ধতি | কিভাবে কাজ করে | সুবিধা | অসুবিধা |
|--------|----------------|--------|---------|
| **Auto-increment + Base62** | DB auto-increment ID → Base62 encode | সিম্পল, কোনো কলিশন নেই | পরবর্তী URL অনুমানযোগ্য, সিকুয়েন্শিয়াল |
| **MD5/SHA256 + Truncate** | URL-এর hash → প্রথম ৭ ক্যারেক্টার | ডিস্ট্রিবিউটেড ফ্রেন্ডলি | কলিশনের সম্ভাবনা আছে |
| **UUID-based** | UUID জেনারেট → Base62 encode → ৭ char truncate | সম্পূর্ণ র‍্যান্ডম | কলিশন হতে পারে, লম্বা |
| **Snowflake ID** | Twitter-style unique ID → Base62 encode | ইউনিক, টাইম-অর্ডার্ড, ডিস্ট্রিবিউটেড | জটিল সেটআপ |
| **Counter Range (Pre-allocated)** | প্রতি সার্ভারকে একটা ID range দেওয়া | কলিশন-মুক্ত, ফাস্ট | রেঞ্জ ম্যানেজমেন্ট দরকার |

#### আমাদের পছন্দ: Counter Range + Base62

এই পদ্ধতিতে একটা **Counter Service** (ZooKeeper বা Redis) প্রতিটি অ্যাপ সার্ভারকে একটা রেঞ্জ বরাদ্দ দেয়:

```
Server-1: 1 - 1,000,000       (10 লক্ষ ID)
Server-2: 1,000,001 - 2,000,000
Server-3: 2,000,001 - 3,000,000
...

প্রতিটি সার্ভার তার রেঞ্জ থেকে sequentially ID নেয়
→ Base62 encode করে → শর্ট কোড পায়

সুবিধা:
✅ কোনো কলিশন নেই (প্রতি সার্ভারের আলাদা রেঞ্জ)
✅ ডাটাবেজে কোনো coordination লাগে না
✅ অত্যন্ত দ্রুত (in-memory counter)
✅ স্কেলেবল (নতুন সার্ভার = নতুন রেঞ্জ)
```

```php
// PHP (Laravel) — Base62 এনকোডিং
class Base62Encoder
{
    private const CHARSET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

    public static function encode(int $number): string
    {
        if ($number === 0) return '0';

        $result = '';
        $base = strlen(self::CHARSET); // 62

        while ($number > 0) {
            $remainder = $number % $base;
            $result = self::CHARSET[$remainder] . $result;
            $number = intdiv($number, $base);
        }

        // ৭ ক্যারেক্টারে প্যাড করা (sho.rt/0000abc → sho.rt/abc হবে না)
        return str_pad($result, 7, '0', STR_PAD_LEFT);
    }

    public static function decode(string $encoded): int
    {
        $number = 0;
        $base = strlen(self::CHARSET);

        for ($i = 0; $i < strlen($encoded); $i++) {
            $char = $encoded[$i];
            $position = strpos(self::CHARSET, $char);

            if ($position === false) {
                throw new \InvalidArgumentException("অবৈধ ক্যারেক্টার: {$char}");
            }

            $number = $number * $base + $position;
        }

        return $number;
    }
}

// ব্যবহার
echo Base62Encoder::encode(123456789);  // "8m0Kx" → "008m0Kx" (প্যাডেড)
echo Base62Encoder::decode('008m0Kx');  // 123456789
```

```javascript
// Node.js — Base62 এনকোডিং
const CHARSET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
const BASE = BigInt(CHARSET.length); // 62n

function base62Encode(num) {
    let n = BigInt(num);
    if (n === 0n) return '0'.padStart(7, '0');

    let result = '';
    while (n > 0n) {
        const remainder = Number(n % BASE);
        result = CHARSET[remainder] + result;
        n = n / BASE;
    }

    return result.padStart(7, '0');
}

function base62Decode(str) {
    let result = 0n;
    for (const char of str) {
        const pos = BigInt(CHARSET.indexOf(char));
        if (pos === -1n) throw new Error(`অবৈধ ক্যারেক্টার: ${char}`);
        result = result * BASE + pos;
    }
    return result;
}

// ব্যবহার
console.log(base62Encode(123456789));   // "008m0Kx"
console.log(base62Decode('008m0Kx'));   // 123456789n
```

---

### ২. ডাটাবেজ স্কিমা

#### PostgreSQL (প্রাইমারি ডাটা স্টোর)

```sql
-- ইউজার টেবিল
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    api_key         VARCHAR(64) UNIQUE NOT NULL,
    tier            VARCHAR(20) DEFAULT 'free',     -- free, pro, enterprise
    daily_limit     INT DEFAULT 100,                -- bKash merchant: unlimited
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_api_key ON users(api_key);

-- URL টেবিল (মূল টেবিল)
CREATE TABLE urls (
    id              BIGSERIAL PRIMARY KEY,
    short_code      VARCHAR(15) UNIQUE NOT NULL,    -- Base62 কোড বা কাস্টম অ্যালিয়াস
    original_url    TEXT NOT NULL,
    user_id         BIGINT REFERENCES users(id),
    is_custom       BOOLEAN DEFAULT FALSE,          -- কাস্টম অ্যালিয়াস কিনা
    click_count     BIGINT DEFAULT 0,               -- ডিনরমালাইজড কাউন্টার (ফাস্ট রিড)
    expires_at      TIMESTAMPTZ,                    -- NULL = কখনো expire হবে না
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- সবচেয়ে গুরুত্বপূর্ণ ইনডেক্স — রিডাইরেক্টের সময় এটা ব্যবহার হয়
CREATE UNIQUE INDEX idx_urls_short_code ON urls(short_code) WHERE is_active = TRUE;
CREATE INDEX idx_urls_user_id ON urls(user_id);
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;

-- অ্যানালিটিক্স টেবিল (প্রতিটি ক্লিকের তথ্য)
-- এই টেবিল অনেক বড় হবে — পার্টিশনিং জরুরি
CREATE TABLE click_events (
    id              BIGSERIAL,
    short_code      VARCHAR(15) NOT NULL,
    clicked_at      TIMESTAMPTZ DEFAULT NOW(),
    ip_hash         VARCHAR(64),                    -- Privacy: পূর্ণ IP রাখি না
    country_code    CHAR(2),                        -- 'BD', 'US', etc.
    city            VARCHAR(100),
    device_type     VARCHAR(20),                    -- mobile, desktop, tablet
    browser         VARCHAR(50),
    os              VARCHAR(50),
    referrer_domain VARCHAR(255),
    PRIMARY KEY (id, clicked_at)
) PARTITION BY RANGE (clicked_at);

-- মাসিক পার্টিশন তৈরি (প্রতি মাসের ডাটা আলাদা)
CREATE TABLE click_events_2024_01 PARTITION OF click_events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE click_events_2024_02 PARTITION OF click_events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE INDEX idx_clicks_short_code ON click_events(short_code, clicked_at);
```

#### NoSQL বিকল্প (DynamoDB / Cassandra)

বিলিয়ন স্কেলে PostgreSQL-এ সমস্যা হতে পারে। DynamoDB বিকল্প হিসেবে দারুণ:

```
DynamoDB টেবিল ডিজাইন:

urls টেবিল:
  Partition Key: short_code
  Attributes: original_url, user_id, click_count, expires_at, created_at

click_events টেবিল:
  Partition Key: short_code
  Sort Key: clicked_at
  Attributes: country_code, device_type, browser, referrer_domain

সুবিধা:
  - Partition Key দিয়ে single-digit ms লেটেন্সি
  - Auto-scaling (ঈদের স্পাইকে নিজে নিজেই স্কেল)
  - TTL সাপোর্ট (expires_at → DynamoDB নিজেই ডিলিট করবে)
```

---

### ৩. API ডিজাইন

#### RESTful API এন্ডপয়েন্টস

```
┌──────────┬──────────────────────┬──────────────────────────────────┐
│ মেথড     │ এন্ডপয়েন্ট          │ বিবরণ                            │
├──────────┼──────────────────────┼──────────────────────────────────┤
│ POST     │ /api/v1/shorten      │ URL শর্ট করা                    │
│ GET      │ /:shortCode          │ রিডাইরেক্ট (301/302)           │
│ GET      │ /api/v1/urls/:code   │ URL-এর বিস্তারিত তথ্য          │
│ DELETE   │ /api/v1/urls/:code   │ URL নিষ্ক্রিয় করা              │
│ GET      │ /api/v1/analytics/:c │ অ্যানালিটিক্স ডাটা              │
│ PATCH    │ /api/v1/urls/:code   │ URL আপডেট (expiry, etc.)       │
└──────────┴──────────────────────┴──────────────────────────────────┘
```

#### 301 vs 302 — কোনটা কখন?

এটা একটা গুরুত্বপূর্ণ ডিজাইন ডিসিশন:

```
301 (Moved Permanently):
  → ব্রাউজার ক্যাশ করে রাখে → পরের বার সরাসরি অরিজিনাল URL-এ যায়
  → আমাদের সার্ভারে আর রিকোয়েস্ট আসে না
  → অ্যানালিটিক্স ট্র্যাক করা যায় না!
  ✅ ব্যবহার: স্থায়ী URL যেখানে অ্যানালিটিক্স দরকার নেই

302 (Found / Temporary Redirect):
  → ব্রাউজার ক্যাশ করে না → প্রতিবার আমাদের সার্ভারে আসে
  → প্রতিটি ক্লিক ট্র্যাক করা যায়
  → সার্ভারে বেশি লোড
  ✅ ব্যবহার: bKash প্রমোশন লিংক, যেখানে ক্লিক কাউন্ট দরকার

আমাদের সিদ্ধান্ত:
  - ডিফল্ট: 302 (অ্যানালিটিক্স চাই)
  - অপশনাল: 301 (ইউজার চাইলে সেট করতে পারবে)
```

#### API রিকোয়েস্ট/রেসপন্স উদাহরণ

```json
// POST /api/v1/shorten
// Request:
{
    "url": "https://www.bkash.com/campaign/eid-cashback-2024?ref=sms&utm_source=bulk",
    "custom_alias": "bkash-eid",        // ঐচ্ছিক
    "expires_in": "7d",                  // ঐচ্ছিক — 7 দিন
    "redirect_type": 302                 // ঐচ্ছিক — 301 বা 302
}

// Response (201 Created):
{
    "success": true,
    "data": {
        "short_url": "https://sho.com.bd/bkash-eid",
        "short_code": "bkash-eid",
        "original_url": "https://www.bkash.com/campaign/eid-cashback-2024?ref=sms&utm_source=bulk",
        "expires_at": "2024-04-17T00:00:00Z",
        "created_at": "2024-04-10T12:00:00Z"
    }
}

// GET /api/v1/analytics/bkash-eid
// Response:
{
    "short_code": "bkash-eid",
    "total_clicks": 1458923,
    "unique_visitors": 892341,
    "created_at": "2024-04-10T12:00:00Z",
    "top_countries": [
        { "code": "BD", "clicks": 1420000, "percentage": 97.3 },
        { "code": "US", "clicks": 15000, "percentage": 1.0 }
    ],
    "devices": {
        "mobile": 89.2,
        "desktop": 8.5,
        "tablet": 2.3
    },
    "clicks_by_day": [
        { "date": "2024-04-10", "clicks": 523000 },
        { "date": "2024-04-11", "clicks": 412000 }
    ],
    "top_referrers": [
        { "domain": "facebook.com", "clicks": 650000 },
        { "domain": "direct", "clicks": 430000 }
    ]
}
```

---

### ৪. Read Path (রিডাইরেক্ট ফ্লো)

রিডাইরেক্ট হলো সবচেয়ে ক্রিটিক্যাল পাথ — এটা প্রতি সেকেন্ডে হাজার হাজার বার হয়। এখানে প্রতিটি মিলিসেকেন্ড গুরুত্বপূর্ণ।

```
    User ক্লিক: sho.com.bd/bkash-eid
                    │
                    ▼
    ┌───────────────────────────────┐
    │     Load Balancer (Nginx)     │
    │  - Rate limit check           │
    │  - Forward to Node.js         │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │     Node.js Read Service      │
    │                               │
    │  1. short_code = "bkash-eid"  │
    │     extract from URL path     │
    │                               │
    │  2. Redis GET url:bkash-eid   │──── HIT ──┐
    │     (ক্যাশে আছে?)            │           │
    │           │                   │           │
    │         MISS                  │           │
    │           │                   │           │
    │  3. PostgreSQL SELECT         │           │
    │     WHERE short_code =        │           │
    │     'bkash-eid'               │           │
    │     AND is_active = TRUE      │           │
    │           │                   │           │
    │      ┌────┴────┐              │           │
    │    Found    Not Found         │           │
    │      │         │              │           │
    │      │     404 Error          │           │
    │      │                        │           │
    │  4. expires_at চেক            │           │
    │     (মেয়াদ শেষ?)             │           │
    │      │                        │           │
    │  ┌───┴───┐                    │           │
    │ Valid  Expired                │           │
    │  │       │                    │           │
    │  │    410 Gone                │           │
    │  │                            │           │
    │  5. Redis SET (ক্যাশে রাখো)   │           │
    │     TTL: 24 hours             │           │
    │           │                   │           │
    │           ├───────────────────┘           │
    │           │                               │
    │  6. Analytics Event (Async)   │           │
    │     Redis INCR clicks:bkash-  │◄──────────┘
    │     Kafka publish click event │
    │           │                   │
    │  7. HTTP 302 Redirect         │
    │     Location: original_url    │
    └───────────────────────────────┘
```

```javascript
// Node.js — Read Path ইমপ্লিমেন্টেশন
const express = require('express');
const Redis = require('ioredis');
const { Pool } = require('pg');

const redis = new Redis({ host: 'redis-cluster', port: 6379 });
const db = new Pool({ connectionString: process.env.DATABASE_URL });

async function redirectHandler(req, res) {
    const { shortCode } = req.params;
    const cacheKey = `url:${shortCode}`;

    try {
        // ধাপ ১: Redis ক্যাশ চেক
        let cached = await redis.get(cacheKey);

        if (cached) {
            const urlData = JSON.parse(cached);

            // মেয়াদ চেক (ক্যাশে থাকা ডাটায়)
            if (urlData.expires_at && new Date(urlData.expires_at) < new Date()) {
                await redis.del(cacheKey);
                return res.status(410).json({ error: 'এই লিংকের মেয়াদ শেষ হয়ে গেছে' });
            }

            // অ্যাসিঙ্ক্রোনাস অ্যানালিটিক্স — রেসপন্স ব্লক করবে না
            trackClick(shortCode, req).catch(console.error);

            return res.redirect(urlData.redirect_type || 302, urlData.original_url);
        }

        // ধাপ ২: ডাটাবেজ থেকে খোঁজো
        const result = await db.query(
            `SELECT original_url, expires_at, redirect_type
             FROM urls
             WHERE short_code = $1 AND is_active = TRUE`,
            [shortCode]
        );

        if (result.rows.length === 0) {
            return res.status(404).json({ error: 'শর্ট URL পাওয়া যায়নি' });
        }

        const urlData = result.rows[0];

        // মেয়াদ চেক
        if (urlData.expires_at && new Date(urlData.expires_at) < new Date()) {
            return res.status(410).json({ error: 'এই লিংকের মেয়াদ শেষ হয়ে গেছে' });
        }

        // ধাপ ৩: ক্যাশে সেট করো (24 ঘণ্টার TTL)
        await redis.setex(cacheKey, 86400, JSON.stringify(urlData));

        // অ্যানালিটিক্স ট্র্যাক
        trackClick(shortCode, req).catch(console.error);

        return res.redirect(urlData.redirect_type || 302, urlData.original_url);

    } catch (error) {
        console.error('রিডাইরেক্ট এরর:', error);
        return res.status(500).json({ error: 'সার্ভার সমস্যা' });
    }
}

async function trackClick(shortCode, req) {
    const pipeline = redis.pipeline();

    // ক্লিক কাউন্ট বাড়াও (অ্যাটমিক)
    pipeline.incr(`clicks:${shortCode}`);
    // আজকের ক্লিক আলাদাভাবে
    pipeline.incr(`clicks:${shortCode}:${new Date().toISOString().slice(0, 10)}`);

    await pipeline.exec();

    // বিস্তারিত ইভেন্ট Kafka-তে পাঠাও (বা Redis Stream-এ)
    const event = {
        short_code: shortCode,
        timestamp: new Date().toISOString(),
        ip_hash: hashIP(req.ip),
        user_agent: req.headers['user-agent'],
        referrer: req.headers['referer'] || 'direct',
        country: req.headers['cf-ipcountry'] || 'XX', // Cloudflare হেডার
    };

    await redis.xadd('click_stream', '*',
        'data', JSON.stringify(event)
    );
}
```

---

### ৫. Write Path (শর্টেন ফ্লো)

```
    User: POST /api/v1/shorten
    Body: { url: "https://bkash.com/...", custom_alias: "bkash-eid" }
                    │
                    ▼
    ┌───────────────────────────────────┐
    │     Laravel Write Service         │
    │                                   │
    │  1. ভ্যালিডেশন                    │
    │     - URL ফরম্যাট ঠিক আছে?       │
    │     - ম্যালিশাস URL চেক           │
    │     - রেট লিমিট চেক              │
    │           │                       │
    │     ┌─────┴─────┐                 │
    │   Valid      Invalid              │
    │     │           │                 │
    │     │       422 Error             │
    │     │                             │
    │  2. কাস্টম অ্যালিয়াস আছে?        │
    │     ┌─────┴─────┐                 │
    │    হ্যাঁ          না               │
    │     │           │                 │
    │  3a. অ্যালিয়াস  3b. Counter      │
    │   ইউনিক চেক     থেকে নতুন ID    │
    │     │           │                 │
    │  ┌──┴──┐     Base62 Encode       │
    │ Free  Taken     │                │
    │  │      │       │                │
    │  │   409 Error  │                │
    │  │              │                │
    │  4. ডুপ্লিকেট URL চেক            │
    │     (একই URL আগে শর্ট হয়েছে?)   │
    │     ┌─────┴─────┐                │
    │   Exists     New                 │
    │     │           │                │
    │  Return      5. DB INSERT        │
    │  existing       │                │
    │                 │                │
    │  6. Redis ক্যাশে সেট              │
    │                 │                │
    │  7. 201 Created Response         │
    └───────────────────────────────────┘
```

```php
// PHP (Laravel) — Write Path ইমপ্লিমেন্টেশন

// app/Http/Controllers/UrlController.php
namespace App\Http\Controllers;

use App\Models\Url;
use App\Services\ShortCodeGenerator;
use App\Services\UrlValidator;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\RateLimiter;

class UrlController extends Controller
{
    public function __construct(
        private ShortCodeGenerator $codeGenerator,
        private UrlValidator $validator
    ) {}

    public function shorten(Request $request)
    {
        // ধাপ ১: ইনপুট ভ্যালিডেশন
        $validated = $request->validate([
            'url'           => 'required|url|max:2048',
            'custom_alias'  => 'nullable|string|min:3|max:30|regex:/^[a-zA-Z0-9_-]+$/',
            'expires_in'    => 'nullable|string|in:1h,1d,7d,30d,1y',
            'redirect_type' => 'nullable|integer|in:301,302',
        ]);

        // ধাপ ২: ম্যালিশাস URL চেক
        if ($this->validator->isMalicious($validated['url'])) {
            return response()->json([
                'error' => 'এই URL নিরাপদ নয়'
            ], 422);
        }

        // ধাপ ৩: রেট লিমিট চেক
        $userId = $request->user()?->id ?? $request->ip();
        if (RateLimiter::tooManyAttempts("shorten:{$userId}", 100)) {
            return response()->json([
                'error' => 'অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন'
            ], 429);
        }
        RateLimiter::hit("shorten:{$userId}", 3600);

        // ধাপ ৪: ডুপ্লিকেট চেক (একই URL আগে শর্ট করা হয়েছে কিনা)
        if (empty($validated['custom_alias'])) {
            $existing = Url::where('original_url', $validated['url'])
                ->where('user_id', $request->user()?->id)
                ->where('is_active', true)
                ->first();

            if ($existing) {
                return response()->json([
                    'success' => true,
                    'data' => $this->formatUrlResponse($existing),
                    'message' => 'এই URL আগেই শর্ট করা হয়েছে'
                ]);
            }
        }

        // ধাপ ৫: শর্ট কোড জেনারেট
        $shortCode = $validated['custom_alias']
            ?? $this->codeGenerator->generate();

        // ধাপ ৬: ডাটাবেজে সেভ (ট্রানজ্যাকশন ব্যবহার)
        try {
            $url = DB::transaction(function () use ($validated, $shortCode, $request) {
                return Url::create([
                    'short_code'    => $shortCode,
                    'original_url'  => $validated['url'],
                    'user_id'       => $request->user()?->id,
                    'is_custom'     => !empty($validated['custom_alias']),
                    'redirect_type' => $validated['redirect_type'] ?? 302,
                    'expires_at'    => $this->calculateExpiry($validated['expires_in'] ?? null),
                ]);
            });
        } catch (\Illuminate\Database\UniqueConstraintViolationException $e) {
            // কাস্টম অ্যালিয়াসে কলিশন
            return response()->json([
                'error' => "'{$shortCode}' ইতিমধ্যে ব্যবহৃত হয়েছে। অন্য একটি বেছে নিন।"
            ], 409);
        }

        // ধাপ ৭: ক্যাশে প্রি-ওয়ার্ম (নতুন URL এখনই ক্যাশে রাখো)
        Cache::store('redis')->put(
            "url:{$shortCode}",
            json_encode([
                'original_url'  => $url->original_url,
                'expires_at'    => $url->expires_at?->toISOString(),
                'redirect_type' => $url->redirect_type,
            ]),
            now()->addDay()
        );

        return response()->json([
            'success' => true,
            'data' => $this->formatUrlResponse($url),
        ], 201);
    }

    private function calculateExpiry(?string $duration): ?\Carbon\Carbon
    {
        return match ($duration) {
            '1h'    => now()->addHour(),
            '1d'    => now()->addDay(),
            '7d'    => now()->addDays(7),
            '30d'   => now()->addDays(30),
            '1y'    => now()->addYear(),
            default => null, // কখনো expire হবে না
        };
    }

    private function formatUrlResponse(Url $url): array
    {
        return [
            'short_url'     => config('app.short_domain') . '/' . $url->short_code,
            'short_code'    => $url->short_code,
            'original_url'  => $url->original_url,
            'expires_at'    => $url->expires_at?->toISOString(),
            'created_at'    => $url->created_at->toISOString(),
            'click_count'   => $url->click_count,
        ];
    }
}
```

```php
// app/Services/ShortCodeGenerator.php
namespace App\Services;

use App\Helpers\Base62Encoder;
use Illuminate\Support\Facades\Redis;

class ShortCodeGenerator
{
    private const RANGE_SIZE = 1_000_000;
    private const COUNTER_KEY = 'url_counter:current';
    private const RANGE_KEY = 'url_counter:range_end';

    /**
     * Counter Range পদ্ধতি — প্রতিটি সার্ভার ইনস্ট্যান্স একটা রেঞ্জ পায়
     * কোনো কলিশন নেই, কোনো DB lookup নেই
     */
    public function generate(): string
    {
        $counter = Redis::incr(self::COUNTER_KEY);
        $rangeEnd = Redis::get(self::RANGE_KEY);

        // রেঞ্জ শেষ হলে নতুন রেঞ্জ নাও
        if (!$rangeEnd || $counter >= (int) $rangeEnd) {
            $this->allocateNewRange();
            $counter = Redis::incr(self::COUNTER_KEY);
        }

        return Base62Encoder::encode($counter);
    }

    private function allocateNewRange(): void
    {
        // গ্লোবাল কাউন্টার থেকে নতুন রেঞ্জ অ্যালোকেট (atomic)
        $rangeStart = Redis::incrby('global_url_counter', self::RANGE_SIZE);
        Redis::set(self::COUNTER_KEY, $rangeStart - self::RANGE_SIZE);
        Redis::set(self::RANGE_KEY, $rangeStart);
    }
}
```

---

### ৬. ক্যাশিং স্ট্র্যাটেজি

Redis হলো আমাদের সিস্টেমের "সুপারপাওয়ার" — 80-90% রিকোয়েস্ট ক্যাশ থেকেই সার্ভ হবে।

#### Cache-Aside প্যাটার্ন

```
Cache-Aside (Lazy Loading) — আমাদের মূল প্যাটার্ন:

READ:
  1. ক্যাশে খোঁজো (Redis GET)
  2. পেলে → সরাসরি ব্যবহার করো (ক্যাশ হিট)
  3. না পেলে → DB থেকে পড়ো → ক্যাশে রাখো → ব্যবহার করো (ক্যাশ মিস)

WRITE:
  1. DB-তে লেখো
  2. ক্যাশে নতুন ডাটা সেট করো (Write-through)

DELETE/UPDATE:
  1. DB আপডেট করো
  2. ক্যাশ invalidate করো (ডিলিট)

কেন Cache-Aside?
  - সিম্পল ইমপ্লিমেন্টেশন
  - শুধু যেসব URL অ্যাক্সেস হয় সেগুলোই ক্যাশে থাকে
  - ক্যাশ ফেইলিওর হলেও সিস্টেম কাজ করে (DB ফলব্যাক)
```

#### Redis ডাটা স্ট্রাকচার

```
Key Design:

url:{shortCode}          → URL ডাটা (String/JSON)
  TTL: 24 hours
  Value: {"original_url":"...","expires_at":"...","redirect_type":302}

clicks:{shortCode}       → মোট ক্লিক কাউন্ট (Counter)
  TTL: none (persistent)

clicks:{shortCode}:{date} → দৈনিক ক্লিক (Counter)
  TTL: 90 days

bloom:urls               → Bloom Filter (কলিশন চেক)
  Probabilistic data structure

ratelimit:{userId}       → রেট লিমিট কাউন্টার
  TTL: 1 hour

Eviction Policy: allkeys-lru
  যখন মেমোরি ফুল হয়ে যায়, সবচেয়ে পুরোনো অ্যাক্সেসড key মুছে যায়
  জনপ্রিয় URL (bKash অফার) সবসময় ক্যাশে থাকবে
```

```javascript
// Node.js — Redis ক্যাশ লেয়ার (প্রোডাকশন-রেডি)
const Redis = require('ioredis');

class UrlCache {
    constructor() {
        // Redis Cluster — হাই অ্যাভেইলেবিলিটির জন্য
        this.redis = new Redis.Cluster([
            { host: 'redis-1', port: 6379 },
            { host: 'redis-2', port: 6379 },
            { host: 'redis-3', port: 6379 },
        ], {
            redisOptions: {
                password: process.env.REDIS_PASSWORD,
                maxRetriesPerRequest: 3,
            },
            scaleReads: 'slave', // রিড অপারেশন slave থেকে
        });
    }

    async getUrl(shortCode) {
        const data = await this.redis.get(`url:${shortCode}`);
        if (!data) return null;

        const parsed = JSON.parse(data);

        // মেয়াদ চেক ক্যাশ লেভেলেই
        if (parsed.expires_at && new Date(parsed.expires_at) < new Date()) {
            await this.redis.del(`url:${shortCode}`);
            return { expired: true };
        }

        return parsed;
    }

    async setUrl(shortCode, urlData, ttlSeconds = 86400) {
        await this.redis.setex(
            `url:${shortCode}`,
            ttlSeconds,
            JSON.stringify(urlData)
        );
    }

    async incrementClicks(shortCode) {
        const today = new Date().toISOString().slice(0, 10);
        const pipeline = this.redis.pipeline();

        pipeline.incr(`clicks:${shortCode}`);
        pipeline.incr(`clicks:${shortCode}:${today}`);
        // দৈনিক কাউন্টারের TTL: 90 দিন
        pipeline.expire(`clicks:${shortCode}:${today}`, 86400 * 90);

        return pipeline.exec();
    }

    async getClickCount(shortCode) {
        return parseInt(await this.redis.get(`clicks:${shortCode}`) || '0');
    }

    async invalidate(shortCode) {
        await this.redis.del(`url:${shortCode}`);
    }
}

module.exports = new UrlCache();
```

---

### ৭. অ্যানালিটিক্স সিস্টেম

অ্যানালিটিক্স হলো URL শর্টনারের সবচেয়ে ভ্যালুয়েবল ফিচার। bKash যখন একটা প্রমোশনাল লিংক ছাড়ে, তারা জানতে চায় — কত লোক ক্লিক করেছে, কোন জেলা থেকে, কোন ফোন দিয়ে?

#### অ্যানালিটিক্স পাইপলাইন

```
    Click Event
        │
        ▼
    ┌─────────────────┐
    │  Redis Counter   │  ← তাৎক্ষণিক কাউন্ট (INCR)
    │  INCR clicks:x   │    এটা real-time — ক্লিক হওয়ার সাথে সাথেই বাড়ে
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │  Redis Stream    │  ← বিস্তারিত ইভেন্ট ডাটা
    │  XADD click_     │    geo, device, referrer info
    │  stream          │
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │  Consumer Worker │  ← ব্যাকগ্রাউন্ড প্রসেসর
    │  (Node.js)       │    প্রতি 5 সেকেন্ডে ব্যাচ রিড
    └────────┬────────┘
             │
    ┌────────▼────────────────┐
    │  ClickHouse / TimescaleDB│  ← অ্যানালিটিক্স DB
    │  (Columnar Storage)      │    বিলিয়ন রো-তেও ফাস্ট
    │                          │    aggregation query
    └────────┬────────────────┘
             │
    ┌────────▼────────┐
    │  API Response    │  ← ড্যাশবোর্ডে দেখানো
    │  (Aggregated)    │
    └─────────────────┘

কেন Redis → ClickHouse?
  Redis-এ সব ক্লিক ডাটা রাখা অসম্ভব (মেমোরি সীমিত)।
  ClickHouse columnar DB — বিলিয়ন রো aggregation মিলিসেকেন্ডে করে।
  Redis শুধু কাউন্টার ও বাফার হিসেবে কাজ করে।
```

```php
// PHP (Laravel) — অ্যানালিটিক্স Consumer (চলবে Queue Worker হিসেবে)

// app/Jobs/ProcessClickEvents.php
namespace App\Jobs;

use App\Models\ClickEvent;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;

class ProcessClickEvents implements ShouldQueue
{
    use Queueable;

    public function handle(): void
    {
        // Redis Stream থেকে ব্যাচে ইভেন্ট পড়ো
        $events = Redis::xrange('click_stream', '-', '+', 1000);

        if (empty($events)) return;

        $clickRows = [];
        $processedIds = [];

        foreach ($events as $id => $fields) {
            $data = json_decode($fields['data'], true);
            $processedIds[] = $id;

            // User-Agent পার্সিং
            $deviceInfo = $this->parseUserAgent($data['user_agent'] ?? '');

            $clickRows[] = [
                'short_code'      => $data['short_code'],
                'clicked_at'      => $data['timestamp'],
                'ip_hash'         => $data['ip_hash'],
                'country_code'    => $data['country'] ?? 'XX',
                'device_type'     => $deviceInfo['type'],
                'browser'         => $deviceInfo['browser'],
                'os'              => $deviceInfo['os'],
                'referrer_domain' => $this->extractDomain($data['referrer'] ?? ''),
            ];
        }

        // ব্যাচ INSERT (একটা একটা করার চেয়ে অনেক দ্রুত)
        DB::table('click_events')->insert($clickRows);

        // DB-তে click_count আপডেট (ব্যাচে)
        $clickCounts = collect($clickRows)
            ->groupBy('short_code')
            ->map->count();

        foreach ($clickCounts as $shortCode => $count) {
            DB::table('urls')
                ->where('short_code', $shortCode)
                ->increment('click_count', $count);
        }

        // প্রসেস করা ইভেন্ট মুছে ফেলো
        foreach ($processedIds as $id) {
            Redis::xdel('click_stream', $id);
        }
    }

    private function parseUserAgent(string $ua): array
    {
        $type = 'unknown';
        if (preg_match('/Mobile|Android|iPhone/i', $ua)) $type = 'mobile';
        elseif (preg_match('/Tablet|iPad/i', $ua)) $type = 'tablet';
        elseif (!empty($ua)) $type = 'desktop';

        $browser = 'unknown';
        if (str_contains($ua, 'Chrome')) $browser = 'Chrome';
        elseif (str_contains($ua, 'Firefox')) $browser = 'Firefox';
        elseif (str_contains($ua, 'Safari')) $browser = 'Safari';

        $os = 'unknown';
        if (str_contains($ua, 'Android')) $os = 'Android';
        elseif (str_contains($ua, 'iPhone') || str_contains($ua, 'iOS')) $os = 'iOS';
        elseif (str_contains($ua, 'Windows')) $os = 'Windows';
        elseif (str_contains($ua, 'Linux')) $os = 'Linux';

        return compact('type', 'browser', 'os');
    }

    private function extractDomain(string $url): string
    {
        if ($url === 'direct' || empty($url)) return 'direct';
        return parse_url($url, PHP_URL_HOST) ?? 'unknown';
    }
}
```

---

## 🔥 Advanced Topics

### ১. কলিশন হ্যান্ডলিং

কলিশন মানে — দুইটা আলাদা URL-এর জন্য একই শর্ট কোড তৈরি হওয়া। Counter Range পদ্ধতিতে কলিশন হয় না, তবে হ্যাশ-বেজড পদ্ধতিতে হতে পারে।

#### Bloom Filter দিয়ে দ্রুত কলিশন চেক

Bloom Filter হলো একটা probabilistic data structure যেটা বলতে পারে "এই কোড **হয়তো** আছে" অথবা "**নিশ্চিতভাবে** নেই"।

```
Bloom Filter কিভাবে কাজ করে:

1. নতুন কোড "abc123" চেক করতে চাই
2. Bloom Filter বলে "নেই" → নিশ্চিত! ব্যবহার করো
3. Bloom Filter বলে "হয়তো আছে" → DB-তে গিয়ে নিশ্চিত হও

False Positive Rate: ~1% (কনফিগারেবল)
মেমোরি: 1 বিলিয়ন URL-এর জন্য মাত্র ~1.2 GB

DB query লাগবে শুধু 1% ক্ষেত্রে (Bloom Filter "maybe" বললে)
বাকি 99% শুধু in-memory check!
```

```php
// PHP — Bloom Filter দিয়ে কলিশন হ্যান্ডলিং (Redis ব্যবহার করে)
namespace App\Services;

use Illuminate\Support\Facades\Redis;

class CollisionHandler
{
    private const BLOOM_KEY = 'bloom:short_codes';
    private const MAX_RETRIES = 5;

    /**
     * Hash-based পদ্ধতিতে কলিশন-ফ্রি কোড জেনারেট
     */
    public function generateUniqueCode(string $originalUrl): string
    {
        for ($attempt = 0; $attempt < self::MAX_RETRIES; $attempt++) {
            // প্রতিটি attempt-এ আলাদা seed যোগ করি
            $input = $originalUrl . '|' . $attempt . '|' . microtime(true);
            $hash = md5($input);
            $code = $this->hashToBase62(substr($hash, 0, 12));

            // ধাপ ১: Bloom Filter চেক (99% ক্ষেত্রে এখানেই শেষ)
            if (!$this->bloomMightExist($code)) {
                $this->bloomAdd($code);
                return $code;
            }

            // ধাপ ২: Bloom Filter "maybe" বললে DB চেক (1% ক্ষেত্রে)
            $existsInDb = \App\Models\Url::where('short_code', $code)->exists();

            if (!$existsInDb) {
                // False positive ছিল — কোড আসলে নেই
                return $code;
            }

            // সত্যিই কলিশন — পরবর্তী attempt-এ যাও
        }

        throw new \RuntimeException('কলিশন সমাধান ব্যর্থ। ' . self::MAX_RETRIES . ' বার চেষ্টা করা হয়েছে।');
    }

    private function bloomMightExist(string $code): bool
    {
        // Redis-এর BF.EXISTS কমান্ড (RedisBloom মডিউল)
        return (bool) Redis::executeRaw(['BF.EXISTS', self::BLOOM_KEY, $code]);
    }

    private function bloomAdd(string $code): void
    {
        Redis::executeRaw(['BF.ADD', self::BLOOM_KEY, $code]);
    }

    private function hashToBase62(string $hexHash): string
    {
        $decimal = hexdec($hexHash);
        return Base62Encoder::encode($decimal);
    }
}
```

### ২. কাস্টম অ্যালিয়াস ভ্যালিডেশন

```php
// app/Services/UrlValidator.php
namespace App\Services;

class UrlValidator
{
    // নিষিদ্ধ শব্দ (বাংলাদেশ কনটেক্সট সহ)
    private const BLOCKED_WORDS = [
        'admin', 'api', 'static', 'assets', 'health',
        'login', 'signup', 'dashboard', 'analytics',
    ];

    // রিজার্ভড পাথ (সিস্টেম ব্যবহার করে)
    private const RESERVED_PATHS = [
        'api', 'health', 'metrics', 'status', 'robots.txt', 'favicon.ico',
    ];

    public function validateAlias(string $alias): array
    {
        $errors = [];

        // দৈর্ঘ্য চেক
        if (strlen($alias) < 3 || strlen($alias) > 30) {
            $errors[] = 'অ্যালিয়াস ৩-৩০ ক্যারেক্টারের মধ্যে হতে হবে';
        }

        // অনুমোদিত ক্যারেক্টার
        if (!preg_match('/^[a-zA-Z0-9_-]+$/', $alias)) {
            $errors[] = 'শুধু ইংরেজি অক্ষর, সংখ্যা, হাইফেন (-) ও আন্ডারস্কোর (_) ব্যবহার করুন';
        }

        // নিষিদ্ধ শব্দ
        $lowerAlias = strtolower($alias);
        foreach (self::BLOCKED_WORDS as $word) {
            if (str_contains($lowerAlias, $word)) {
                $errors[] = "'{$word}' শব্দটি ব্যবহার করা যাবে না";
                break;
            }
        }

        // রিজার্ভড পাথ
        if (in_array($lowerAlias, self::RESERVED_PATHS)) {
            $errors[] = "এই নামটি সিস্টেমের জন্য সংরক্ষিত";
        }

        return $errors;
    }

    public function isMalicious(string $url): bool
    {
        // পরিচিত ম্যালওয়্যার ডোমেইন চেক (Google Safe Browsing API ব্যবহার করা উচিত)
        $blockedDomains = ['malware.com', 'phishing.net'];
        $host = parse_url($url, PHP_URL_HOST);

        foreach ($blockedDomains as $blocked) {
            if (str_ends_with($host, $blocked)) return true;
        }

        // সন্দেহজনক URL প্যাটার্ন চেক
        if (preg_match('/\.(exe|bat|cmd|scr|vbs|js)$/i', parse_url($url, PHP_URL_PATH) ?? '')) {
            return true;
        }

        return false;
    }
}
```

### ৩. এক্সপিরেশন ও ক্লিনআপ

```php
// app/Console/Commands/CleanExpiredUrls.php — Cron Job
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;

class CleanExpiredUrls extends Command
{
    protected $signature = 'urls:clean-expired';
    protected $description = 'মেয়াদোত্তীর্ণ URL গুলো নিষ্ক্রিয় করো';

    /**
     * প্রতি ঘণ্টায় চলবে: * /1 * * * (cron schedule)
     * ব্যাচে কাজ করে — মিলিয়ন expired URL থাকলেও সমস্যা নেই
     */
    public function handle(): void
    {
        $totalCleaned = 0;
        $batchSize = 1000;

        do {
            // ব্যাচে expired URL খুঁজে is_active = false করো
            $affected = DB::table('urls')
                ->where('is_active', true)
                ->where('expires_at', '<', now())
                ->limit($batchSize)
                ->update(['is_active' => false, 'updated_at' => now()]);

            $totalCleaned += $affected;

            // প্রতিটি ব্যাচের expired URL-এর ক্যাশ invalidate করো
            $expiredCodes = DB::table('urls')
                ->where('is_active', false)
                ->where('updated_at', '>=', now()->subMinute())
                ->pluck('short_code');

            foreach ($expiredCodes as $code) {
                Cache::store('redis')->forget("url:{$code}");
            }

            // CPU-তে চাপ কমাতে একটু বিরতি
            if ($affected === $batchSize) {
                usleep(100000); // 100ms
            }

        } while ($affected === $batchSize);

        $this->info("✅ {$totalCleaned}টি মেয়াদোত্তীর্ণ URL নিষ্ক্রিয় করা হয়েছে");
    }
}

// app/Console/Kernel.php — Schedule রেজিস্ট্রেশন
// $schedule->command('urls:clean-expired')->hourly();
```

### ৪. ডাটাবেজ শার্ডিং

যখন বিলিয়ন URL হবে, একটা PostgreSQL-এ সব রাখা যাবে না। শার্ডিং দরকার।

```
শার্ডিং স্ট্র্যাটেজি: Hash Range

short_code → MD5 hash → প্রথম 2 byte → shard নম্বর (0-255)

উদাহরণ:
  "abc123" → MD5 → "a1b2..." → 0xa1 = 161 → Shard 161

256টি logical shard → 4টি physical DB server:
  Server 1: Shard 0-63     (a-p দিয়ে শুরু)
  Server 2: Shard 64-127   (q-z দিয়ে শুরু)
  Server 3: Shard 128-191  (A-P দিয়ে শুরু)
  Server 4: Shard 192-255  (Q-Z, 0-9 দিয়ে শুরু)

সুবিধা:
  ✅ সমান ডাটা ডিস্ট্রিবিউশন (hash-based)
  ✅ নতুন shard যোগ করা সহজ (consistent hashing)
  ✅ প্রতিটি query শুধু একটা shard-এ যায় (short_code জানা থাকলে)

অসুবিধা:
  ❌ cross-shard query কঠিন (যেমন ইউজারের সব URL)
  ❌ সেটআপ জটিল
```

```javascript
// Node.js — ডাটাবেজ শার্ড রাউটার
const crypto = require('crypto');
const { Pool } = require('pg');

class ShardRouter {
    constructor() {
        this.shards = [
            new Pool({ connectionString: process.env.SHARD_0_URL }),
            new Pool({ connectionString: process.env.SHARD_1_URL }),
            new Pool({ connectionString: process.env.SHARD_2_URL }),
            new Pool({ connectionString: process.env.SHARD_3_URL }),
        ];
    }

    getShardIndex(shortCode) {
        const hash = crypto.createHash('md5').update(shortCode).digest();
        // প্রথম byte → 0-255 → 4 দিয়ে ভাগ → shard 0-3
        return hash[0] % this.shards.length;
    }

    getPool(shortCode) {
        return this.shards[this.getShardIndex(shortCode)];
    }

    async queryUrl(shortCode) {
        const pool = this.getPool(shortCode);
        const result = await pool.query(
            'SELECT original_url, expires_at, redirect_type FROM urls WHERE short_code = $1 AND is_active = TRUE',
            [shortCode]
        );
        return result.rows[0] || null;
    }

    async insertUrl(shortCode, originalUrl, userId, expiresAt) {
        const pool = this.getPool(shortCode);
        return pool.query(
            `INSERT INTO urls (short_code, original_url, user_id, expires_at)
             VALUES ($1, $2, $3, $4) RETURNING *`,
            [shortCode, originalUrl, userId, expiresAt]
        );
    }
}

module.exports = new ShardRouter();
```

### ৫. রেট লিমিটিং

```
রেট লিমিটিং স্ট্র্যাটেজি:

┌────────────┬──────────────────────┬────────────────┐
│ Tier       │ URL শর্টেন (রাইট)    │ রিডাইরেক্ট     │
├────────────┼──────────────────────┼────────────────┤
│ Anonymous  │ 10/hour per IP       │ No limit       │
│ Free       │ 100/day per user     │ No limit       │
│ Pro        │ 10,000/day per user  │ No limit       │
│ Enterprise │ Unlimited            │ No limit       │
│ bKash API  │ 100,000/day          │ No limit       │
└────────────┴──────────────────────┴────────────────┘

রিডাইরেক্টে রেট লিমিট দিই না — কারণ:
  1. রিডাইরেক্ট ব্লক করলে ইউজার এক্সপেরিয়েন্স খারাপ হয়
  2. bKash অফারে লক্ষ ক্লিক আসতে পারে — সব ভ্যালিড
  3. DDoS প্রটেকশন Cloudflare/Nginx লেভেলে করি (L7)
```

```php
// app/Http/Middleware/ApiRateLimiter.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class ApiRateLimiter
{
    public function handle(Request $request, Closure $next)
    {
        $user = $request->user();
        $key = $user
            ? "ratelimit:user:{$user->id}"
            : "ratelimit:ip:{$request->ip()}";

        $limit = $this->getLimit($user);
        $window = $this->getWindow($user);

        // Sliding Window Counter (Redis)
        $current = (int) Redis::get($key);

        if ($current >= $limit) {
            $ttl = Redis::ttl($key);
            return response()->json([
                'error' => 'রেট লিমিট অতিক্রম করেছেন',
                'limit' => $limit,
                'retry_after' => $ttl,
            ], 429)->header('Retry-After', $ttl);
        }

        // আটমিক increment + TTL সেট
        $pipe = Redis::pipeline();
        $pipe->incr($key);
        $pipe->expire($key, $window);
        $pipe->exec();

        // রেসপন্স হেডারে লিমিট ইনফো
        $response = $next($request);
        return $response
            ->header('X-RateLimit-Limit', $limit)
            ->header('X-RateLimit-Remaining', max(0, $limit - $current - 1));
    }

    private function getLimit($user): int
    {
        if (!$user) return 10;

        return match ($user->tier) {
            'free'       => 100,
            'pro'        => 10000,
            'enterprise' => 1000000,
            default      => 10,
        };
    }

    private function getWindow($user): int
    {
        return $user ? 86400 : 3600; // ইউজার: ২৪ ঘণ্টা, anonymous: ১ ঘণ্টা
    }
}
```

### ৬. বিলিয়ন URL-এ স্কেলিং

```
স্কেলিং রোডম্যাপ:

Phase 1: Single Server (0-10M URLs)
  ├── 1x Laravel (Write) + 1x Node.js (Read)
  ├── 1x PostgreSQL
  ├── 1x Redis (6 GB)
  └── ~$50/month (DigitalOcean/Vultr)

Phase 2: Basic Scaling (10M-100M URLs)
  ├── 2x Laravel + 4x Node.js (behind Nginx LB)
  ├── PostgreSQL Primary + 2 Read Replicas
  ├── Redis Sentinel (3 nodes, 8 GB each)
  ├── Cloudflare CDN (edge caching)
  └── ~$500/month

Phase 3: High Scale (100M-1B URLs)
  ├── Auto-scaling group (Laravel + Node.js, 4-20 instances)
  ├── PostgreSQL Sharded (4 shards × Primary + Replica)
  ├── Redis Cluster (6 nodes, 16 GB each)
  ├── Kafka (analytics pipeline)
  ├── ClickHouse (analytics storage)
  └── ~$5,000/month (AWS/GCP)

Phase 4: Massive Scale (1B+ URLs)
  ├── Multi-region deployment (Singapore + Mumbai)
  ├── DynamoDB Global Tables (auto-replication)
  ├── ElastiCache Global Datastore
  ├── CloudFront edge locations
  ├── Dedicated analytics cluster
  └── ~$20,000+/month

বাংলাদেশ অপটিমাইজেশন:
  - Singapore region ব্যবহার (BD-তে সবচেয়ে কম latency)
  - Cloudflare Dhaka PoP ব্যবহার
  - bKash/Nagad webhook integration
  - বাংলা ড্যাশবোর্ড (জেলাভিত্তিক অ্যানালিটিক্স)
```

---

## 💻 Full Implementation

### Laravel প্রজেক্ট সেটআপ

```php
// routes/api.php
use App\Http\Controllers\UrlController;
use App\Http\Controllers\AnalyticsController;

Route::middleware('throttle:api')->group(function () {
    Route::post('/v1/shorten', [UrlController::class, 'shorten']);
    Route::get('/v1/urls/{code}', [UrlController::class, 'show']);
    Route::delete('/v1/urls/{code}', [UrlController::class, 'destroy']);
    Route::patch('/v1/urls/{code}', [UrlController::class, 'update']);
    Route::get('/v1/analytics/{code}', [AnalyticsController::class, 'show']);
    Route::get('/v1/analytics/{code}/timeseries', [AnalyticsController::class, 'timeseries']);
});

// routes/web.php (রিডাইরেক্ট — Node.js ব্যবহার না করলে)
Route::get('/{shortCode}', [UrlController::class, 'redirect'])
    ->where('shortCode', '[a-zA-Z0-9_-]{3,30}');
```

```php
// app/Models/Url.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Url extends Model
{
    protected $fillable = [
        'short_code', 'original_url', 'user_id', 'is_custom',
        'redirect_type', 'expires_at', 'is_active',
    ];

    protected $casts = [
        'expires_at' => 'datetime',
        'is_custom'  => 'boolean',
        'is_active'  => 'boolean',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function clickEvents(): HasMany
    {
        return $this->hasMany(ClickEvent::class, 'short_code', 'short_code');
    }

    public function isExpired(): bool
    {
        return $this->expires_at && $this->expires_at->isPast();
    }

    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeNotExpired($query)
    {
        return $query->where(function ($q) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', now());
        });
    }
}
```

```php
// app/Http/Controllers/AnalyticsController.php
namespace App\Http\Controllers;

use App\Models\Url;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;

class AnalyticsController extends Controller
{
    public function show(string $code)
    {
        $url = Url::where('short_code', $code)->firstOrFail();

        // ক্যাশড অ্যানালিটিক্স (৫ মিনিট)
        $analytics = Cache::remember("analytics:{$code}", 300, function () use ($code, $url) {
            return [
                'short_code'      => $code,
                'original_url'    => $url->original_url,
                'total_clicks'    => $url->click_count,
                'created_at'      => $url->created_at->toISOString(),
                'top_countries'   => $this->getTopCountries($code),
                'devices'         => $this->getDeviceBreakdown($code),
                'top_referrers'   => $this->getTopReferrers($code),
                'clicks_by_day'   => $this->getClicksByDay($code, 30),
            ];
        });

        return response()->json(['success' => true, 'data' => $analytics]);
    }

    public function timeseries(string $code, Request $request)
    {
        $days = min((int) $request->get('days', 7), 90);

        $data = DB::table('click_events')
            ->where('short_code', $code)
            ->where('clicked_at', '>=', now()->subDays($days))
            ->selectRaw("DATE(clicked_at) as date, COUNT(*) as clicks")
            ->groupByRaw("DATE(clicked_at)")
            ->orderBy('date')
            ->get();

        return response()->json(['success' => true, 'data' => $data]);
    }

    private function getTopCountries(string $code): array
    {
        return DB::table('click_events')
            ->where('short_code', $code)
            ->selectRaw("country_code, COUNT(*) as clicks")
            ->groupBy('country_code')
            ->orderByDesc('clicks')
            ->limit(10)
            ->get()
            ->toArray();
    }

    private function getDeviceBreakdown(string $code): array
    {
        $total = DB::table('click_events')
            ->where('short_code', $code)
            ->count();

        if ($total === 0) return [];

        return DB::table('click_events')
            ->where('short_code', $code)
            ->selectRaw("device_type, COUNT(*) as clicks, ROUND(COUNT(*) * 100.0 / ?, 1) as percentage", [$total])
            ->groupBy('device_type')
            ->orderByDesc('clicks')
            ->get()
            ->toArray();
    }

    private function getTopReferrers(string $code): array
    {
        return DB::table('click_events')
            ->where('short_code', $code)
            ->selectRaw("referrer_domain, COUNT(*) as clicks")
            ->groupBy('referrer_domain')
            ->orderByDesc('clicks')
            ->limit(10)
            ->get()
            ->toArray();
    }

    private function getClicksByDay(string $code, int $days): array
    {
        return DB::table('click_events')
            ->where('short_code', $code)
            ->where('clicked_at', '>=', now()->subDays($days))
            ->selectRaw("DATE(clicked_at) as date, COUNT(*) as clicks")
            ->groupByRaw("DATE(clicked_at)")
            ->orderBy('date')
            ->get()
            ->toArray();
    }
}
```

### Node.js Express — সম্পূর্ণ Read Service

```javascript
// server.js — Node.js Read Service (Ultra-fast redirect)
const express = require('express');
const Redis = require('ioredis');
const { Pool } = require('pg');
const crypto = require('crypto');

const app = express();
app.use(express.json());

// ── কানেকশন সেটআপ ─────────────────────────────────────────
const redis = new Redis.Cluster([
    { host: process.env.REDIS_HOST_1 || 'localhost', port: 6379 },
], {
    scaleReads: 'slave',
    redisOptions: { password: process.env.REDIS_PASSWORD, connectTimeout: 5000 },
});

const db = new Pool({
    connectionString: process.env.DATABASE_URL,
    max: 20,
    idleTimeoutMillis: 30000,
});

// ── হেলথ চেক ─────────────────────────────────────────────
app.get('/health', async (req, res) => {
    try {
        await redis.ping();
        await db.query('SELECT 1');
        res.json({ status: 'ok', timestamp: new Date().toISOString() });
    } catch (err) {
        res.status(503).json({ status: 'unhealthy', error: err.message });
    }
});

// ── মেইন রিডাইরেক্ট হ্যান্ডলার ──────────────────────────────
app.get('/:shortCode([a-zA-Z0-9_-]{3,30})', async (req, res) => {
    const { shortCode } = req.params;
    const startTime = Date.now();

    try {
        // ১. Redis ক্যাশ চেক
        const cached = await redis.get(`url:${shortCode}`);

        if (cached) {
            const data = JSON.parse(cached);

            if (data.expires_at && new Date(data.expires_at) < new Date()) {
                redis.del(`url:${shortCode}`).catch(() => {});
                return res.status(410).json({
                    error: 'এই লিংকের মেয়াদ শেষ',
                    expired_at: data.expires_at,
                });
            }

            recordClick(shortCode, req);
            logLatency('cache_hit', startTime);
            return res.redirect(data.redirect_type || 302, data.original_url);
        }

        // ২. DB ফলব্যাক
        const result = await db.query(
            `SELECT original_url, expires_at, redirect_type
             FROM urls WHERE short_code = $1 AND is_active = TRUE`,
            [shortCode]
        );

        if (result.rows.length === 0) {
            return res.status(404).json({ error: 'URL পাওয়া যায়নি' });
        }

        const urlData = result.rows[0];

        if (urlData.expires_at && new Date(urlData.expires_at) < new Date()) {
            return res.status(410).json({ error: 'মেয়াদ শেষ' });
        }

        // ৩. ক্যাশে সেভ
        redis.setex(`url:${shortCode}`, 86400, JSON.stringify(urlData)).catch(() => {});

        recordClick(shortCode, req);
        logLatency('cache_miss', startTime);
        return res.redirect(urlData.redirect_type || 302, urlData.original_url);

    } catch (error) {
        console.error(`[ERROR] Redirect failed for ${shortCode}:`, error.message);
        return res.status(500).json({ error: 'সার্ভার সমস্যা' });
    }
});

// ── অ্যানালিটিক্স রেকর্ডিং (non-blocking) ──────────────────
function recordClick(shortCode, req) {
    const event = {
        short_code: shortCode,
        timestamp: new Date().toISOString(),
        ip_hash: crypto.createHash('sha256').update(req.ip || '').digest('hex').slice(0, 16),
        user_agent: req.headers['user-agent'] || '',
        referrer: req.headers['referer'] || 'direct',
        country: req.headers['cf-ipcountry'] || req.headers['x-country'] || 'XX',
    };

    const pipeline = redis.pipeline();
    pipeline.incr(`clicks:${shortCode}`);
    pipeline.incr(`clicks:${shortCode}:${event.timestamp.slice(0, 10)}`);
    pipeline.expire(`clicks:${shortCode}:${event.timestamp.slice(0, 10)}`, 86400 * 90);
    pipeline.xadd('click_stream', '*', 'data', JSON.stringify(event));
    pipeline.exec().catch(err => console.error('[ANALYTICS ERROR]', err.message));
}

function logLatency(type, startTime) {
    const latency = Date.now() - startTime;
    if (latency > 100) {
        console.warn(`[SLOW] ${type}: ${latency}ms`);
    }
}

// ── সার্ভার স্টার্ট ──────────────────────────────────────────
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`🔗 URL Redirect Service চালু আছে পোর্ট ${PORT}-এ`);
});

module.exports = app;
```

### Laravel Migration

```php
// database/migrations/create_url_shortener_tables.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('urls', function (Blueprint $table) {
            $table->id();
            $table->string('short_code', 15)->unique();
            $table->text('original_url');
            $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();
            $table->boolean('is_custom')->default(false);
            $table->integer('redirect_type')->default(302);
            $table->bigInteger('click_count')->default(0);
            $table->timestamp('expires_at')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();

            $table->index(['user_id', 'created_at']);
            $table->index('expires_at');
        });

        Schema::create('click_events', function (Blueprint $table) {
            $table->id();
            $table->string('short_code', 15)->index();
            $table->timestamp('clicked_at')->useCurrent();
            $table->string('ip_hash', 64)->nullable();
            $table->char('country_code', 2)->nullable();
            $table->string('city', 100)->nullable();
            $table->string('device_type', 20)->nullable();
            $table->string('browser', 50)->nullable();
            $table->string('os', 50)->nullable();
            $table->string('referrer_domain', 255)->nullable();

            $table->index(['short_code', 'clicked_at']);
            $table->index(['clicked_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('click_events');
        Schema::dropIfExists('urls');
    }
};
```

### Docker Compose (ডেভেলপমেন্ট)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Laravel Write Service
  laravel:
    build:
      context: ./laravel-api
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      DB_CONNECTION: pgsql
      DB_HOST: postgres
      DB_DATABASE: url_shortener
      DB_USERNAME: postgres
      DB_PASSWORD: secret
      REDIS_HOST: redis
      SHORT_DOMAIN: "http://localhost:3000"
    depends_on:
      - postgres
      - redis

  # Node.js Read Service
  node-redirect:
    build:
      context: ./node-redirect
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: "postgresql://postgres:secret@postgres:5432/url_shortener"
      REDIS_HOST_1: redis
      REDIS_PASSWORD: ""
    depends_on:
      - postgres
      - redis

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: url_shortener
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru

  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - laravel
      - node-redirect

volumes:
  pgdata:
```

```nginx
# nginx.conf — Write/Read সার্ভিস রাউটিং
events { worker_connections 4096; }

http {
    upstream write_service {
        server laravel:8000;
    }

    upstream read_service {
        server node-redirect:3000;
    }

    server {
        listen 80;
        server_name sho.com.bd;

        # API কল → Laravel (Write Service)
        location /api/ {
            proxy_pass http://write_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # বাকি সব → Node.js (Redirect Service)
        location / {
            proxy_pass http://read_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Country $geoip2_data_country_code;
        }
    }
}
```

---

## 📋 সারসংক্ষেপ

### মূল ডিজাইন ডিসিশন

```
┌───────────────────────┬──────────────────────────────────────┐
│ ডিসিশন               │ পছন্দ ও কারণ                        │
├───────────────────────┼──────────────────────────────────────┤
│ শর্ট কোড জেনারেশন    │ Counter Range + Base62               │
│                       │ (কলিশন-মুক্ত, ডিস্ট্রিবিউটেড)     │
├───────────────────────┼──────────────────────────────────────┤
│ Read/Write বিভাজন     │ Node.js (Read) + Laravel (Write)    │
│                       │ (প্রতিটি নিজ শক্তিতে)              │
├───────────────────────┼──────────────────────────────────────┤
│ ক্যাশ                │ Redis (Cache-Aside, LRU)             │
│                       │ (80-90% হিট রেট)                    │
├───────────────────────┼──────────────────────────────────────┤
│ রিডাইরেক্ট           │ 302 (ডিফল্ট) — অ্যানালিটিক্স চাই    │
├───────────────────────┼──────────────────────────────────────┤
│ অ্যানালিটিক্স        │ Redis Counter → Stream → ClickHouse │
│                       │ (real-time + batch)                  │
├───────────────────────┼──────────────────────────────────────┤
│ শার্ডিং              │ Hash-based (short_code MD5)          │
│                       │ (সমান ডিস্ট্রিবিউশন)               │
├───────────────────────┼──────────────────────────────────────┤
│ এক্সপিরেশন           │ Hourly cron + ক্যাশ TTL             │
│                       │ (ব্যাচে ক্লিনআপ)                   │
└───────────────────────┴──────────────────────────────────────┘
```

### পারফরম্যান্স টার্গেট বনাম অর্জন

```
             টার্গেট          প্রত্যাশিত অর্জন
রিডাইরেক্ট   < 100ms          ~5ms (cache hit), ~30ms (cache miss)
শর্টেন        < 500ms          ~50ms (avg)
ক্যাশ হিট     > 80%            ~85-90%
আপটাইম        99.99%           99.99% (multi-node + failover)
```

### ইন্টারভিউ টিপস

**যখন URL শর্টনার ডিজাইন করতে বলবে, এই ক্রমে উত্তর দিন:**

1. **Requirements clarify করুন** — কত URL/day? কাস্টম অ্যালিয়াস? অ্যানালিটিক্স?
2. **Back-of-envelope** — ট্রাফিক, স্টোরেজ, ক্যাশ সাইজ হিসাব দেখান
3. **High-level architecture** — ASCII ডায়াগ্রাম আঁকুন
4. **শর্ট কোড জেনারেশন** — Base62 + Counter Range ব্যাখ্যা করুন (সবচেয়ে গুরুত্বপূর্ণ!)
5. **Read path** — Redis → DB → Redirect ফ্লো দেখান
6. **Deep dive** — কলিশন, শার্ডিং, অ্যানালিটিক্স পাইপলাইন

> **মনে রাখুন**: URL শর্টনার দেখতে সহজ লাগলেও এটা distributed systems-এর মৌলিক ধারণাগুলো (caching, sharding, rate limiting, async processing) চমৎকারভাবে শেখায়। এই একটা সিস্টেম ভালোভাবে ডিজাইন করতে পারলে — বেশিরভাগ সিস্টেম ডিজাইন ইন্টারভিউতে আত্মবিশ্বাসী থাকবেন। 🚀
