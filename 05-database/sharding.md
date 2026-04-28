# 🧩 শার্ডিং (Database Sharding)

## 📌 সংজ্ঞা ও মূল ধারণা

### Sharding কী?

Sharding হলো একটি database-এর data-কে **একাধিক independent database instance (shard)**-এ ভাগ করে রাখার কৌশল। প্রতিটি shard মোট data-র একটি অংশ (subset) ধারণ করে এবং নিজে একটি সম্পূর্ণ database হিসেবে কাজ করে। একে **horizontal partitioning across multiple servers** বলা যায়।

যখন একটি single database server-এ data এত বেশি হয় যে vertical scaling (CPU, RAM, Disk বাড়ানো) আর সম্ভব নয়, তখন sharding-এর মাধ্যমে **horizontal scaling** করা হয়।

### কেন Sharding প্রয়োজন?

| সমস্যা | Sharding-এর সমাধান |
|---|---|
| **Data volume** | Terabyte-scale data একটি server-এ রাখা অসম্ভব |
| **Write throughput** | একটি master-এ write bottleneck, sharding-এ multiple write node |
| **Query performance** | ছোট dataset-এ query চালালে দ্রুত result |
| **Memory limitation** | প্রতিটি shard-এ কম data = বেশি data RAM-এ cache হয় |
| **Backup/Restore** | ছোট shard দ্রুত backup ও restore হয় |

---

## 🏠 বাস্তব উদাহরণ

**লাইব্রেরির বই সাজানো:**

কল্পনা করুন একটি বিশাল লাইব্রেরিতে ১০ লক্ষ বই আছে। একটি মাত্র ঘরে সব বই রাখলে:
- বই খুঁজে পেতে অনেক সময় লাগবে
- একসাথে অনেক পাঠক এলে ভিড় হবে
- ঘরের জায়গা ফুরিয়ে যাবে

**সমাধান**: বিষয় অনুযায়ী আলাদা ঘরে ভাগ করুন:
- ঘর ১ (Shard 1): বিজ্ঞান বই (A-M লেখক)
- ঘর ২ (Shard 2): বিজ্ঞান বই (N-Z লেখক)
- ঘর ৩ (Shard 3): সাহিত্য বই
- ঘর ৪ (Shard 4): ইতিহাস বই

প্রতিটি ঘরে নিজস্ব librarian (database server), নিজস্ব catalogue (index)। পাঠক নির্দিষ্ট ঘরে গিয়ে দ্রুত বই খুঁজে পান।

---

## 🏗️ Sharding-এর ধরন

### Horizontal Sharding (Row-based)

একই table-এর **row গুলো** বিভিন্ন shard-এ ভাগ হয়। সবচেয়ে সাধারণ sharding পদ্ধতি।

```
┌──────────────────────────────────────────────────────┐
│              Horizontal Sharding                      │
│                                                      │
│  users table (100M rows)                             │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   Shard 1    │  │   Shard 2    │  │  Shard 3   │ │
│  │ user_id 1-33M│  │user_id 34-66M│  │user_id 67M+│ │
│  │  (Server A)  │  │  (Server B)  │  │ (Server C) │ │
│  │              │  │              │  │            │ │
│  │ id | name    │  │ id | name    │  │ id | name  │ │
│  │ 1  | Rahim   │  │ 34M| Karim  │  │ 67M| Jamal │ │
│  │ 2  | Fatima  │  │ 34M+1|Nadia │  │ 67M+1|Sara │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
└──────────────────────────────────────────────────────┘
```

### Vertical Sharding (Column/Table-based)

বিভিন্ন **table বা column group** আলাদা server-এ রাখা হয়।

```
┌────────────────────────────────────────────────────┐
│              Vertical Sharding                      │
│                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  Server A   │  │   Server B   │  │ Server C  │ │
│  │  users      │  │   orders     │  │ products  │ │
│  │  profiles   │  │   payments   │  │ inventory │ │
│  │  auth       │  │   invoices   │  │ reviews   │ │
│  └─────────────┘  └──────────────┘  └───────────┘ │
│                                                    │
│  সুবিধা: Domain অনুযায়ী আলাদা, সহজ                │
│  অসুবিধা: Cross-table JOIN করা কঠিন               │
└────────────────────────────────────────────────────┘
```

---

## 🔑 Shard Key Selection

Shard key হলো সেই column যার value দিয়ে নির্ধারণ হয় কোন row কোন shard-এ যাবে। **সঠিক shard key নির্বাচন sharding-এর সবচেয়ে গুরুত্বপূর্ণ সিদ্ধান্ত।**

### সাধারণ Sharding Strategy

| Strategy | পদ্ধতি | উদাহরণ |
|---|---|---|
| **Range-based** | Key-এর value range অনুযায়ী ভাগ | user_id 1-1M → Shard 1, 1M-2M → Shard 2 |
| **Hash-based** | Key-এর hash value mod shard_count | hash(user_id) % 3 → Shard 0/1/2 |
| **Directory-based** | Lookup table-এ কোন key কোন shard তা রাখা | mapping service → shard lookup |
| **Geographic** | User-এর location অনুযায়ী | Bangladesh → Shard-BD, India → Shard-IN |

### ভালো Shard Key-এর বৈশিষ্ট্য

- **High cardinality**: অনেক ধরনের unique value (user_id ✅, boolean ❌)
- **Even distribution**: Data সব shard-এ সমানভাবে ভাগ হওয়া
- **Query alignment**: বেশিরভাগ query-তে shard key উপস্থিত থাকা
- **Immutable**: Shard key change হলে row re-shard করতে হয়

---

## 🔄 Consistent Hashing

সাধারণ hash-based sharding-এ shard সংখ্যা পরিবর্তন হলে (যেমন 3 → 4 shard) প্রায় সব data re-distribute করতে হয়। **Consistent hashing** এই সমস্যার সমাধান।

```
┌─────────────────────────────────────────────────────┐
│              Consistent Hashing Ring                  │
│                                                     │
│                    Shard A (0°)                      │
│                      ●                              │
│                   ╱     ╲                            │
│                 ╱         ╲                          │
│    Shard D   ●             ● Shard B                │
│    (270°)     ╲           ╱   (90°)                 │
│                 ╲       ╱                           │
│                   ╲   ╱                             │
│                     ●                               │
│                  Shard C (180°)                      │
│                                                     │
│  Key → hash → ring-এ clockwise নিকটতম shard-এ যায় │
│  নতুন shard যোগে শুধু প্রতিবেশী shard-এর data move │
└─────────────────────────────────────────────────────┘
```

**সুবিধা**: নতুন shard যোগ/বাদ দিলে গড়ে মাত্র `K/N` key re-distribute হয় (K = মোট key, N = shard সংখ্যা)।

---

## 💻 PHP Code Example — Shard Routing

```php
<?php

class ShardManager
{
    /** @var PDO[] */
    private array $shards = [];
    private int $shardCount;

    public function __construct(array $shardConfigs)
    {
        $this->shardCount = count($shardConfigs);

        foreach ($shardConfigs as $index => $config) {
            $this->shards[$index] = new PDO(
                "mysql:host={$config['host']};dbname={$config['db']};charset=utf8mb4",
                $config['user'],
                $config['password'],
                [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
            );
        }
    }

    /**
     * Hash-based shard key calculation
     * crc32 ব্যবহার করে consistent numeric hash পাওয়া যায়
     */
    public function getShardIndex(int|string $shardKey): int
    {
        $hash = crc32((string) $shardKey);
        return abs($hash) % $this->shardCount;
    }

    public function getShardConnection(int|string $shardKey): PDO
    {
        $index = $this->getShardIndex($shardKey);
        return $this->shards[$index];
    }

    /**
     * নির্দিষ্ট shard-এ user insert করা
     */
    public function insertUser(int $userId, string $name, string $email): void
    {
        $shard = $this->getShardConnection($userId);
        $stmt = $shard->prepare(
            "INSERT INTO users (id, name, email) VALUES (:id, :name, :email)"
        );
        $stmt->execute(['id' => $userId, 'name' => $name, 'email' => $email]);
    }

    /**
     * User ID দিয়ে সঠিক shard থেকে data fetch
     */
    public function getUser(int $userId): ?array
    {
        $shard = $this->getShardConnection($userId);
        $stmt = $shard->prepare("SELECT * FROM users WHERE id = :id");
        $stmt->execute(['id' => $userId]);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return $result ?: null;
    }

    /**
     * সব shard-এ query চালানো (scatter-gather pattern)
     * সাবধান: এটি ধীর — সব shard-এ parallel query চালাতে হয়
     */
    public function searchAllShards(string $email): ?array
    {
        foreach ($this->shards as $shard) {
            $stmt = $shard->prepare("SELECT * FROM users WHERE email = :email");
            $stmt->execute(['email' => $email]);
            $result = $stmt->fetch(PDO::FETCH_ASSOC);
            if ($result) {
                return $result;
            }
        }
        return null;
    }
}

// ব্যবহার
$manager = new ShardManager([
    ['host' => 'shard1.db.local', 'db' => 'users_shard', 'user' => 'app', 'password' => 'secret'],
    ['host' => 'shard2.db.local', 'db' => 'users_shard', 'user' => 'app', 'password' => 'secret'],
    ['host' => 'shard3.db.local', 'db' => 'users_shard', 'user' => 'app', 'password' => 'secret'],
]);

// User insert — shard key (user_id) অনুযায়ী সঠিক shard-এ যাবে
$manager->insertUser(12345, 'রহিম উদ্দিন', 'rahim@example.com');
$manager->insertUser(67890, 'করিম সাহেব', 'karim@example.com');

// User fetch — user_id দিয়ে সঠিক shard-এ query
$user = $manager->getUser(12345);

// Cross-shard search — সব shard-এ query (expensive!)
$found = $manager->searchAllShards('karim@example.com');
```

---

## 💻 JavaScript Code Example — Shard Key Calculation ও Routing

```javascript
const mysql = require('mysql2/promise');
const crypto = require('crypto');

class ShardRouter {
    constructor(shardConfigs) {
        this.shardCount = shardConfigs.length;
        this.pools = shardConfigs.map(config =>
            mysql.createPool({
                host: config.host,
                user: config.user,
                password: config.password,
                database: config.database,
                connectionLimit: 10,
            })
        );
    }

    /**
     * Consistent hashing ব্যবহার করে shard index নির্ধারণ
     */
    getShardIndex(shardKey) {
        const hash = crypto
            .createHash('md5')
            .update(String(shardKey))
            .digest('hex');
        // প্রথম 8 character hex → integer → mod shard count
        const numericHash = parseInt(hash.substring(0, 8), 16);
        return numericHash % this.shardCount;
    }

    getPool(shardKey) {
        const index = this.getShardIndex(shardKey);
        return this.pools[index];
    }

    async insertUser(userId, name, email) {
        const pool = this.getPool(userId);
        await pool.execute(
            'INSERT INTO users (id, name, email) VALUES (?, ?, ?)',
            [userId, name, email]
        );
        console.log(`User ${userId} → Shard ${this.getShardIndex(userId)}`);
    }

    async getUser(userId) {
        const pool = this.getPool(userId);
        const [rows] = await pool.execute(
            'SELECT * FROM users WHERE id = ?',
            [userId]
        );
        return rows[0] || null;
    }

    /**
     * Scatter-Gather: সব shard-এ parallel query চালিয়ে result merge
     */
    async queryAllShards(sql, params = []) {
        const results = await Promise.all(
            this.pools.map(async (pool) => {
                const [rows] = await pool.execute(sql, params);
                return rows;
            })
        );
        return results.flat(); // সব shard-এর result একত্রিত
    }

    /**
     * Aggregation across shards (যেমন মোট user সংখ্যা)
     */
    async countAllUsers() {
        const results = await Promise.all(
            this.pools.map(async (pool) => {
                const [rows] = await pool.execute(
                    'SELECT COUNT(*) as cnt FROM users'
                );
                return rows[0].cnt;
            })
        );
        return results.reduce((sum, count) => sum + count, 0);
    }

    async close() {
        await Promise.all(this.pools.map(pool => pool.end()));
    }
}

// ব্যবহার
(async () => {
    const router = new ShardRouter([
        { host: 'shard1.db.local', user: 'app', password: 'secret', database: 'users_shard' },
        { host: 'shard2.db.local', user: 'app', password: 'secret', database: 'users_shard' },
        { host: 'shard3.db.local', user: 'app', password: 'secret', database: 'users_shard' },
    ]);

    // Insert — shard key অনুযায়ী routing
    await router.insertUser(1001, 'রহিম', 'rahim@example.com');
    await router.insertUser(2002, 'করিম', 'karim@example.com');

    // Single-shard read
    const user = await router.getUser(1001);

    // Cross-shard aggregation
    const totalUsers = await router.countAllUsers();
    console.log(`Total users across all shards: ${totalUsers}`);

    // Cross-shard search (scatter-gather)
    const recentUsers = await router.queryAllShards(
        'SELECT * FROM users WHERE created_at > ? ORDER BY created_at DESC LIMIT 10',
        ['2024-01-01']
    );

    await router.close();
})();
```

---

## ⚠️ Sharding-এর চ্যালেঞ্জ

| চ্যালেঞ্জ | ব্যাখ্যা |
|---|---|
| **Cross-shard JOIN** | দুই shard-এর table JOIN করা সরাসরি সম্ভব নয়, application-level-এ করতে হয় |
| **Cross-shard transaction** | Distributed transaction (2PC) ধীর ও জটিল |
| **Re-sharding** | Shard সংখ্যা পরিবর্তনে data migration প্রয়োজন |
| **Hotspot** | ভুল shard key-তে কিছু shard-এ অতিরিক্ত load |
| **Operational complexity** | N shard = N গুণ বেশি monitoring, backup, schema migration |

---

## ✅ কখন ব্যবহার করবেন

- **Data volume**: Single server-এ data ধরছে না (multi-TB)
- **Write scalability**: Replication দিয়ে write scale হচ্ছে না
- **Multi-tenant SaaS**: প্রতি tenant আলাদা shard (data isolation + performance)
- **Regulatory compliance**: নির্দিষ্ট দেশের data সেই দেশের server-এ রাখা বাধ্যতামূলক
- **Query performance**: Index optimization-এও query slow, data volume-ই সমস্যা

## ❌ কখন ব্যবহার করবেন না

- **Data volume কম**: GB-level data-তে sharding অপ্রয়োজনীয় জটিলতা
- **Complex JOIN প্রয়োজন**: Cross-shard JOIN অত্যন্ত কঠিন ও ধীর
- **Read scaling যথেষ্ট**: Replication দিয়ে read scale করা সম্ভব হলে sharding এড়ান
- **ছোট দল**: Sharding-এর operational complexity সামলানোর মতো team না থাকলে
- **Vertical scaling সম্ভব**: আরো বড় server কিনে সমাধান হলে সেটি সহজ

---

## 🔑 মনে রাখার বিষয়

1. **Shard key সবচেয়ে গুরুত্বপূর্ণ**: ভুল shard key = hotspot + cross-shard query
2. **Shard nothing architecture আগে চেষ্টা করুন**: Read replica, caching, query optimization আগে
3. **Application-level sharding vs Middleware**: ProxySQL, Vitess, Citus ব্যবহার করে complexity কমানো যায়
4. **Schema changes কঠিন**: সব shard-এ একই সাথে migration চালানো চ্যালেঞ্জিং
5. **Global unique ID**: Auto-increment কাজ করবে না, UUID বা Snowflake ID ব্যবহার করুন
