# 📰 নিউজফিড সিস্টেম ডিজাইন (কেস স্টাডি)

> **লক্ষ্য:** বাংলাদেশের জন্য একটি Facebook-সদৃশ নিউজফিড সিস্টেম ডিজাইন করা — যেখানে ৫০ কোটি ব্যবহারকারী বাংলা কনটেন্ট পড়বে, পোস্ট করবে, লাইক-কমেন্ট করবে এবং রিয়েল-টাইমে আপডেট পাবে।

---

## 📌 সেকশন ১: Requirements Analysis

### ফাংশনাল Requirements

| # | Feature | বিবরণ |
|---|---------|--------|
| ১ | **Post Creation** | টেক্সট, ইমেজ, ভিডিও সহ পোস্ট তৈরি |
| ২ | **Feed Generation** | প্রতিটি ব্যবহারকারীর জন্য পার্সোনালাইজড ফিড |
| ৩ | **Follow/Unfollow** | অন্য ব্যবহারকারীকে ফলো/আনফলো করা |
| ৪ | **Likes/Comments** | পোস্টে লাইক ও কমেন্ট করার সুবিধা |
| ৫ | **Media Upload** | ছবি/ভিডিও আপলোড ও প্রসেসিং |
| ৬ | **Real-time Updates** | নতুন পোস্ট তাৎক্ষণিকভাবে ফিডে দেখানো |
| ৭ | **Notifications** | লাইক, কমেন্ট, ফলো ইভেন্টের নোটিফিকেশন |
| ৮ | **Trending** | বাংলাদেশে ট্রেন্ডিং হ্যাশট্যাগ ও টপিক |

### নন-ফাংশনাল Requirements

```
✅ Availability    → ৯৯.৯৯% uptime (বাংলাদেশে সবসময় অ্যাক্সেসযোগ্য)
✅ Latency         → ফিড লোড < ২০০ms (ঢাকা থেকে)
✅ Scalability     → ৫০ কোটি ব্যবহারকারী সাপোর্ট
✅ Consistency     → Eventually consistent (ফিডের জন্য)
✅ Partition Tol.   → Network partition সহ্য করতে পারবে
✅ Bangla Support  → Unicode (UTF-8) বাংলা কনটেন্ট পুরোপুরি সাপোর্ট
```

---

## 📊 সেকশন ২: Capacity Estimation

### ব্যবহারকারী ও ট্র্যাফিক অনুমান

```
মোট ব্যবহারকারী:          ৫০০ মিলিয়ন (৫০ কোটি)
দৈনিক সক্রিয় (DAU):       ২৫০ মিলিয়ন (২৫ কোটি)
প্রতি ব্যবহারকারী ফিড রিড: ১০ বার/দিন
প্রতি ব্যবহারকারী পোস্ট:   ০.৫ টি/দিন (প্রতি ২ জনে ১ পোস্ট)

দৈনিক ফিড রিড:  ২৫০M × ১০ = ২.৫ বিলিয়ন reads/day
                = ২.৫B / ৮৬,৪০০ ≈ ২৯,০০০ reads/sec

দৈনিক পোস্ট:    ২৫০M × ০.৫ = ১২৫ মিলিয়ন posts/day
                = ১২৫M / ৮৬,৪০০ ≈ ১,৪৫০ writes/sec

পিক ট্র্যাফিক (৩× গড়):
  পিক reads:  ~৮৭,০০০/sec
  পিক writes: ~৪,৩৫০/sec
```

### স্টোরেজ অনুমান

```
প্রতিটি পোস্ট (টেক্সট):     ~১ KB (বাংলা UTF-8 বেশি জায়গা নেয়)
প্রতিটি পোস্ট (মিডিয়া সহ):  ~৫০০ KB গড়ে
মেটাডেটা প্রতি পোস্ট:        ~২০০ bytes

দৈনিক স্টোরেজ (টেক্সট):      ১২৫M × ১KB = ১২৫ GB/day
দৈনিক স্টোরেজ (মিডিয়া সহ):  ১২৫M × ৫০০KB = ~৬২ TB/day
বাৎসরিক স্টোরেজ:             ~২২.৫ PB/year (মিডিয়া সহ)

ফিড ক্যাশ (Redis):
  প্রতি ব্যবহারকারী ৫০০টি post ID = ৫০০ × ৮ bytes = ৪ KB
  ২৫০M DAU × ৪ KB = ~১ TB Redis মেমরি
```

### ব্যান্ডউইথ অনুমান

```
ইনকামিং (writes):  ১,৪৫০ × ৫০০KB = ~৭২৫ MB/sec
আউটগোয়িং (reads): ২৯,০০০ × ২MB (ফিড পেজ) = ~৫৮ GB/sec
                    (CDN ক্যাশিং ছাড়া)
CDN ক্যাশ হিট ৯০%: আউটগোয়িং ~৫.৮ GB/sec অরিজিন থেকে
```

---

## 🏗️ সেকশন ৩: High-Level Architecture

```
                        ┌──────────────────────────────────────────┐
                        │              CDN (BunnyCDN/CF)            │
                        │   বাংলাদেশ Edge (ঢাকা, চট্টগ্রাম)        │
                        └──────────────┬───────────────────────────┘
                                       │
                        ┌──────────────▼───────────────────────────┐
                        │          API Gateway / Load Balancer      │
                        │     (Rate Limiting, Auth, Routing)        │
                        └──┬─────┬──────┬──────┬──────┬──────┬─────┘
                           │     │      │      │      │      │
              ┌────────────▼┐ ┌──▼────┐ │  ┌───▼───┐  │  ┌───▼──────┐
              │   Post      │ │ Feed  │ │  │ Media │  │  │Notification│
              │  Service    │ │Service│ │  │Service│  │  │  Service   │
              │  (Laravel)  │ │(Node) │ │  │(Node) │  │  │  (Node)    │
              └──────┬──────┘ └───┬───┘ │  └───┬───┘  │  └─────┬─────┘
                     │            │     │      │      │        │
                     │     ┌──────▼──┐  │      │      │        │
                     │     │  User   │  │      │   ┌──▼─────┐  │
                     │     │ Service │  │      │   │ Search │  │
                     │     │(Laravel)│  │      │   │Service │  │
                     │     └────┬────┘  │      │   └────────┘  │
                     │          │       │      │               │
          ┌──────────▼──────────▼───────▼──────▼───────────────▼──┐
          │              Message Queue (Kafka / RabbitMQ)           │
          │   Topics: post.created, feed.fanout, notification.send │
          └──────┬──────────────┬──────────────┬───────────────────┘
                 │              │              │
          ┌──────▼──────┐ ┌────▼────┐ ┌───────▼──────┐
          │  Feed       │ │Analytics│ │ Moderation   │
          │  Worker     │ │ Worker  │ │   Worker     │
          │  (Fan-out)  │ │(Pipeline│ │ (ML-based)   │
          └──────┬──────┘ └────┬────┘ └──────────────┘
                 │              │
    ┌────────────▼──────────────▼─────────────────────────┐
    │                   Data Layer                         │
    │  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐ │
    │  │PostgreSQL│ │  Redis   │ │Cassandra│ │   S3/    │ │
    │  │ (Posts,  │ │ (Feed   │ │ (Feed  │ │ Object   │ │
    │  │  Users)  │ │  Cache) │ │  Store)│ │ Storage  │ │
    │  └──────────┘ └──────────┘ └────────┘ └──────────┘ │
    └─────────────────────────────────────────────────────┘
```

### সার্ভিস দায়িত্ব বিভাজন

| সার্ভিস | দায়িত্ব | টেকনোলজি |
|----------|---------|-----------|
| **Post Service** | পোস্ট CRUD, ভ্যালিডেশন | Laravel (PHP) |
| **Feed Service** | ফিড জেনারেশন ও ডেলিভারি | Node.js |
| **User Service** | প্রোফাইল, ফলো/আনফলো | Laravel (PHP) |
| **Media Service** | আপলোড, রিসাইজ, ট্রান্সকোড | Node.js + FFmpeg |
| **Notification Service** | পুশ নোটিফিকেশন, ইন-অ্যাপ | Node.js + WebSocket |
| **Search Service** | পোস্ট/ইউজার সার্চ, ট্রেন্ডিং | Elasticsearch |

---

## 💻 সেকশন ৪: Detailed Design

### ১. Feed Generation Strategy

নিউজফিড জেনারেট করার দুটি মূল পদ্ধতি আছে — **Fan-out on Write** এবং **Fan-out on Read**। প্রতিটির সুবিধা-অসুবিধা আছে এবং বাস্তবে হাইব্রিড পদ্ধতি ব্যবহৃত হয়।

#### Fan-out on Write (Push Model)

পোস্ট তৈরির সময়ই সকল ফলোয়ারের ফিডে পোস্ট পুশ করা হয়।

```
ব্যবহারকারী "করিম" পোস্ট করলো
           │
           ▼
    ┌──────────────┐
    │ Post Service  │──▶ DB-তে পোস্ট সেভ
    └──────┬───────┘
           │
           ▼ (Kafka event: post.created)
    ┌──────────────┐
    │ Feed Worker   │
    └──────┬───────┘
           │
           ▼
    করিমের ফলোয়ার লিস্ট পড়ো
    [রহিম, সালমা, তানিয়া, ...]
           │
           ▼  (প্রতিটি ফলোয়ারের জন্য)
    ┌──────────────────────────────────┐
    │  Redis ZADD feed:{user_id}      │
    │    score=timestamp              │
    │    member=post_id               │
    │                                  │
    │  রহিমের ফিড  ← post_id যোগ     │
    │  সালমার ফিড  ← post_id যোগ     │
    │  তানিয়ার ফিড ← post_id যোগ     │
    └──────────────────────────────────┘
```

**সুবিধা:**
- ফিড পড়া অত্যন্ত দ্রুত — শুধু Redis থেকে read
- রিয়েল-টাইম অনুভূতি — পোস্ট তাৎক্ষণিক ফিডে আসে
- Read-heavy সিস্টেমের জন্য আদর্শ (আমাদের read:write = ২০:১)

**অসুবিধা:**
- সেলিব্রিটি সমস্যা — ১ কোটি ফলোয়ার থাকলে ১ কোটি write অপারেশন
- স্টোরেজ বেশি লাগে — প্রতিটি ব্যবহারকারীর জন্য আলাদা ফিড রাখতে হয়
- নিষ্ক্রিয় ব্যবহারকারীদের ফিডও আপডেট হয় — অপচয়

#### Fan-out on Read (Pull Model)

ব্যবহারকারী ফিড দেখতে চাইলে তখনই ফলো-করা মানুষদের পোস্ট থেকে ফিড তৈরি হয়।

```
ব্যবহারকারী "রহিম" ফিড রিকোয়েস্ট করলো
           │
           ▼
    ┌──────────────┐
    │ Feed Service  │
    └──────┬───────┘
           │
           ▼
    রহিম যাদের ফলো করে তাদের লিস্ট পড়ো
    [করিম, সালমা, জাহিদ, ...]
           │
           ▼
    ┌──────────────────────────────────┐
    │  প্রতিটি ফলোয়িং-এর সাম্প্রতিক    │
    │  পোস্ট DB থেকে fetch করো         │
    │                                  │
    │  করিমের পোস্ট  → [p1, p2, p3]   │
    │  সালমার পোস্ট  → [p4, p5]       │
    │  জাহিদের পোস্ট → [p6, p7, p8]   │
    │                                  │
    │  সব merge করো + sort by time    │
    │  → [p6, p1, p4, p7, p2, ...]    │
    └──────────────────────────────────┘
```

**সুবিধা:**
- সেলিব্রিটি সমস্যা নেই — লেখার সময় কোনো fan-out নেই
- স্টোরেজ কম — শুধু পোস্টগুলোই রাখতে হয়
- নিষ্ক্রিয় ব্যবহারকারীদের জন্য কোনো কাজ হয় না

**অসুবিধা:**
- ফিড পড়া ধীর — অনেক query চালাতে হয় রিয়েল-টাইমে
- Read-heavy সিস্টেমে বিপুল লোড তৈরি করে
- কমপ্লেক্স ranking ও merging রিয়েল-টাইমে ব্যয়বহুল

---

### ২. Hybrid Approach (Twitter-স্টাইল)

বাস্তবে Twitter ও Facebook হাইব্রিড পদ্ধতি ব্যবহার করে। ধারণাটি সহজ — সাধারণ ব্যবহারকারীদের জন্য **fan-out on write** এবং সেলিব্রিটিদের জন্য **fan-out on read**।

```
                    পোস্ট তৈরি হলো
                         │
                         ▼
                 ┌───────────────┐
                 │  ফলোয়ার সংখ্যা │
                 │   চেক করো      │
                 └───────┬───────┘
                         │
              ┌──────────┴──────────┐
              │                     │
     ফলোয়ার < ৫০,০০০       ফলোয়ার ≥ ৫০,০০০
     (সাধারণ ব্যবহারকারী)     (সেলিব্রিটি/ইনফ্লুয়েন্সার)
              │                     │
              ▼                     ▼
    ┌──────────────┐      ┌──────────────────┐
    │ Fan-out on   │      │ শুধু পোস্ট DB-তে │
    │ Write (Push) │      │ সেভ করো          │
    │              │      │ (কোনো fan-out নেই)│
    │ সকল ফলোয়ারের│      └──────────────────┘
    │ ফিডে push   │
    └──────────────┘

    ──────── ফিড পড়ার সময় ────────

    ┌──────────────────────────────────────┐
    │  ১. Redis থেকে pre-computed ফিড পড়ো │
    │     (সাধারণ ব্যবহারকারীদের পোস্ট)     │
    │                                      │
    │  ২. সেলিব্রিটি ফলোয়িং লিস্ট থেকে     │
    │     তাদের সাম্প্রতিক পোস্ট fetch করো  │
    │                                      │
    │  ৩. দুটো merge + rank করো            │
    │     → চূড়ান্ত ফিড ডেলিভার করো        │
    └──────────────────────────────────────┘
```

**Threshold নির্ধারণ:**
- বাংলাদেশ কনটেক্সটে ৫০,০০০+ ফলোয়ার হলে সেলিব্রিটি হিসেবে বিবেচনা
- এই threshold কনফিগারেবল — সিস্টেম পারফরম্যান্সের উপর নির্ভর করে পরিবর্তন করা যায়
- কিছু সিস্টেমে ১০,০০০ threshold ব্যবহার হয়, আবার কিছুতে ১,০০,০০০

---

### ৩. Post Storage Design

#### Posts টেবিল (PostgreSQL)

```sql
CREATE TABLE posts (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    content         TEXT,                              -- বাংলা কনটেন্ট (UTF-8)
    post_type       VARCHAR(20) DEFAULT 'text',        -- text, image, video, link
    visibility      VARCHAR(20) DEFAULT 'public',      -- public, friends, private
    like_count      INT DEFAULT 0,
    comment_count   INT DEFAULT 0,
    share_count     INT DEFAULT 0,
    is_deleted      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),

    -- পার্টিশনিং-এর জন্য
    created_date    DATE GENERATED ALWAYS AS (created_at::DATE) STORED
);

-- ইনডেক্স: ব্যবহারকারীর পোস্ট সময় অনুযায়ী
CREATE INDEX idx_posts_user_created ON posts (user_id, created_at DESC);

-- ইনডেক্স: ফিড জেনারেশনের জন্য
CREATE INDEX idx_posts_created ON posts (created_at DESC)
    WHERE is_deleted = FALSE;

-- টেবিল পার্টিশনিং (মাস অনুযায়ী)
-- সময়ের সাথে পুরনো পার্টিশন আর্কাইভ করা যায়
```

#### Media টেবিল

```sql
CREATE TABLE post_media (
    id              BIGSERIAL PRIMARY KEY,
    post_id         BIGINT NOT NULL REFERENCES posts(id),
    media_type      VARCHAR(10) NOT NULL,    -- image, video, gif
    original_url    TEXT NOT NULL,            -- S3 অরিজিনাল ফাইল
    cdn_url         TEXT,                     -- CDN URL (প্রসেসিং-এর পরে)
    thumbnail_url   TEXT,                     -- থাম্বনেইল
    width           INT,
    height          INT,
    file_size       BIGINT,                  -- bytes
    duration        INT,                     -- ভিডিওর জন্য (সেকেন্ডে)
    processing_status VARCHAR(20) DEFAULT 'pending',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_media_post ON post_media (post_id);
CREATE INDEX idx_media_processing ON post_media (processing_status)
    WHERE processing_status != 'completed';
```

#### Likes এবং Comments টেবিল

```sql
CREATE TABLE likes (
    user_id     BIGINT NOT NULL REFERENCES users(id),
    post_id     BIGINT NOT NULL REFERENCES posts(id),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

CREATE INDEX idx_likes_post ON likes (post_id, created_at DESC);

CREATE TABLE comments (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT NOT NULL REFERENCES posts(id),
    user_id     BIGINT NOT NULL REFERENCES users(id),
    parent_id   BIGINT REFERENCES comments(id),     -- নেস্টেড কমেন্ট
    content     TEXT NOT NULL,                       -- বাংলা কমেন্ট
    like_count  INT DEFAULT 0,
    is_deleted  BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_comments_post ON comments (post_id, created_at ASC);
CREATE INDEX idx_comments_parent ON comments (parent_id);
```

---

### ৪. Feed Storage

#### Pre-computed Feed (Redis Sorted Set)

প্রতিটি ব্যবহারকারীর জন্য Redis-এ একটি sorted set রাখা হয় যেখানে **score = timestamp** এবং **member = post_id**।

```
Redis Key: feed:{user_id}
Type: Sorted Set (ZSET)

উদাহরণ — রহিমের ফিড (user_id = 42):

feed:42 = {
    score: 1700000100,  member: "post:8831"    ← সর্বশেষ পোস্ট
    score: 1700000050,  member: "post:8829"
    score: 1699999900,  member: "post:8825"
    score: 1699999800,  member: "post:8820"
    ...
    (সর্বোচ্চ ৫০০ এন্ট্রি রাখা হয়)
}

কমান্ড:
  যোগ করা:    ZADD feed:42 1700000100 "post:8831"
  ফিড পড়া:   ZREVRANGE feed:42 0 19  (সর্বশেষ ২০টি)
  পেজিনেশন:  ZREVRANGEBYSCORE feed:42 <last_score> -inf LIMIT 0 20
  ট্রিম:      ZREMRANGEBYRANK feed:42 0 -501  (৫০০-এর বেশি হলে পুরনো মুছে ফেলো)
```

#### On-demand Feed Query (Fallback)

যখন Redis-এ ফিড নেই (নতুন ব্যবহারকারী বা ক্যাশ মিস), তখন DB থেকে সরাসরি ফিড তৈরি:

```sql
-- রহিম (user_id=42) যাদের ফলো করে তাদের সাম্প্রতিক পোস্ট
SELECT p.id, p.content, p.user_id, p.created_at, p.like_count
FROM posts p
INNER JOIN follows f ON f.following_id = p.user_id
WHERE f.follower_id = 42
  AND p.is_deleted = FALSE
  AND p.created_at > NOW() - INTERVAL '7 days'
ORDER BY p.created_at DESC
LIMIT 50;
```

---

### ৫. Social Graph (Follow/Follower Storage)

#### Adjacency List (PostgreSQL)

```sql
CREATE TABLE follows (
    follower_id     BIGINT NOT NULL REFERENCES users(id),
    following_id    BIGINT NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, following_id)
);

-- কে কাকে ফলো করে (ব্যবহারকারীর following list)
CREATE INDEX idx_follows_follower ON follows (follower_id, created_at DESC);

-- কার কতজন ফলোয়ার (ব্যবহারকারীর follower list)
CREATE INDEX idx_follows_following ON follows (following_id, created_at DESC);

-- ফলোয়ার/ফলোয়িং কাউন্ট ক্যাশ
CREATE TABLE user_stats (
    user_id         BIGINT PRIMARY KEY REFERENCES users(id),
    follower_count  INT DEFAULT 0,
    following_count INT DEFAULT 0,
    post_count      INT DEFAULT 0
);
```

#### Graph DB বিবেচনা (Neo4j)

বৃহৎ স্কেলে social graph query-র জন্য Graph Database বেশি কার্যকর:

```
// Neo4j Cypher Query উদাহরণ

// করিমের ফলোয়ার যারা রহিমকেও ফলো করে (mutual connections)
MATCH (karim:User {id: 1})<-[:FOLLOWS]-(mutual)-[:FOLLOWS]->(rahim:User {id: 2})
RETURN mutual.name

// "তুমি হয়তো চেনো" সাজেশন — ফলোয়ারের ফলোয়ার (2-hop)
MATCH (me:User {id: 42})-[:FOLLOWS]->(:User)-[:FOLLOWS]->(suggestion:User)
WHERE NOT (me)-[:FOLLOWS]->(suggestion) AND me <> suggestion
RETURN suggestion.name, COUNT(*) AS mutual_count
ORDER BY mutual_count DESC
LIMIT 10

// এই ধরনের graph traversal PostgreSQL-এ recursive CTE দিয়ে
// করা যায়, কিন্তু কোটি কোটি এজ-এ Graph DB অনেক দ্রুত
```

```
       ┌───────┐  FOLLOWS   ┌───────┐
       │ করিম  │──────────▶│ রহিম  │
       └───┬───┘            └───┬───┘
           │                    │
    FOLLOWS│              FOLLOWS│
           ▼                    ▼
       ┌───────┐           ┌───────┐
       │ সালমা │◀──────────│তানিয়া│
       └───────┘  FOLLOWS  └───────┘

  সম্পর্কের ধরন: Directed graph (A→B মানে A ফলো করে B-কে)
  Undirected হলে "বন্ধু" হতো (Facebook-style)
```

---

### ৬. Feed Ranking Algorithm

#### Chronological (সময়ভিত্তিক)

সবচেয়ে সহজ — শুধু সময় অনুযায়ী সাজানো। কিন্তু বর্তমানে কোনো বড় প্ল্যাটফর্ম শুধু এটি ব্যবহার করে না।

#### Algorithmic Ranking (Engagement-based)

বাংলাদেশ কনটেক্সটে একটি ranking formula:

```
Feed Score = w₁ × Affinity + w₂ × Engagement + w₃ × Recency + w₄ × ContentType

যেখানে:
  Affinity (সম্পর্কের ঘনিষ্ঠতা):
    - কতবার প্রোফাইল ভিজিট করেছো          (weight: 0.3)
    - কতবার পোস্টে লাইক/কমেন্ট করেছো      (weight: 0.4)
    - সরাসরি মেসেজ করেছো কিনা              (weight: 0.2)
    - mutual friends সংখ্যা                (weight: 0.1)

  Engagement (পোস্টের জনপ্রিয়তা):
    - লাইক সংখ্যা / ইম্প্রেশন               (like rate)
    - কমেন্ট সংখ্যা / ইম্প্রেশন             (comment rate)
    - শেয়ার সংখ্যা / ইম্প্রেশন              (share rate)
    - ভিউ সময়কাল (dwell time)              (গুরুত্বপূর্ণ সিগন্যাল)

  Recency (সময়ের তাজা-পনা):
    - decay = 1 / (1 + hours_since_posted × decay_rate)
    - ২৪ ঘণ্টার মধ্যে সবচেয়ে বেশি weight

  ContentType (কনটেন্ট ধরন):
    - বাংলাদেশে ভিডিও বেশি জনপ্রিয় → ভিডিও boost
    - ইমেজ পোস্ট > শুধু টেক্সট
    - লিংক পোস্টে সবচেয়ে কম engagement

চূড়ান্ত স্কোর (উদাহরণ weights):
  score = 0.35 × affinity + 0.25 × engagement + 0.30 × recency + 0.10 × content_boost
```

---

### ৭. Media Pipeline

```
   ব্যবহারকারী আপলোড করলো
          │
          ▼
   ┌──────────────┐
   │ API Gateway   │ → ফাইল সাইজ/টাইপ ভ্যালিডেশন
   └──────┬───────┘   (Max ১০MB ইমেজ, ১০০MB ভিডিও)
          │
          ▼
   ┌──────────────┐
   │ Media Service │ → Presigned URL দিয়ে সরাসরি S3-তে আপলোড
   └──────┬───────┘   (সার্ভার দিয়ে রাউট না করে ক্লায়েন্ট→S3)
          │
          ▼
   ┌──────────────┐
   │  S3 Bucket   │ → অরিজিনাল ফাইল সংরক্ষণ
   │  (Original)  │
   └──────┬───────┘
          │ (S3 Event → Kafka)
          ▼
   ┌──────────────────────────────────────┐
   │        Media Processing Worker        │
   │                                       │
   │  ইমেজ:                               │
   │    ├─ Thumbnail (150×150)             │
   │    ├─ Small (320×320)                 │
   │    ├─ Medium (640×640)                │
   │    └─ Large (1080×1080)               │
   │                                       │
   │  ভিডিও:                              │
   │    ├─ 360p (কম ব্যান্ডউইথ — BD-তে    │
   │    │        গ্রামীণ এলাকার জন্য)       │
   │    ├─ 480p                            │
   │    ├─ 720p                            │
   │    └─ Thumbnail (ভিডিও থেকে)         │
   │                                       │
   │  + EXIF মেটাডেটা স্ট্রিপ (প্রাইভেসি)  │
   │  + Content safety check (NSFW ফিল্টার)│
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────┐
   │  S3 (Processed) →    │
   │  CDN (BunnyCDN/CF)   │
   │  BD Edge: ঢাকা, চট্ট.│
   └──────────────────────┘
```

---

### ৮. Real-time Feed Updates

#### WebSocket / SSE দিয়ে লাইভ ফিড আপডেট

```
   ক্লায়েন্ট (ব্রাউজার/অ্যাপ)
          │
          │ WebSocket Connection
          │ ws://feed.example.com/ws?token=xxx
          ▼
   ┌──────────────────┐
   │  WebSocket Server │ ← Node.js (Socket.io / ws)
   │  (Connection Pool)│
   └────────┬─────────┘
            │
            │ Redis Pub/Sub Subscribe
            │ Channel: feed_updates:{user_id}
            ▼
   ┌──────────────────┐
   │   Redis Pub/Sub   │
   └────────┬─────────┘
            │
            │ Publish (Feed Worker থেকে)
            │
   নতুন পোস্ট fan-out হলে:
     PUBLISH feed_updates:42 '{"type":"new_post","post_id":"8831"}'

   ক্লায়েন্ট পায়:
     → "৩টি নতুন পোস্ট" ব্যানার দেখায়
     → ক্লিক করলে ফিড রিফ্রেশ হয়
```

#### Fallback: Long Polling

কিছু নেটওয়ার্ক (বাংলাদেশের কিছু ISP) WebSocket ব্লক করে। সেক্ষেত্রে long polling fallback:

```
ক্লায়েন্ট → GET /api/feed/updates?since=1700000100&timeout=30
সার্ভার: ৩০ সেকেন্ড পর্যন্ত অপেক্ষা করে
  - নতুন পোস্ট আসলে → তাৎক্ষণিক রেসপন্স
  - না আসলে → ৩০ সেকেন্ড পর empty রেসপন্স
ক্লায়েন্ট আবার রিকোয়েস্ট পাঠায়
```

---

### ৯. Notification System

```
   ইভেন্ট ঘটলো (লাইক/কমেন্ট/ফলো)
          │
          ▼
   ┌──────────────────┐
   │   Kafka Topic:    │
   │ notification.send │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────────────────────┐
   │     Notification Worker           │
   │                                   │
   │  ১. Dedup চেক (একই ইভেন্ট বারবার │
   │     নোটিফাই না করা)               │
   │                                   │
   │  ২. Aggregation:                  │
   │     "করিম ও আরো ৫ জন আপনার      │
   │      পোস্টে লাইক দিয়েছেন"          │
   │                                   │
   │  ৩. User preference চেক           │
   │     (নোটিফিকেশন বন্ধ থাকলে স্কিপ)│
   │                                   │
   │  ৪. Multi-channel delivery:       │
   │     ├─ In-app (DB + WebSocket)    │
   │     ├─ Push (FCM/APNs)           │
   │     └─ Email (গুরুত্বপূর্ণ হলে)    │
   └──────────────────────────────────┘
```

```sql
CREATE TABLE notifications (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    actor_id        BIGINT NOT NULL REFERENCES users(id),
    type            VARCHAR(30) NOT NULL,    -- like, comment, follow, mention
    entity_type     VARCHAR(20),             -- post, comment
    entity_id       BIGINT,
    is_read         BOOLEAN DEFAULT FALSE,
    is_seen         BOOLEAN DEFAULT FALSE,
    group_key       VARCHAR(100),            -- aggregation key
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notif_user ON notifications (user_id, created_at DESC)
    WHERE is_read = FALSE;
```

---

### ১০. Feed Cache Strategy

```
                    ফিড রিকোয়েস্ট
                         │
                         ▼
                ┌────────────────┐
                │  L1: Local     │ ← অ্যাপ সার্ভারের মেমরি ক্যাশ
                │  Cache (LRU)   │   TTL: ৩০ সেকেন্ড
                │  হিট হলে →     │   শুধু hot users (সাম্প্রতিক অ্যাক্টিভ)
                └───────┬────────┘
                   মিস  │
                        ▼
                ┌────────────────┐
                │  L2: Redis     │ ← প্রি-কম্পিউটেড ফিড
                │  Sorted Set    │   TTL: ২৪ ঘণ্টা
                │  হিট হলে →     │   সর্বোচ্চ ৫০০ post IDs
                └───────┬────────┘
                   মিস  │
                        ▼
                ┌────────────────┐
                │  L3: DB Query  │ ← On-demand feed generation
                │  + ক্যাশ সেট   │   ফলাফল Redis-এ ক্যাশ করো
                └────────────────┘

Hot User Caching:
  - সবচেয়ে বেশি অ্যাক্টিভ ১% ব্যবহারকারী → L1 ক্যাশ
  - ৫০% ফিড রিকোয়েস্ট এই ১% থেকে আসে
  - Redis Sorted Set দিয়ে hot user ট্র্যাকিং:
    ZINCRBY hot_users 1 "user:42"  (প্রতি রিকোয়েস্টে)
    ZREVRANGE hot_users 0 9999     (টপ ১০,০০০ hot users)
```

---

## 🔥 সেকশন ৫: Advanced Topics

### ১. Celebrity/Influencer সমস্যা

বাংলাদেশে সেলিব্রিটি (শাকিব খান, তামিম ইকবাল) বা পপুলার পেজের লক্ষ লক্ষ ফলোয়ার থাকে। একটি পোস্টে ১ কোটি fan-out write অসম্ভব।

**সমাধান কৌশল:**

```
সেলিব্রিটি Tier System:

Tier 1: > ১০ লক্ষ ফলোয়ার (মেগা সেলিব্রিটি)
  → সম্পূর্ণ fan-out on read
  → পোস্ট একটি special "celebrity_posts" ক্যাশে যায়
  → ফিড পড়ার সময় এই ক্যাশ থেকে merge হয়

Tier 2: ৫০K - ১০ লক্ষ ফলোয়ার (মাইক্রো সেলিব্রিটি)
  → শুধু সক্রিয় ফলোয়ারদের কাছে fan-out (গত ৭ দিনে login)
  → বাকিদের জন্য fan-out on read

Tier 3: < ৫০K ফলোয়ার (সাধারণ ব্যবহারকারী)
  → সম্পূর্ণ fan-out on write

  ┌─────────────────────────────────────────────┐
  │ Celebrity Post Flow:                        │
  │                                             │
  │  পোস্ট → celebrity_posts:{user_id} (Redis)  │
  │        → TTL ৪৮ ঘণ্টা                       │
  │        → ZSET: score=timestamp              │
  │                                             │
  │ ফিড পড়ার সময়:                              │
  │  regular_feed + celebrity_posts merge        │
  │  → unified ranking → ডেলিভার                │
  └─────────────────────────────────────────────┘
```

---

### ২. Feed Pagination (Cursor-based)

অফসেট-ভিত্তিক pagination-এ নতুন পোস্ট আসলে duplicate বা missed পোস্ট হয়। তাই **cursor-based pagination** ব্যবহার করি:

```
প্রথম রিকোয়েস্ট:
  GET /api/feed?limit=20

রেসপন্স:
{
  "posts": [...২০টি পোস্ট...],
  "cursor": {
    "next": "eyJ0IjoxNzAwMDAwMTAwLCJpIjoiODgzMSJ9",
    "has_more": true
  }
}

পরবর্তী পেজ:
  GET /api/feed?limit=20&cursor=eyJ0IjoxNzAwMDAwMTAwLCJpIjoiODgzMSJ9

Cursor ডিকোড করলে:
{
  "t": 1700000100,    ← শেষ পোস্টের timestamp
  "i": "8831"         ← শেষ পোস্টের ID (tie-breaker)
}

Redis Query:
  ZREVRANGEBYSCORE feed:42 (1700000100 -inf LIMIT 0 20

এতে:
  ✅ নতুন পোস্ট আসলেও duplicate হবে না
  ✅ কোনো পোস্ট মিস হবে না
  ✅ যেকোনো পয়েন্ট থেকে রিজিউম করা যায়
```

---

### ৩. Content Moderation Pipeline

বাংলা কনটেন্ট মডারেশন বিশেষভাবে গুরুত্বপূর্ণ — বাংলা ভাষায় hate speech, misinformation ডিটেকশন জটিল।

```
   পোস্ট তৈরি হলো
        │
        ▼
   ┌──────────────────────────────────────┐
   │  Stage 1: Automated (Real-time)      │
   │  ├─ Spam ফিল্টার (keyword + ML)     │
   │  ├─ NSFW ইমেজ ডিটেকশন (Vision AI)  │
   │  ├─ বাংলা profanity ফিল্টার         │
   │  ├─ URL safety check (phishing)      │
   │  └─ Duplicate content detection      │
   │                                      │
   │  Score: 0.0 (নিরাপদ) → 1.0 (ক্ষতিকর)│
   └──────────┬───────────────────────────┘
              │
     ┌────────┼────────┐
     │        │        │
  <0.3     0.3-0.7   >0.7
  পাস       রিভিউ    ব্লক
     │        │        │
     ▼        ▼        ▼
  ফিডে যায়  Queue-এ  সাথে সাথে
            যায়      লুকানো হয়
              │
              ▼
   ┌──────────────────────┐
   │  Stage 2: Manual      │
   │  Human Review Team    │
   │  (বাংলা বোঝে এমন)    │
   │                       │
   │  সিদ্ধান্ত:           │
   │  ├─ Approve → ফিডে   │
   │  ├─ Reject → মুছে ফেলো│
   │  └─ Warn → সতর্কতা    │
   └──────────────────────┘
```

---

### ৪. Trending/Hashtag System

```
বাংলাদেশ-কেন্দ্রিক ট্রেন্ডিং:

   পোস্ট থেকে হ্যাশট্যাগ extract:
     "#বাংলাদেশ_ক্রিকেট" → hashtag টেবিলে কাউন্ট বাড়াও

   Sliding Window Counting:
     - গত ১ ঘণ্টায় কতবার ব্যবহৃত হয়েছে
     - গত ২৪ ঘণ্টায় কতবার
     - velocity (বৃদ্ধির হার) — হঠাৎ spike ট্রেন্ডিং-এ আসে

   Redis Implementation:
     ZINCRBY trending:hourly:{hour} 1 "#বিপিএল_২০২৪"
     ZINCRBY trending:daily:{date}  1 "#বিপিএল_২০২৪"

   ট্রেন্ডিং স্কোর:
     trending_score = hourly_count × 5 + velocity × 10 + daily_count × 1

   Location-based Trending:
     trending:dhaka:hourly:{hour}     → ঢাকায় ট্রেন্ডিং
     trending:chittagong:hourly:{hour} → চট্টগ্রামে ট্রেন্ডিং
     trending:bd:hourly:{hour}        → সারা বাংলাদেশে ট্রেন্ডিং
```

---

### ৫. Sharding Strategy

#### Posts Sharding

```
কৌশল: user_id ভিত্তিক hash sharding

  shard_id = hash(user_id) % total_shards

  সুবিধা:
    - একজন ব্যবহারকারীর সব পোস্ট একই shard-এ
    - ব্যবহারকারীর টাইমলাইন query দ্রুত

  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
  │user 0,4,8│  │user 1,5,9│  │user 2,6  │  │user 3,7  │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘

  সমস্যা: সেলিব্রিটি hotspot
  সমাধান: consistent hashing + virtual nodes
```

#### Feed Cache Sharding

```
কৌশল: user_id ভিত্তিক Redis Cluster sharding

  ১৬,৩৮৪টি hash slot-এ বিভক্ত (Redis Cluster)
  slot = CRC16(feed:{user_id}) % 16384

  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
  │ Redis Node 1  │  │ Redis Node 2  │  │ Redis Node 3  │
  │ Slot 0-5460   │  │ Slot 5461-10922│ │ Slot 10923-   │
  │               │  │               │  │  16383        │
  │ + Replica     │  │ + Replica     │  │ + Replica     │
  └───────────────┘  └───────────────┘  └───────────────┘
```

---

### ৬. Analytics Pipeline

```
   ব্যবহারকারীর অ্যাকশন (view, click, scroll, dwell time)
          │
          ▼
   ┌──────────────┐
   │ Client SDK   │ → ইভেন্ট ব্যাচ করে পাঠায় (প্রতি ৩০ সেকেন্ডে)
   └──────┬───────┘
          │
          ▼
   ┌──────────────┐
   │ Analytics    │ → Kafka topic: analytics.events
   │ Collector    │
   └──────┬───────┘
          │
     ┌────┴────┐
     │         │
     ▼         ▼
   Real-time  Batch
   (Flink)    (Spark)
     │         │
     ▼         ▼
   Redis      Data
   (Live      Warehouse
   Metrics)   (BigQuery/
              Redshift)

ট্র্যাক করা মেট্রিক্স:
  - Impressions: কতবার পোস্ট ফিডে দেখানো হয়েছে
  - Engagement Rate: (likes + comments + shares) / impressions
  - Dwell Time: কতক্ষণ পোস্টে স্ক্রল থামিয়ে রেখেছে
  - Click-through Rate: লিংক/ইমেজে ক্লিক রেট
  - Feed Scroll Depth: গড়ে কতটুকু স্ক্রল করে

বাংলাদেশ-specific মেট্রিক্স:
  - বাংলা vs ইংরেজি কনটেন্ট engagement তুলনা
  - মোবাইল vs ডেস্কটপ ব্যবহার (BD-তে ৯০%+ মোবাইল)
  - লো-ব্যান্ডউইথ ব্যবহারকারীদের experience
  - জেলা-ভিত্তিক ব্যবহারের প্যাটার্ন
```

---

## 💻 সেকশন ৬: Implementation

### Post Service (Laravel/PHP)

```php
<?php

namespace App\Services;

use App\Models\Post;
use App\Models\PostMedia;
use App\Events\PostCreated;
use App\Jobs\FanoutPostToFollowers;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Cache;

class PostService
{
    private const CELEBRITY_THRESHOLD = 50000;
    private const FEED_MAX_SIZE = 500;

    /**
     * নতুন পোস্ট তৈরি — ভ্যালিডেশন, সেভ, ফ্যানআউট trigger
     */
    public function createPost(int $userId, array $data): Post
    {
        return DB::transaction(function () use ($userId, $data) {
            $post = Post::create([
                'user_id'    => $userId,
                'content'    => $data['content'],       // বাংলা কনটেন্ট সাপোর্ট
                'post_type'  => $data['type'] ?? 'text',
                'visibility' => $data['visibility'] ?? 'public',
            ]);

            if (!empty($data['media_ids'])) {
                PostMedia::whereIn('id', $data['media_ids'])
                    ->whereNull('post_id')
                    ->update(['post_id' => $post->id]);
            }

            $this->extractAndStoreHashtags($post);
            $this->triggerFanout($post, $userId);

            event(new PostCreated($post));

            return $post->load('user', 'media');
        });
    }

    /**
     * হাইব্রিড ফ্যানআউট — সেলিব্রিটি কিনা চেক করে সিদ্ধান্ত নেয়
     */
    private function triggerFanout(Post $post, int $userId): void
    {
        $followerCount = $this->getFollowerCount($userId);

        if ($followerCount >= self::CELEBRITY_THRESHOLD) {
            // সেলিব্রিটি: fan-out on read — শুধু celebrity cache-এ রাখো
            $this->addToCelebrityCache($userId, $post->id, $post->created_at);
        } else {
            // সাধারণ ব্যবহারকারী: fan-out on write — async job dispatch
            FanoutPostToFollowers::dispatch($post->id, $userId)
                ->onQueue('feed-fanout');
        }
    }

    private function addToCelebrityCache(int $userId, int $postId, $timestamp): void
    {
        $key = "celebrity_posts:{$userId}";
        $score = $timestamp->timestamp;

        Redis::zadd($key, $score, $postId);
        Redis::zremrangebyrank($key, 0, -(self::FEED_MAX_SIZE + 1));
        Redis::expire($key, 172800); // ৪৮ ঘণ্টা TTL
    }

    /**
     * বাংলা হ্যাশট্যাগ extract ও ট্রেন্ডিং-এ যোগ
     */
    private function extractAndStoreHashtags(Post $post): void
    {
        // বাংলা ও ইংরেজি উভয় হ্যাশট্যাগ ম্যাচ করে
        preg_match_all('/#([\p{Bengali}\w]+)/u', $post->content, $matches);

        if (empty($matches[1])) {
            return;
        }

        $hour = now()->format('Y-m-d-H');
        $date = now()->format('Y-m-d');

        foreach ($matches[1] as $tag) {
            $normalizedTag = mb_strtolower($tag);

            DB::table('hashtags')->updateOrInsert(
                ['tag' => $normalizedTag],
                ['post_count' => DB::raw('post_count + 1')]
            );

            // ট্রেন্ডিং কাউন্ট আপডেট
            Redis::zincrby("trending:bd:hourly:{$hour}", 1, $normalizedTag);
            Redis::expire("trending:bd:hourly:{$hour}", 7200);

            Redis::zincrby("trending:bd:daily:{$date}", 1, $normalizedTag);
            Redis::expire("trending:bd:daily:{$date}", 172800);
        }
    }

    private function getFollowerCount(int $userId): int
    {
        return Cache::remember(
            "follower_count:{$userId}",
            300, // ৫ মিনিট ক্যাশ
            fn () => DB::table('follows')
                ->where('following_id', $userId)
                ->count()
        );
    }

    /**
     * পোস্টে লাইক — optimistic counter update + notification
     */
    public function likePost(int $userId, int $postId): bool
    {
        $inserted = DB::table('likes')->insertOrIgnore([
            'user_id'    => $userId,
            'post_id'    => $postId,
            'created_at' => now(),
        ]);

        if ($inserted) {
            Post::where('id', $postId)->increment('like_count');

            // ক্যাশড কাউন্ট ইনভ্যালিডেট
            Cache::forget("post_engagement:{$postId}");

            // নোটিফিকেশন ইভেন্ট dispatch
            $post = Post::find($postId);
            if ($post && $post->user_id !== $userId) {
                \App\Jobs\SendNotification::dispatch(
                    $post->user_id,
                    $userId,
                    'like',
                    'post',
                    $postId
                )->onQueue('notifications');
            }
        }

        return (bool) $inserted;
    }
}
```

### Fan-out Worker (Laravel Job)

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Log;

class FanoutPostToFollowers implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public int $tries = 3;
    public int $backoff = 60;

    private const BATCH_SIZE = 1000;
    private const FEED_MAX_SIZE = 500;

    public function __construct(
        private int $postId,
        private int $authorId
    ) {}

    /**
     * ফলোয়ারদের ফিডে পোস্ট push করা — ব্যাচে ভাগ করে
     */
    public function handle(): void
    {
        $post = DB::table('posts')->find($this->postId);

        if (!$post || $post->is_deleted) {
            return;
        }

        $score = strtotime($post->created_at);
        $processedCount = 0;

        // ফলোয়ার চাঙ্ক করে প্রসেস — মেমরি সাশ্রয়
        DB::table('follows')
            ->where('following_id', $this->authorId)
            ->orderBy('follower_id')
            ->chunk(self::BATCH_SIZE, function ($followers) use ($score, &$processedCount) {
                $pipeline = Redis::pipeline();

                foreach ($followers as $follower) {
                    $feedKey = "feed:{$follower->follower_id}";

                    $pipeline->zadd($feedKey, $score, $this->postId);
                    // ফিড সাইজ সীমিত রাখো
                    $pipeline->zremrangebyrank($feedKey, 0, -(self::FEED_MAX_SIZE + 1));
                    // TTL রিফ্রেশ
                    $pipeline->expire($feedKey, 86400);

                    $processedCount++;
                }

                $pipeline->exec();

                // রিয়েল-টাইম নোটিফিকেশন — WebSocket সার্ভারকে জানাও
                foreach ($followers as $follower) {
                    Redis::publish(
                        "feed_updates:{$follower->follower_id}",
                        json_encode([
                            'type'    => 'new_post',
                            'post_id' => $this->postId,
                            'author'  => $this->authorId,
                        ])
                    );
                }
            });

        Log::info("Fanout complete", [
            'post_id'   => $this->postId,
            'followers' => $processedCount,
        ]);
    }
}
```

### Feed Service (Node.js)

```javascript
// feed-service/src/feedService.js

const Redis = require('ioredis');
const { Pool } = require('pg');

const redis = new Redis.Cluster([
  { host: 'redis-1.internal', port: 6379 },
  { host: 'redis-2.internal', port: 6379 },
  { host: 'redis-3.internal', port: 6379 },
]);

const pgPool = new Pool({
  host: process.env.PG_HOST,
  database: 'newsfeed',
  max: 50,
});

const CELEBRITY_THRESHOLD = 50000;
const FEED_PAGE_SIZE = 20;
const FEED_CACHE_TTL = 86400;

class FeedService {
  /**
   * হাইব্রিড ফিড তৈরি — pre-computed + celebrity merge + ranking
   * @param {number} userId - যে ব্যবহারকারীর ফিড চাই
   * @param {string|null} cursor - পেজিনেশন cursor (base64 encoded)
   * @returns {Object} - posts array ও next cursor
   */
  async getFeed(userId, cursor = null) {
    const { maxScore, lastPostId } = this.decodeCursor(cursor);

    // ১. Pre-computed ফিড Redis থেকে পড়ো (সাধারণ ব্যবহারকারীদের পোস্ট)
    const regularPosts = await this.getRegularFeedPosts(userId, maxScore);

    // ২. সেলিব্রিটি পোস্ট on-the-fly merge করো
    const celebrityPosts = await this.getCelebrityPosts(userId, maxScore);

    // ৩. সব পোস্ট merge ও rank করো
    let allPostIds = this.mergeAndDeduplicate(regularPosts, celebrityPosts);

    // ৪. DB থেকে পূর্ণ পোস্ট ডেটা fetch করো
    const posts = await this.hydratePostsWithDetails(allPostIds);

    // ৫. Ranking algorithm প্রয়োগ করো
    const rankedPosts = await this.rankPosts(posts, userId);

    // ৬. পেজিনেশন cursor সহ রেসপন্স
    const page = rankedPosts.slice(0, FEED_PAGE_SIZE);
    const nextCursor = page.length === FEED_PAGE_SIZE
      ? this.encodeCursor(page[page.length - 1])
      : null;

    return {
      posts: page,
      cursor: { next: nextCursor, has_more: !!nextCursor },
    };
  }

  /**
   * Redis Sorted Set থেকে pre-computed ফিড পড়া
   */
  async getRegularFeedPosts(userId, maxScore) {
    const feedKey = `feed:${userId}`;
    const args = maxScore
      ? [feedKey, maxScore, '-inf', 'LIMIT', 0, FEED_PAGE_SIZE * 3]
      : [feedKey, '+inf', '-inf', 'LIMIT', 0, FEED_PAGE_SIZE * 3];

    const results = await redis.zrevrangebyscore(
      ...args, 'WITHSCORES'
    );

    const posts = [];
    for (let i = 0; i < results.length; i += 2) {
      posts.push({
        postId: parseInt(results[i]),
        score: parseFloat(results[i + 1]),
      });
    }

    return posts;
  }

  /**
   * সেলিব্রিটি পোস্ট — ফলো-করা সেলিব্রিটিদের ক্যাশ থেকে merge
   */
  async getCelebrityPosts(userId, maxScore) {
    const celebrityFollowings = await this.getCelebrityFollowings(userId);

    if (celebrityFollowings.length === 0) return [];

    const pipeline = redis.pipeline();
    const args = maxScore
      ? ['+inf', maxScore, 'LIMIT', 0, 10]
      : ['+inf', '-inf', 'LIMIT', 0, 10];

    for (const celeb of celebrityFollowings) {
      pipeline.zrevrangebyscore(
        `celebrity_posts:${celeb.id}`, ...args, 'WITHSCORES'
      );
    }

    const results = await pipeline.exec();
    const allPosts = [];

    for (const [err, result] of results) {
      if (err || !result) continue;
      for (let i = 0; i < result.length; i += 2) {
        allPosts.push({
          postId: parseInt(result[i]),
          score: parseFloat(result[i + 1]),
        });
      }
    }

    return allPosts;
  }

  /**
   * Ranking: affinity + engagement + recency ভিত্তিক score
   */
  async rankPosts(posts, userId) {
    if (posts.length === 0) return [];

    const userAffinities = await this.getUserAffinities(userId);
    const now = Date.now() / 1000;

    const scored = posts.map(post => {
      const hoursSincePosted = (now - post.created_at_ts) / 3600;
      const recency = 1 / (1 + hoursSincePosted * 0.1);

      const affinity = userAffinities[post.user_id] || 0;

      const impressions = Math.max(post.impression_count || 1, 1);
      const engagement =
        (post.like_count * 1 + post.comment_count * 3 + post.share_count * 5) / impressions;

      const contentBoost = this.getContentTypeBoost(post.post_type);

      post.rankScore =
        0.35 * affinity +
        0.25 * Math.min(engagement, 1) +
        0.30 * recency +
        0.10 * contentBoost;

      return post;
    });

    return scored.sort((a, b) => b.rankScore - a.rankScore);
  }

  /**
   * কনটেন্ট টাইপ অনুযায়ী boost — বাংলাদেশ কনটেক্সটে ভিডিও বেশি জনপ্রিয়
   */
  getContentTypeBoost(postType) {
    const boosts = {
      video: 1.0,
      image: 0.7,
      text: 0.4,
      link: 0.3,
    };
    return boosts[postType] || 0.4;
  }

  /**
   * ব্যবহারকারীর সাথে অন্যদের affinity score (ক্যাশড)
   */
  async getUserAffinities(userId) {
    const cacheKey = `affinities:${userId}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const result = await pgPool.query(`
      SELECT
        target_user_id,
        (
          profile_visits * 0.3 +
          post_interactions * 0.4 +
          message_count * 0.2 +
          mutual_friends * 0.1
        ) / GREATEST(
          profile_visits + post_interactions + message_count + mutual_friends, 1
        ) AS affinity
      FROM user_interactions
      WHERE user_id = $1
      ORDER BY affinity DESC
      LIMIT 200
    `, [userId]);

    const affinities = {};
    for (const row of result.rows) {
      affinities[row.target_user_id] = parseFloat(row.affinity);
    }

    await redis.setex(cacheKey, 3600, JSON.stringify(affinities));
    return affinities;
  }

  /**
   * ফলো-করা সেলিব্রিটিদের লিস্ট
   */
  async getCelebrityFollowings(userId) {
    const cacheKey = `celeb_followings:${userId}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const result = await pgPool.query(`
      SELECT f.following_id AS id
      FROM follows f
      JOIN user_stats us ON us.user_id = f.following_id
      WHERE f.follower_id = $1 AND us.follower_count >= $2
    `, [userId, CELEBRITY_THRESHOLD]);

    const celebs = result.rows;
    await redis.setex(cacheKey, 1800, JSON.stringify(celebs));
    return celebs;
  }

  mergeAndDeduplicate(regular, celebrity) {
    const seen = new Set();
    const merged = [];

    for (const p of [...regular, ...celebrity]) {
      if (!seen.has(p.postId)) {
        seen.add(p.postId);
        merged.push(p);
      }
    }

    return merged.sort((a, b) => b.score - a.score);
  }

  /**
   * পোস্ট IDs থেকে পূর্ণ পোস্ট ডেটা DB থেকে লোড + ক্যাশ
   */
  async hydratePostsWithDetails(postEntries) {
    const postIds = postEntries.map(p => p.postId);
    if (postIds.length === 0) return [];

    // Multi-get ক্যাশ চেক
    const cacheKeys = postIds.map(id => `post:${id}`);
    const cached = await redis.mget(...cacheKeys);

    const found = {};
    const missingIds = [];

    cached.forEach((val, idx) => {
      if (val) found[postIds[idx]] = JSON.parse(val);
      else missingIds.push(postIds[idx]);
    });

    // ক্যাশ মিস হলে DB থেকে fetch
    if (missingIds.length > 0) {
      const placeholders = missingIds.map((_, i) => `$${i + 1}`).join(',');
      const result = await pgPool.query(`
        SELECT p.*, u.name AS author_name, u.avatar_url,
               array_agg(pm.cdn_url) FILTER (WHERE pm.cdn_url IS NOT NULL) AS media_urls
        FROM posts p
        JOIN users u ON u.id = p.user_id
        LEFT JOIN post_media pm ON pm.post_id = p.id
        WHERE p.id IN (${placeholders}) AND p.is_deleted = FALSE
        GROUP BY p.id, u.name, u.avatar_url
      `, missingIds);

      const pipeline = redis.pipeline();
      for (const row of result.rows) {
        found[row.id] = row;
        found[row.id].created_at_ts = new Date(row.created_at).getTime() / 1000;
        pipeline.setex(`post:${row.id}`, 3600, JSON.stringify(found[row.id]));
      }
      await pipeline.exec();
    }

    return postEntries
      .map(entry => found[entry.postId])
      .filter(Boolean);
  }

  decodeCursor(cursor) {
    if (!cursor) return { maxScore: null, lastPostId: null };
    const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString());
    return { maxScore: decoded.t, lastPostId: decoded.i };
  }

  encodeCursor(post) {
    const data = { t: post.created_at_ts, i: post.id };
    return Buffer.from(JSON.stringify(data)).toString('base64');
  }
}

module.exports = FeedService;
```

### WebSocket Real-time Updates (Node.js)

```javascript
// feed-service/src/realtimeServer.js

const WebSocket = require('ws');
const Redis = require('ioredis');
const jwt = require('jsonwebtoken');

const subscriber = new Redis(process.env.REDIS_URL);
const JWT_SECRET = process.env.JWT_SECRET;

// userId → Set<WebSocket> ম্যাপিং
const userConnections = new Map();

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', async (ws, req) => {
  const token = new URL(req.url, 'http://localhost').searchParams.get('token');

  let userId;
  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    userId = decoded.sub;
  } catch {
    ws.close(4001, 'Invalid token');
    return;
  }

  // কানেকশন রেজিস্টার করো
  if (!userConnections.has(userId)) {
    userConnections.set(userId, new Set());

    // এই ব্যবহারকারীর চ্যানেলে subscribe করো
    subscriber.subscribe(`feed_updates:${userId}`);
  }
  userConnections.get(userId).add(ws);

  console.log(`ব্যবহারকারী ${userId} কানেক্টেড। মোট: ${userConnections.get(userId).size}`);

  // Heartbeat
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });

  ws.on('close', () => {
    const connections = userConnections.get(userId);
    if (connections) {
      connections.delete(ws);
      if (connections.size === 0) {
        userConnections.delete(userId);
        subscriber.unsubscribe(`feed_updates:${userId}`);
      }
    }
  });
});

// Redis Pub/Sub থেকে মেসেজ পেলে WebSocket দিয়ে পাঠাও
subscriber.on('message', (channel, message) => {
  const match = channel.match(/^feed_updates:(\d+)$/);
  if (!match) return;

  const userId = parseInt(match[1]);
  const connections = userConnections.get(userId);
  if (!connections) return;

  for (const ws of connections) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'feed_update',
        data: JSON.parse(message),
      }));
    }
  }
});

// Dead connection cleanup — প্রতি ৩০ সেকেন্ডে
setInterval(() => {
  wss.clients.forEach(ws => {
    if (!ws.isAlive) {
      ws.terminate();
      return;
    }
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);

console.log('WebSocket সার্ভার পোর্ট 8080-এ চালু');
```

### Feed API Routes (Laravel)

```php
<?php

// routes/api.php
use App\Http\Controllers\FeedController;
use App\Http\Controllers\PostController;

Route::middleware('auth:sanctum')->group(function () {
    // ফিড
    Route::get('/feed', [FeedController::class, 'index']);

    // পোস্ট CRUD
    Route::post('/posts', [PostController::class, 'store']);
    Route::delete('/posts/{post}', [PostController::class, 'destroy']);

    // লাইক/কমেন্ট
    Route::post('/posts/{post}/like', [PostController::class, 'like']);
    Route::delete('/posts/{post}/like', [PostController::class, 'unlike']);
    Route::post('/posts/{post}/comments', [PostController::class, 'comment']);

    // ফলো
    Route::post('/users/{user}/follow', [PostController::class, 'follow']);
    Route::delete('/users/{user}/follow', [PostController::class, 'unfollow']);

    // ট্রেন্ডিং
    Route::get('/trending', [FeedController::class, 'trending']);
});
```

```php
<?php

namespace App\Http\Controllers;

use App\Services\PostService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class FeedController extends Controller
{
    public function __construct(private PostService $postService) {}

    /**
     * হাইব্রিড ফিড — Node.js Feed Service-কে internal call করে ফিড আনে
     * অথবা সরাসরি Redis + DB থেকে build করে (simplified version)
     */
    public function index(Request $request)
    {
        $userId = $request->user()->id;
        $cursor = $request->query('cursor');

        // Internal HTTP call to Node.js Feed Service
        $response = \Http::get(config('services.feed.url') . '/feed', [
            'user_id' => $userId,
            'cursor'  => $cursor,
        ]);

        return response()->json($response->json());
    }

    /**
     * বাংলাদেশে ট্রেন্ডিং হ্যাশট্যাগ
     */
    public function trending(Request $request)
    {
        $region = $request->query('region', 'bd');
        $hour = now()->format('Y-m-d-H');
        $date = now()->format('Y-m-d');

        $hourlyKey = "trending:{$region}:hourly:{$hour}";
        $dailyKey  = "trending:{$region}:daily:{$date}";

        // ঘণ্টাভিত্তিক ও দৈনিক ট্রেন্ড combine
        $hourlyTrends = Redis::zrevrange($hourlyKey, 0, 19, 'WITHSCORES');
        $dailyTrends  = Redis::zrevrange($dailyKey, 0, 19, 'WITHSCORES');

        $combined = [];
        foreach ($hourlyTrends as $tag => $count) {
            $dailyCount = $dailyTrends[$tag] ?? 0;
            $combined[$tag] = ($count * 5) + ($dailyCount * 1);
        }

        arsort($combined);

        $trending = [];
        foreach (array_slice($combined, 0, 10, true) as $tag => $score) {
            $trending[] = [
                'hashtag'      => "#{$tag}",
                'score'        => round($score, 2),
                'hourly_count' => (int) ($hourlyTrends[$tag] ?? 0),
                'daily_count'  => (int) ($dailyTrends[$tag] ?? 0),
            ];
        }

        return response()->json([
            'region'   => $region,
            'trending' => $trending,
        ]);
    }
}
```

### Follow/Unfollow Service (Laravel)

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Cache;
use App\Jobs\SendNotification;
use App\Jobs\BackfillFeedForNewFollowing;

class FollowService
{
    /**
     * ফলো করা — ফিড ব্যাকফিল + নোটিফিকেশন
     */
    public function follow(int $followerId, int $followingId): bool
    {
        if ($followerId === $followingId) {
            return false;
        }

        $inserted = DB::table('follows')->insertOrIgnore([
            'follower_id'  => $followerId,
            'following_id' => $followingId,
            'created_at'   => now(),
        ]);

        if (!$inserted) {
            return false;
        }

        // কাউন্টার আপডেট
        DB::table('user_stats')
            ->where('user_id', $followerId)
            ->increment('following_count');

        DB::table('user_stats')
            ->where('user_id', $followingId)
            ->increment('follower_count');

        // ক্যাশ ইনভ্যালিডেট
        Cache::forget("follower_count:{$followingId}");
        Cache::forget("celeb_followings:{$followerId}");

        // নতুন ফলো-করা ব্যক্তির সাম্প্রতিক পোস্ট ফিডে ব্যাকফিল
        BackfillFeedForNewFollowing::dispatch($followerId, $followingId)
            ->onQueue('feed-fanout');

        // ফলো নোটিফিকেশন
        SendNotification::dispatch($followingId, $followerId, 'follow', 'user', $followingId)
            ->onQueue('notifications');

        return true;
    }

    /**
     * আনফলো — ফিড থেকে পোস্ট সরানো (optional)
     */
    public function unfollow(int $followerId, int $followingId): bool
    {
        $deleted = DB::table('follows')
            ->where('follower_id', $followerId)
            ->where('following_id', $followingId)
            ->delete();

        if (!$deleted) {
            return false;
        }

        DB::table('user_stats')
            ->where('user_id', $followerId)
            ->decrement('following_count');

        DB::table('user_stats')
            ->where('user_id', $followingId)
            ->decrement('follower_count');

        Cache::forget("follower_count:{$followingId}");
        Cache::forget("celeb_followings:{$followerId}");

        return true;
    }
}
```

---

## 📋 সারসংক্ষেপ

### মূল ডিজাইন সিদ্ধান্তসমূহ

| বিষয় | সিদ্ধান্ত | কারণ |
|-------|-----------|------|
| **Feed Generation** | হাইব্রিড (Push + Pull) | সেলিব্রিটি সমস্যা এড়ানো + দ্রুত read |
| **Primary DB** | PostgreSQL (sharded) | ACID compliance + JSON support |
| **Feed Cache** | Redis Sorted Set | O(log N) insert + range query |
| **Message Queue** | Kafka | High throughput fan-out + event sourcing |
| **Real-time** | WebSocket + SSE fallback | তাৎক্ষণিক আপডেট, BD ISP compatibility |
| **Media Storage** | S3 + CDN (BD edge) | লো-ল্যাটেন্সি মিডিয়া ডেলিভারি |
| **Social Graph** | PostgreSQL + Neo4j | সহজ query + complex traversal |
| **Ranking** | Affinity + Engagement + Recency | পার্সোনালাইজড অভিজ্ঞতা |
| **Pagination** | Cursor-based | Duplicate/miss সমস্যা এড়ানো |
| **Sharding** | user_id hash-based | ডেটা locality বজায় রাখা |

### স্কেলিং রোডম্যাপ

```
Phase 1: ১ কোটি ব্যবহারকারী
  → Single region, basic fan-out on write
  → PostgreSQL + Redis (single cluster)

Phase 2: ১০ কোটি ব্যবহারকারী
  → হাইব্রিড fan-out চালু
  → DB sharding শুরু
  → CDN edge ঢাকা + চট্টগ্রাম

Phase 3: ৫০ কোটি ব্যবহারকারী
  → Multi-region deployment
  → Cassandra feed store
  → ML-based ranking
  → Real-time analytics pipeline
  → Content moderation at scale
```

### বাংলাদেশ-নির্দিষ্ট বিবেচনা

```
১. ভাষা: UTF-8 বাংলা সাপোর্ট সর্বত্র (DB, search, rendering)
২. নেটওয়ার্ক: লো-ব্যান্ডউইথ অপ্টিমাইজেশন (adaptive media quality)
৩. মোবাইল: ৯০%+ মোবাইল ব্যবহারকারী — মোবাইল-ফার্স্ট ডিজাইন
৪. CDN Edge: ঢাকা ও চট্টগ্রামে edge server — ল্যাটেন্সি কমানো
৫. পেমেন্ট: bKash/Nagad ইন্টিগ্রেশন (বুস্ট পোস্ট/বিজ্ঞাপনের জন্য)
৬. কনটেন্ট: বাংলা hate speech ডিটেকশন মডেল
৭. ট্রেন্ডিং: জেলা-ভিত্তিক লোকাল ট্রেন্ডিং (ঢাকা, চট্টগ্রাম, সিলেট)
```

---

> **এই ডিজাইনটি একটি production-grade নিউজফিড সিস্টেমের সম্পূর্ণ blueprint। বাস্তবায়নে প্রতিটি component আলাদা microservice হিসেবে deploy করা উচিত এবং ধাপে ধাপে স্কেল করা উচিত।**
