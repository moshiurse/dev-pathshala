# 🔄 রেপ্লিকেশন (Database Replication)

## 📌 সংজ্ঞা ও মূল ধারণা

### Replication কী?

Database replication হলো এমন একটি প্রক্রিয়া যেখানে একটি database server-এর data স্বয়ংক্রিয়ভাবে এক বা একাধিক অন্য server-এ copy করা হয়। এটি **high availability**, **fault tolerance**, এবং **read scalability** নিশ্চিত করার জন্য ব্যবহৃত হয়।

Production system-এ single database server একটি **Single Point of Failure (SPOF)**। Server crash হলে পুরো application বন্ধ। Replication এই সমস্যার সমাধান করে — একটি server down হলে অন্যটি থেকে service চালু থাকে।

### Replication-এর মূল উদ্দেশ্য

| উদ্দেশ্য | ব্যাখ্যা |
|---|---|
| **High Availability** | Master down হলে slave/replica promote হয়ে service চালু রাখে |
| **Read Scalability** | Read-heavy workload একাধিক replica-তে distribute করা যায় |
| **Disaster Recovery** | ভিন্ন geographic location-এ replica রেখে data loss প্রতিরোধ |
| **Backup Offloading** | Replica থেকে backup নিলে master-এর performance প্রভাবিত হয় না |
| **Latency Reduction** | User-এর কাছাকাছি replica থেকে read করলে latency কমে |

---

## 🏠 বাস্তব উদাহরণ

**সংবাদপত্রের ছাপাখানা:**

কল্পনা করুন একটি জাতীয় সংবাদপত্র। ঢাকায় মূল সম্পাদকীয় অফিসে (Master) খবর লেখা ও সম্পাদনা হয়। কিন্তু পুরো দেশে পত্রিকা পৌঁছানোর জন্য চট্টগ্রাম, রাজশাহী, সিলেটে আলাদা ছাপাখানা (Replica/Slave) আছে। মূল অফিস থেকে চূড়ান্ত কপি পাঠানো হয় এবং প্রতিটি ছাপাখানা সেটি ছাপিয়ে বিতরণ করে।

- **Master** = ঢাকার সম্পাদকীয় অফিস (write/সম্পাদনা)
- **Slave/Replica** = আঞ্চলিক ছাপাখানা (read/বিতরণ)
- **Synchronous replication** = ছাপাখানায় কপি পৌঁছানোর নিশ্চিতকরণ পাওয়ার আগে পরবর্তী সংস্করণ প্রকাশ হবে না
- **Asynchronous replication** = কপি পাঠিয়ে দেওয়ার পর নিশ্চিতকরণের অপেক্ষা না করে পরবর্তী কাজ শুরু

---

## 🏗️ Replication Architecture

### Master-Slave (Primary-Replica) Replication

সবচেয়ে সাধারণ pattern। একটি master node সব write গ্রহণ করে, slave/replica node শুধু read handle করে।

```
┌─────────────────────────────────────────────────────┐
│              Master-Slave Replication                │
│                                                     │
│         ┌──────────────┐                            │
│         │   Master     │  ◄── All WRITES go here    │
│         │  (Primary)   │                            │
│         └──────┬───────┘                            │
│                │                                    │
│         Replication Stream (Binary Log / WAL)       │
│                │                                    │
│       ┌────────┼────────┐                           │
│       ▼        ▼        ▼                           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│  │ Slave 1 │ │ Slave 2 │ │ Slave 3 │ ◄── READs     │
│  │(Replica)│ │(Replica)│ │(Replica)│     distributed│
│  └─────────┘ └─────────┘ └─────────┘               │
└─────────────────────────────────────────────────────┘
```

### Master-Master (Multi-Primary) Replication

দুটি বা তার বেশি node সবাই write গ্রহণ করতে পারে। প্রতিটি node অন্য node-এর changes replicate করে।

```
┌─────────────────────────────────────────────────────┐
│           Master-Master Replication                  │
│                                                     │
│  ┌──────────────┐     ┌──────────────┐              │
│  │   Master A   │◄───►│   Master B   │              │
│  │  (Read/Write)│     │  (Read/Write)│              │
│  └──────────────┘     └──────────────┘              │
│         │                     │                     │
│     Writes +              Writes +                  │
│     Reads                 Reads                     │
│                                                     │
│  ⚠️ Conflict Resolution প্রয়োজন হতে পারে          │
│  ⚠️ Auto-increment collision সমস্যা হতে পারে       │
└─────────────────────────────────────────────────────┘
```

**Master-Master-এ সমস্যা:**
- **Write conflict**: দুই master-এ একই row একই সময়ে update হলে conflict
- **Auto-increment collision**: উভয় master একই ID generate করতে পারে
- **সমাধান**: Odd/even ID strategy, UUID ব্যবহার, conflict resolution logic

---

## ⚡ Synchronous vs Asynchronous Replication

### Synchronous Replication

Master প্রতিটি write operation-এর পর **সব replica acknowledge করার অপেক্ষা করে** তারপর client-কে success জানায়।

```
Client ──WRITE──► Master ──replicate──► Slave 1 ──ACK──►
                                    ──► Slave 2 ──ACK──► Master ──OK──► Client
```

**সুবিধা**: Zero data loss guarantee, strong consistency
**অসুবিধা**: Higher write latency, একটি slow replica পুরো system ধীর করে

### Asynchronous Replication

Master write commit করার পর **সাথে সাথে client-কে success জানায়**, background-এ replica-তে data পাঠায়।

```
Client ──WRITE──► Master ──OK──► Client
                     │
                     └──(background)──► Slave 1
                                    ──► Slave 2
```

**সুবিধা**: Low write latency, replica down থাকলেও master কাজ করে
**অসুবিধা**: **Replication lag** — slave-এ পুরানো data থাকতে পারে, master crash-এ data loss সম্ভব

### Semi-Synchronous Replication

Master **অন্তত একটি replica acknowledge করলেই** client-কে success জানায়।

```
Client ──WRITE──► Master ──replicate──► Slave 1 ──ACK──► Master ──OK──► Client
                                    ──► Slave 2 (async, ACK পরে)
```

এটি synchronous ও asynchronous-এর মধ্যবর্তী — data safety ও performance-এর ভারসাম্য।

---

## 🛡️ Failover Strategies

### Automatic Failover

```
┌─────────────────────────────────────────────────────────┐
│                 Automatic Failover Flow                  │
│                                                         │
│  1. Master DOWN detected (health check fails)           │
│                    ↓                                    │
│  2. Election: সবচেয়ে up-to-date slave নির্বাচন         │
│                    ↓                                    │
│  3. Slave PROMOTED to new Master                        │
│                    ↓                                    │
│  4. অন্য slave-রা নতুন master-কে follow করে             │
│                    ↓                                    │
│  5. Application connection নতুন master-এ redirect       │
│                                                         │
│  Tools: MHA, Orchestrator, Patroni, ProxySQL            │
└─────────────────────────────────────────────────────────┘
```

### Manual Failover

DBA নিজে slave promote করেন। ছোট system বা planned maintenance-এ ব্যবহৃত।

---

## 💻 PHP Code Example — Read/Write Splitting with PDO

```php
<?php

class ReplicationAwareConnection
{
    private PDO $master;
    /** @var PDO[] */
    private array $slaves;
    private int $currentSlaveIndex = 0;

    public function __construct(array $masterConfig, array $slaveConfigs)
    {
        // Master connection (writes)
        $this->master = new PDO(
            "mysql:host={$masterConfig['host']};dbname={$masterConfig['db']};charset=utf8mb4",
            $masterConfig['user'],
            $masterConfig['password'],
            [
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_EMULATE_PREPARES => false,
            ]
        );

        // Slave connections (reads)
        foreach ($slaveConfigs as $config) {
            $this->slaves[] = new PDO(
                "mysql:host={$config['host']};dbname={$config['db']};charset=utf8mb4",
                $config['user'],
                $config['password'],
                [
                    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                    PDO::ATTR_EMULATE_PREPARES => false,
                ]
            );
        }
    }

    /**
     * Write query master-এ execute হবে
     */
    public function executeWrite(string $sql, array $params = []): bool
    {
        $stmt = $this->master->prepare($sql);
        return $stmt->execute($params);
    }

    /**
     * Read query round-robin ভাবে slave-এ execute হবে
     */
    public function executeRead(string $sql, array $params = []): array
    {
        $slave = $this->getNextSlave();
        $stmt = $slave->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    /**
     * Critical read — master থেকে পড়তে হবে (write-এর পরপরই read হলে)
     * Replication lag এড়াতে ব্যবহৃত
     */
    public function executeReadFromMaster(string $sql, array $params = []): array
    {
        $stmt = $this->master->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    private function getNextSlave(): PDO
    {
        if (empty($this->slaves)) {
            return $this->master; // slave না থাকলে master থেকে read
        }
        $slave = $this->slaves[$this->currentSlaveIndex];
        $this->currentSlaveIndex = ($this->currentSlaveIndex + 1) % count($this->slaves);
        return $slave;
    }
}

// ব্যবহার
$db = new ReplicationAwareConnection(
    ['host' => '10.0.1.1', 'db' => 'app', 'user' => 'root', 'password' => 'secret'],
    [
        ['host' => '10.0.1.2', 'db' => 'app', 'user' => 'reader', 'password' => 'secret'],
        ['host' => '10.0.1.3', 'db' => 'app', 'user' => 'reader', 'password' => 'secret'],
    ]
);

// Write → master-এ যাবে
$db->executeWrite(
    "INSERT INTO orders (user_id, total) VALUES (:uid, :total)",
    ['uid' => 42, 'total' => 1500.00]
);

// Read → slave থেকে আসবে (round-robin)
$orders = $db->executeRead(
    "SELECT * FROM orders WHERE user_id = :uid",
    ['uid' => 42]
);

// Critical read → master থেকে (write-এর পরপরই replication lag এড়াতে)
$latestOrder = $db->executeReadFromMaster(
    "SELECT * FROM orders WHERE user_id = :uid ORDER BY created_at DESC LIMIT 1",
    ['uid' => 42]
);
```

---

## 💻 JavaScript Code Example — Connection Routing with mysql2

```javascript
const mysql = require('mysql2/promise');

class ReplicaSetConnection {
    constructor(masterConfig, slaveConfigs) {
        // Master pool — writes
        this.masterPool = mysql.createPool({
            host: masterConfig.host,
            user: masterConfig.user,
            password: masterConfig.password,
            database: masterConfig.database,
            waitForConnections: true,
            connectionLimit: 10,
        });

        // Slave pools — reads
        this.slavePools = slaveConfigs.map(config =>
            mysql.createPool({
                host: config.host,
                user: config.user,
                password: config.password,
                database: config.database,
                waitForConnections: true,
                connectionLimit: 20, // read-heavy তাই বেশি connection
            })
        );

        this.currentSlaveIndex = 0;
    }

    /**
     * SQL query analyze করে স্বয়ংক্রিয়ভাবে master বা slave-এ route করে
     */
    async query(sql, params = []) {
        const isWriteQuery = /^\s*(INSERT|UPDATE|DELETE|CREATE|ALTER|DROP|TRUNCATE)/i.test(sql);

        if (isWriteQuery) {
            return this.writeQuery(sql, params);
        }
        return this.readQuery(sql, params);
    }

    async writeQuery(sql, params = []) {
        const [rows] = await this.masterPool.execute(sql, params);
        return rows;
    }

    async readQuery(sql, params = []) {
        const pool = this._getNextSlavePool();
        try {
            const [rows] = await pool.execute(sql, params);
            return rows;
        } catch (error) {
            // slave fail হলে master থেকে read (fallback)
            console.warn('Slave read failed, falling back to master:', error.message);
            const [rows] = await this.masterPool.execute(sql, params);
            return rows;
        }
    }

    /**
     * Write-এর পরপরই read — master থেকে পড়ে replication lag এড়ায়
     */
    async readAfterWrite(sql, params = []) {
        const [rows] = await this.masterPool.execute(sql, params);
        return rows;
    }

    _getNextSlavePool() {
        if (this.slavePools.length === 0) return this.masterPool;
        const pool = this.slavePools[this.currentSlaveIndex];
        this.currentSlaveIndex = (this.currentSlaveIndex + 1) % this.slavePools.length;
        return pool;
    }

    async close() {
        await this.masterPool.end();
        await Promise.all(this.slavePools.map(pool => pool.end()));
    }
}

// ব্যবহার
(async () => {
    const db = new ReplicaSetConnection(
        { host: '10.0.1.1', user: 'root', password: 'secret', database: 'app' },
        [
            { host: '10.0.1.2', user: 'reader', password: 'secret', database: 'app' },
            { host: '10.0.1.3', user: 'reader', password: 'secret', database: 'app' },
        ]
    );

    // Auto-routing: INSERT → master
    await db.query(
        'INSERT INTO orders (user_id, total) VALUES (?, ?)',
        [42, 1500.00]
    );

    // Auto-routing: SELECT → slave
    const orders = await db.query(
        'SELECT * FROM orders WHERE user_id = ?',
        [42]
    );

    // Replication lag এড়াতে master থেকে read
    const latest = await db.readAfterWrite(
        'SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 1',
        [42]
    );

    await db.close();
})();
```

---

## ⏱️ Replication Lag Monitoring

Replication lag হলো master-এ write হওয়ার পর slave-এ সেই data আসতে যে সময় লাগে।

```sql
-- MySQL: Replication lag পরীক্ষা
SHOW SLAVE STATUS\G
-- দেখুন: Seconds_Behind_Master

-- PostgreSQL: Replication lag পরীক্ষা
SELECT
    client_addr,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    replay_lag
FROM pg_stat_replication;
```

### Lag কমানোর কৌশল

| কৌশল | ব্যাখ্যা |
|---|---|
| **Parallel replication** | একাধিক thread-এ replication apply করা (MySQL 5.7+) |
| **Semi-sync replication** | অন্তত একটি slave ACK দিলে write complete |
| **Read-your-writes consistency** | Write-এর পর সেই user-এর read master থেকে করা |
| **Network optimization** | Master ও slave-এর মধ্যে low-latency network নিশ্চিত করা |

---

## ✅ কখন ব্যবহার করবেন

- **Read-heavy application**: 80%+ query যদি SELECT হয় (e-commerce product listing, news site)
- **High availability দরকার**: Downtime সহ্য করা সম্ভব নয় এমন critical system
- **Geographic distribution**: বিভিন্ন region-এ user থাকলে কাছের replica থেকে serve করা
- **Reporting/Analytics**: Report generation-এর load master থেকে সরিয়ে replica-তে দেওয়া
- **Backup strategy**: Replica থেকে backup নিলে master performance intact থাকে

## ❌ কখন ব্যবহার করবেন না

- **Write-heavy workload**: প্রতিটি write সব replica-তে propagate হয়, scalability লাভ কম
- **Strong consistency দরকার**: Async replication-এ stale read হতে পারে
- **ছোট application**: Complexity ও operational cost বৃদ্ধি পায়, ছোট app-এ অপ্রয়োজনীয়
- **Limited budget**: প্রতিটি replica আলাদা server cost, monitoring, maintenance যোগ করে
- **Schema migration চলাকালীন**: DDL changes replication-এ সমস্যা তৈরি করতে পারে

---

## 🔑 মনে রাখার বিষয়

1. **CAP theorem**: Replication-এ Consistency ও Availability-র মধ্যে trade-off আছে
2. **Replication ≠ Backup**: Replica-তে accidental DELETE-ও replicate হয়, আলাদা backup রাখুন
3. **Monitor lag**: Replication lag monitor না করলে stale data serve হতে পারে
4. **Test failover**: Production-এ failover আগে rehearsal করুন
5. **Read-after-write**: Critical read master থেকে করুন, blind slave read করবেন না
