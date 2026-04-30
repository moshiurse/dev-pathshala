# 🔴 Redis Deep Dive — In-Memory ডেটা স্ট্রাকচার সার্ভার

## 📌 সংজ্ঞা ও মূল ধারণা

**Redis** (REmote DIctionary Server) হলো একটি **in-memory data structure server** — শুধু key-value store নয়, বরং একটি পূর্ণাঙ্গ ডেটা স্ট্রাকচার সার্ভার যেখানে strings, lists, hashes, sets, sorted sets, streams, geospatial index, HyperLogLog, bitmaps, এমনকি probabilistic data structures (Bloom, Cuckoo, Top-K) — সব নেটওয়ার্কের ওপর দিয়ে অ্যাক্সেস করা যায়।

> Redis = "Memcached + অনেক বেশি data structures + persistence + replication + scripting + pub/sub + streams + cluster"

### কেন Redis এত জনপ্রিয়?

```
┌────────────────────────────────────────────────────────────────┐
│  Daraz Bangladesh — একটি প্রোডাক্ট পেজ লোডের গল্প              │
│                                                                │
│  ❌ Redis ছাড়া:                                                │
│     User → App → MySQL (50ms) → JOIN 5 tables → 250ms          │
│     RPS 10k হলে DB CPU 100% → site down 💀                    │
│                                                                │
│  ✅ Redis সহ (cache-aside):                                    │
│     User → App → Redis GET (0.4ms) → done                      │
│     Hit ratio 95% → DB load 95% কমে গেল                       │
│     P99 latency 250ms → 3ms                                    │
└────────────────────────────────────────────────────────────────┘
```

Redis fast কারণ:
1. **In-memory** — RAM-এ সব data, ডিস্ক I/O নেই hot path-এ।
2. **Single-threaded event loop** — lock contention নেই, context switch নেই।
3. **Optimized data structures** — প্রতিটি data type-এর জন্য বিশেষ encoding (ziplist, intset, listpack, skiplist)।
4. **Simple protocol (RESP)** — text-based, parse করতে cheap।
5. **Pipelining** — অনেক command একসাথে পাঠানো যায়।

---

## 🏗️ Architecture & Internals

### Single-threaded Event Loop

Redis-এর মূল command processing **একটি thread**-এ হয় — এটাই Redis-কে predictable ও lock-free করে।

```
┌────────────────────────────────────────────────────────────────┐
│                Redis Single-threaded Event Loop                │
│                                                                │
│   ┌──────────┐                                                 │
│   │ Client 1 │──┐                                              │
│   └──────────┘  │                                              │
│   ┌──────────┐  │   ┌──────────────────────────────────┐       │
│   │ Client 2 │──┼──▶│  epoll/kqueue (Multiplexing)     │       │
│   └──────────┘  │   │                                  │       │
│   ┌──────────┐  │   │  ┌────────────────────────────┐  │       │
│   │ Client N │──┘   │  │  Single Main Thread         │  │       │
│   └──────────┘      │  │  ┌──────────────────────┐   │  │       │
│                     │  │  │ 1. Read RESP frames   │   │  │       │
│                     │  │  │ 2. Parse command      │   │  │       │
│                     │  │  │ 3. Execute (in RAM)   │   │  │       │
│                     │  │  │ 4. Write reply        │   │  │       │
│                     │  │  └──────────────────────┘   │  │       │
│                     │  └────────────────────────────┘  │       │
│                     └──────────────────────────────────┘       │
│                              │                                 │
│      ┌───────────────────────┼───────────────────┐             │
│      ▼                       ▼                   ▼             │
│ ┌─────────┐         ┌─────────────┐      ┌─────────────┐      │
│ │ BIO     │         │ AOF rewrite │      │ RDB save    │      │
│ │ threads │         │ (forked     │      │ (forked     │      │
│ │ (close, │         │   child)    │      │   child)    │      │
│ │ unlink) │         └─────────────┘      └─────────────┘      │
│ └─────────┘                                                    │
└────────────────────────────────────────────────────────────────┘
```

**কেন single-thread fast?**
- প্রতিটি command in-memory hash table lookup — typically <1μs।
- কোনো mutex/lock নেই → cache-line bouncing, deadlock নেই।
- Latency predictable: P50 ~0.2ms, P99 ~1ms (LAN-এ)।

**Bottleneck কোথায়?**
- CPU-bound: একটি core saturate হয়ে গেলে আর scale হয় না → Redis Cluster দরকার।
- Network I/O — অনেক ছোট command হলে syscall overhead → I/O threads (Redis 6+)।
- Big keys (1MB string বা 1M-element list) — single-thread block করে দেয়।

### Redis 6+ I/O Threads

Redis 6 থেকে **I/O multiplexing** আলাদা thread-এ করা যায় (command execution তবু single-thread):

```ini
# redis.conf
io-threads 4              # I/O threads (read/write socket)
io-threads-do-reads yes   # read-ও multi-thread
```

এতে network heavy workload-এ throughput 2x হতে পারে। কিন্তু latency-sensitive workload-এ off রাখাই ভালো।

### Memory Resident — RAM-ই সব

Redis সব data RAM-এ রাখে। ডিস্ক শুধু persistence-এর জন্য (RDB/AOF)। এর মানে:
- ✅ Read/write < 1ms।
- ❌ Dataset RAM-এর চেয়ে বড় হলে চলবে না।
- 💸 RAM expensive — 100GB Redis = expensive instance।

> Bangladesh context: Daraz-এর entire active product catalog (~5M SKU × ~2KB = 10GB) Redis-এ ফিট করে। কিন্তু order history (300M rows) Redis-এ রাখা insane — সেটা MySQL-এ থাকে।

---

## 💾 Persistence — ডেটা টেকসই করার উপায়

Redis crash বা restart হলে data হারাবে — তাই persistence দরকার (যদি cache-only ব্যবহার না করো)।

### RDB (Redis Database) — Snapshot

নির্দিষ্ট সময় পর পর memory-র সম্পূর্ণ snapshot ডিস্কে লেখা।

```ini
# redis.conf
save 900 1      # 900 sec (15min)-এ অন্তত 1 key change হলে save
save 300 10     # 300 sec-এ 10 keys
save 60 10000   # 60 sec-এ 10000 keys
dbfilename dump.rdb
dir /var/lib/redis
```

```
┌──────────────────────────────────────────────────────────┐
│              RDB Snapshot Process                        │
│                                                          │
│   Main Process            Forked Child                   │
│   ─────────────           ────────────────               │
│   1. fork()  ─────────▶   COW (copy-on-write)            │
│   2. Continue serving                                    │
│      requests             3. Iterate all keys            │
│                           4. Write to dump.rdb.tmp       │
│                           5. rename → dump.rdb           │
│                           6. exit                        │
└──────────────────────────────────────────────────────────┘
```

**সুবিধা:**
- Compact binary format।
- Restore fast (memory-এ load)।
- Backup-friendly (একটা file copy করলেই হলো)।

**অসুবিধা:**
- Snapshot interval-এ crash হলে শেষ snapshot-এর পরের সব data lost।
- Fork() expensive বড় instance-এ (10GB RAM = ~1s pause + COW memory pressure)।

### AOF (Append-Only File) — Write-ahead log

প্রতিটি write command-এর log রাখা।

```ini
appendonly yes
appendfilename "appendonly.aof"
appendfsync always       # প্রতি write-এ fsync (slowest, safest)
appendfsync everysec     # প্রতি সেকেন্ডে fsync (default, balanced)
appendfsync no           # OS ঠিক করুক (fastest, riskiest)
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

| `appendfsync` | Durability | Latency impact | Use case |
|---------------|------------|----------------|----------|
| `always` | Zero data loss | High (fsync per write) | Financial: bKash transaction log |
| `everysec` | Max 1 sec loss | Low | **Default — recommended** |
| `no` | OS-dependent (~30s) | None | Cache only |

**AOF Rewrite:** AOF file বড় হয়ে গেলে background-এ rewrite হয় — শুধু minimum commands রাখা হয় (e.g., 100 INCR → একটা SET)।

### Hybrid Persistence (Redis 4+)

```ini
aof-use-rdb-preamble yes
```

AOF file-এর শুরুতে RDB binary snapshot, তারপর incremental AOF commands। দ্রুত restart + low data loss।

### কোনটি ব্যবহার করবে?

```
┌──────────────────────────────────────────────────────────┐
│  Use case                       │  Recommendation        │
├──────────────────────────────────────────────────────────┤
│  Pure cache (rebuildable)       │  No persistence        │
│  Session store                  │  RDB only              │
│  Job queue                      │  AOF everysec          │
│  Source-of-truth (rare)         │  AOF always + replica  │
│  Production default             │  Hybrid (RDB+AOF)      │
└──────────────────────────────────────────────────────────┘
```

---

## 🔄 Replication — Master/Replica

Redis **asynchronous replication** ব্যবহার করে — master commit করার পর replica-তে পাঠায়।

```
                      ┌────────────────┐
   Writes ──────────▶│     MASTER      │
                      │  (read+write)  │
                      └───────┬────────┘
                              │ async replication
                              │ (PSYNC + repl backlog)
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
         ┌──────────┐  ┌──────────┐  ┌──────────┐
         │ REPLICA1 │  │ REPLICA2 │  │ REPLICA3 │
         │  (read)  │  │  (read)  │  │  (read)  │
         └──────────┘  └──────────┘  └──────────┘
                ▲             ▲             ▲
                │             │             │
   Reads ───────┴─────────────┴─────────────┘
```

```bash
# Replica config
replicaof 10.0.0.1 6379
replica-read-only yes
masterauth <password>
```

**Replication consistency:**
- **Async** → replica কিছু ms পেছনে থাকে (replication lag)।
- Master ack পাঠায় write-এর সাথে সাথে — replica-এ পৌঁছানোর আগেই।
- যদি master crash করে আর replica-এ data না পৌঁছায় → **data loss** সম্ভব।

`min-replicas-to-write 1` + `min-replicas-max-lag 10` — অন্তত 1 replica 10s lag-এর মধ্যে না থাকলে master write reject করবে।

---

## 🧠 Memory Model

Redis **jemalloc** ব্যবহার করে (default Linux) — fragmentation reduce করে।

### MEMORY USAGE & OBJECT ENCODING

```bash
127.0.0.1:6379> SET user:1 "Karim"
OK
127.0.0.1:6379> MEMORY USAGE user:1
(integer) 56
127.0.0.1:6379> OBJECT ENCODING user:1
"embstr"             # ছোট string (≤44 bytes) embedded

127.0.0.1:6379> HSET product:101 name "iPhone" price 130000
127.0.0.1:6379> OBJECT ENCODING product:101
"listpack"           # ছোট hash listpack-এ
# ↑ hash field count > hash-max-listpack-entries (default 128)
#   বা value > hash-max-listpack-value (default 64) হলে → "hashtable"
```

**Encoding optimization:**
```ini
hash-max-listpack-entries 128
hash-max-listpack-value 64
list-max-listpack-size -2     # 8KB
set-max-intset-entries 512
zset-max-listpack-entries 128
```

ছোট collections compact listpack-এ থাকে → 5-10x কম memory।

### Memory Fragmentation

```bash
127.0.0.1:6379> INFO memory
used_memory: 1073741824           # Redis যা allocate করেছে
used_memory_rss: 1610612736       # OS-এ actual RSS
mem_fragmentation_ratio: 1.50     # > 1.5 মানে fragmentation আছে
```

`activedefrag yes` — runtime defragmentation (Redis 4+)।

### Eviction Policies — `maxmemory-policy`

```ini
maxmemory 4gb
maxmemory-policy allkeys-lru
```

| Policy | Behavior | Use case |
|--------|----------|----------|
| `noeviction` | OOM error returned | Source-of-truth, কখনো data হারানো যাবে না |
| `allkeys-lru` | যেকোনো key, least recently used evict | **General cache (default recommendation)** |
| `allkeys-lfu` | যেকোনো key, least frequently used | Hot keys preserve করতে — Daraz product cache |
| `volatile-lru` | শুধু TTL-যুক্ত keys, LRU | Cache + persistent data মিশ্রিত |
| `volatile-lfu` | শুধু TTL keys, LFU | একই কিন্তু LFU |
| `allkeys-random` | Random key evict | Workload uniform হলে |
| `volatile-random` | TTL keys random | Rare |
| `volatile-ttl` | শীঘ্রই expire হবে এমন evict | Predictable expiration |

> 💡 **LRU vs LFU**: LRU recently-accessed keep করে, LFU frequently-accessed keep করে। Daraz-এ "Galaxy S24" দিনে 1M বার access হয় কিন্তু কেউ scroll করে গেলে last access purano হতে পারে — LFU ভালো।

---

## 📚 Data Structures — Redis-এর হৃদপিণ্ড

### 1. Strings — সবচেয়ে basic

Maximum 512MB। শুধু text নয়, যেকোনো binary।

```bash
SET user:1:name "Karim"
GET user:1:name
SET counter 100
INCR counter             # → 101
INCRBY counter 10        # → 111
INCRBYFLOAT price 99.5
APPEND log "new entry\n"
SETEX session:abc 3600 "{...}"   # 1 hour TTL
SET key value NX EX 60   # Set if not exists, 60s TTL (atomic)
MSET k1 v1 k2 v2 k3 v3   # multi-set
MGET k1 k2 k3
GETRANGE log 0 100       # substring
STRLEN user:1:name
```

#### Bitmap Operations — ছোট জায়গায় বিশাল data

```bash
# Daraz: প্রতি ইউজার online কিনা track করতে
SETBIT users:online:2024-01-15 12345 1   # user 12345 online
SETBIT users:online:2024-01-15 99999 1
BITCOUNT users:online:2024-01-15         # মোট online users
GETBIT users:online:2024-01-15 12345     # → 1

# 10M users = 10M bits = 1.25 MB মাত্র!

# AND/OR/XOR/NOT — সাত দিনের retention
BITOP AND users:active:week users:online:2024-01-15 \
                              users:online:2024-01-16 \
                              ...
BITCOUNT users:active:week
```

### 2. Lists — Linked list / Deque

```bash
LPUSH queue:tasks "task1"        # left push
RPUSH queue:tasks "task2"        # right push
LPOP queue:tasks                 # → "task1"
RPOP queue:tasks
LRANGE queue:tasks 0 -1          # all elements
LLEN queue:tasks
LTRIM logs 0 999                 # শুধু সর্বশেষ 1000 রাখো
BLPOP queue:tasks 30             # blocking pop, 30s timeout
LMOVE src dst LEFT RIGHT         # atomic move (reliable queue)
```

**Use case:** simple FIFO queue, recent activity log।

> ⚠️ Production queue-এর জন্য Streams ব্যবহার করো — list-এ ack/replay নেই।

### 3. Hashes — Object storage

```bash
HSET user:1 name "Karim" age 28 city "Dhaka"
HGET user:1 name                 # → "Karim"
HMGET user:1 name age
HGETALL user:1                   # → {name, Karim, age, 28, city, Dhaka}
HINCRBY user:1 age 1             # age = 29
HEXISTS user:1 email             # → 0
HDEL user:1 city
HSCAN user:1 0 MATCH "n*"        # cursor-based scan
```

**Daraz product cache:**
```bash
HSET product:9876 name "iPhone 15" price 145000 stock 23 brand "Apple"
EXPIRE product:9876 600
```

### 4. Sets — Unique unordered

```bash
SADD pathao:riders:available "rider:101" "rider:102" "rider:103"
SISMEMBER pathao:riders:available "rider:101"   # → 1
SCARD pathao:riders:available                   # cardinality
SREM pathao:riders:available "rider:101"
SMEMBERS pathao:riders:available

# Set algebra
SINTER online:dhaka kyc:verified                # intersection
SUNION users:premium users:vip
SDIFF users:all users:banned
SRANDMEMBER lottery:participants 3              # random sample
SPOP queue:retry                                # remove + return random
```

### 5. Sorted Sets (ZSET) — Score-ranked

প্রতিটি member-এ একটা score (double)। skiplist + hashtable দিয়ে implement।

```bash
# Daraz top sellers leaderboard
ZADD leaderboard 1500 "seller:1"
ZADD leaderboard 2300 "seller:2" 1800 "seller:3"
ZINCRBY leaderboard 100 "seller:1"              # score += 100
ZRANGE leaderboard 0 9 WITHSCORES REV           # top 10
ZRANGEBYSCORE leaderboard 1000 2000             # score range
ZRANK leaderboard "seller:1"                    # rank (0-indexed)
ZREVRANK leaderboard "seller:1"                 # rank from top

# Time-window rate limiter
ZADD requests:user:42 1700000000 "req-uuid-1"
ZREMRANGEBYSCORE requests:user:42 0 1699999940  # 60s window
ZCARD requests:user:42                           # current count
```

### 6. Streams (Redis 5+) — Append-only log

Kafka-like, কিন্তু Redis-এ। ছোট-মাঝারি স্কেলে Kafka-র বিকল্প।

```bash
# Producer
XADD orders * order_id 1001 user 42 amount 1500
# auto-generated ID: 1700000000000-0

# Simple consumer (latest)
XREAD COUNT 10 BLOCK 5000 STREAMS orders $

# Consumer group (load balanced + ack)
XGROUP CREATE orders payment-svc $ MKSTREAM
XREADGROUP GROUP payment-svc consumer-1 \
  COUNT 10 BLOCK 5000 STREAMS orders >
XACK orders payment-svc 1700000000000-0

# Pending (not yet ack'd) entries
XPENDING orders payment-svc
XCLAIM orders payment-svc consumer-2 60000 1700000000000-0
# ↑ 60s এর বেশি pending হলে অন্য consumer দখল নিতে পারে

XLEN orders
XTRIM orders MAXLEN ~ 10000      # cap stream size
```

> 🔗 Stream vs Kafka comparison-এর জন্য `/18-kafka/README.md` দেখো। মূল পার্থক্য: Streams clustering-এ partitioning, retention, throughput-এ Kafka-র চেয়ে কম।

### 7. HyperLogLog — Approximate cardinality

12KB-এ billions of unique items count, ~0.81% error।

```bash
# Daraz: কতজন unique visitor আজ?
PFADD visitors:2024-01-15 "user:1" "user:2" "user:3"
PFADD visitors:2024-01-15 "user:1"   # duplicate, ignored
PFCOUNT visitors:2024-01-15          # ~unique count
PFMERGE visitors:week visitors:2024-01-15 visitors:2024-01-16 ...
PFCOUNT visitors:week                # weekly uniques
```

### 8. Geospatial — GeoHash on sorted sets

Pathao-এর nearest rider খুঁজতে:

```bash
GEOADD pathao:riders 90.4125 23.8103 "rider:101"   # lon lat member
GEOADD pathao:riders 90.4150 23.8120 "rider:102"

# 2km radius-এ সব rider
GEOSEARCH pathao:riders FROMLONLAT 90.4125 23.8103 \
          BYRADIUS 2 km ASC COUNT 10 WITHCOORD WITHDIST

GEODIST pathao:riders "rider:101" "rider:102" km
GEOHASH pathao:riders "rider:101"
```

### 9. Bitfields

বিভিন্ন size-এর integer একটি string-এ pack:

```bash
BITFIELD user:42:counters \
  SET u32 #0 100 \
  INCRBY u32 #0 1 \
  GET u32 #0
```

### 10. Probabilistic Data Structures (RedisBloom module)

```bash
# Bloom filter — "user এই product দেখেছে কি?" (false positive ok)
BF.RESERVE seen:user:42 0.001 100000
BF.ADD seen:user:42 "product:9876"
BF.EXISTS seen:user:42 "product:9876"   # → 1 (definite/probable)

# Cuckoo filter — Bloom + delete support
CF.ADD spam:emails "spam@bad.com"

# Top-K — Heavy Hitters
TOPK.RESERVE trending:products 10 2000 7 0.925
TOPK.INCRBY trending:products "product:9876" 1
TOPK.LIST trending:products

# Count-Min Sketch — frequency estimation
CMS.INITBYPROB freq 0.001 0.01
CMS.INCRBY freq "search:phone" 1

# t-digest — quantile estimation
TDIGEST.CREATE latency
TDIGEST.ADD latency 100 250 500
TDIGEST.QUANTILE latency 0.99   # P99 latency
```

### 11. RedisJSON & RediSearch (modules)

```bash
# RedisJSON — native JSON
JSON.SET user:1 $ '{"name":"Karim","orders":[{"id":1,"total":500}]}'
JSON.GET user:1 $.orders[*].total
JSON.ARRAPPEND user:1 $.orders '{"id":2,"total":700}'

# RediSearch — full-text + secondary indexes
FT.CREATE idx:products ON HASH PREFIX 1 product: \
  SCHEMA name TEXT WEIGHT 5 brand TAG price NUMERIC SORTABLE
FT.SEARCH idx:products "@brand:{Apple} @price:[100000 200000]"
```

### 12. Pub/Sub — Fire-and-forget messaging

```bash
# Subscriber
SUBSCRIBE notifications:user:42
PSUBSCRIBE notifications:*           # pattern

# Publisher
PUBLISH notifications:user:42 "{\"type\":\"order_shipped\"}"
```

**Limitations:**
- ❌ No persistence — subscribe না থাকলে message harano।
- ❌ No replay।
- ❌ No ack।
- ✅ Use case: real-time presence, "someone typing", live dashboard ticker।

বড় কাজে **Streams + consumer groups** ব্যবহার করো।

---

## 🔐 Transactions — MULTI/EXEC/WATCH

Redis transaction = command batching + optimistic locking, **ACID-এর সব নয়** (rollback নেই)।

```bash
MULTI
SET balance:karim 5000
INCRBY balance:karim -1000
INCRBY balance:rahim 1000
EXEC          # সব command একসাথে atomic execute (single-thread guarantees)
```

```
┌──────────────────────────────────────────────────────┐
│  WATCH-based optimistic lock                         │
│                                                      │
│  WATCH balance:karim                                 │
│  bal = GET balance:karim                             │
│  if bal >= 1000:                                     │
│      MULTI                                           │
│      DECRBY balance:karim 1000                       │
│      INCRBY balance:rahim 1000                       │
│      EXEC                                            │
│      # যদি অন্য কেউ ইতিমধ্যে balance:karim modify    │
│      # করে থাকে → EXEC returns nil, retry            │
└──────────────────────────────────────────────────────┘
```

#### PHP (predis) — Transaction with WATCH

```php
<?php
require 'vendor/autoload.php';
use Predis\Client;

$redis = new Client(['host' => '127.0.0.1', 'port' => 6379]);

function transferMoney(Client $r, string $from, string $to, int $amt): bool {
    for ($i = 0; $i < 5; $i++) {  // retry up to 5 times
        $r->watch($from);
        $bal = (int)$r->get($from);
        if ($bal < $amt) {
            $r->unwatch();
            return false;
        }
        $resp = $r->transaction(function ($tx) use ($from, $to, $amt) {
            $tx->decrby($from, $amt);
            $tx->incrby($to, $amt);
        });
        if ($resp !== null) return true;  // EXEC succeeded
        // EXEC failed (WATCH key changed), retry
        usleep(10000);  // 10ms backoff
    }
    return false;
}
```

#### Node.js (ioredis)

```js
const Redis = require('ioredis');
const redis = new Redis();

async function transfer(from, to, amount) {
    for (let i = 0; i < 5; i++) {
        await redis.watch(from);
        const bal = parseInt(await redis.get(from)) || 0;
        if (bal < amount) { await redis.unwatch(); return false; }

        const result = await redis.multi()
            .decrby(from, amount)
            .incrby(to, amount)
            .exec();
        if (result !== null) return true;
        await new Promise(r => setTimeout(r, 10));
    }
    return false;
}
```

> ⚠️ **Redis transaction-এ rollback নেই।** EXEC-এর মাঝে কোনো command fail করলে বাকিগুলো execute হবে — তোমাকে নিজে compensate করতে হবে।

---

## 📜 Lua Scripting — Atomic Multi-Op

Redis embedded Lua interpreter চালায়। Script execution **atomic** (single-thread block), অন্য কোনো command সমান্তরালে চলে না।

### Sliding-window rate limiter (Lua)

```lua
-- rate_limit.lua
-- KEYS[1] = bucket key
-- ARGV[1] = window seconds, ARGV[2] = limit, ARGV[3] = now (ms), ARGV[4] = unique id
local key = KEYS[1]
local window = tonumber(ARGV[1]) * 1000
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)
if count >= limit then
    return 0
end
redis.call('ZADD', key, now, ARGV[4])
redis.call('PEXPIRE', key, window)
return 1
```

```bash
redis-cli SCRIPT LOAD "$(cat rate_limit.lua)"
# → "abc123def456..."
redis-cli EVALSHA abc123def456 1 user:42 60 100 1700000000000 req-xyz
```

#### PHP integration — bKash OTP rate limiter

```php
<?php
$lua = file_get_contents('rate_limit.lua');
$sha = $redis->script('LOAD', $lua);

function checkOtpLimit(string $msisdn): bool {
    global $redis, $sha;
    $now = (int)(microtime(true) * 1000);
    $allowed = $redis->evalsha(
        $sha,
        ['otp:limit:' . $msisdn],         // KEYS
        [3600, 5, $now, uniqid('', true)] // ARGV: 1h window, 5 OTP/h
    );
    return (bool)$allowed;
}

// bKash: একই নম্বরে 1 ঘণ্টায় সর্বোচ্চ 5 OTP
if (!checkOtpLimit('01711xxxxxx')) {
    http_response_code(429);
    exit('Too many OTP requests');
}
```

### Pipelining vs Transactions vs Lua

```
┌────────────────────────────────────────────────────────────────┐
│  Feature           │ Pipeline │ MULTI/EXEC │ Lua Script        │
├────────────────────────────────────────────────────────────────┤
│  Network round-trip│ 1        │ 1          │ 1                 │
│  Atomic execution  │ ❌       │ ✅          │ ✅                │
│  Conditional logic │ ❌       │ ❌ (WATCH) │ ✅ (full control)  │
│  Rollback          │ ❌       │ ❌          │ ❌                │
│  Use case          │ Bulk     │ Simple atom│ Complex atomic ops│
└────────────────────────────────────────────────────────────────┘
```

> 💡 Lua script যেন long না হয় — এটা single-thread block করে। 50ms-এর বেশি হলে অন্য সব client wait করবে।

---

## ⏰ TTL & Expiration

```bash
EXPIRE session:abc 3600        # 1 hour
PEXPIRE key 1000               # 1 second (ms)
EXPIREAT key 1700000000        # absolute unix timestamp
TTL session:abc                # → 3592 (seconds remaining)
PTTL session:abc               # → 3592145 (ms)
PERSIST session:abc            # remove TTL
SET key value EX 60            # set with TTL atomically
SET key value KEEPTTL          # update value, keep existing TTL (Redis 6+)
```

**Lazy + Active expiration:**
- **Lazy:** কেউ key access করলে check করে — expired হলে delete।
- **Active:** প্রতি সেকেন্ডে 10 বার, 20 random key sample করে — expired হলে delete। 25%-এর বেশি expired হলে আবার sample।

> ⚠️ TTL set না করে 10M keys রাখলে → memory ভরে গিয়ে eviction শুরু হবে (যদি `maxmemory-policy` থাকে), নইলে OOM।

---

## 🌐 Clustering & High Availability

### Redis Sentinel — Master/Replica + Auto-failover

Sentinel নিজেই data store নয় — এটা monitoring + failover orchestrator।

```
┌───────────────────────────────────────────────────────────┐
│              Sentinel-managed Redis Cluster               │
│                                                           │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│   │Sentinel 1│◀──▶│Sentinel 2│◀──▶│Sentinel 3│            │
│   │(quorum=2)│    └──────────┘    └──────────┘            │
│   └────┬─────┘         │               │                  │
│        │ monitor       │ monitor       │ monitor          │
│        ▼               ▼               ▼                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│   │  Master  │───▶│ Replica1 │    │ Replica2 │            │
│   │  :6379   │    │  :6380   │    │  :6381   │            │
│   └──────────┘    └──────────┘    └──────────┘            │
│                                                           │
│  Master ডাউন হলে:                                          │
│   1. Sentinels detect (down-after-milliseconds)            │
│   2. Quorum agree → SDOWN → ODOWN                         │
│   3. Leader Sentinel election (Raft-like)                 │
│   4. Best replica → promote to master                     │
│   5. Other replicas → reconfigure to follow new master    │
│   6. Clients notified via Sentinel API                    │
└───────────────────────────────────────────────────────────┘
```

```ini
# sentinel.conf (3 Sentinel nodes, quorum=2)
port 26379
sentinel monitor mymaster 10.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

```bash
redis-sentinel /etc/redis/sentinel.conf
```

#### Client-side (PHP predis with Sentinel)

```php
$redis = new Predis\Client([
    ['host' => '10.0.0.10', 'port' => 26379],
    ['host' => '10.0.0.11', 'port' => 26379],
    ['host' => '10.0.0.12', 'port' => 26379],
], ['replication' => 'sentinel', 'service' => 'mymaster']);
```

**কখন Sentinel?**
- Single master যথেষ্ট (data RAM-এ ফিট করে)।
- HA দরকার, কিন্তু sharding দরকার নেই।
- Foodpanda-র session store: 16GB, single master + 2 replicas + 3 Sentinels।

### Redis Cluster — Sharding + HA

16384 hash slots, প্রতিটি master কিছু slots-এর owner। Client gossip protocol দিয়ে topology জানে।

```
┌──────────────────────────────────────────────────────────────┐
│                    Redis Cluster Layout                      │
│                                                              │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│   │ Master A    │    │ Master B    │    │ Master C    │      │
│   │ slots 0-5460│    │5461-10922   │    │10923-16383  │      │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘      │
│          │                  │                  │             │
│   ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐      │
│   │ Replica A'  │    │ Replica B'  │    │ Replica C'  │      │
│   └─────────────┘    └─────────────┘    └─────────────┘      │
│                                                              │
│   Gossip protocol (port +10000) — node discovery + failure  │
│   detection + slot map propagation                          │
└──────────────────────────────────────────────────────────────┘

Key routing: slot = CRC16(key) mod 16384

GET user:42  → CRC16("user:42") mod 16384 = 7000 → Master B
GET user:99  → slot 14000 → Master C
```

#### Cluster setup

```bash
# 6 nodes (3 masters + 3 replicas) — port 7000-7005
for p in 7000 7001 7002 7003 7004 7005; do
    redis-server --port $p --cluster-enabled yes \
                 --cluster-config-file nodes-$p.conf \
                 --cluster-node-timeout 5000 \
                 --appendonly yes &
done

redis-cli --cluster create \
    127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
    127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
    --cluster-replicas 1
```

#### MOVED & ASK redirection

```bash
# Client connects to wrong node:
127.0.0.1:7000> GET user:42
(error) MOVED 7000 127.0.0.1:7001     # পুনরায় try করো 7001-এ

# Resharding চলাকালে:
(error) ASK 7000 127.0.0.1:7002       # এইবারের জন্য 7002-এ যাও
```

Smart clients (ioredis Cluster, Lettuce, Predis cluster) এই redirects automatically handle করে।

#### Hash tags — Multi-key ops in cluster

```bash
# different slots — fails
MGET user:42 product:9876   # CROSSSLOT error in cluster

# Force same slot using {tag}
SET {order:1001}:items "..."
SET {order:1001}:total 1500
MGET {order:1001}:items {order:1001}:total   # ✅ same slot
```

> 💡 Daraz cart system: একই user-এর সব cart items একই slot-এ রাখতে `cart:{user:42}:item:1`, `cart:{user:42}:item:2` — যাতে atomic ops সম্ভব।

#### Cluster commands

```bash
CLUSTER NODES
CLUSTER INFO
CLUSTER SLOTS
CLUSTER COUNTKEYSINSLOT 7000
CLUSTER GETKEYSINSLOT 7000 10

# Reshard
redis-cli --cluster reshard 127.0.0.1:7000 \
    --cluster-from <src-node-id> --cluster-to <dst-node-id> \
    --cluster-slots 1000 --cluster-yes

# Add node
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# Rebalance
redis-cli --cluster rebalance 127.0.0.1:7000
```

### Sentinel vs Cluster vs Single-master — Decision Table

| Feature | Single Master | Sentinel | Cluster |
|---------|---------------|----------|---------|
| **HA / Failover** | ❌ | ✅ Auto | ✅ Auto |
| **Sharding** | ❌ | ❌ | ✅ |
| **Max data** | RAM of 1 node | RAM of 1 node | Sum of all masters |
| **Multi-key ops** | ✅ All | ✅ All | ⚠️ Same slot only (hash tag) |
| **Pub/Sub** | ✅ | ✅ | ⚠️ Sharded pub/sub or use 1 master |
| **Lua across keys** | ✅ | ✅ | ⚠️ All keys same slot |
| **Client complexity** | Simple | Sentinel-aware | Cluster-aware |
| **Use case** | Dev / small cache | Most prod ≤ 50GB | Massive cache (>100GB) |
| **Ops complexity** | Low | Medium | High |

### Active-Active Geo Replication

- **Redis Enterprise (CRDT-based):** Conflict-free replicated data types — multi-master across regions, eventually consistent।
- **KeyDB:** Open-source multi-master replication।
- Use case: Global app — Daraz Pakistan + Bangladesh + Myanmar regions, regional latency কমাতে।

### Split-brain Risks

Network partition হলে দুই master active হতে পারে → conflicting writes।

```ini
min-replicas-to-write 1
min-replicas-max-lag 10
# অন্তত 1 replica ≤10s lag-এ না থাকলে master লিখতে দেবে না
# → split-brain prevention
```

---

## 🚀 Performance & Tuning

### maxmemory & lazyfree

```ini
maxmemory 8gb
maxmemory-policy allkeys-lfu

# DEL big key main thread block করে (1M-element list)
# UNLINK বা lazyfree-* — async delete
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
replica-lazy-flush yes
```

### Slow log

```bash
CONFIG SET slowlog-log-slower-than 10000   # 10ms
CONFIG SET slowlog-max-len 128
SLOWLOG GET 10
SLOWLOG RESET
```

### Latency monitoring

```bash
LATENCY DOCTOR              # human-readable diagnosis
LATENCY HISTORY event-loop
LATENCY LATEST
LATENCY RESET
```

### Connection pool sizing

```php
// Predis with pool (e.g., via predis-pool or persistent connection)
$redis = new Predis\Client([
    'scheme' => 'tcp',
    'host'   => '127.0.0.1',
    'port'   => 6379,
    'persistent' => true,
]);
```

```js
// ioredis defaults to single connection per client; for high concurrency:
const Redis = require('ioredis');
const redis = new Redis({
    host: '127.0.0.1',
    port: 6379,
    maxRetriesPerRequest: 3,
    enableAutoPipelining: true,   // batches commands automatically
});
```

**Rule of thumb:** PHP-FPM-এ pool size ≈ FPM workers count। Node.js-এ 1 connection + auto-pipelining যথেষ্ট 50k req/s পর্যন্ত।

### Pipelining — Latency hack

```php
$pipe = $redis->pipeline();
for ($i = 0; $i < 1000; $i++) {
    $pipe->set("k:$i", "v:$i");
}
$pipe->execute();   // 1 RTT, not 1000
```

10ms RTT × 1000 commands = 10s sequentially → 10ms with pipeline। 1000x speedup।

### BIG keys — silent killer

```bash
redis-cli --bigkeys
redis-cli --memkeys          # by memory
redis-cli MEMORY USAGE bigkey

# 1M-element hash → HGETALL takes seconds → blocks event loop
# → use HSCAN, or split into chunks
```

### SCAN over KEYS

```bash
# ❌ KEYS production-এ NEVER:
KEYS user:*                  # O(N), blocks server

# ✅ SCAN — cursor-based, non-blocking:
SCAN 0 MATCH user:* COUNT 100
# → returns next cursor; loop until cursor=0
```

```php
$cursor = '0';
do {
    [$cursor, $keys] = $redis->scan($cursor, ['MATCH' => 'user:*', 'COUNT' => 1000]);
    foreach ($keys as $k) { /* process */ }
} while ($cursor !== '0');
```

---

## 🔒 Security

### ACL (Redis 6+)

User-level fine-grained permission।

```bash
# users.acl
user default off
user webapp on >StrongPass! ~cache:* ~session:* +@read +@write -@dangerous
user metrics on >MetricsPass ~* +ping +info +client
user admin on >AdminPass ~* +@all

ACL LIST
ACL WHOAMI
ACL GETUSER webapp
```

```bash
redis-cli --user webapp --pass StrongPass!
```

### TLS

```ini
tls-port 6380
port 0                               # disable plaintext
tls-cert-file /etc/redis/redis.crt
tls-key-file /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes
```

### Legacy AUTH

```ini
requirepass MySuperSecret        # pre-Redis 6
```

### Rename dangerous commands

```ini
rename-command FLUSHALL ""           # disable
rename-command CONFIG "CFG_x9k2m"   # rename
rename-command DEBUG ""
rename-command SHUTDOWN "SHUTDOWN_x9k2m"
```

### Network isolation

```ini
bind 10.0.0.1 127.0.0.1            # internal IP only
protected-mode yes                 # default since 3.2
```

---

## 🎨 Common Patterns — Production Code

### 1. Cache-Aside (Lazy Loading) — Daraz product

```php
<?php
function getProduct(int $id): array {
    global $redis, $db;
    $key = "product:$id";
    $cached = $redis->get($key);
    if ($cached) return json_decode($cached, true);

    $product = $db->query("SELECT * FROM products WHERE id=?", [$id])->fetch();
    if ($product) {
        $redis->setex($key, 600, json_encode($product));   // 10 min
    }
    return $product;
}

function updateProduct(int $id, array $data): void {
    global $redis, $db;
    $db->update('products', $data, ['id' => $id]);
    $redis->del("product:$id");                  // invalidate
}
```

### 2. Write-Through

```js
async function saveProduct(p) {
    await db.query('INSERT INTO products SET ?', p);
    await redis.setex(`product:${p.id}`, 600, JSON.stringify(p));
    return p;
}
```

### 3. Distributed Lock — SET NX EX (Redlock-lite)

```lua
-- release.lua — only release if value matches (avoid releasing other's lock)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```php
function acquireLock(string $key, int $ttl): ?string {
    global $redis;
    $token = bin2hex(random_bytes(16));
    $ok = $redis->set($key, $token, ['NX', 'EX' => $ttl]);
    return $ok ? $token : null;
}

function releaseLock(string $key, string $token): bool {
    global $redis;
    static $sha;
    $sha ??= $redis->script('LOAD', file_get_contents('release.lua'));
    return (bool)$redis->evalsha($sha, [$key], [$token]);
}

// Chaldal flash sale stock decrement
$tok = acquireLock('lock:product:9876', 5);
if (!$tok) throw new RuntimeException('Locked');
try {
    $stock = (int)$redis->get('stock:9876');
    if ($stock <= 0) throw new RuntimeException('Out of stock');
    $redis->decr('stock:9876');
} finally {
    releaseLock('lock:product:9876', $tok);
}
```

> ⚠️ **Redlock caveat (Martin Kleppmann critique):** Single-instance lock-এ clock drift, GC pause, network partition — সব risky। Critical correctness-এর জন্য DB-backed fencing token + Redis lock, বা Zookeeper/etcd ব্যবহার করো। সাধারণ throughput protection-এর জন্য Redis lock যথেষ্ট।

### 4. Atomic Inventory Decrement (Lua)

```lua
-- decr_stock.lua
local stock = tonumber(redis.call('GET', KEYS[1]) or "0")
if stock <= 0 then return -1 end
redis.call('DECR', KEYS[1])
return stock - 1
```

```php
$remaining = $redis->eval(
    file_get_contents('decr_stock.lua'),
    1, 'stock:9876'
);
if ($remaining < 0) { /* sold out */ }
```

### 5. Sliding-window Rate Limiter (Sorted Set)

(উপরে Lua script দেখো — bKash OTP)

### 6. Token Bucket (Lua)

```lua
-- token_bucket.lua: KEYS[1]=key, ARGV: rate, capacity, now_ms, requested
local key = KEYS[1]
local rate = tonumber(ARGV[1])         -- tokens/sec
local cap  = tonumber(ARGV[2])
local now  = tonumber(ARGV[3])
local req  = tonumber(ARGV[4])
local data = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(data[1]) or cap
local ts     = tonumber(data[2]) or now
local delta  = math.max(0, now - ts) / 1000
tokens = math.min(cap, tokens + delta * rate)
local allowed = tokens >= req and 1 or 0
if allowed == 1 then tokens = tokens - req end
redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', key, math.ceil(cap / rate * 1000))
return allowed
```

### 7. Session store

**PHP — `session.save_handler=redis`:**
```ini
; php.ini
session.save_handler = redis
session.save_path    = "tcp://127.0.0.1:6379?auth=pass&database=1"
```

**Express + connect-redis:**
```js
const session = require('express-session');
const RedisStore = require('connect-redis').default;
app.use(session({
    store: new RedisStore({ client: redis, prefix: 'sess:' }),
    secret: process.env.SESSION_SECRET,
    saveUninitialized: false,
    resave: false,
    cookie: { maxAge: 86400000, secure: true, httpOnly: true },
}));
```

### 8. Job/Task Queue — BullMQ (Node)

```js
const { Queue, Worker } = require('bullmq');
const conn = { connection: { host: '127.0.0.1', port: 6379 } };

const emailQueue = new Queue('emails', conn);
await emailQueue.add('order-confirmation', { orderId: 1001, to: 'a@b.com' },
    { attempts: 5, backoff: { type: 'exponential', delay: 5000 } });

new Worker('emails', async job => {
    await sendEmail(job.data.to, job.name, job.data);
}, conn);
```

Laravel Queue Redis driver: `queue.php` → `'default' => 'redis'`। Behind the scenes RPUSH/BLPOP + Lua reliable queue।

### 9. Real-time Leaderboard

```php
// Daraz: top sellers এই মাসে
$redis->zincrby('leaderboard:2024-01', 1500, 'seller:42');
$top10 = $redis->zrevrange('leaderboard:2024-01', 0, 9, ['withscores' => true]);
$myRank = $redis->zrevrank('leaderboard:2024-01', 'seller:42');
```

### 10. Pub/Sub for service notifications

```js
// Publisher
await pub.publish('order.shipped', JSON.stringify({ orderId: 1001 }));

// Subscriber (e.g., SSE bridge)
sub.subscribe('order.shipped');
sub.on('message', (ch, msg) => sseClients.forEach(c => c.write(`data: ${msg}\n\n`)));
```

> 🔗 SSE pattern-এর জন্য `06-api-design/server-sent-events.md` দেখো।

### 11. Streams + Consumer Groups — Reliable processing

**Foodpanda real-time order events:**

```js
// Producer: POS app
await redis.xadd('orders:dhaka', '*',
    'orderId', '12345', 'status', 'placed', 'restaurant', 'r:99');

// Consumer: kitchen-display-svc
await redis.xgroup('CREATE', 'orders:dhaka', 'kds', '$', 'MKSTREAM');
while (true) {
    const res = await redis.xreadgroup('GROUP', 'kds', `c-${process.pid}`,
        'COUNT', 10, 'BLOCK', 5000, 'STREAMS', 'orders:dhaka', '>');
    if (!res) continue;
    for (const [stream, msgs] of res) {
        for (const [id, fields] of msgs) {
            try {
                await processOrder(fieldsToObj(fields));
                await redis.xack('orders:dhaka', 'kds', id);
            } catch (e) {
                // retry-able: leave pending; XCLAIM by janitor later
            }
        }
    }
}
```

### 12. Cache Stampede Protection

**Problem:** 10000 concurrent requests, hot key expire → সবাই DB hit।

**Solution A — single-flight lock:**
```php
function getCached(string $key, callable $loader, int $ttl): array {
    global $redis;
    $cached = $redis->get($key);
    if ($cached) return json_decode($cached, true);

    $lockKey = "lock:$key";
    $token = bin2hex(random_bytes(8));
    if ($redis->set($lockKey, $token, ['NX', 'EX' => 10])) {
        try {
            $val = $loader();
            $redis->setex($key, $ttl, json_encode($val));
            return $val;
        } finally {
            releaseLock($lockKey, $token);
        }
    }
    // অন্য কেউ load করছে — wait + retry
    usleep(50000);
    return getCached($key, $loader, $ttl);
}
```

**Solution B — probabilistic early expiration (XFetch):**
TTL-এর কাছাকাছি random হলে আগেই refresh — stampede prevent।

```js
function shouldRefresh(ttlMs, recompMs, beta = 1.0) {
    return Math.random() * recompMs * beta * Math.log(Math.random()) >= ttlMs;
}
```

---

## 🔀 Variants & Alternatives

| Product | License | Threading | Highlights |
|---------|---------|-----------|------------|
| **Redis OSS** (≤7.2) | BSD | Single-thread + I/O threads | Standard reference |
| **Redis (≥7.4)** | RSALv2/SSPL (source-available) | Same | License change Mar 2024 |
| **Valkey** | BSD-3 | Same as OSS | **Linux Foundation fork** of Redis 7.2 — community choice now |
| **KeyDB** | BSD | Multi-threaded master | 5x throughput, multi-master replication |
| **Dragonfly** | BSL | Multi-threaded, shared-nothing | 25x throughput claimed, RESP-compatible |
| **Garnet** (Microsoft) | MIT | .NET, multi-threaded | RESP-compatible, tiered storage |
| **AWS ElastiCache** | Managed | Redis OSS / Valkey | Cluster mode, encryption |
| **Azure Cache** | Managed | Redis Enterprise tier available | Geo-replication |
| **GCP Memorystore** | Managed | Redis / Valkey | Native VPC integration |
| **Upstash** | Managed serverless | REST + RESP | Per-request pricing — perfect for BD startups |

> 📌 **License timeline:** Redis 7.4+ moved to RSALv2/SSPL (cloud providers cannot resell)। AWS/GCP/Microsoft এর জবাবে **Valkey** fork (BSD) তৈরি করেছে। নতুন প্রজেক্টে Valkey বিবেচনা করো।

---

## 📊 Observability

### INFO

```bash
INFO server
INFO clients
INFO memory
INFO stats
INFO replication
INFO commandstats
INFO latencystats
INFO keyspace
```

### MONITOR (caution!)

```bash
MONITOR    # সব command realtime stream — production-এ massive perf hit
```

### Client list

```bash
CLIENT LIST
CLIENT KILL ID 42
CLIENT NO-EVICT on   # important client-এর data evict করবে না
```

### Memory stats

```bash
MEMORY STATS
MEMORY USAGE key
MEMORY DOCTOR
DEBUG OBJECT key      # encoding, refcount
```

### Prometheus + Grafana

```yaml
# docker-compose snippet
redis-exporter:
  image: oliver006/redis_exporter
  command: --redis.addr=redis://redis:6379 --redis.password=$REDIS_PASS
  ports: ["9121:9121"]
```

Grafana dashboard ID **763** (Redis Dashboard) — INFO + commandstats সব visualize।

**Key metrics to alert on:**
- `redis_memory_used_bytes / redis_memory_max_bytes > 0.9`
- `redis_connected_clients > 5000`
- `redis_commands_duration_seconds_total{cmd=...} P99 > 10ms`
- `redis_replication_lag_seconds > 5`
- `redis_evicted_keys_total` rate spike → cache too small
- `redis_keyspace_hits / (hits + misses) < 0.85` → low hit ratio

---

## 🚫 Anti-patterns

```
┌──────────────────────────────────────────────────────────────┐
│  ❌ KEYS *                       — production-এ NEVER         │
│  ❌ SUNIONSTORE on huge sets     — main thread block          │
│  ❌ HGETALL on million-field hash — same                     │
│  ❌ Big keys (>100KB string,      — uneven cluster, slowlog   │
│     >10k elements)                                           │
│  ❌ Pub/Sub on busy master       — blocking ops affect biz   │
│  ❌ No maxmemory-policy          — OOM in prod                │
│  ❌ Single-instance prod (no HA) — single point of failure    │
│  ❌ Redis as durable DB          — async replication, AOF     │
│     for financial truth             everysec → can lose 1s   │
│  ❌ Mixing cache + queue + DB    — different SLA contention   │
│     in same instance                                         │
│  ❌ FLUSHALL in prod              — disable via rename        │
│  ❌ Long Lua scripts             — block event loop          │
│  ❌ Forgetting TTL                — silent memory leak        │
│  ❌ Storing entire JSON blob     — use hash for partial    │
│     when you only update 1 field   updates                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 🇧🇩 Bangladesh Case Studies

### Daraz — Product/Category Cache + Cart Sessions
- **Layout:** ElastiCache Redis Cluster, 6 shards × (1 master + 1 replica), 96GB total RAM।
- **Patterns:**
  - Product catalog: `product:{id}` HSET, TTL 10min, cache-aside।
  - Category: `category:{slug}:page:{n}` JSON, TTL 5min।
  - Cart: `cart:{user:42}:item:{sku}` (hash tag-এ user → same slot, atomic ops)।
  - Hot products LFU cache, eviction policy `allkeys-lfu`।
- **Stampede protection:** Lua single-flight lock for top-100 products during sale।

### Pathao — Rider Availability + Geo
- `pathao:riders:available` SET — currently online riders।
- `pathao:riders:geo` — GEO sorted set with location।
- Trip request flow:
  ```
  GEOSEARCH pathao:riders:geo FROMLONLAT 90.41 23.81 BYRADIUS 3 km ASC COUNT 20
  → SINTER with pathao:riders:available
  → SINTER with pathao:riders:car_class:economy
  → notify top 5
  ```
- Per-rider lock: `lock:rider:{id}` SET NX EX 30 — যাতে দুই রিকোয়েস্ট একই rider-কে assign না হয়।

### bKash — OTP Rate Limiting
- Lua sliding-window: `otp:limit:{msisdn}` → 5/hour, 10/day।
- Failed attempts: `otp:fail:{msisdn}` INCR + EXPIRE — 5 fail → lock 15min।
- AOF `appendfsync always` — financial compliance।

### Foodpanda — Real-time Order Status
- Streams: `orders:{city}` → `kds`, `delivery`, `analytics` consumer groups।
- WebSocket fan-out via Redis Pub/Sub: `order:status:{id}` → user app + restaurant tablet + rider app।
- `XPENDING` monitoring — stuck consumer detect → PagerDuty alert।

### Chaldal — Flash Sale + Inventory
- Atomic stock decrement Lua: race-condition-free।
- Distributed lock for cart checkout per SKU।
- HyperLogLog: `flashsale:{date}:visitors` — unique visitor count।
- Bloom filter: "এই user এই sale-এ ইতিমধ্যে কিনেছে?" duplicate prevention।

---

## 📝 Summary

Redis = in-memory data structure server, single-threaded event loop, microsecond latency, persistence via RDB/AOF/hybrid, replication async, Sentinel for HA, Cluster for sharding, ACL+TLS for security, Lua for atomic ops, Streams for reliable messaging। সঠিক eviction policy, TTL discipline, big-key avoidance এবং proper observability — production Redis-এর ভিত্তি।

---

## ✅ Production Checklist

- [ ] **Persistence:** RDB + AOF (everysec) hybrid; cache-only হলে disable।
- [ ] **maxmemory** set + appropriate eviction policy (`allkeys-lru`/`-lfu`)।
- [ ] **TTL** discipline — every cache key-এ TTL।
- [ ] **No KEYS in prod** — SCAN ব্যবহার।
- [ ] **No FLUSHALL in prod** — rename-command।
- [ ] **HA:** Sentinel (3 nodes, quorum=2) বা Cluster (3+ masters)।
- [ ] **Replicas:** ≥2; `min-replicas-to-write 1` for split-brain protection।
- [ ] **Backups:** RDB to S3 hourly; off-site copy।
- [ ] **Monitoring:** redis-exporter + Grafana; alert on memory %, latency P99, evictions, replication lag, hit ratio।
- [ ] **Slowlog:** `slowlog-log-slower-than 10000` (10ms)।
- [ ] **Big keys audit:** weekly `redis-cli --bigkeys`।
- [ ] **Encoding tuning:** review `*-max-listpack-*` for memory savings।
- [ ] **Lazy free:** all `lazyfree-lazy-*` enabled।
- [ ] **Network:** bind to internal IP, `protected-mode yes`।
- [ ] **Auth:** ACL users (per-app), strong passwords, separate read/write users।
- [ ] **TLS:** enabled for cross-AZ/cross-VPC traffic।
- [ ] **Connection pooling** sized correctly (PHP-FPM workers / Node concurrency)।
- [ ] **Pipelining:** batch ops > 10 commands।
- [ ] **Lua scripts:** loaded via `SCRIPT LOAD` + `EVALSHA`; <50ms each।
- [ ] **Distributed lock:** SET NX EX with token + Lua release; consider fencing for correctness-critical।
- [ ] **Rate limiter:** Lua-based sliding window in place for OTP, login, write APIs।
- [ ] **Cache stampede:** single-flight lock or probabilistic refresh on hot keys।
- [ ] **Hash tags `{...}`** for multi-key ops in cluster।
- [ ] **READONLY** routing to replicas for read-heavy workloads।
- [ ] **CLIENT NO-EVICT** for critical pub/sub or admin clients।
- [ ] **Keyspace notifications** off unless used (overhead)।
- [ ] **Pub/Sub** on dedicated instance, not shared with hot business data।
- [ ] **Streams retention:** `XTRIM MAXLEN ~ N` to cap memory।
- [ ] **Cluster:** node-timeout tuned (5s default), gossip ports open।
- [ ] **Resharding:** scripted, off-peak only।
- [ ] **Sentinel:** quorum, down-after, failover-timeout, parallel-syncs sane।
- [ ] **Upgrade strategy:** rolling replica → failover → master।
- [ ] **OS tuning:** `vm.overcommit_memory=1`, transparent huge pages disabled, swap off।
- [ ] **CPU pinning** on dedicated host (NUMA aware)।
- [ ] **Disk:** SSD for AOF/RDB; separate from system disk।
- [ ] **Network:** ≥10Gbps for cluster nodes; low-latency private network।
- [ ] **Disaster recovery** runbook: restore from RDB, replica promotion।
- [ ] **License awareness:** Redis 7.4+ RSALv2/SSPL → consider Valkey for new projects।
- [ ] **Deprecation watch:** mirrored queues/keyspace events/etc. — track release notes।
- [ ] **Capacity plan:** RAM growth projection, sharding plan ahead of 70% utilization।
- [ ] **Chaos test:** kill master, kill Sentinel, partition network — verify failover।
- [ ] **Document** key naming conventions (e.g., `app:domain:entity:id:attr`)।
- [ ] **Audit log:** sensitive ACL changes via `CONFIG REWRITE` + git tracking।

---

> 🔗 সংশ্লিষ্ট ফাইল:
> - `/03-system-design/caching.md` — caching strategies overall।
> - `/03-system-design/message-queue.md` — broker comparison।
> - `/03-system-design/rabbitmq-amqp.md` — RabbitMQ deep dive।
> - `/18-kafka/kafka-fundamentals.md` — Kafka vs Redis Streams।
> - `/12-concurrency/saga-pattern.md` — distributed transaction।
> - `/05-database/nosql.md` — NoSQL landscape।
