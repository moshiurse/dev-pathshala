# 🌐 Distributed Cache Consistency

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [Redis Cluster vs Redis Sentinel](#redis-cluster-vs-redis-sentinel)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII Diagram](#ascii-diagram)
- [Consistent Hashing](#consistent-hashing)
- [Hot Key Problem](#hot-key-problem)
- [Cache Penetration, Breakdown, Avalanche](#cache-penetration-breakdown-avalanche)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Distributed Cache** হলো একাধিক server/node-এ বিভক্ত (partitioned) এবং replicated cache system যা high availability, scalability, এবং fault tolerance প্রদান করে। একটি single Redis server-এর memory limit (সাধারণত 64-256GB) অতিক্রম করতে এবং single point of failure এড়াতে distributed caching ব্যবহার করা হয়।

### 🔑 মূল Concepts:

| Concept | বর্ণনা |
|---|---|
| **Partitioning/Sharding** | Data কে multiple nodes-এ ভাগ করা |
| **Replication** | প্রতিটি partition-এর copy রাখা (fault tolerance) |
| **Consistent Hashing** | Node add/remove-এ minimal data redistribution |
| **Split-Brain** | Network partition-এ দুটি "master" তৈরি হওয়া |
| **Quorum** | Majority agreement for read/write |
| **Gossip Protocol** | Nodes পরস্পরকে health/state জানানো |

### 📊 Redis Cluster vs Sentinel vs Standalone:

| Feature | Standalone | Sentinel | Cluster |
|---|---|---|---|
| **Capacity** | Single node | Single node (replicated) | Multiple nodes (sharded) |
| **HA** | ❌ No | ✅ Auto-failover | ✅ Auto-failover |
| **Scalability** | ❌ Vertical only | ❌ Vertical only | ✅ Horizontal |
| **Data Distribution** | Single | Replicated | Sharded + Replicated |
| **Multi-key ops** | ✅ | ✅ | ⚠️ Same slot only |
| **Use Case** | Dev/Small | Medium traffic | Large scale |

### 📊 Memcached vs Redis:

| Feature | Memcached | Redis |
|---|---|---|
| **Data Structures** | Key-Value only | Strings, Lists, Sets, Hashes, Sorted Sets, Streams |
| **Persistence** | ❌ | ✅ RDB + AOF |
| **Replication** | ❌ | ✅ Master-Replica |
| **Clustering** | Client-side | Server-side (Redis Cluster) |
| **Memory Efficiency** | ✅ Better (slab allocator) | ⚠️ Slightly more overhead |
| **Multi-threaded** | ✅ | ⚠️ Single-threaded (I/O threads in 6.0+) |
| **Pub/Sub** | ❌ | ✅ |
| **Lua Scripting** | ❌ | ✅ |
| **Use Case** | Simple caching | Caching + Data structures + Pub/Sub |

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🛒 Daraz Flash Sale Scenario:

```
সমস্যা:
- Flash Sale: iPhone 15 মাত্র ৫০,০০০ টাকা (৫০% ছাড়!)
- Stock: মাত্র ১০০টি unit
- ১০ লক্ষ user একই সময়ে product page load করছে
- সব request একই cache key "product:iphone15" hit করছে

Challenge:
- একটি Redis node-এ সব traffic → HOT KEY problem!
- Node overloaded হলে সব user-ই product দেখতে পাচ্ছে না
- Stock count cache ও real stock mismatch (overselling!)
- Flash sale শেষ হলে cache invalidation → thundering herd

Real Numbers:
- Peak QPS: ৫০,০০০ requests/second একটি key-তে
- Single Redis node max: ~১০০,০০০ QPS (সব key মিলে)
- এক key-তে ৫০% traffic → performance degradation
```

### 💰 bKash Distributed Balance Cache:

```
সমস্যা:
- ৫ কোটি+ users
- প্রতি সেকেন্ডে ১০,০০০+ transactions
- Balance data multiple datacenter-এ cached
- Dhaka DC আর Chittagong DC-এ balance different হলে?
- Network partition (split-brain) হলে কী হবে?

Design:
- Redis Cluster: 6 masters + 6 replicas (2 datacenters)
- Balance read: Local datacenter-এর replica থেকে
- Balance write: Master node-এ (cross-DC replication)
- Critical ops: Quorum read (majority agreement)
```

### 📱 Grameenphone Session Cache:

```
- ৮ কোটি subscriber
- MyGP app session data distributed cache-এ
- User Dhaka-তে login করেছে → Chittagong-এ গেলেও session valid
- Cross-datacenter session replication needed
- Cache node fail হলে user log out হওয়া যাবে না
```

---

## 📊 ASCII Diagram

### Redis Cluster Architecture:

```
┌──────────────────────────────────────────────────────────────┐
│                    REDIS CLUSTER (6 nodes)                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Hash Slots: 0 ────────────────────────────────── 16383      │
│              |← Master-1 →|← Master-2 →|← Master-3 →|       │
│              [0-5460]      [5461-10922]  [10923-16383]        │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Master-1   │  │  Master-2   │  │  Master-3   │          │
│  │ Slots:0-5460│  │Slots:5461-  │  │Slots:10923- │          │
│  │             │  │     10922   │  │     16383   │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                 │                 │                 │
│         ▼                 ▼                 ▼                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Replica-1  │  │  Replica-2  │  │  Replica-3  │          │
│  │(Master-1    │  │(Master-2    │  │(Master-3    │          │
│  │  backup)    │  │  backup)    │  │  backup)    │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                               │
│  Key → Slot mapping: CRC16(key) % 16384                      │
│  "product:123" → CRC16("product:123") % 16384 = 8721        │
│  → Goes to Master-2 (slot 5461-10922)                        │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Redis Sentinel Architecture:

```
┌──────────────────────────────────────────────────────────────┐
│                    REDIS SENTINEL                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐            │
│  │ Sentinel-1│    │ Sentinel-2│    │ Sentinel-3│            │
│  │(monitor)  │    │(monitor)  │    │(monitor)  │            │
│  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘            │
│        │                 │                 │                  │
│        └────────────┬────┴────────────────┘                  │
│                     │ Monitoring                              │
│                     ▼                                         │
│            ┌──────────────┐                                   │
│            │    Master    │ ◄── All writes go here            │
│            │  (Primary)   │                                   │
│            └──────┬───────┘                                   │
│                   │ Replication                               │
│         ┌─────────┼─────────┐                                │
│         ▼         ▼         ▼                                │
│  ┌──────────┐┌──────────┐┌──────────┐                       │
│  │ Replica-1││ Replica-2││ Replica-3│ ◄── Reads can go here │
│  └──────────┘└──────────┘└──────────┘                       │
│                                                               │
│  Master fail হলে:                                             │
│  1. Sentinel detect করে (quorum vote)                         │
│  2. একটি Replica কে Master promote করে                       │
│  3. অন্য Replica গুলো নতুন Master-এ point করে                │
│  4. Client নতুন Master-এর address পায়                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Consistent Hashing Ring:

```
┌──────────────────────────────────────────────────────────────┐
│                CONSISTENT HASHING RING                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│                      0°                                        │
│                      │                                        │
│              Node-C ●│● key-1                                 │
│                 /    │    \                                    │
│               /      │      \                                 │
│        270° /        │        \ 90°                           │
│       ─────●─────────┼─────────●─────                        │
│      Node-B \        │        / Node-A                       │
│               \      │      / ● key-3                        │
│                 \    │    /                                    │
│              key-2 ●│●                                        │
│                      │                                        │
│                    180°                                        │
│                                                               │
│  Mapping: key → closest node clockwise                        │
│  key-1 → Node-C (next clockwise)                             │
│  key-2 → Node-B (next clockwise)                             │
│  key-3 → Node-A (next clockwise)                             │
│                                                               │
│  Node-D add হলে:                                              │
│  শুধু Node-D এর আগের keys move হবে                           │
│  (NOT all keys!) → Minimal redistribution                    │
│                                                               │
│  Virtual Nodes (150 per physical node):                       │
│  Node-A → vnode-A1, vnode-A2, ... vnode-A150                 │
│  → Even distribution of keys                                  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Split-Brain Problem:

```
┌──────────────────────────────────────────────────────────────┐
│                    SPLIT-BRAIN PROBLEM                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│         Dhaka DC                    Chittagong DC             │
│  ┌─────────────────┐        ┌─────────────────┐             │
│  │   Master-1      │   ╳    │   Replica-1     │             │
│  │   (original)    │ Network│   (promoted to  │             │
│  │                 │Partition│    Master!)     │             │
│  │   balance:5000  │   ╳    │   balance:4500  │             │
│  └─────────────────┘        └─────────────────┘             │
│         │                          │                         │
│         ▼                          ▼                         │
│  Client-A writes                Client-B writes              │
│  balance = 4000                 balance = 3000               │
│                                                              │
│  ⚠️ TWO masters accepting writes!                            │
│  ⚠️ Data divergence!                                         │
│                                                              │
│  Network recover হলে:                                         │
│  - কার data সঠিক? Dhaka=4000 নাকি CTG=3000?                 │
│  - Data loss inevitable (one side's writes lost)             │
│                                                              │
│  সমাধান:                                                      │
│  1. min-replicas-to-write = 1 (majority needed)              │
│  2. Fencing tokens (older master can't write)                │
│  3. Manual resolution for critical data                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 Hot Key Problem

### সমস্যা:

```
┌──────────────────────────────────────────────────────────────┐
│                    HOT KEY PROBLEM                             │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Daraz Flash Sale: "product:iphone15"                         │
│                                                               │
│  Normal Distribution:                                         │
│  ┌────────┐  ┌────────┐  ┌────────┐                         │
│  │Node-1  │  │Node-2  │  │Node-3  │                         │
│  │33% load│  │33% load│  │33% load│                         │
│  └────────┘  └────────┘  └────────┘                         │
│                                                               │
│  With Hot Key (Flash Sale):                                   │
│  ┌────────┐  ┌────────┐  ┌────────┐                         │
│  │Node-1  │  │Node-2  │  │Node-3  │                         │
│  │10% load│  │80% load│  │10% load│  ← Node-2 overloaded!  │
│  └────────┘  └─┬──────┘  └────────┘                         │
│                 │                                             │
│     "product:iphone15" lives on Node-2                        │
│     ৫০,০০০ req/s hitting ONE node!                           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### সমাধান:

```
┌──────────────────────────────────────────────────────────────┐
│              HOT KEY SOLUTIONS                                 │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Solution 1: Local Cache (L1)                                 │
│  ┌────────────────────────────────────┐                      │
│  │ App Server                          │                      │
│  │ ┌──────────────┐                   │                      │
│  │ │ Local Cache  │ ← 1s TTL          │                      │
│  │ │ (in-memory)  │                   │                      │
│  │ └──────┬───────┘                   │                      │
│  │        │ Miss only                  │                      │
│  │        ▼                            │                      │
│  │    Redis Cluster                    │                      │
│  └────────────────────────────────────┘                      │
│  → ৯৯% requests local cache-এ serve হবে                      │
│                                                               │
│  Solution 2: Key Splitting/Replication                         │
│  ┌────────────────────────────────────┐                      │
│  │ Original: "product:iphone15"       │                      │
│  │                                     │                      │
│  │ Split into:                         │                      │
│  │ "product:iphone15:shard:0" → Node-1│                      │
│  │ "product:iphone15:shard:1" → Node-2│                      │
│  │ "product:iphone15:shard:2" → Node-3│                      │
│  │                                     │                      │
│  │ Read: random shard select           │                      │
│  │ Write: all shards update            │                      │
│  └────────────────────────────────────┘                      │
│  → Load সমানভাবে distribute হবে                              │
│                                                               │
│  Solution 3: Read Replicas                                    │
│  ┌────────────────────────────────────┐                      │
│  │ Master: writes only                 │                      │
│  │ Replica-1: reads (33%)             │                      │
│  │ Replica-2: reads (33%)             │                      │
│  │ Replica-3: reads (33%)             │                      │
│  └────────────────────────────────────┘                      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💥 Cache Penetration, Breakdown, Avalanche

### তিনটি সমস্যা ও সমাধান:

```
┌──────────────────────────────────────────────────────────────┐
│  CACHE PENETRATION (অস্তিত্বহীন data-র query)               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Attacker: GET /product/99999999 (exists না)                  │
│                                                               │
│  Cache: MISS → DB: NOT FOUND → Cache: nothing stored          │
│  → Every request hits DB! (bypass cache completely)           │
│                                                               │
│  সমাধান:                                                      │
│  1. Bloom Filter: request আসার আগেই check করো exist করে কিনা│
│  2. Null caching: না পেলেও null cache করো (short TTL)        │
│  3. Request validation: invalid ID format reject              │
│                                                               │
│  ┌──────┐    ┌───────────┐    ┌───────┐    ┌─────┐          │
│  │Client│──►│Bloom Filter│──►│ Cache │──►│ DB  │          │
│  │      │    │ Exists? NO│    │       │    │     │          │
│  │      │◄──│ → REJECT  │    │       │    │     │          │
│  └──────┘    └───────────┘    └───────┘    └─────┘          │
│                                                               │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  CACHE BREAKDOWN (Hot key expire)                             │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Popular key "flash_sale:iphone" expire হলো!                  │
│  → হাজার হাজার concurrent request DB-তে!                     │
│                                                               │
│  সমাধান:                                                      │
│  1. Mutex lock: একজনই DB query করবে                          │
│  2. Never expire: hot keys-এ TTL দেবেন না, manual invalidate │
│  3. Logical expiration: value-র ভেতরে expiry time রাখুন      │
│                                                               │
│  Logical Expiration:                                          │
│  cache_value = {                                              │
│    "data": { actual product data },                           │
│    "logical_expiry": 1700000000,                              │
│    "redis_ttl": NEVER (or very long)                          │
│  }                                                            │
│  → Redis key never expires                                    │
│  → App checks logical_expiry, refreshes in background         │
│                                                               │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  CACHE AVALANCHE (অনেক key একসাথে expire)                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  সমস্যা: Deploy-এর পর সব cache set করলাম TTL=3600            │
│  → ১ ঘণ্টা পর সব expire → massive DB load!                   │
│                                                               │
│  ┌──────────────────────────────────────┐                    │
│  │   Time ─────────────────────────►    │                    │
│  │                                      │                    │
│  │   TTL set    All expire!   DB crash! │                    │
│  │   ──────────|────────────|────────── │                    │
│  │   t=0       t=3600       t=3601      │                    │
│  └──────────────────────────────────────┘                    │
│                                                               │
│  সমাধান:                                                      │
│  1. Jittered TTL: TTL = base + random(0, 600)                │
│  2. Multi-level cache: L1 expire হলে L2 ধরবে                │
│  3. Circuit breaker: DB overload detect করলে fallback        │
│  4. Rate limiting: DB-তে concurrent queries limit            │
│  5. Warm-up: High-traffic keys কখনো expire হতে দেবেন না     │
│                                                               │
│  Jittered Expiry:                                             │
│  ┌──────────────────────────────────────┐                    │
│  │  Key-1: TTL = 3600 + 120 = 3720     │                    │
│  │  Key-2: TTL = 3600 + 45  = 3645     │                    │
│  │  Key-3: TTL = 3600 + 580 = 4180     │                    │
│  │  Key-4: TTL = 3600 + 200 = 3800     │                    │
│  │  → Spread out over 10 minutes!       │                    │
│  └──────────────────────────────────────┘                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Distributed Cache Service
 * Daraz Flash Sale - Hot Key Protection + Cache Patterns
 */

class DistributedCacheService
{
    private array $redisNodes = [];
    private $localCache = [];
    private int $virtualNodes = 150;
    private array $hashRing = [];
    
    public function __construct(array $redisConfigs)
    {
        foreach ($redisConfigs as $config) {
            $this->redisNodes[$config['name']] = new \Redis();
            $this->redisNodes[$config['name']]->connect($config['host'], $config['port']);
        }
        $this->buildHashRing();
    }
    
    // ===========================
    // CONSISTENT HASHING
    // ===========================
    
    /**
     * Hash Ring তৈরি করা
     * Virtual nodes ব্যবহার করে even distribution
     */
    private function buildHashRing(): void
    {
        $this->hashRing = [];
        
        foreach (array_keys($this->redisNodes) as $nodeName) {
            for ($i = 0; $i < $this->virtualNodes; $i++) {
                $virtualNodeKey = "{$nodeName}:vnode:{$i}";
                $hash = crc32($virtualNodeKey);
                $this->hashRing[$hash] = $nodeName;
            }
        }
        
        ksort($this->hashRing); // Sort by hash position
    }
    
    /**
     * Key-এর জন্য সঠিক node খুঁজে বের করা
     */
    private function getNodeForKey(string $key): string
    {
        $keyHash = crc32($key);
        
        // Clockwise-এ পরবর্তী node খুঁজুন
        foreach ($this->hashRing as $hash => $nodeName) {
            if ($hash >= $keyHash) {
                return $nodeName;
            }
        }
        
        // Wrap around (ring-এর শুরুতে যান)
        return reset($this->hashRing);
    }
    
    /**
     * Distributed GET - consistent hashing ব্যবহার করে
     */
    public function get(string $key): ?string
    {
        $node = $this->getNodeForKey($key);
        return $this->redisNodes[$node]->get($key) ?: null;
    }
    
    /**
     * Distributed SET - consistent hashing ব্যবহার করে
     */
    public function set(string $key, string $value, int $ttl = 3600): bool
    {
        $node = $this->getNodeForKey($key);
        return $this->redisNodes[$node]->setex($key, $ttl, $value);
    }
    
    // ===========================
    // HOT KEY PROTECTION
    // ===========================
    
    /**
     * Hot Key Read - Local Cache + Key Splitting
     * Daraz flash sale product page
     */
    public function getHotKey(string $key, int $shardCount = 5): ?string
    {
        // Strategy 1: Local in-memory cache (L1)
        $localKey = "local:{$key}";
        if (isset($this->localCache[$localKey])) {
            $cached = $this->localCache[$localKey];
            if ($cached['expires_at'] > time()) {
                return $cached['value']; // L1 hit! No network call
            }
            unset($this->localCache[$localKey]);
        }
        
        // Strategy 2: Random shard selection (distribute load)
        $shardIndex = mt_rand(0, $shardCount - 1);
        $shardKey = "{$key}:shard:{$shardIndex}";
        
        $node = $this->getNodeForKey($shardKey);
        $value = $this->redisNodes[$node]->get($shardKey);
        
        if ($value !== false) {
            // L1 cache set (very short TTL - 1-2 seconds)
            $this->localCache[$localKey] = [
                'value' => $value,
                'expires_at' => time() + 2
            ];
            return $value;
        }
        
        return null;
    }
    
    /**
     * Hot Key Write - সব shards-এ write করা
     */
    public function setHotKey(string $key, string $value, int $shardCount = 5, int $ttl = 60): void
    {
        for ($i = 0; $i < $shardCount; $i++) {
            $shardKey = "{$key}:shard:{$i}";
            $node = $this->getNodeForKey($shardKey);
            // Jittered TTL (avalanche prevention)
            $jitter = mt_rand(0, (int)($ttl * 0.2));
            $this->redisNodes[$node]->setex($shardKey, $ttl + $jitter, $value);
        }
        
        // Local cache invalidate
        unset($this->localCache["local:{$key}"]);
    }
    
    // ===========================
    // CACHE PENETRATION PROTECTION
    // ===========================
    
    /**
     * Bloom Filter based cache penetration protection
     * Non-existent product query block করা
     */
    public function getWithBloomFilter(string $key, callable $dbFetchFn): ?string
    {
        // Step 1: Bloom filter check (probabilistic, no false negatives)
        if (!$this->bloomFilterContains($key)) {
            return null; // Definitely doesn't exist - don't hit DB
        }
        
        // Step 2: Cache check
        $cached = $this->get($key);
        if ($cached !== null) {
            if ($cached === '__NULL__') return null; // Cached null
            return $cached;
        }
        
        // Step 3: DB fetch with mutex
        $lockKey = "lock:{$key}";
        $node = $this->getNodeForKey($lockKey);
        $locked = $this->redisNodes[$node]->set($lockKey, '1', ['NX', 'EX' => 5]);
        
        if (!$locked) {
            usleep(100000); // 100ms wait
            return $this->get($key);
        }
        
        try {
            $value = $dbFetchFn($key);
            
            if ($value === null) {
                // Null caching (short TTL to prevent penetration)
                $this->set($key, '__NULL__', 60);
                return null;
            }
            
            $this->set($key, $value, 3600);
            return $value;
        } finally {
            $this->redisNodes[$node]->del($lockKey);
        }
    }
    
    /**
     * Simple Bloom Filter implementation
     */
    private function bloomFilterContains(string $key): bool
    {
        $bloomKey = 'bloom:products';
        $node = $this->getNodeForKey($bloomKey);
        
        // Multiple hash functions
        $hashes = [
            crc32($key) % 100000,
            crc32("salt1:{$key}") % 100000,
            crc32("salt2:{$key}") % 100000,
        ];
        
        foreach ($hashes as $bit) {
            if (!$this->redisNodes[$node]->getBit($bloomKey, $bit)) {
                return false; // Definitely not in set
            }
        }
        
        return true; // Probably in set (may be false positive)
    }
    
    public function bloomFilterAdd(string $key): void
    {
        $bloomKey = 'bloom:products';
        $node = $this->getNodeForKey($bloomKey);
        
        $hashes = [
            crc32($key) % 100000,
            crc32("salt1:{$key}") % 100000,
            crc32("salt2:{$key}") % 100000,
        ];
        
        foreach ($hashes as $bit) {
            $this->redisNodes[$node]->setBit($bloomKey, $bit, 1);
        }
    }
    
    // ===========================
    // CACHE-ASIDE + WRITE-THROUGH HYBRID
    // ===========================
    
    /**
     * Hybrid Pattern: Read = Cache-Aside, Write = Write-Through
     * bKash balance management
     */
    public function hybridRead(string $key, callable $dbFetchFn): ?string
    {
        // Cache-Aside: cache থেকে পড়ো, miss হলে DB
        $cached = $this->get($key);
        if ($cached !== null) {
            return $cached;
        }
        
        // DB fetch
        $value = $dbFetchFn($key);
        if ($value !== null) {
            $this->set($key, $value, 3600);
        }
        
        return $value;
    }
    
    /**
     * Write-Through: DB ও Cache দুটোতেই write
     */
    public function hybridWrite(string $key, string $value, callable $dbWriteFn): bool
    {
        // Step 1: DB write first (source of truth)
        $success = $dbWriteFn($key, $value);
        
        if (!$success) {
            return false;
        }
        
        // Step 2: Cache update (write-through)
        $this->set($key, $value, 3600);
        
        // Step 3: Invalidate related caches
        $this->invalidateRelated($key);
        
        return true;
    }
    
    // ===========================
    // CROSS-DATACENTER SYNC
    // ===========================
    
    /**
     * Cross-DC cache replication
     * Dhaka DC → Chittagong DC sync
     */
    public function crossDCWrite(string $key, string $value, int $ttl = 3600): array
    {
        $results = [];
        
        // Write to local DC
        $localNode = $this->getNodeForKey($key);
        $this->redisNodes[$localNode]->setex($key, $ttl, $value);
        $results['local'] = 'written';
        
        // Async replication to remote DC
        $replicationPayload = json_encode([
            'key' => $key,
            'value' => $value,
            'ttl' => $ttl,
            'source_dc' => 'dhaka',
            'timestamp' => microtime(true),
        ]);
        
        // Push to replication queue (processed by background worker)
        $this->redisNodes[$localNode]->rpush('cross_dc_replication_queue', $replicationPayload);
        $results['remote'] = 'queued';
        
        return $results;
    }
    
    /**
     * Cross-DC replication worker
     * Background process: remote DC-তে data push করে
     */
    public function processCrossDCReplication(array $remoteDCRedis): int
    {
        $processed = 0;
        $localNode = array_key_first($this->redisNodes);
        
        while (true) {
            $item = $this->redisNodes[$localNode]->lpop('cross_dc_replication_queue');
            if ($item === null) break;
            
            $payload = json_decode($item, true);
            
            try {
                $remoteDCRedis->setex(
                    $payload['key'],
                    $payload['ttl'],
                    $payload['value']
                );
                $processed++;
            } catch (\Exception $e) {
                // Re-queue failed items
                $this->redisNodes[$localNode]->rpush('cross_dc_replication_queue', $item);
                break;
            }
        }
        
        return $processed;
    }
    
    private function invalidateRelated(string $key): void
    {
        // Related keys invalidation logic
    }
}

/**
 * Flash Sale Cache Manager
 * Daraz flash sale specific optimizations
 */
class FlashSaleCacheManager
{
    private DistributedCacheService $cache;
    private $db;
    
    public function __construct(DistributedCacheService $cache, $db)
    {
        $this->cache = $cache;
        $this->db = $db;
    }
    
    /**
     * Flash sale product - stock count management
     * Atomic decrement with distributed lock
     */
    public function decrementStock(string $productId): array
    {
        $stockKey = "flash_sale:stock:{$productId}";
        
        // Read current stock from hot key (with local cache)
        $currentStock = (int) $this->cache->getHotKey($stockKey);
        
        if ($currentStock <= 0) {
            return ['success' => false, 'reason' => 'sold_out'];
        }
        
        // Atomic decrement (Redis DECR is atomic)
        $node = $this->cache->get($stockKey); // Get correct node
        // In real implementation, use Redis DECR directly
        $newStock = $currentStock - 1;
        
        if ($newStock < 0) {
            return ['success' => false, 'reason' => 'race_condition_sold_out'];
        }
        
        // Update all shards
        $this->cache->setHotKey($stockKey, (string) $newStock, 5, 300);
        
        return ['success' => true, 'remaining_stock' => $newStock];
    }
    
    /**
     * Pre-warm flash sale cache
     * Sale শুরুর আগে cache warm করা
     */
    public function prewarmFlashSale(string $saleId): array
    {
        $products = $this->db->query(
            "SELECT * FROM flash_sale_products WHERE sale_id = ?",
            [$saleId]
        );
        
        $warmed = [];
        foreach ($products as $product) {
            $productKey = "flash_sale:product:{$product['id']}";
            $stockKey = "flash_sale:stock:{$product['id']}";
            
            // Product data → hot key (multiple shards)
            $this->cache->setHotKey($productKey, json_encode($product), 5, 600);
            
            // Stock count → hot key
            $this->cache->setHotKey($stockKey, (string) $product['stock'], 5, 600);
            
            // Bloom filter-এ add করো
            $this->cache->bloomFilterAdd($productKey);
            
            $warmed[] = $product['id'];
        }
        
        return $warmed;
    }
}

// ব্যবহার উদাহরণ:
$cache = new DistributedCacheService([
    ['name' => 'node-1', 'host' => 'redis-1.daraz.internal', 'port' => 6379],
    ['name' => 'node-2', 'host' => 'redis-2.daraz.internal', 'port' => 6379],
    ['name' => 'node-3', 'host' => 'redis-3.daraz.internal', 'port' => 6379],
]);

// Hot key read (Daraz flash sale)
$product = $cache->getHotKey('flash_sale:product:iphone15');

// Cache penetration protection
$product = $cache->getWithBloomFilter('product:99999', function($key) use ($db) {
    return $db->query("SELECT * FROM products WHERE id = ?", [str_replace('product:', '', $key)]);
});

// Hybrid pattern (bKash balance)
$balance = $cache->hybridRead('balance:user123', function($key) use ($db) {
    return $db->query("SELECT balance FROM users WHERE id = ?", ['user123']);
});
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Distributed Cache Manager
 * Complete implementation with all patterns
 * Example: Daraz E-commerce Platform
 */

const Redis = require('ioredis');
const crypto = require('crypto');

class ConsistentHashRing {
    constructor(nodes, virtualNodesPerNode = 150) {
        this.ring = new Map(); // hash → node
        this.sortedHashes = [];
        this.nodes = new Set(nodes);
        this.virtualNodesPerNode = virtualNodesPerNode;
        
        this.build();
    }

    build() {
        this.ring.clear();
        this.sortedHashes = [];
        
        for (const node of this.nodes) {
            for (let i = 0; i < this.virtualNodesPerNode; i++) {
                const virtualKey = `${node}:vnode:${i}`;
                const hash = this.hash(virtualKey);
                this.ring.set(hash, node);
                this.sortedHashes.push(hash);
            }
        }
        
        this.sortedHashes.sort((a, b) => a - b);
    }

    hash(key) {
        const md5 = crypto.createHash('md5').update(key).digest('hex');
        return parseInt(md5.substring(0, 8), 16);
    }

    getNode(key) {
        if (this.sortedHashes.length === 0) return null;
        
        const keyHash = this.hash(key);
        
        // Binary search for next node clockwise
        let low = 0, high = this.sortedHashes.length - 1;
        
        while (low < high) {
            const mid = Math.floor((low + high) / 2);
            if (this.sortedHashes[mid] < keyHash) {
                low = mid + 1;
            } else {
                high = mid;
            }
        }
        
        // Wrap around
        const index = this.sortedHashes[low] >= keyHash ? low : 0;
        return this.ring.get(this.sortedHashes[index]);
    }

    /**
     * Node add/remove - minimal redistribution
     */
    addNode(node) {
        this.nodes.add(node);
        this.build();
    }

    removeNode(node) {
        this.nodes.delete(node);
        this.build();
    }
}

class DistributedCacheManager {
    constructor(nodeConfigs) {
        this.clients = new Map();
        this.localCache = new Map(); // L1 in-process cache
        this.localCacheTTL = 2000; // 2 seconds
        
        const nodeNames = nodeConfigs.map(c => c.name);
        this.hashRing = new ConsistentHashRing(nodeNames);
        
        // Redis connections
        for (const config of nodeConfigs) {
            this.clients.set(config.name, new Redis({
                host: config.host,
                port: config.port,
                retryStrategy: (times) => Math.min(times * 50, 2000)
            }));
        }
    }

    getClient(key) {
        const nodeName = this.hashRing.getNode(key);
        return this.clients.get(nodeName);
    }

    // ===========================
    // BASIC OPERATIONS
    // ===========================

    async get(key) {
        const client = this.getClient(key);
        return await client.get(key);
    }

    async set(key, value, ttl = 3600) {
        const client = this.getClient(key);
        // Jittered TTL (cache avalanche prevention)
        const jitter = Math.floor(Math.random() * ttl * 0.1);
        await client.setex(key, ttl + jitter, value);
    }

    async del(key) {
        const client = this.getClient(key);
        await client.del(key);
    }

    // ===========================
    // HOT KEY MANAGEMENT
    // ===========================

    /**
     * Hot Key Read with L1 Cache + Sharding
     * Daraz Flash Sale: ৫০,০০০ req/s একটি product-এ
     */
    async getHotKey(key, options = {}) {
        const { shardCount = 5, localTTL = 2000 } = options;

        // L1: In-process cache (fastest — no network)
        const localEntry = this.localCache.get(key);
        if (localEntry && localEntry.expiresAt > Date.now()) {
            return localEntry.value;
        }

        // L2: Random shard read (distribute across nodes)
        const shardIndex = Math.floor(Math.random() * shardCount);
        const shardKey = `${key}:shard:${shardIndex}`;
        
        const client = this.getClient(shardKey);
        const value = await client.get(shardKey);

        if (value) {
            // Update L1 cache
            this.localCache.set(key, {
                value,
                expiresAt: Date.now() + localTTL
            });
        }

        return value;
    }

    /**
     * Hot Key Write - সব shards-এ atomic update
     */
    async setHotKey(key, value, options = {}) {
        const { shardCount = 5, ttl = 60 } = options;

        const pipeline = [];
        for (let i = 0; i < shardCount; i++) {
            const shardKey = `${key}:shard:${i}`;
            const client = this.getClient(shardKey);
            const jitter = Math.floor(Math.random() * ttl * 0.2);
            pipeline.push(client.setex(shardKey, ttl + jitter, value));
        }

        await Promise.all(pipeline);

        // Invalidate local cache
        this.localCache.delete(key);
    }

    // ===========================
    // CACHE PENETRATION PROTECTION
    // ===========================

    /**
     * Bloom Filter + Null Caching
     * Invalid product ID request block
     */
    async getWithPenetrationProtection(key, fetchFn) {
        // Step 1: Bloom filter check
        const mightExist = await this.bloomFilterCheck(key);
        if (!mightExist) {
            return null; // Definitely doesn't exist
        }

        // Step 2: Cache read
        const cached = await this.get(key);
        if (cached !== null) {
            if (cached === '__NULL__') return null; // Cached negative
            return JSON.parse(cached);
        }

        // Step 3: Mutex lock → DB fetch
        const lockKey = `mutex:${key}`;
        const lockClient = this.getClient(lockKey);
        const lockId = `${process.pid}-${Date.now()}-${Math.random()}`;
        
        const acquired = await lockClient.set(lockKey, lockId, 'NX', 'EX', 5);

        if (acquired === 'OK') {
            try {
                const data = await fetchFn(key);
                
                if (data === null || data === undefined) {
                    // Null caching (prevent repeated penetration)
                    await this.set(key, '__NULL__', 60);
                    return null;
                }

                await this.set(key, JSON.stringify(data), 3600);
                return data;
            } finally {
                // Release lock (only if we still own it)
                const script = `
                    if redis.call("get", KEYS[1]) == ARGV[1] then
                        return redis.call("del", KEYS[1])
                    else
                        return 0
                    end
                `;
                await lockClient.eval(script, 1, lockKey, lockId);
            }
        }

        // Lock না পেলে wait ও retry
        await new Promise(resolve => setTimeout(resolve, 100));
        const retryValue = await this.get(key);
        if (retryValue && retryValue !== '__NULL__') {
            return JSON.parse(retryValue);
        }
        return null;
    }

    async bloomFilterCheck(key) {
        const bloomKey = 'bloom:all_products';
        const client = this.getClient(bloomKey);
        
        const hashes = [
            crypto.createHash('md5').update(key).digest('hex'),
            crypto.createHash('md5').update(`s1:${key}`).digest('hex'),
            crypto.createHash('md5').update(`s2:${key}`).digest('hex'),
        ];

        for (const hash of hashes) {
            const bit = parseInt(hash.substring(0, 8), 16) % 1000000;
            const exists = await client.getbit(bloomKey, bit);
            if (!exists) return false;
        }
        return true; // Probably exists
    }

    async bloomFilterAdd(key) {
        const bloomKey = 'bloom:all_products';
        const client = this.getClient(bloomKey);
        
        const hashes = [
            crypto.createHash('md5').update(key).digest('hex'),
            crypto.createHash('md5').update(`s1:${key}`).digest('hex'),
            crypto.createHash('md5').update(`s2:${key}`).digest('hex'),
        ];

        const pipeline = client.pipeline();
        for (const hash of hashes) {
            const bit = parseInt(hash.substring(0, 8), 16) % 1000000;
            pipeline.setbit(bloomKey, bit, 1);
        }
        await pipeline.exec();
    }

    // ===========================
    // CACHE AVALANCHE PREVENTION
    // ===========================

    /**
     * Batch set with jittered TTL
     * Deploy-এর পর mass cache warm করতে
     */
    async batchSetWithJitter(items, baseTTL = 3600) {
        const promises = items.map(({ key, value }) => {
            // Random jitter: ±20% of base TTL
            const jitter = Math.floor(Math.random() * baseTTL * 0.4) - (baseTTL * 0.2);
            const finalTTL = Math.max(60, baseTTL + jitter);
            
            return this.set(key, JSON.stringify(value), finalTTL);
        });

        await Promise.all(promises);
    }

    /**
     * Circuit Breaker for DB protection
     * Cache avalanche-এ DB crash prevention
     */
    createCircuitBreaker(options = {}) {
        const {
            failureThreshold = 5,
            recoveryTimeout = 30000,
            halfOpenRequests = 3
        } = options;

        let state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        let failures = 0;
        let lastFailureTime = 0;
        let halfOpenAttempts = 0;

        return {
            async execute(fn) {
                if (state === 'OPEN') {
                    if (Date.now() - lastFailureTime > recoveryTimeout) {
                        state = 'HALF_OPEN';
                        halfOpenAttempts = 0;
                    } else {
                        throw new Error('Circuit breaker OPEN - DB protected');
                    }
                }

                if (state === 'HALF_OPEN' && halfOpenAttempts >= halfOpenRequests) {
                    state = 'OPEN';
                    throw new Error('Circuit breaker OPEN - half-open failed');
                }

                try {
                    const result = await fn();
                    
                    if (state === 'HALF_OPEN') {
                        halfOpenAttempts++;
                        if (halfOpenAttempts >= halfOpenRequests) {
                            state = 'CLOSED';
                            failures = 0;
                        }
                    } else {
                        failures = 0;
                    }
                    
                    return result;
                } catch (error) {
                    failures++;
                    lastFailureTime = Date.now();
                    
                    if (failures >= failureThreshold) {
                        state = 'OPEN';
                    }
                    throw error;
                }
            },
            
            getState: () => state
        };
    }

    // ===========================
    // CACHE-ASIDE + WRITE-THROUGH HYBRID
    // ===========================

    /**
     * Hybrid Pattern for critical data
     * bKash balance: consistency + performance
     */
    async hybridGet(key, fetchFn) {
        // L1 local cache
        const local = this.localCache.get(key);
        if (local && local.expiresAt > Date.now()) {
            return local.value;
        }

        // L2 distributed cache (cache-aside read)
        const cached = await this.get(key);
        if (cached) {
            const value = JSON.parse(cached);
            this.localCache.set(key, { value, expiresAt: Date.now() + 5000 });
            return value;
        }

        // L3 database
        const value = await fetchFn(key);
        if (value !== null) {
            await this.set(key, JSON.stringify(value), 3600);
            this.localCache.set(key, { value, expiresAt: Date.now() + 5000 });
        }
        return value;
    }

    async hybridSet(key, value, writeFn) {
        // Write-through: DB first, then cache
        await writeFn(key, value);
        await this.set(key, JSON.stringify(value), 3600);
        
        // Invalidate L1
        this.localCache.delete(key);
        
        // Publish invalidation (other servers)
        const pubClient = this.getClient('__invalidation_channel__');
        await pubClient.publish('cache:invalidate', JSON.stringify({ key }));
    }

    // ===========================
    // MONITORING & HEALTH
    // ===========================

    /**
     * Cache health metrics
     */
    async getClusterHealth() {
        const health = {};
        
        for (const [name, client] of this.clients) {
            try {
                const info = await client.info('memory');
                const keyspace = await client.info('keyspace');
                const stats = await client.info('stats');
                
                health[name] = {
                    status: 'healthy',
                    usedMemory: info.match(/used_memory_human:(.+)/)?.[1]?.trim(),
                    keys: keyspace.match(/keys=(\d+)/)?.[1] || '0',
                    hitRate: this.parseHitRate(stats),
                };
            } catch (error) {
                health[name] = { status: 'unhealthy', error: error.message };
            }
        }
        
        return health;
    }

    parseHitRate(stats) {
        const hits = parseInt(stats.match(/keyspace_hits:(\d+)/)?.[1] || '0');
        const misses = parseInt(stats.match(/keyspace_misses:(\d+)/)?.[1] || '0');
        const total = hits + misses;
        return total > 0 ? ((hits / total) * 100).toFixed(2) + '%' : 'N/A';
    }

    /**
     * Hot key detection
     * কোন keys সবচেয়ে বেশি access হচ্ছে
     */
    async detectHotKeys(sampleSize = 100) {
        const keyFrequency = new Map();
        
        for (const [name, client] of this.clients) {
            // Redis MONITOR বা OBJECT FREQ ব্যবহার করে
            // Simplified: random key sampling
            for (let i = 0; i < sampleSize; i++) {
                const key = await client.randomkey();
                if (key) {
                    const freq = await client.object('FREQ', key).catch(() => 0);
                    keyFrequency.set(key, (keyFrequency.get(key) || 0) + freq);
                }
            }
        }

        // Top 10 hot keys
        return [...keyFrequency.entries()]
            .sort((a, b) => b[1] - a[1])
            .slice(0, 10)
            .map(([key, freq]) => ({ key, frequency: freq }));
    }
}

// ====== ব্যবহার উদাহরণ ======

const cacheManager = new DistributedCacheManager([
    { name: 'redis-dhaka-1', host: '10.0.1.1', port: 6379 },
    { name: 'redis-dhaka-2', host: '10.0.1.2', port: 6379 },
    { name: 'redis-ctg-1', host: '10.0.2.1', port: 6379 },
    { name: 'redis-ctg-2', host: '10.0.2.2', port: 6379 },
]);

// === Daraz Flash Sale ===

// Pre-warm flash sale products
const flashSaleProducts = [
    { key: 'product:iphone15', value: { name: 'iPhone 15', price: 50000, stock: 100 } },
    { key: 'product:samsung-s24', value: { name: 'Samsung S24', price: 45000, stock: 200 } },
];

await cacheManager.batchSetWithJitter(flashSaleProducts, 600);

// Hot key read during sale (৫০,০০০ req/s)
const product = await cacheManager.getHotKey('product:iphone15', {
    shardCount: 10,  // ১০টি shard-এ distribute
    localTTL: 1000   // 1s local cache
});

// Stock decrement
const stockKey = 'stock:iphone15';
const client = cacheManager.getClient(stockKey);
const remaining = await client.decr(stockKey); // Atomic!
if (remaining < 0) {
    await client.incr(stockKey); // Rollback
    console.log('Sold out!');
}

// === Cache Penetration Protection ===
const productData = await cacheManager.getWithPenetrationProtection(
    'product:invalid-id-99999',
    async (key) => {
        // DB query
        return await db.findProduct(key.replace('product:', ''));
    }
);

// === Circuit Breaker (Avalanche Protection) ===
const breaker = cacheManager.createCircuitBreaker({
    failureThreshold: 10,
    recoveryTimeout: 60000
});

try {
    const data = await breaker.execute(async () => {
        return await db.query('SELECT * FROM products WHERE id = ?', [productId]);
    });
} catch (error) {
    if (error.message.includes('Circuit breaker')) {
        // Serve stale cache or default response
        console.log('DB protected by circuit breaker, serving degraded response');
        return { message: 'Service temporarily degraded' };
    }
}

// === Health Check ===
const health = await cacheManager.getClusterHealth();
console.log('Cluster health:', JSON.stringify(health, null, 2));

// === Hot Key Detection ===
const hotKeys = await cacheManager.detectHotKeys();
console.log('Hot keys detected:', hotKeys);
// Auto-shard if frequency too high
for (const { key, frequency } of hotKeys) {
    if (frequency > 10000) {
        console.log(`Auto-sharding hot key: ${key}`);
        const value = await cacheManager.get(key);
        if (value) {
            await cacheManager.setHotKey(key, value, { shardCount: 10 });
        }
    }
}
```

---

## 🔄 Cross-Datacenter Cache Synchronization

```
┌──────────────────────────────────────────────────────────────┐
│        CROSS-DATACENTER CACHE SYNC                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│     Dhaka Datacenter              Chittagong Datacenter       │
│  ┌─────────────────────┐      ┌─────────────────────┐       │
│  │  ┌──────┐ ┌──────┐  │      │  ┌──────┐ ┌──────┐  │       │
│  │  │Redis │ │Redis │  │      │  │Redis │ │Redis │  │       │
│  │  │Master│ │Replica│  │      │  │Master│ │Replica│  │       │
│  │  └──┬───┘ └──────┘  │      │  └──┬───┘ └──────┘  │       │
│  │     │                │      │     │                │       │
│  │  ┌──┴───────────┐   │      │  ┌──┴───────────┐   │       │
│  │  │ Replication  │   │      │  │ Replication  │   │       │
│  │  │ Queue (Kafka)│◄──┼──────┼──│ Queue (Kafka)│   │       │
│  │  └──────────────┘   │      │  └──────────────┘   │       │
│  └─────────────────────┘      └─────────────────────┘       │
│                                                               │
│  Sync Strategy:                                               │
│  1. Write → Local DC Redis                                    │
│  2. Publish to Kafka topic "cache-sync"                       │
│  3. Remote DC consumer reads from Kafka                       │
│  4. Remote DC Redis updated                                   │
│                                                               │
│  Conflict Resolution:                                         │
│  - Last-Write-Wins (timestamp comparison)                     │
│  - For counters: CRDT (merge, don't overwrite)               │
│  - For critical data: Primary DC wins                         │
│                                                               │
│  Latency: ~50-200ms (Dhaka ↔ Chittagong)                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## ✅❌ কখন ব্যবহার করবেন / করবেন না

### ✅ Distributed Cache ব্যবহার করবেন:

| Scenario | Solution | উদাহরণ |
|---|---|---|
| Data > single node memory | Redis Cluster (sharding) | Daraz product catalog |
| High availability needed | Sentinel or Cluster | bKash balance cache |
| Multi-region deployment | Cross-DC replication | Grameenphone subscriber data |
| Hot key problem | Key splitting + L1 cache | Flash sale products |
| Cache penetration attack | Bloom filter + null caching | Invalid API requests |
| Traffic spikes (flash sale) | Pre-warming + auto-scaling | Daraz 11.11 sale |

### ❌ Distributed Cache এড়িয়ে চলুন:

| Scenario | Alternative | কারণ |
|---|---|---|
| Small dataset (< 1GB) | Single Redis instance | Complexity অপ্রয়োজনীয় |
| Strong consistency required | Database (no cache) | Cache inherently eventual |
| Simple key-value (no HA) | Memcached single node | Simpler, faster |
| Very transient data (< 1s) | In-process cache only | Network overhead > benefit |
| Write-heavy, read-rare | Database directly | Cache will always be stale |

### 🎯 Technology Selection:

```
┌──────────────────────────────────────────────────────────┐
│               WHEN TO USE WHAT?                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Simple caching (< 10GB):                                │
│  → Redis Standalone + Sentinel for HA                    │
│    Example: Pathao driver location cache                 │
│                                                          │
│  Large scale (> 10GB, horizontal scaling):               │
│  → Redis Cluster (auto-sharding)                         │
│    Example: Daraz full product catalog                   │
│                                                          │
│  Ultra-high throughput, simple data:                      │
│  → Memcached (multi-threaded)                            │
│    Example: Session storage, simple counters             │
│                                                          │
│  Complex data structures + Pub/Sub:                      │
│  → Redis (Sorted Sets, Streams, Pub/Sub)                 │
│    Example: Leaderboards, real-time feeds                │
│                                                          │
│  Global distribution:                                     │
│  → Redis Enterprise (Active-Active geo-replication)      │
│    Example: bKash multi-region deployment                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### ⚠️ Common Pitfalls:

```
┌──────────────────────────────────────────────────────────┐
│            DISTRIBUTED CACHE PITFALLS                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ❌ All keys same TTL → Cache Avalanche                  │
│  ✅ Jittered TTL: base + random(0, base*0.2)            │
│                                                          │
│  ❌ No null caching → Cache Penetration                  │
│  ✅ Cache null/empty with short TTL (60s)                │
│                                                          │
│  ❌ No lock on hot key expire → Stampede                 │
│  ✅ Mutex lock or probabilistic early refresh            │
│                                                          │
│  ❌ Ignoring split-brain → Data corruption               │
│  ✅ min-replicas-to-write + fencing tokens               │
│                                                          │
│  ❌ No monitoring → Silent failures                      │
│  ✅ Hit rate, memory usage, latency alerts               │
│                                                          │
│  ❌ Cache as source of truth → Data loss                 │
│  ✅ Database = truth, cache = performance layer          │
│                                                          │
│  ❌ No eviction policy → OOM crash                       │
│  ✅ maxmemory-policy: allkeys-lru or volatile-lfu       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 📝 সারসংক্ষেপ

> **মনে রাখবেন**: Distributed cache হলো performance layer, truth নয়। Database সবসময় source of truth।

- **Redis Cluster** = horizontal scaling (sharding) + HA
- **Redis Sentinel** = HA only (auto-failover), no sharding
- **Consistent Hashing** = node add/remove-এ minimal data movement
- **Hot Key** = L1 local cache + key splitting দিয়ে সমাধান
- **Cache Penetration** = Bloom filter + null caching
- **Cache Breakdown** = Mutex lock + logical expiration
- **Cache Avalanche** = Jittered TTL + circuit breaker
- **Split-Brain** = min-replicas-to-write + fencing tokens
- **Cross-DC** = Async replication via message queue (Kafka)
- সবসময় monitoring রাখুন: hit rate, memory, latency
