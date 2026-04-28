# 🏊 কানেকশন পুলিং (Connection Pooling)

## 📌 সংজ্ঞা ও মূল ধারণা

### Connection Pooling কী?

Connection pooling হলো একটি কৌশল যেখানে database connection গুলো আগে থেকেই তৈরি করে একটি **pool**-এ রেখে দেওয়া হয় এবং application-এর প্রয়োজনে সেই pool থেকে connection ধার নিয়ে ব্যবহার শেষে ফেরত দেওয়া হয়। প্রতিটি request-এ নতুন connection তৈরি ও বন্ধ করার overhead এড়ানো এর মূল উদ্দেশ্য।

### কেন Connection Pooling প্রয়োজন?

প্রতিটি database connection তৈরি করা একটি **expensive operation**:

```
┌─────────────────────────────────────────────────────┐
│        Connection তৈরির ধাপসমূহ (TCP + Auth)        │
│                                                     │
│  Application              Database Server           │
│      │                         │                    │
│      │── TCP SYN ──────────►   │   (1) TCP handshake│
│      │◄─ TCP SYN-ACK ─────    │                    │
│      │── TCP ACK ──────────►   │                    │
│      │                         │                    │
│      │── Auth Request ─────►   │   (2) Authentication│
│      │◄─ Auth Challenge ───    │                    │
│      │── Auth Response ────►   │                    │
│      │◄─ Auth OK ──────────    │                    │
│      │                         │                    │
│      │── SET charset, tz ──►   │   (3) Session setup│
│      │◄─ OK ───────────────    │                    │
│      │                         │                    │
│      │   সময়: 3-10ms (local), 50-200ms (remote)    │
│      │   প্রতি সেকেন্ডে 1000 request = 1000 বার!   │
└─────────────────────────────────────────────────────┘
```

| Pool ছাড়া | Pool সহ |
|---|---|
| প্রতি request-এ নতুন connection | আগে থেকে তৈরি connection ব্যবহার |
| 3-200ms connection overhead | ~0ms (pool থেকে নেওয়া) |
| Server-এ connection limit দ্রুত শেষ | সীমিত সংখ্যক connection পুনর্ব্যবহার |
| Memory spike (প্রতি connection ~5-10MB) | স্থিতিশীল memory usage |

---

## 🏠 বাস্তব উদাহরণ

**রেস্তোরাঁর টেবিল ব্যবস্থাপনা:**

একটি রেস্তোরাঁয় ২০টি টেবিল আছে (connection pool size = 20)। অতিথিরা (application requests) আসেন, খালি টেবিলে বসেন, খাওয়া শেষে টেবিল ছেড়ে দেন — পরের অতিথি সেই টেবিল ব্যবহার করেন।

- **Pool size** = টেবিল সংখ্যা (20)
- **Active connection** = ব্যস্ত টেবিল (অতিথি বসে আছেন)
- **Idle connection** = খালি টেবিল (পরবর্তী অতিথির জন্য প্রস্তুত)
- **Connection wait** = সব টেবিল ভর্তি হলে বাইরে অপেক্ষা (queue)
- **Max pool size** = সর্বোচ্চ টেবিল সংখ্যা (আর বাড়ানো সম্ভব নয়)
- **Connection timeout** = নির্দিষ্ট সময় অপেক্ষার পর অতিথি চলে যান

যদি প্রতি অতিথির জন্য নতুন টেবিল বানাতে হতো (pool ছাড়া), সেটা অত্যন্ত ধীর ও অবাস্তব হতো!

---

## 🏗️ Pool Architecture

```
┌─────────────────────────────────────────────────────────┐
│                Connection Pool Architecture              │
│                                                         │
│  Application Threads/Requests                           │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                    │
│  │ R1 │ │ R2 │ │ R3 │ │ R4 │ │ R5 │                    │
│  └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘                    │
│     │      │      │      │      │                       │
│  ┌──▼──────▼──────▼──────▼──────▼──┐                    │
│  │       Connection Pool            │                    │
│  │                                  │                    │
│  │  ┌─────┐ ┌─────┐ ┌─────┐       │                    │
│  │  │Conn1│ │Conn2│ │Conn3│ Active │                    │
│  │  │(R1) │ │(R2) │ │(R3) │       │                    │
│  │  └─────┘ └─────┘ └─────┘       │                    │
│  │                                  │                    │
│  │  ┌─────┐ ┌─────┐               │                    │
│  │  │Conn4│ │Conn5│  Idle          │                    │
│  │  │     │ │     │  (R4, R5 এদের │                    │
│  │  └─────┘ └─────┘   পাবে)       │                    │
│  └──────────────────────────────────┘                    │
│                    │                                     │
│            ┌───────▼────────┐                            │
│            │ Database Server│                            │
│            │ max_connections│                            │
│            └────────────────┘                            │
└─────────────────────────────────────────────────────────┘
```

---

## 📐 Pool Size নির্ধারণ

### PostgreSQL-এর জন্য সূত্র (Pool Sizing Formula)

```
Optimal Pool Size = (core_count * 2) + effective_spindle_count
```

- `core_count` = CPU core সংখ্যা
- `effective_spindle_count` = disk spindle সংখ্যা (SSD হলে 1)

**উদাহরণ**: 4-core CPU + SSD → (4 × 2) + 1 = **9 connections**

### সাধারণ নিয়ম

| পরিস্থিতি | Pool Size |
|---|---|
| **Small app** (single server) | 5-10 |
| **Medium app** (few app servers) | 10-20 per server |
| **Large app** (many app servers) | PgBouncer/ProxySQL ব্যবহার করুন |

### ⚠️ সতর্কতা

Pool size বেশি হলেই ভালো নয়! Database server-এর `max_connections` সীমিত। ১০টি app server প্রতিটিতে ৫০ connection pool = ৫০০ connection — database সামলাতে পারবে না।

**সমাধান**: External connection pooler ব্যবহার করুন — **PgBouncer** (PostgreSQL), **ProxySQL** (MySQL)।

---

## 💻 PHP Code Example — PDO Persistent Connections

PHP-তে traditional connection pooling নেই কারণ PHP request-based architecture ব্যবহার করে। তবে **PDO persistent connections** দিয়ে connection reuse করা যায়।

```php
<?php

class DatabasePool
{
    private static ?PDO $connection = null;
    private static array $config = [];

    public static function configure(array $config): void
    {
        self::$config = $config;
    }

    /**
     * Persistent connection ব্যবহার করে connection reuse করা
     * PDO::ATTR_PERSISTENT = true হলে PHP process শেষ হলেও
     * connection বন্ধ হয় না, পরবর্তী request-এ পুনর্ব্যবহৃত হয়
     */
    public static function getConnection(): PDO
    {
        if (self::$connection === null) {
            $dsn = sprintf(
                "mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4",
                self::$config['host'],
                self::$config['port'] ?? 3306,
                self::$config['database']
            );

            self::$connection = new PDO($dsn, self::$config['user'], self::$config['password'], [
                PDO::ATTR_PERSISTENT => true,          // persistent connection সক্রিয়
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_EMULATE_PREPARES => false,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_TIMEOUT => 5,                // connection timeout 5s
            ]);
        }

        return self::$connection;
    }

    /**
     * Connection health check — stale connection detect করা
     */
    public static function healthCheck(): bool
    {
        try {
            $conn = self::getConnection();
            $conn->query('SELECT 1');
            return true;
        } catch (PDOException $e) {
            self::$connection = null; // stale connection reset
            return false;
        }
    }
}

// Laravel-style connection pool configuration (config/database.php তে)
// Laravel internally PDO persistent connection ও connection pooling handle করে
$laravelConfig = [
    'mysql' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'database' => env('DB_DATABASE', 'app'),
        'username' => env('DB_USERNAME', 'root'),
        'password' => env('DB_PASSWORD', ''),
        'options' => [
            PDO::ATTR_PERSISTENT => true,
        ],
        // Swoole/RoadRunner-এর সাথে octane ব্যবহার করলে real pooling পাওয়া যায়
    ],
];

// ব্যবহার
DatabasePool::configure([
    'host' => '127.0.0.1',
    'database' => 'myapp',
    'user' => 'root',
    'password' => 'secret',
]);

$db = DatabasePool::getConnection();
$stmt = $db->prepare("SELECT * FROM users WHERE id = :id");
$stmt->execute(['id' => 42]);
$user = $stmt->fetch();

// PHP-FPM + persistent connection কাজের ধারা:
// Request 1: connection তৈরি → query → request শেষ (connection বন্ধ হয় না)
// Request 2: আগের connection পুনর্ব্যবহার → query → request শেষ
// Request 3: আগের connection পুনর্ব্যবহার → query → request শেষ
```

---

## 💻 JavaScript Code Example — Node.js pg-pool

```javascript
const { Pool } = require('pg');

// Pool configuration
const pool = new Pool({
    host: '127.0.0.1',
    port: 5432,
    database: 'myapp',
    user: 'app_user',
    password: 'secret',

    // Pool size configuration
    min: 2,              // সর্বনিম্ন idle connection রাখবে
    max: 10,             // সর্বোচ্চ connection সংখ্যা
    idleTimeoutMillis: 30000,       // 30s idle থাকলে connection বন্ধ
    connectionTimeoutMillis: 5000,  // 5s-এ connection না পেলে error
    maxUses: 7500,       // একটি connection সর্বোচ্চ 7500 বার ব্যবহার হবে, তারপর নতুন
});

// Pool events — monitoring
pool.on('connect', (client) => {
    console.log(`New connection created. Total: ${pool.totalCount}`);
});

pool.on('acquire', (client) => {
    console.log(`Connection acquired. Idle: ${pool.idleCount}, Waiting: ${pool.waitingCount}`);
});

pool.on('remove', (client) => {
    console.log(`Connection removed. Total: ${pool.totalCount}`);
});

pool.on('error', (err, client) => {
    console.error('Unexpected pool error:', err.message);
});

/**
 * সাধারণ query — pool থেকে connection নিয়ে আবার ছেড়ে দেয়
 */
async function getUser(userId) {
    // pool.query() internally connection acquire → query → release করে
    const result = await pool.query(
        'SELECT * FROM users WHERE id = $1',
        [userId]
    );
    return result.rows[0];
}

/**
 * Transaction — একটি connection ধরে রেখে একাধিক query চালানো
 */
async function transferMoney(fromId, toId, amount) {
    // Transaction-এ manually connection acquire করতে হয়
    const client = await pool.connect();

    try {
        await client.query('BEGIN');

        await client.query(
            'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
            [amount, fromId]
        );

        await client.query(
            'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
            [amount, toId]
        );

        await client.query('COMMIT');
        console.log(`Transferred ${amount} from ${fromId} to ${toId}`);
    } catch (error) {
        await client.query('ROLLBACK');
        console.error('Transfer failed:', error.message);
        throw error;
    } finally {
        client.release(); // গুরুত্বপূর্ণ: connection pool-এ ফেরত দিন!
    }
}

/**
 * Pool status monitoring
 */
function getPoolStats() {
    return {
        totalConnections: pool.totalCount,    // মোট connection (active + idle)
        idleConnections: pool.idleCount,      // ব্যবহৃত হচ্ছে না এমন
        waitingRequests: pool.waitingCount,    // connection-এর জন্য অপেক্ষারত
    };
}

// mysql2 pool example
const mysql = require('mysql2/promise');

const mysqlPool = mysql.createPool({
    host: '127.0.0.1',
    user: 'root',
    password: 'secret',
    database: 'myapp',
    connectionLimit: 10,         // max connections
    queueLimit: 0,               // unlimited queue (0 = no limit)
    waitForConnections: true,    // pool full হলে queue-তে অপেক্ষা
    enableKeepAlive: true,       // TCP keep-alive
    keepAliveInitialDelay: 10000,// 10s পর keep-alive শুরু
});

async function main() {
    // Simple query
    const user = await getUser(42);
    console.log(user);

    // Transaction
    await transferMoney(1, 2, 500);

    // Pool stats
    console.log('Pool stats:', getPoolStats());

    // Graceful shutdown — সব connection বন্ধ
    await pool.end();
}

main().catch(console.error);
```

---

## 📊 Pool Monitoring ও Troubleshooting

### গুরুত্বপূর্ণ Metrics

| Metric | বিবরণ | Alert Threshold |
|---|---|---|
| **Active connections** | বর্তমানে ব্যবহৃত connection | > 80% of max |
| **Idle connections** | অব্যবহৃত connection | < min pool size |
| **Wait count** | Connection-এর জন্য অপেক্ষারত request | > 0 sustained |
| **Wait time** | Connection পেতে গড় সময় | > 100ms |
| **Connection errors** | Timeout, refused, reset | > 0 |
| **Connection lifetime** | একটি connection কতক্ষণ ধরে pool-এ আছে | Database-dependent |

### Database Server-এ Connection দেখা

```sql
-- MySQL: বর্তমান connection সংখ্যা
SHOW STATUS LIKE 'Threads_connected';
SHOW PROCESSLIST;
SHOW VARIABLES LIKE 'max_connections';

-- PostgreSQL: বর্তমান connection সংখ্যা
SELECT count(*) FROM pg_stat_activity;
SELECT * FROM pg_stat_activity WHERE state = 'idle';
SHOW max_connections;
```

---

## ✅ কখন ব্যবহার করবেন

- **Web application**: প্রতি HTTP request-এ database query চালাতে হয়
- **High-concurrency system**: শত শত simultaneous request handle করতে হয়
- **Microservices**: প্রতিটি service-এর নিজস্ব pool দিয়ে resource isolation
- **Long-running application**: Node.js, Java, Go server যা ক্রমাগত চলে
- **Remote database**: Network latency বেশি হলে connection reuse বেশি লাভজনক

## ❌ কখন ব্যবহার করবেন না

- **CLI scripts / one-shot jobs**: একটি query চালিয়ে শেষ, pool-এর overhead অপ্রয়োজনীয়
- **Serverless functions** (Lambda): Short-lived function-এ pool maintain সম্ভব নয়, **RDS Proxy** বা external pooler ব্যবহার করুন
- **PHP traditional** (mod_php / PHP-FPM without persistent): প্রতি request-এ process শেষ হয়, pool কাজ করে না — persistent connection ব্যবহার করুন
- **Connection-per-tenant isolation**: Security কারণে প্রতি tenant আলাদা credential ব্যবহার করলে pooling সীমিত

---

## 🔑 মনে রাখার বিষয়

1. **Connection leak ধরুন**: `finally` block-এ connection release না করলে pool শেষ হয়ে যাবে
2. **Pool size অতিরিক্ত বড় করবেন না**: Database-র `max_connections` সীমিত
3. **Health check সক্রিয় রাখুন**: Stale/broken connection detect করতে periodic health check
4. **Idle timeout সেট করুন**: অব্যবহৃত connection-এ resource waste হয়
5. **External pooler বিবেচনা করুন**: PgBouncer, ProxySQL — অনেক app server থাকলে অপরিহার্য
