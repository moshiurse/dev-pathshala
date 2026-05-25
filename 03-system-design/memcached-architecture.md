# 🧊 Memcached Architecture — In-Memory Caching Deep Dive

> **"Memcached হলো distributed caching-এর grandfather — সহজ, দ্রুত, এবং battle-tested।"**
>
> Facebook, Twitter, YouTube, Wikipedia — সব Memcached দিয়ে শুরু করেছিল। Redis জনপ্রিয় হলেও Memcached-এর নিজস্ব জায়গা আছে।

---

## 📌 সংজ্ঞা

**Memcached** (Memory Cache Daemon) হলো একটি high-performance, distributed, in-memory key-value cache system। এটি database বা API call-এর ফলাফল cache করে application-এর response time drastically কমায়।

### মূল দর্শন

```
┌──────────────────────────────────────────────────────────┐
│  Memcached Philosophy:                                    │
│                                                           │
│  ✅ এক কাজ, ভালোভাবে: key-value caching                  │
│  ✅ Simple protocol: GET, SET, DELETE                      │
│  ✅ Multi-threaded: CPU cores efficiently ব্যবহার         │
│  ✅ No persistence: শুধু RAM, disk নয়                     │
│  ✅ No replication: client-side distribution              │
│  ✅ LRU eviction: memory ভরলে পুরানো data মুছে যায়       │
│                                                           │
│  "Do one thing, do it fast, do it at scale."             │
└──────────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture

### Single Node Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   Memcached Server (Single Node)               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Network Layer                          │  │
│  │        TCP/UDP listener (port 11211)                     │  │
│  │        libevent-based event loop                         │  │
│  └────────────────────────┬────────────────────────────────┘  │
│                           │                                    │
│  ┌────────────────────────▼────────────────────────────────┐  │
│  │              Thread Pool (Worker Threads)                │  │
│  │                                                          │  │
│  │  Thread 1 │ Thread 2 │ Thread 3 │ Thread 4 │ ... │ N   │  │
│  │  ─────────────────────────────────────────────────────── │  │
│  │  প্রতিটি thread স্বতন্ত্রভাবে connection handle করে     │  │
│  │  CAS (Compare And Swap) lock-free operations            │  │
│  └────────────────────────┬────────────────────────────────┘  │
│                           │                                    │
│  ┌────────────────────────▼────────────────────────────────┐  │
│  │                Hash Table (Global)                       │  │
│  │                                                          │  │
│  │  key hash → bucket → linked list of items               │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │  │
│  │  │Bucket 0│ │Bucket 1│ │Bucket 2│ │Bucket N│           │  │
│  │  │ item→  │ │ item→  │ │ item→  │ │ item→  │           │  │
│  │  │  item→ │ │  NULL  │ │  item→ │ │  NULL  │           │  │
│  │  └────────┘ └────────┘ └────────┘ └────────┘           │  │
│  └────────────────────────┬────────────────────────────────┘  │
│                           │                                    │
│  ┌────────────────────────▼────────────────────────────────┐  │
│  │              Slab Allocator (Memory Management)          │  │
│  │                                                          │  │
│  │  Slab Class 1 (64B)  │ Slab Class 2 (128B) │ ... │ 1MB │  │
│  │  ┌──┐┌──┐┌──┐┌──┐    │ ┌───┐┌───┐┌───┐     │     │     │  │
│  │  │64││64││64││64│    │ │128││128││128│     │     │     │  │
│  │  └──┘└──┘└──┘└──┘    │ └───┘└───┘└───┘     │     │     │  │
│  │                       │                      │     │     │  │
│  │  প্রতিটি slab class নির্দিষ্ট size-এর chunk ধরে        │  │
│  │  Memory fragmentation prevent করে                       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                 LRU (Eviction)                           │  │
│  │                                                          │  │
│  │  HOT ←──── WARM ←──── COLD ←──── (evict from here)     │  │
│  │                                                          │  │
│  │  Per-slab-class LRU chain                               │  │
│  │  Segmented LRU (modern Memcached 1.6+)                  │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Distributed Architecture (Multi-Node)

Memcached নিজে কোনো distribution logic রাখে না — **client library** responsible।

```
┌──────────────────────────────────────────────────────────────┐
│              Memcached Distributed Architecture                │
│                                                               │
│  ┌────────────────────────────────────────────────────┐       │
│  │              Application Servers                     │       │
│  │                                                     │       │
│  │  App1, App2, App3 ... (all share same Memcached)   │       │
│  │  ┌──────────────────────────────────────────────┐  │       │
│  │  │         Memcached Client Library              │  │       │
│  │  │                                              │  │       │
│  │  │  key → hash(key) → server selection          │  │       │
│  │  │                                              │  │       │
│  │  │  Consistent Hashing Ring:                    │  │       │
│  │  │    hash(key) = 0x3A2F → MC Server 2          │  │       │
│  │  └──────────────────────────────────────────────┘  │       │
│  └───────────────────────┬────────────────────────────┘       │
│                          │                                     │
│            ┌─────────────┼─────────────┐                      │
│            ▼             ▼             ▼                       │
│     ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│     │MC Server 1 │ │MC Server 2 │ │MC Server 3 │             │
│     │ 32GB RAM   │ │ 32GB RAM   │ │ 32GB RAM   │             │
│     │ Keys A-G   │ │ Keys H-P   │ │ Keys Q-Z   │             │
│     └────────────┘ └────────────┘ └────────────┘             │
│                                                               │
│     মোট cache capacity = 96GB (additive!)                    │
│     Server 2 down → keys H-P → cache miss → DB fallback     │
│     ⚠️ কোনো replication নেই — server down = data gone       │
└──────────────────────────────────────────────────────────────┘
```

### Slab Allocator — Memory Management

```
┌──────────────────────────────────────────────────────────────┐
│                    Slab Allocation System                      │
│                                                               │
│  কেন Slab? → malloc/free করলে memory fragmentation হয়      │
│  Slab → fixed-size chunk → fragmentation শূন্য!              │
│                                                               │
│  Slab Class 1 (96 bytes):                                    │
│  ┌──────────────────────────────────────────────────┐        │
│  │ Page (1MB)                                       │        │
│  │ ┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐ ... ×10922│        │
│  │ │ 96B││ 96B││ 96B││ 96B││ 96B││ 96B│           │        │
│  │ └────┘└────┘└────┘└────┘└────┘└────┘           │        │
│  └──────────────────────────────────────────────────┘        │
│                                                               │
│  Slab Class 2 (120 bytes):                                   │
│  ┌──────────────────────────────────────────────────┐        │
│  │ Page (1MB)                                       │        │
│  │ ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐ ... ×8738 │        │
│  │ │120B ││120B ││120B ││120B ││120B │            │        │
│  │ └─────┘└─────┘└─────┘└─────┘└─────┘            │        │
│  └──────────────────────────────────────────────────┘        │
│                                                               │
│  ... Class N (1MB max item size)                             │
│                                                               │
│  Item 80 bytes → Slab Class 1 (96B)-তে যাবে                 │
│  Item 100 bytes → Slab Class 2 (120B)-তে যাবে               │
│  16 bytes wasted (internal fragmentation) — tradeoff         │
│                                                               │
│  Growth Factor (default 1.25):                               │
│    96 → 120 → 152 → 192 → 240 → 304 → ...                  │
└──────────────────────────────────────────────────────────────┘
```

---

## ⚡ Multi-threading Model

```
┌──────────────────────────────────────────────────────────────┐
│           Memcached Multi-threaded Architecture                │
│                                                               │
│  Main Thread (listener):                                     │
│    - TCP/UDP port-এ listen করে                               │
│    - নতুন connection accept করে                              │
│    - Worker thread-এ dispatch করে (round-robin)              │
│                                                               │
│  Worker Threads (N, default = 4):                            │
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │  Thread 1   │  │  Thread 2   │  │  Thread 3   │        │
│    │  conn A,D,G │  │  conn B,E,H │  │  conn C,F,I │        │
│    │  libevent   │  │  libevent   │  │  libevent   │        │
│    │  event loop │  │  event loop │  │  event loop │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                               │
│  Global Hash Table + Per-item locks:                         │
│    - Threads share hash table                                │
│    - Fine-grained locking (per-item, not per-table)          │
│    - CAS (Check And Set) for atomic updates                  │
│                                                               │
│  Redis vs Memcached threading:                               │
│    Redis: single-threaded command processing                 │
│    Memcached: multi-threaded → better CPU utilization        │
│              on multi-core machines                           │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 Memcached vs Redis

| বৈশিষ্ট্য | Memcached | Redis |
|-----------|-----------|-------|
| **Threading** | Multi-threaded | Single-threaded (6.0+ I/O threads) |
| **Data types** | String only (key-value) | String, List, Hash, Set, Sorted Set, Stream |
| **Persistence** | ❌ None | ✅ RDB + AOF |
| **Replication** | ❌ None (client-side) | ✅ Built-in master-slave |
| **Cluster** | Client-side sharding | Server-side cluster |
| **Max item size** | 1MB (default) | 512MB |
| **Memory efficiency** | Slab allocator (predictable) | jemalloc (flexible) |
| **Eviction** | LRU per slab class | LRU, LFU, volatile-* |
| **Pub/Sub** | ❌ | ✅ |
| **Scripting** | ❌ | ✅ Lua |
| **Transactions** | ❌ | ✅ MULTI/EXEC |
| **Use case** | Pure caching | Caching + data structure server |
| **Memory overhead** | কম (per-item ~56 bytes) | বেশি (per-key ~70+ bytes) |

### কখন Memcached ব্যবহার করবেন?

```
✅ Memcached Choose করুন যদি:
  - শুধু simple key-value caching দরকার
  - Multi-threaded performance চান (many cores)
  - Memory efficiency critical (uniform small objects)
  - Persistence দরকার নেই
  - Facebook-style large-scale caching pool

✅ Redis Choose করুন যদি:
  - Complex data structures দরকার (list, set, sorted set)
  - Persistence চান
  - Pub/Sub দরকার
  - Built-in replication/cluster চান
  - Lua scripting দরকার
```

---

## 🌐 Facebook's Memcached Architecture (TAO/mcrouter)

Facebook বিশ্বের সবচেয়ে বড় Memcached deployment চালায় — **trillions of requests/day**।

```
┌──────────────────────────────────────────────────────────────┐
│       Facebook Memcached Architecture (Simplified)             │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Web Server Tier                          │     │
│  │   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐               │     │
│  │   │Web 1│  │Web 2│  │Web 3│  │Web N│               │     │
│  │   └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘               │     │
│  └──────┼────────┼────────┼────────┼───────────────────┘     │
│         │        │        │        │                          │
│  ┌──────▼────────▼────────▼────────▼───────────────────┐     │
│  │              mcrouter (Routing Proxy)                 │     │
│  │   - Consistent hashing                              │     │
│  │   - Connection pooling                              │     │
│  │   - Replication (for hot keys)                      │     │
│  │   - Failover routing                                │     │
│  └──────┬────────────────────────────────┬─────────────┘     │
│         │                                │                    │
│  ┌──────▼──────────────┐    ┌────────────▼────────────┐      │
│  │  Frontend Cluster   │    │  Backend Cluster         │      │
│  │  (hot data, small)  │    │  (warm/cold data, large) │      │
│  │  MC1, MC2, MC3      │    │  MC4, MC5, ... MC100     │      │
│  └─────────────────────┘    └─────────────────────────┘      │
│                                                               │
│  Multi-Region:                                               │
│  ┌────────────────┐   lease ──►  ┌────────────────┐         │
│  │ Region A       │              │ Region B       │         │
│  │ (Master)       │   invalidate │ (Replica)      │         │
│  │ MC + MySQL     │ ◄──────────  │ MC + MySQL     │         │
│  └────────────────┘              └────────────────┘         │
│                                                               │
│  Key innovations:                                            │
│  1. Lease tokens — thundering herd prevention               │
│  2. mcrouter — intelligent routing layer                    │
│  3. Gutter pool — failover servers for down nodes           │
│  4. Multi-region invalidation via MySQL replication          │
└──────────────────────────────────────────────────────────────┘
```

### Lease Mechanism (Thundering Herd Prevention)

```
┌──────────────────────────────────────────────────────────────┐
│  Problem: Thundering Herd                                     │
│                                                               │
│  Cache miss → 1000 concurrent requests → ALL hit DB → 💀    │
│                                                               │
│  Solution: Lease Token                                        │
│                                                               │
│  1. Client A: GET key → MISS + lease_token_123               │
│  2. Client B: GET key → MISS + "wait, lease exists"          │
│  3. Client C: GET key → MISS + "wait, lease exists"          │
│  4. Client A: SET key value (with lease_token_123) → OK      │
│  5. Client B, C: GET key → HIT ✅                            │
│                                                               │
│  শুধু একটি client DB query করে, বাকিরা wait করে             │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP Code Example

```php
<?php
// Memcached client with consistent hashing and failover

class CacheManager
{
    private \Memcached $mc;

    public function __construct(array $servers)
    {
        $this->mc = new \Memcached('pool');
        $this->mc->setOption(\Memcached::OPT_DISTRIBUTION, \Memcached::DISTRIBUTION_CONSISTENT);
        $this->mc->setOption(\Memcached::OPT_LIBKETAMA_COMPATIBLE, true);
        $this->mc->setOption(\Memcached::OPT_REMOVE_FAILED_SERVERS, true);
        $this->mc->setOption(\Memcached::OPT_RETRY_TIMEOUT, 2);
        $this->mc->setOption(\Memcached::OPT_CONNECT_TIMEOUT, 100);  // 100ms
        $this->mc->setOption(\Memcached::OPT_NO_BLOCK, true);

        // persistent connection pool — connection reuse
        if (!count($this->mc->getServerList())) {
            $this->mc->addServers($servers);
            // [['mc1', 11211, 33], ['mc2', 11211, 33], ['mc3', 11211, 34]]
        }
    }

    /**
     * Cache-aside pattern with stampede prevention
     */
    public function getOrSet(string $key, callable $loader, int $ttl = 3600): mixed
    {
        $value = $this->mc->get($key);

        if ($this->mc->getResultCode() === \Memcached::RES_SUCCESS) {
            return $value; // Cache hit
        }

        // CAS-based lock to prevent thundering herd
        $lockKey = "lock:{$key}";
        $token = uniqid('', true);

        if ($this->mc->add($lockKey, $token, 5)) {
            // Got the lock — we fetch from DB
            try {
                $value = $loader();
                $this->mc->set($key, $value, $ttl);
            } finally {
                // Release lock (CAS to ensure we own it)
                $this->mc->delete($lockKey);
            }
            return $value;
        }

        // Someone else is fetching — wait and retry
        usleep(50000); // 50ms
        $value = $this->mc->get($key);
        return $value ?: $loader(); // fallback if still miss
    }

    /**
     * Multi-get for batch operations
     */
    public function getMulti(array $keys): array
    {
        $result = $this->mc->getMulti($keys);
        return $result ?: [];
    }

    /**
     * Increment with auto-initialization
     */
    public function increment(string $key, int $step = 1, int $initial = 0, int $ttl = 0): int|false
    {
        $result = $this->mc->increment($key, $step, $initial, $ttl);
        return $result;
    }

    /**
     * Delete with multi-server broadcast (for replicated keys)
     */
    public function delete(string $key): bool
    {
        return $this->mc->delete($key);
    }
}

// Usage — Daraz product cache
$cache = new CacheManager([
    ['mc-node1.internal', 11211, 33],
    ['mc-node2.internal', 11211, 33],
    ['mc-node3.internal', 11211, 34], // weight 34 — বেশি RAM
]);

// Cache-aside with stampede prevention
$product = $cache->getOrSet("product:12345", function () {
    return DB::table('products')
        ->join('categories', 'products.category_id', '=', 'categories.id')
        ->where('products.id', 12345)
        ->first();
}, ttl: 1800);

// Counter
$cache->increment("page_views:product:12345");

// Multi-get (batch)
$products = $cache->getMulti(['product:1', 'product:2', 'product:3']);
```

---

## 💻 JavaScript Code Example

```javascript
import Memcached from 'memcached';

class MemcachedCache {
  constructor(servers, options = {}) {
    // Consistent hashing built into the client
    this.client = new Memcached(servers, {
      retries: 2,
      retry: 1000,
      remove: true,         // remove dead servers
      failOverServers: options.failover || [],
      timeout: 100,         // 100ms connection timeout
      idle: 30000,          // idle connection timeout
      poolSize: 20,         // connection pool size per server
    });
  }

  async get(key) {
    return new Promise((resolve, reject) => {
      this.client.get(key, (err, data) => {
        if (err) return reject(err);
        resolve(data); // undefined if miss
      });
    });
  }

  async set(key, value, ttl = 3600) {
    return new Promise((resolve, reject) => {
      this.client.set(key, value, ttl, (err) => {
        if (err) return reject(err);
        resolve(true);
      });
    });
  }

  /**
   * Cache-aside with lock-based stampede prevention
   */
  async getOrLoad(key, loader, ttl = 3600) {
    const cached = await this.get(key);
    if (cached !== undefined) return cached;

    // Attempt lock
    const lockKey = `lock:${key}`;
    const locked = await this.add(lockKey, '1', 5);

    if (locked) {
      try {
        const value = await loader();
        await this.set(key, value, ttl);
        return value;
      } finally {
        await this.del(lockKey);
      }
    }

    // Wait for other loader
    await new Promise(r => setTimeout(r, 50));
    const retry = await this.get(key);
    return retry !== undefined ? retry : loader();
  }

  async getMulti(keys) {
    return new Promise((resolve, reject) => {
      this.client.getMulti(keys, (err, data) => {
        if (err) return reject(err);
        resolve(data || {});
      });
    });
  }

  async add(key, value, ttl) {
    return new Promise((resolve) => {
      this.client.add(key, value, ttl, (err) => resolve(!err));
    });
  }

  async del(key) {
    return new Promise((resolve) => {
      this.client.del(key, () => resolve(true));
    });
  }

  async incr(key, amount = 1) {
    return new Promise((resolve, reject) => {
      this.client.incr(key, amount, (err, result) => {
        if (err) return reject(err);
        resolve(result);
      });
    });
  }
}

// Usage
const cache = new MemcachedCache(
  ['mc1:11211', 'mc2:11211', 'mc3:11211'],
  { failover: ['mc-backup1:11211'] }
);

// Cache-aside
const product = await cache.getOrLoad('product:12345', async () => {
  return db.query('SELECT * FROM products WHERE id = ?', [12345]);
}, 1800);

// Batch fetch
const products = await cache.getMulti(['product:1', 'product:2', 'product:3']);
```

---

## 🔧 Production Configuration

```bash
# /etc/memcached.conf (Production settings)

# Memory allocation
-m 32768          # 32GB RAM
-I 2m             # Max item size 2MB (default 1MB)

# Threading
-t 8              # 8 worker threads (match CPU cores)
-R 200            # Max requests per event (default 20)

# Networking
-p 11211          # TCP port
-U 0              # Disable UDP (security)
-l 10.0.0.5       # Bind to internal IP only
-c 4096           # Max connections (default 1024)
-b 2048           # Connection backlog

# Slab allocator
-f 1.25           # Growth factor (chunk size multiplier)
-n 48             # Minimum item size (bytes)
-L                # Use large memory pages (hugepages)

# Security
-S                # Enable SASL authentication

# Performance
-o modern,no_modern       # Modern features
-o slab_reassign          # Enable slab rebalancing
-o slab_automove=1        # Auto slab rebalancing
-o lru_maintainer         # Background LRU maintenance
-o lru_crawler            # Expired item cleanup
-o hash_algorithm=xxhash  # Faster hashing
```

---

## 📈 Performance Characteristics

```
┌──────────────────────────────────────────────────────────┐
│  Memcached Performance (typical production):              │
│                                                           │
│  Latency:                                                │
│    - GET: 0.1 - 0.5ms (same datacenter)                 │
│    - SET: 0.1 - 0.5ms                                    │
│    - Multi-GET (100 keys): 0.5 - 2ms                    │
│                                                           │
│  Throughput (per server, 8 cores):                       │
│    - GET: 200K - 1M ops/sec                              │
│    - SET: 200K - 800K ops/sec                            │
│    - Mix (80% GET, 20% SET): 500K ops/sec               │
│                                                           │
│  Memory efficiency:                                      │
│    - Overhead per item: ~56 bytes                        │
│    - 32GB server, 1KB avg item → ~30M items             │
│                                                           │
│  Cache hit ratio target: ≥ 95%                           │
│  Eviction rate target: < 10 evictions/sec                │
└──────────────────────────────────────────────────────────┘
```

---

## ⚠️ Common Pitfalls ও Solutions

| সমস্যা | কারণ | সমাধান |
|---------|------|--------|
| **Slab imbalance** | কিছু slab class full, কিছু empty | `slab_reassign` + `slab_automove` |
| **Thundering herd** | Cache miss-এ সবাই DB hit করে | Lease token / lock pattern |
| **Hot key** | একটি key অতিরিক্ত request পায় | Client-side local cache / replicate key |
| **Cold start** | Restart-এ সব cache empty | Warm-up script / gradual traffic shift |
| **Large item** | 1MB limit exceed | Chunk করে store / item size বাড়ান |
| **Connection exhaustion** | Too many app servers | Connection pooling / mcrouter |

---

## 🔑 মনে রাখার বিষয়

1. **Simple but powerful** — Memcached শুধু key-value, কিন্তু এটাই এর শক্তি
2. **Multi-threaded** — Redis-এর চেয়ে multi-core-এ better raw throughput
3. **No persistence** — Server restart = data gone; critical data-র জন্য Redis
4. **Client-side sharding** — Consistent hashing client library-তে হয়
5. **Slab allocator** — Memory fragmentation prevent করে, কিন্তু internal waste আছে
6. **Stampede prevention** — Production-এ lease/lock pattern mandatory
7. **Facebook scale** — Billions of requests/day serve করতে পারে mcrouter দিয়ে
