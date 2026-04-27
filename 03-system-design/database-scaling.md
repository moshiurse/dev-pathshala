# 📈 ডাটাবেস স্কেলিং (Database Scaling Deep Dive)

> **"যখন একটি সার্ভারে সব ডেটা রাখা অসম্ভব হয়ে যায়, তখনই স্কেলিং-এর যাত্রা শুরু।"**

---

## 📌 সংজ্ঞা ও প্রয়োজনীয়তা

### ডাটাবেস স্কেলিং কী?

ডাটাবেস স্কেলিং হলো এমন একটি কৌশলগত প্রক্রিয়া যার মাধ্যমে একটি ডাটাবেস সিস্টেমকে ক্রমবর্ধমান লোড, ডেটা ভলিউম এবং ট্র্যাফিক সামলাতে সক্ষম করা হয়। বাংলাদেশের প্রেক্ষাপটে ভাবুন — bKash-এর দৈনিক কোটি কোটি ট্রানজেকশন, ১৭ কোটি মানুষের জাতীয় পরিচয়পত্র ডাটাবেস, বা ঈদের দিনে মোবাইল রিচার্জের spike — এসব ক্ষেত্রে একটি মাত্র ডাটাবেস সার্ভার দিয়ে কাজ চলে না।

### কেন স্কেল করতে হয়?

```
সমস্যা                          পরিমাপ (Metric)
─────────────────────────────────────────────────────
ধীর গতির Query                → Response time > 500ms
CPU সীমায় পৌঁছানো              → CPU usage > 85%
ডিস্ক I/O bottleneck           → Disk IOPS saturated
Memory-তে সব data ধরছে না      → Cache hit ratio < 60%
Connection limit পূর্ণ          → max_connections reached
Write throughput অপর্যাপ্ত      → Write latency spikes
ডেটা আকার TB পর্যায়ে           → Storage > single disk
```

### স্কেলিং-এর দুটি মূল মাত্রা (Dimensions)

```
                    ┌─────────────────────┐
                    │   Database Scaling   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                  │
     ┌────────▼────────┐              ┌─────────▼─────────┐
     │   Vertical (↑)  │              │  Horizontal (→)   │
     │   Scale UP       │              │  Scale OUT         │
     │                  │              │                    │
     │ • বড় সার্ভার    │              │ • একাধিক সার্ভার   │
     │ • বেশি RAM/CPU  │              │ • Sharding         │
     │ • সীমিত সীমা    │              │ • Replication      │
     │ • সহজ           │              │ • Partitioning     │
     └─────────────────┘              └────────────────────┘
```

---

## 📊 Scaling Strategies Architecture

### সম্পূর্ণ স্কেলিং আর্কিটেকচার

```
                            ┌──────────────┐
                            │   Clients    │
                            │  (170M users)│
                            └──────┬───────┘
                                   │
                            ┌──────▼───────┐
                            │ Load Balancer │
                            └──────┬───────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                              │
             ┌──────▼──────┐              ┌───────▼───────┐
             │  App Server  │              │  App Server   │
             │  (Laravel)   │              │  (Node.js)    │
             └──────┬───────┘              └───────┬───────┘
                    │                              │
                    └──────────────┬───────────────┘
                                   │
                            ┌──────▼───────┐
                            │ DB Proxy     │
                            │ (ProxySQL)   │
                            └──────┬───────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
     ┌────────▼────────┐  ┌───────▼───────┐  ┌─────────▼──────┐
     │  Shard 1        │  │  Shard 2      │  │  Shard 3       │
     │  (users A-H)    │  │  (users I-Q)  │  │  (users R-Z)   │
     │                 │  │               │  │                │
     │ ┌─────┐ ┌────┐ │  │ ┌─────┐┌────┐ │  │ ┌─────┐ ┌────┐ │
     │ │Write│ │Read│ │  │ │Write││Read│ │  │ │Write│ │Read│ │
     │ │(M)  │ │(S) │ │  │ │(M)  ││(S) │ │  │ │(M)  │ │(S) │ │
     │ └─────┘ └────┘ │  │ └─────┘└────┘ │  │ └─────┘ └────┘ │
     └─────────────────┘  └───────────────┘  └────────────────┘
```

### ডেটা ফ্লো ডায়াগ্রাম

```
  Write Request                          Read Request
  ─────────────                          ────────────
       │                                      │
       ▼                                      ▼
  ┌─────────┐                           ┌─────────┐
  │ ProxySQL │──── Route to ───────────→│ ProxySQL │
  └────┬─────┘    Master                └────┬─────┘
       │                                     │
       ▼                                     ▼
  ┌──────────┐     Async Replication    ┌──────────┐
  │  Master  │ ─────────────────────→  │  Replica  │
  │  (Write) │                          │  (Read)   │
  └──────────┘                          └──────────┘
```

---

## 💻 Scaling Techniques

### ১. Vertical Scaling (Scale-Up)

ভার্টিক্যাল স্কেলিং মানে হলো বিদ্যমান সার্ভারকে আরও শক্তিশালী করা — বেশি RAM, দ্রুত CPU, SSD/NVMe storage ইত্যাদি যোগ করা। এটি সবচেয়ে সহজ পদ্ধতি কারণ কোনো কোড পরিবর্তন লাগে না।

```
  Vertical Scaling Limits
  ═══════════════════════

  ┌─────────────────────────────────────────┐
  │                                         │
  │  Performance                            │
  │     ▲                                   │
  │     │            ╱─── Diminishing        │
  │     │          ╱      Returns            │
  │     │        ╱                           │
  │     │      ╱                             │
  │     │    ╱                               │
  │     │  ╱  ← Linear Phase                │
  │     │╱                                   │
  │     └──────────────────→ Cost ($$$)      │
  │                                         │
  │  AWS RDS Max: 128 vCPU, 4TB RAM         │
  │  খরচ: ~$50,000/month (db.x2iedn.metal) │
  └─────────────────────────────────────────┘
```

**কখন ভার্টিক্যাল স্কেলিং ব্যবহার করবেন:**
- ডেটা সাইজ < 1TB এবং QPS < 10,000
- দ্রুত সমাধান দরকার, কোড পরিবর্তন করার সময় নেই
- Strong consistency দরকার (সব ডেটা এক জায়গায়)
- বাজেট আছে কিন্তু ইঞ্জিনিয়ারিং রিসোর্স কম

**সীমাবদ্ধতা:**
- হার্ডওয়্যারের একটি চূড়ান্ত সীমা আছে (single point of failure)
- খরচ exponentially বাড়ে
- Downtime লাগে upgrade করতে
- ১৭ কোটি ব্যবহারকারীর ডেটাবেসের জন্য শুধু vertical scaling যথেষ্ট নয়

---

### ২. Read Replicas (Master-Slave Replication)

Read Replica হলো মূল ডাটাবেসের (Master/Primary) হুবহু কপি যেখানে শুধু READ query যায়। বেশিরভাগ অ্যাপ্লিকেশনে Read:Write ratio প্রায় 80:20 বা 90:10, তাই Read Replica যোগ করলে বিশাল পারফরম্যান্স উন্নতি হয়।

```
  Master-Slave Architecture
  ═════════════════════════

  ┌────────────────┐
  │   Application  │
  └───────┬────────┘
          │
     ┌────┴────┐
     │         │
  WRITE     READ
     │         │
     ▼         ▼
  ┌──────┐  ┌──────────────────────────────┐
  │Master│  │        Read Replicas          │
  │(RW)  │  │                              │
  │      │──│──→ Replica 1 (Dhaka DC)      │
  │      │──│──→ Replica 2 (Chittagong DC) │
  │      │──│──→ Replica 3 (Analytics)     │
  └──────┘  └──────────────────────────────┘
     │              ▲  ▲  ▲
     │              │  │  │
     └──────────────┘──┘──┘
       Async Replication (binlog)
```

#### PHP Laravel — Read/Write Splitting Configuration

```php
// config/database.php
// Laravel-তে read/write splitting খুবই সহজ — আলাদা connection config দিলেই হয়

return [
    'connections' => [
        'mysql' => [
            'read' => [
                'host' => [
                    env('DB_READ_HOST_1', '10.0.1.11'), // Replica 1 — Dhaka DC
                    env('DB_READ_HOST_2', '10.0.1.12'), // Replica 2 — Chittagong DC
                    env('DB_READ_HOST_3', '10.0.1.13'), // Replica 3 — Analytics
                ],
            ],
            'write' => [
                'host' => [
                    env('DB_WRITE_HOST', '10.0.1.10'), // Master
                ],
            ],
            'sticky'    => true, // Write-এর পর একই request-এ read ও master থেকে যাবে
            'driver'    => 'mysql',
            'database'  => env('DB_DATABASE', 'bkash_transactions'),
            'username'  => env('DB_USERNAME', 'app_user'),
            'password'  => env('DB_PASSWORD'),
            'charset'   => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix'    => '',
            'strict'    => true,
        ],
    ],
];
```

```php
// app/Services/TransactionService.php
// bKash-এর মতো ট্রানজেকশন সিস্টেমে read/write আলাদা করা

namespace App\Services;

use App\Models\Transaction;
use Illuminate\Support\Facades\DB;

class TransactionService
{
    /**
     * ট্রানজেকশন তৈরি — সবসময় Master-এ যাবে
     */
    public function createTransaction(array $data): Transaction
    {
        return DB::transaction(function () use ($data) {
            $transaction = Transaction::create([
                'sender_wallet'   => $data['sender'],
                'receiver_wallet' => $data['receiver'],
                'amount'          => $data['amount'],
                'type'            => $data['type'], // send_money, cashout, payment
                'status'          => 'pending',
                'reference_id'    => $this->generateReferenceId(),
            ]);

            // ব্যালেন্স আপডেট — Write query, Master-এ যাবে
            DB::table('wallets')
                ->where('phone', $data['sender'])
                ->decrement('balance', $data['amount']);

            DB::table('wallets')
                ->where('phone', $data['receiver'])
                ->increment('balance', $data['amount']);

            $transaction->update(['status' => 'completed']);

            return $transaction;
        });
    }

    /**
     * ট্রানজেকশন হিস্ট্রি — Read Replica থেকে যাবে
     */
    public function getTransactionHistory(string $walletPhone, int $days = 30)
    {
        return Transaction::where('sender_wallet', $walletPhone)
            ->orWhere('receiver_wallet', $walletPhone)
            ->where('created_at', '>=', now()->subDays($days))
            ->orderByDesc('created_at')
            ->paginate(50);  // স্বয়ংক্রিয়ভাবে Read Replica-তে যাবে
    }

    /**
     * গুরুত্বপূর্ণ — যেখানে replication lag সমস্যা করতে পারে,
     * সেখানে জোর করে Master থেকে পড়তে হবে
     */
    public function getBalanceAfterTransaction(string $walletPhone): float
    {
        // useWritePdo() দিয়ে Master থেকে পড়া — consistency নিশ্চিত
        return DB::table('wallets')
            ->useWritePdo()
            ->where('phone', $walletPhone)
            ->value('balance');
    }

    private function generateReferenceId(): string
    {
        return 'TXN' . date('Ymd') . strtoupper(bin2hex(random_bytes(6)));
    }
}
```

#### Node.js — Read/Write Splitting

```javascript
// db/replicaPool.js
// Node.js-এ mysql2 ব্যবহার করে Read/Write splitting

const mysql = require('mysql2/promise');

class ReplicaAwarePool {
    constructor(config) {
        // Master pool — শুধু Write-এর জন্য
        this.masterPool = mysql.createPool({
            host: config.master.host,
            port: config.master.port || 3306,
            user: config.user,
            password: config.password,
            database: config.database,
            connectionLimit: 20,       // Write কম হয় তাই কম connection
            waitForConnections: true,
            queueLimit: 100,
        });

        // Read Replica pools — Round-robin load balancing
        this.replicaPools = config.replicas.map(replica =>
            mysql.createPool({
                host: replica.host,
                port: replica.port || 3306,
                user: config.user,
                password: config.password,
                database: config.database,
                connectionLimit: 50,   // Read বেশি হয় তাই বেশি connection
                waitForConnections: true,
                queueLimit: 200,
            })
        );

        this.currentReplicaIndex = 0;
    }

    // Round-robin দিয়ে replica নির্বাচন
    getReadPool() {
        const pool = this.replicaPools[this.currentReplicaIndex];
        this.currentReplicaIndex =
            (this.currentReplicaIndex + 1) % this.replicaPools.length;
        return pool;
    }

    async read(sql, params = []) {
        const pool = this.getReadPool();
        const [rows] = await pool.execute(sql, params);
        return rows;
    }

    async write(sql, params = []) {
        const [result] = await this.masterPool.execute(sql, params);
        return result;
    }

    // Master থেকে পড়া — consistency দরকার হলে
    async readFromMaster(sql, params = []) {
        const [rows] = await this.masterPool.execute(sql, params);
        return rows;
    }

    async transaction(callback) {
        const connection = await this.masterPool.getConnection();
        try {
            await connection.beginTransaction();
            const result = await callback(connection);
            await connection.commit();
            return result;
        } catch (error) {
            await connection.rollback();
            throw error;
        } finally {
            connection.release();
        }
    }
}

// ব্যবহার
const db = new ReplicaAwarePool({
    master: { host: '10.0.1.10' },
    replicas: [
        { host: '10.0.1.11' },  // Dhaka
        { host: '10.0.1.12' },  // Chittagong
    ],
    user: 'app_user',
    password: process.env.DB_PASSWORD,
    database: 'bkash_transactions',
});

// Read — Replica থেকে
const history = await db.read(
    'SELECT * FROM transactions WHERE sender_wallet = ? ORDER BY created_at DESC LIMIT 50',
    ['01711000000']
);

// Write — Master-এ
await db.write(
    'INSERT INTO transactions (sender_wallet, receiver_wallet, amount, status) VALUES (?, ?, ?, ?)',
    ['01711000000', '01812000000', 5000, 'pending']
);
```

---

### ৩. Horizontal Sharding

Sharding হলো ডেটাকে একাধিক ডাটাবেস সার্ভারে ভাগ করে রাখা। প্রতিটি shard মোট ডেটার একটি অংশ (subset) ধারণ করে। এটি সবচেয়ে শক্তিশালী কিন্তু সবচেয়ে জটিল স্কেলিং পদ্ধতি।

#### Shard Key নির্বাচনের গুরুত্ব

Shard key নির্বাচন হলো sharding-এর সবচেয়ে critical সিদ্ধান্ত। ভুল shard key নির্বাচন করলে hotspot তৈরি হয়, cross-shard query বেড়ে যায় এবং পুরো সিস্টেম ধীর হয়।

```
  Shard Key Selection Matrix
  ══════════════════════════

  ভালো Shard Key              খারাপ Shard Key
  ───────────────              ────────────────
  ✅ High cardinality          ❌ Low cardinality
     (user_id, phone)             (gender, status)

  ✅ সমান distribution          ❌ Skewed distribution
     (hash of user_id)            (city — ঢাকায় 40% user)

  ✅ Query pattern match        ❌ Cross-shard query বাধ্য
     (যেই column দিয়ে বেশি       (join লাগে অন্য shard-এ)
      query হয়)

  ✅ Monotonic নয়              ❌ Auto-increment ID
     (UUID, hash)                 (সব নতুন data শেষ shard-এ)
```

#### তিন ধরনের Sharding Strategy

```
  ┌─────────────────────────────────────────────────────────────┐
  │              Sharding Strategies তুলনা                      │
  ├──────────────┬──────────────────┬───────────────────────────┤
  │ Range-Based  │  Hash-Based      │  Directory-Based          │
  ├──────────────┼──────────────────┼───────────────────────────┤
  │              │                  │                           │
  │  Shard 1:    │  shard_id =      │  ┌──────────┐            │
  │  A-H         │  hash(key)       │  │ Lookup   │            │
  │              │  % num_shards    │  │ Service  │            │
  │  Shard 2:    │                  │  │          │            │
  │  I-Q         │  সমান বণ্টন      │  │ key→shard│            │
  │              │  নিশ্চিত         │  └──────────┘            │
  │  Shard 3:    │                  │                           │
  │  R-Z         │  Range query     │  সম্পূর্ণ নমনীয়          │
  │              │  কঠিন            │  কিন্তু lookup            │
  │  Range query │                  │  bottleneck হতে পারে      │
  │  সহজ         │                  │                           │
  │              │                  │                           │
  │  Hotspot     │  Resharding      │  Resharding সবচেয়ে       │
  │  সমস্যা      │  কঠিন            │  সহজ                     │
  └──────────────┴──────────────────┴───────────────────────────┘
```

#### PHP Laravel — Sharding Implementation

```php
// app/Database/ShardManager.php
// বাংলাদেশের ১৭ কোটি ব্যবহারকারীর জন্য Shard Manager

namespace App\Database;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Config;

class ShardManager
{
    private array $shardConfigs;
    private int $totalShards;

    public function __construct()
    {
        $this->totalShards = (int) config('database.shard_count', 8);
        $this->initializeShardConfigs();
    }

    /**
     * Hash-based sharding — phone number দিয়ে shard নির্ধারণ
     * bKash-এর ক্ষেত্রে phone number হলো আদর্শ shard key কারণ:
     * ১. প্রতিটি user-এর unique phone আছে (high cardinality)
     * ২. বেশিরভাগ query phone দিয়ে হয়
     * ৩. Hash করলে সমান বণ্টন হয়
     */
    public function getShardId(string $phoneNumber): int
    {
        $hash = crc32($phoneNumber);
        return abs($hash) % $this->totalShards;
    }

    public function getConnection(string $phoneNumber): \Illuminate\Database\Connection
    {
        $shardId = $this->getShardId($phoneNumber);
        $connectionName = "shard_{$shardId}";

        if (!array_key_exists($connectionName, config('database.connections', []))) {
            $this->registerShardConnection($shardId);
        }

        return DB::connection($connectionName);
    }

    private function registerShardConnection(int $shardId): void
    {
        $config = $this->shardConfigs[$shardId];
        Config::set("database.connections.shard_{$shardId}", [
            'driver'    => 'mysql',
            'host'      => $config['host'],
            'port'      => $config['port'],
            'database'  => "bkash_shard_{$shardId}",
            'username'  => config('database.shard_user'),
            'password'  => config('database.shard_password'),
            'charset'   => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
        ]);
    }

    /**
     * Cross-shard query — একাধিক shard থেকে ডেটা সংগ্রহ
     * এটি ব্যয়বহুল, তাই যতটা সম্ভব এড়ানো উচিত
     */
    public function queryAllShards(string $sql, array $bindings = []): array
    {
        $results = [];

        for ($i = 0; $i < $this->totalShards; $i++) {
            $connectionName = "shard_{$i}";
            if (!array_key_exists($connectionName, config('database.connections', []))) {
                $this->registerShardConnection($i);
            }

            $shardResults = DB::connection($connectionName)
                ->select($sql, $bindings);

            $results = array_merge($results, $shardResults);
        }

        return $results;
    }

    /**
     * Scatter-Gather pattern — সব shard-এ parallel query পাঠানো
     * দৈনিক রিপোর্ট বা analytics-এর জন্য দরকার হয়
     */
    public function scatterGather(string $sql, array $bindings = []): array
    {
        $promises = [];

        for ($i = 0; $i < $this->totalShards; $i++) {
            $shardId = $i;
            $promises[] = async(function () use ($shardId, $sql, $bindings) {
                $conn = "shard_{$shardId}";
                return DB::connection($conn)->select($sql, $bindings);
            });
        }

        return array_merge(...await($promises));
    }

    private function initializeShardConfigs(): void
    {
        $this->shardConfigs = [];
        for ($i = 0; $i < $this->totalShards; $i++) {
            $this->shardConfigs[$i] = [
                'host' => config("database.shards.{$i}.host", "shard-{$i}.db.internal"),
                'port' => config("database.shards.{$i}.port", 3306),
            ];
        }
    }
}
```

```php
// app/Repositories/ShardedTransactionRepository.php

namespace App\Repositories;

use App\Database\ShardManager;

class ShardedTransactionRepository
{
    public function __construct(private ShardManager $shardManager) {}

    public function transfer(string $senderPhone, string $receiverPhone, float $amount): array
    {
        $senderShardId = $this->shardManager->getShardId($senderPhone);
        $receiverShardId = $this->shardManager->getShardId($receiverPhone);

        // একই shard-এ থাকলে সহজ transaction
        if ($senderShardId === $receiverShardId) {
            return $this->sameShardTransfer($senderPhone, $receiverPhone, $amount);
        }

        // আলাদা shard — Two-Phase Commit বা Saga pattern লাগবে
        return $this->crossShardTransfer($senderPhone, $receiverPhone, $amount);
    }

    private function sameShardTransfer(string $sender, string $receiver, float $amount): array
    {
        $conn = $this->shardManager->getConnection($sender);

        return $conn->transaction(function ($db) use ($sender, $receiver, $amount) {
            $db->table('wallets')->where('phone', $sender)->decrement('balance', $amount);
            $db->table('wallets')->where('phone', $receiver)->increment('balance', $amount);

            return $db->table('transactions')->insertGetId([
                'sender_wallet'   => $sender,
                'receiver_wallet' => $receiver,
                'amount'          => $amount,
                'status'          => 'completed',
                'created_at'      => now(),
            ]);
        });
    }

    /**
     * Cross-shard transfer — Saga Pattern ব্যবহার করে
     * দুটি shard-এ ACID guarantee নেই, তাই eventually consistent approach
     */
    private function crossShardTransfer(string $sender, string $receiver, float $amount): array
    {
        $txnId = 'TXN' . uniqid('', true);

        $senderConn = $this->shardManager->getConnection($sender);
        $receiverConn = $this->shardManager->getConnection($receiver);

        try {
            // Step 1: Sender থেকে টাকা কাটা
            $senderConn->transaction(function ($db) use ($sender, $amount, $txnId) {
                $db->table('wallets')->where('phone', $sender)->decrement('balance', $amount);
                $db->table('outbox')->insert([
                    'txn_id' => $txnId,
                    'type'   => 'debit',
                    'status' => 'completed',
                ]);
            });

            // Step 2: Receiver-এ টাকা যোগ
            $receiverConn->transaction(function ($db) use ($receiver, $amount, $txnId) {
                $db->table('wallets')->where('phone', $receiver)->increment('balance', $amount);
                $db->table('outbox')->insert([
                    'txn_id' => $txnId,
                    'type'   => 'credit',
                    'status' => 'completed',
                ]);
            });

            return ['txn_id' => $txnId, 'status' => 'completed'];

        } catch (\Throwable $e) {
            // Compensating transaction — rollback
            $this->compensate($txnId, $sender, $receiver, $amount);
            throw $e;
        }
    }

    private function compensate(string $txnId, string $sender, string $receiver, float $amount): void
    {
        // Sender-এর টাকা ফেরত দেওয়া (যদি কেটে থাকে)
        try {
            $senderConn = $this->shardManager->getConnection($sender);
            $debit = $senderConn->table('outbox')
                ->where('txn_id', $txnId)
                ->where('type', 'debit')
                ->first();

            if ($debit) {
                $senderConn->table('wallets')
                    ->where('phone', $sender)
                    ->increment('balance', $amount);
            }
        } catch (\Throwable $e) {
            // Dead letter queue-তে পাঠানো — manual intervention দরকার
            logger()->critical("Compensation failed for {$txnId}", [
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

#### Node.js — Sharding Implementation

```javascript
// sharding/shardRouter.js

const crypto = require('crypto');
const mysql = require('mysql2/promise');

class ShardRouter {
    constructor(shardConfigs) {
        this.totalShards = shardConfigs.length;
        this.pools = shardConfigs.map((config, index) =>
            mysql.createPool({
                host: config.host,
                port: config.port || 3306,
                user: config.user,
                password: config.password,
                database: `bkash_shard_${index}`,
                connectionLimit: 30,
                waitForConnections: true,
            })
        );
    }

    getShardId(key) {
        const hash = crypto.createHash('md5').update(key).digest();
        return hash.readUInt32BE(0) % this.totalShards;
    }

    getPool(key) {
        return this.pools[this.getShardId(key)];
    }

    // একটি shard-এ query
    async queryOnShard(key, sql, params = []) {
        const pool = this.getPool(key);
        const [rows] = await pool.execute(sql, params);
        return rows;
    }

    // সব shard-এ parallel query (scatter-gather)
    async queryAllShards(sql, params = []) {
        const results = await Promise.all(
            this.pools.map(async (pool) => {
                const [rows] = await pool.execute(sql, params);
                return rows;
            })
        );
        return results.flat();
    }

    // Cross-shard transaction — Saga pattern
    async crossShardTransfer(senderPhone, receiverPhone, amount) {
        const senderPool = this.getPool(senderPhone);
        const receiverPool = this.getPool(receiverPhone);
        const txnId = `TXN_${Date.now()}_${crypto.randomBytes(4).toString('hex')}`;

        const senderConn = await senderPool.getConnection();
        const receiverConn = await receiverPool.getConnection();

        try {
            // Step 1: Debit
            await senderConn.beginTransaction();
            await senderConn.execute(
                'UPDATE wallets SET balance = balance - ? WHERE phone = ? AND balance >= ?',
                [amount, senderPhone, amount]
            );
            await senderConn.commit();

            // Step 2: Credit
            try {
                await receiverConn.beginTransaction();
                await receiverConn.execute(
                    'UPDATE wallets SET balance = balance + ? WHERE phone = ?',
                    [amount, receiverPhone]
                );
                await receiverConn.commit();
            } catch (creditError) {
                // Credit ব্যর্থ — Sender-কে refund (compensating transaction)
                await senderConn.beginTransaction();
                await senderConn.execute(
                    'UPDATE wallets SET balance = balance + ? WHERE phone = ?',
                    [amount, senderPhone]
                );
                await senderConn.commit();
                throw creditError;
            }

            return { txnId, status: 'completed' };
        } finally {
            senderConn.release();
            receiverConn.release();
        }
    }
}

module.exports = ShardRouter;
```

---

### ৪. Partitioning (টেবিল পার্টিশনিং)

Partitioning হলো একটি বড় টেবিলকে ছোট ছোট ভৌত অংশে (partitions) ভাগ করা, কিন্তু logical ভাবে একটি টেবিলই থাকে। Sharding-এর মতো আলাদা সার্ভার লাগে না — একই সার্ভারে হয়। Query performance নাটকীয়ভাবে উন্নতি হয় কারণ MySQL শুধু relevant partition scan করে।

```
  Partitioning vs Sharding
  ═══════════════════════

  Partitioning (একই সার্ভার)         Sharding (আলাদা সার্ভার)
  ┌─────────────────────┐         ┌────────┐  ┌────────┐  ┌────────┐
  │ ┌─────┐ ┌─────┐    │         │Shard 1 │  │Shard 2 │  │Shard 3 │
  │ │ P1  │ │ P2  │    │         │Server A│  │Server B│  │Server C│
  │ └─────┘ └─────┘    │         └────────┘  └────────┘  └────────┘
  │ ┌─────┐ ┌─────┐    │
  │ │ P3  │ │ P4  │    │         একাধিক সার্ভারে ডেটা বণ্টন
  │ └─────┘ └─────┘    │         জটিল, কিন্তু অসীম scalable
  │   একটি সার্ভার      │
  └─────────────────────┘
```

#### MySQL Partition Examples

```sql
-- ১. RANGE Partitioning — তারিখ অনুযায়ী
-- bKash ট্রানজেকশন টেবিল — মাসিক পার্টিশন
-- পুরানো ডেটা আলাদা partition-এ থাকে, query দ্রুত হয়

CREATE TABLE transactions (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    sender      VARCHAR(15)     NOT NULL,
    receiver    VARCHAR(15)     NOT NULL,
    amount      DECIMAL(12,2)   NOT NULL,
    type        ENUM('send_money','cashout','payment','recharge') NOT NULL,
    status      ENUM('pending','completed','failed','reversed')   NOT NULL,
    created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, created_at),
    INDEX idx_sender (sender, created_at),
    INDEX idx_receiver (receiver, created_at)
)
PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    PARTITION p202403 VALUES LESS THAN (202404),
    PARTITION p202404 VALUES LESS THAN (202405),
    PARTITION p202405 VALUES LESS THAN (202406),
    PARTITION p202406 VALUES LESS THAN (202407),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- ২. LIST Partitioning — বিভাগ (division) অনুযায়ী
-- বাংলাদেশের ৮ বিভাগে ব্যবহারকারী ভাগ

CREATE TABLE users (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    phone       VARCHAR(15)     NOT NULL UNIQUE,
    name        VARCHAR(100)    NOT NULL,
    division    TINYINT UNSIGNED NOT NULL,
    created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, division)
)
PARTITION BY LIST (division) (
    PARTITION p_dhaka       VALUES IN (1),   -- ঢাকা
    PARTITION p_chittagong  VALUES IN (2),   -- চট্টগ্রাম
    PARTITION p_rajshahi    VALUES IN (3),   -- রাজশাহী
    PARTITION p_khulna      VALUES IN (4),   -- খুলনা
    PARTITION p_sylhet      VALUES IN (5),   -- সিলেট
    PARTITION p_rangpur     VALUES IN (6),   -- রংপুর
    PARTITION p_barisal     VALUES IN (7),   -- বরিশাল
    PARTITION p_mymensingh  VALUES IN (8)    -- ময়মনসিংহ
);

-- ৩. HASH Partitioning — সমান বণ্টনের জন্য

CREATE TABLE wallet_ledger (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    wallet_id   BIGINT UNSIGNED NOT NULL,
    debit       DECIMAL(12,2) DEFAULT 0,
    credit      DECIMAL(12,2) DEFAULT 0,
    balance     DECIMAL(12,2) NOT NULL,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, wallet_id)
)
PARTITION BY HASH (wallet_id)
PARTITIONS 16;

-- ৪. Composite (Sub-partitioning) — RANGE + HASH একসাথে

CREATE TABLE transaction_logs (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    wallet_id   BIGINT UNSIGNED NOT NULL,
    action      VARCHAR(50)     NOT NULL,
    created_at  DATE            NOT NULL,
    PRIMARY KEY (id, wallet_id, created_at)
)
PARTITION BY RANGE (YEAR(created_at))
SUBPARTITION BY HASH (wallet_id)
SUBPARTITIONS 8 (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- পুরানো partition মুছে ফেলা (data archiving)
ALTER TABLE transactions DROP PARTITION p202401;

-- নতুন partition যোগ করা
ALTER TABLE transactions REORGANIZE PARTITION p_future INTO (
    PARTITION p202407 VALUES LESS THAN (202408),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

### ৫. Federation (Functional Partitioning)

Federation হলো বিভিন্ন ফিচার বা ডোমেইনের জন্য আলাদা আলাদা ডাটাবেস ব্যবহার করা। এটি microservices architecture-এর সাথে চমৎকারভাবে মিলে যায়। প্রতিটি সার্ভিস তার নিজস্ব ডাটাবেসের মালিক।

```
  Federation Architecture (bKash উদাহরণ)
  ═══════════════════════════════════════

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  User        │  │  Transaction │  │  Merchant    │
  │  Service     │  │  Service     │  │  Service     │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                  │
         ▼                 ▼                  ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  user_db     │  │ txn_db       │  │ merchant_db  │
  │              │  │              │  │              │
  │ • users      │  │ • txns       │  │ • merchants  │
  │ • profiles   │  │ • ledger     │  │ • stores     │
  │ • kyc_docs   │  │ • settlements│  │ • terminals  │
  │              │  │              │  │              │
  │ PostgreSQL   │  │ MySQL        │  │ PostgreSQL   │
  │ (JSONB       │  │ (InnoDB,     │  │              │
  │  for docs)   │  │  Partitioned)│  │              │
  └──────────────┘  └──────────────┘  └──────────────┘

  ┌──────────────┐  ┌──────────────┐
  │  Notification│  │  Analytics   │
  │  Service     │  │  Service     │
  └──────┬───────┘  └──────┬───────┘
         │                 │
         ▼                 ▼
  ┌──────────────┐  ┌──────────────┐
  │ notif_db     │  │ analytics_db │
  │              │  │              │
  │ Redis +      │  │ ClickHouse   │
  │ MongoDB      │  │ (columnar)   │
  └──────────────┘  └──────────────┘
```

```php
// config/database.php — Federation configuration in Laravel

return [
    'connections' => [
        'users' => [
            'driver'   => 'pgsql',
            'host'     => env('USER_DB_HOST'),
            'database' => 'user_db',
            'username' => env('USER_DB_USER'),
            'password' => env('USER_DB_PASS'),
        ],
        'transactions' => [
            'driver'   => 'mysql',
            'host'     => env('TXN_DB_HOST'),
            'database' => 'txn_db',
            'username' => env('TXN_DB_USER'),
            'password' => env('TXN_DB_PASS'),
        ],
        'merchants' => [
            'driver'   => 'pgsql',
            'host'     => env('MERCHANT_DB_HOST'),
            'database' => 'merchant_db',
            'username' => env('MERCHANT_DB_USER'),
            'password' => env('MERCHANT_DB_PASS'),
        ],
    ],
];
```

```php
// app/Models/Transaction.php — নির্দিষ্ট connection ব্যবহার

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Transaction extends Model
{
    protected $connection = 'transactions'; // Federation — txn_db তে যাবে

    protected $fillable = [
        'sender_wallet', 'receiver_wallet', 'amount', 'type', 'status',
    ];
}
```

---

### ৬. Denormalization for Performance

Denormalization হলো ইচ্ছাকৃতভাবে ডেটা duplicate করা যাতে JOIN কমে এবং read performance বাড়ে। Normalized ডাটাবেসে অনেক JOIN দরকার হয় যা scale-এ bottleneck হয়ে যায়।

```
  Normalized (ধীর)                    Denormalized (দ্রুত)
  ════════════════                    ═══════════════════

  SELECT t.*, u.name, u.phone        SELECT * FROM transactions
  FROM transactions t                 WHERE sender_phone = '017...'
  JOIN users u ON t.sender_id = u.id
  WHERE u.phone = '017...'           ← কোনো JOIN নেই!

  3 টেবিল JOIN                       ১ টেবিলে সব তথ্য
  ┌──────────┐ ┌───────┐ ┌────┐      ┌──────────────────────┐
  │   txns   │─│ users │─│kyc │      │   txns_denormalized  │
  └──────────┘ └───────┘ └────┘      │  + sender_name       │
                                      │  + sender_phone      │
                                      │  + receiver_name     │
                                      └──────────────────────┘
```

```javascript
// denormalization/transactionDenormalizer.js
// Node.js — ট্রানজেকশন তৈরির সময় denormalized data store করা

class TransactionDenormalizer {
    constructor(db) {
        this.db = db;
    }

    async createDenormalizedTransaction(txnData) {
        // User তথ্য আগেই fetch করে transaction-এ embed করা
        const [sender] = await this.db.read(
            'SELECT name, phone, division FROM users WHERE id = ?',
            [txnData.senderId]
        );
        const [receiver] = await this.db.read(
            'SELECT name, phone, division FROM users WHERE id = ?',
            [txnData.receiverId]
        );

        // Denormalized insert — পরে read-এ কোনো JOIN লাগবে না
        return this.db.write(
            `INSERT INTO transactions_denormalized
             (sender_id, sender_name, sender_phone, sender_division,
              receiver_id, receiver_name, receiver_phone, receiver_division,
              amount, type, status, created_at)
             VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, NOW())`,
            [
                txnData.senderId, sender.name, sender.phone, sender.division,
                txnData.receiverId, receiver.name, receiver.phone, receiver.division,
                txnData.amount, txnData.type, 'completed',
            ]
        );
    }

    // User নাম পরিবর্তন হলে সব সংশ্লিষ্ট transaction update করতে হবে
    // এটি denormalization-এর মূল trade-off
    async onUserNameChanged(userId, newName) {
        await Promise.all([
            this.db.write(
                'UPDATE transactions_denormalized SET sender_name = ? WHERE sender_id = ?',
                [newName, userId]
            ),
            this.db.write(
                'UPDATE transactions_denormalized SET receiver_name = ? WHERE receiver_id = ?',
                [newName, userId]
            ),
        ]);
    }
}

module.exports = TransactionDenormalizer;
```

---

### ৭. Database Proxy (ProxySQL / PgBouncer)

Database Proxy হলো application এবং database-এর মাঝে বসা একটি middleware layer। এটি connection pooling, query routing, load balancing এবং query caching করে। High-scale system-এ এটি অপরিহার্য।

```
  ProxySQL Architecture
  ═════════════════════

  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
  │ App Server 1│ │ App Server 2│ │ App Server N│
  │ (100 conns) │ │ (100 conns) │ │ (100 conns) │
  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
         │               │               │
         └───────────────┼───────────────┘
                         │  300 connections
                  ┌──────▼──────┐
                  │  ProxySQL   │
                  │             │
                  │ • Pooling   │  ← ৩০০ → ৫০ তে কমানো
                  │ • Routing   │  ← R/W split
                  │ • Caching   │  ← Hot query cache
                  │ • Failover  │  ← Auto master switch
                  └──────┬──────┘
                         │  50 connections (multiplexing)
              ┌──────────┼──────────┐
              │          │          │
         ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
         │ Master │ │Replica1│ │Replica2│
         └────────┘ └────────┘ └────────┘
```

```sql
-- ProxySQL configuration — Read/Write splitting rules

-- Writer hostgroup (10) এবং Reader hostgroup (20) সেটআপ
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES
    (10, 'master.db.internal', 3306, 1000),
    (20, 'replica1.db.internal', 3306, 500),
    (20, 'replica2.db.internal', 3306, 500);

-- Query rules — SELECT replica-তে, বাকি master-এ
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES
    (1, 1, '^SELECT .* FOR UPDATE', 10, 1),       -- SELECT FOR UPDATE → Master
    (2, 1, '^SELECT',               20, 1),        -- সাধারণ SELECT → Replica
    (3, 1, '.*',                    10, 1);        -- INSERT/UPDATE/DELETE → Master

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

---

## 🔥 Advanced Topics

### ১. Consistent Hashing for Sharding

সাধারণ hash-based sharding-এ shard সংখ্যা পরিবর্তন করলে (৪ → ৫ shard) প্রায় সব key-এর shard বদলে যায়, ফলে massive data migration দরকার হয়। Consistent Hashing এই সমস্যার সমাধান — shard যোগ/বাদ দিলে শুধু `K/N` keys (K = মোট keys, N = মোট nodes) migrate হয়।

```
  Consistent Hashing Ring
  ═══════════════════════

  hash(key) → 0 থেকে 2^32 পর্যন্ত একটি ring-এ map হয়
  প্রতিটি shard-ও ring-এ একটি position পায়
  Key clockwise ঘুরে প্রথম যেই shard পায়, সেখানে যায়

                        0 / 2^32
                          │
                    ──────●──────
                ──        │        ──
             ●(Shard A)   │           ──
           /              │             \
          /               │              ●(Shard B)
         │                │              │
         │    key1 ◆──→ Shard A          │
         │                │              │
         │       key2 ◆──────→ Shard B   │
          \               │             /
           \              │            /
             ●(Shard D)   │       ●(Shard C)
                ──        │     ──
                    ──────●──────
                    key3 ◆──→ Shard C

  Virtual Nodes (VNodes):
  ═══════════════════════
  প্রতিটি physical shard ring-এ একাধিক position নেয়
  এতে load আরও সমানভাবে বণ্টন হয়

                    ──────●──────
                ──   A1   │  B1     ──
             ●            │           ●
           /    C2        │     A2      \
          ●               │              ●
         │   B2           │         C1   │
          ●               │              ●
           \    A3        │     B3      /
             ●            │           ●
                ──   C3   │  D1     ──
                    ──────●──────
```

#### PHP — Consistent Hashing Implementation

```php
// app/Database/ConsistentHash.php

namespace App\Database;

class ConsistentHash
{
    private array $ring = [];           // position → node mapping
    private array $sortedPositions = []; // sorted hash positions
    private int $virtualNodes;          // প্রতিটি physical node-এর virtual copies

    public function __construct(int $virtualNodes = 150)
    {
        $this->virtualNodes = $virtualNodes;
    }

    /**
     * Ring-এ একটি নতুন node (shard) যোগ করা
     * Virtual nodes ব্যবহার করে load সমানভাবে বণ্টন
     */
    public function addNode(string $node): void
    {
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $virtualKey = "{$node}:vnode:{$i}";
            $position = $this->hash($virtualKey);
            $this->ring[$position] = $node;
        }

        $this->sortedPositions = array_keys($this->ring);
        sort($this->sortedPositions);
    }

    /**
     * Ring থেকে একটি node সরানো
     * শুধু এই node-এর keys পরবর্তী node-এ migrate হবে
     */
    public function removeNode(string $node): void
    {
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $virtualKey = "{$node}:vnode:{$i}";
            $position = $this->hash($virtualKey);
            unset($this->ring[$position]);
        }

        $this->sortedPositions = array_keys($this->ring);
        sort($this->sortedPositions);
    }

    /**
     * Key-এর জন্য সঠিক node খুঁজে বের করা
     * Binary search দিয়ে ring-এ clockwise প্রথম node খোঁজা
     */
    public function getNode(string $key): string
    {
        if (empty($this->ring)) {
            throw new \RuntimeException('No nodes in hash ring');
        }

        $hash = $this->hash($key);
        $index = $this->binarySearchCeiling($hash);

        // Ring-এ wrap around — শেষের পরে প্রথমে যাওয়া
        if ($index >= count($this->sortedPositions)) {
            $index = 0;
        }

        $position = $this->sortedPositions[$index];
        return $this->ring[$position];
    }

    /**
     * Key-এর জন্য N টি replica node খোঁজা (replication factor)
     */
    public function getNodes(string $key, int $count = 3): array
    {
        $nodes = [];
        $hash = $this->hash($key);
        $index = $this->binarySearchCeiling($hash);

        while (count($nodes) < $count && count($nodes) < count(array_unique(array_values($this->ring)))) {
            if ($index >= count($this->sortedPositions)) {
                $index = 0;
            }

            $position = $this->sortedPositions[$index];
            $node = $this->ring[$position];

            // একই physical node duplicate না নেওয়া
            if (!in_array($node, $nodes, true)) {
                $nodes[] = $node;
            }

            $index++;
        }

        return $nodes;
    }

    private function hash(string $key): int
    {
        return crc32(md5($key));
    }

    private function binarySearchCeiling(int $target): int
    {
        $low = 0;
        $high = count($this->sortedPositions) - 1;

        while ($low <= $high) {
            $mid = (int)(($low + $high) / 2);
            if ($this->sortedPositions[$mid] < $target) {
                $low = $mid + 1;
            } else {
                $high = $mid - 1;
            }
        }

        return $low;
    }
}
```

```php
// ব্যবহার উদাহরণ
$hashRing = new ConsistentHash(virtualNodes: 150);

$hashRing->addNode('shard-dhaka-1');
$hashRing->addNode('shard-dhaka-2');
$hashRing->addNode('shard-chittagong-1');
$hashRing->addNode('shard-sylhet-1');

// Key → Node mapping
$node = $hashRing->getNode('01711000000'); // → 'shard-dhaka-2'
$node = $hashRing->getNode('01812000000'); // → 'shard-chittagong-1'

// নতুন shard যোগ — শুধু ~20% keys migrate হবে (1/N)
$hashRing->addNode('shard-rajshahi-1');
```

#### JavaScript — Consistent Hashing

```javascript
// consistentHash.js

const crypto = require('crypto');

class ConsistentHashRing {
    constructor(virtualNodes = 150) {
        this.virtualNodes = virtualNodes;
        this.ring = new Map();       // position → node
        this.sortedKeys = [];        // sorted positions
    }

    hash(key) {
        const md5 = crypto.createHash('md5').update(key).digest();
        return md5.readUInt32BE(0);
    }

    addNode(node) {
        for (let i = 0; i < this.virtualNodes; i++) {
            const virtualKey = `${node}:vnode:${i}`;
            const position = this.hash(virtualKey);
            this.ring.set(position, node);
        }
        this.sortedKeys = [...this.ring.keys()].sort((a, b) => a - b);
    }

    removeNode(node) {
        for (let i = 0; i < this.virtualNodes; i++) {
            const virtualKey = `${node}:vnode:${i}`;
            const position = this.hash(virtualKey);
            this.ring.delete(position);
        }
        this.sortedKeys = [...this.ring.keys()].sort((a, b) => a - b);
    }

    getNode(key) {
        if (this.ring.size === 0) throw new Error('Empty ring');

        const hash = this.hash(key);
        // Binary search — clockwise-এ প্রথম node
        let low = 0, high = this.sortedKeys.length - 1;

        while (low <= high) {
            const mid = (low + high) >>> 1;
            if (this.sortedKeys[mid] < hash) low = mid + 1;
            else high = mid - 1;
        }

        const index = low >= this.sortedKeys.length ? 0 : low;
        return this.ring.get(this.sortedKeys[index]);
    }

    // Resharding-এ কোন keys migrate করতে হবে তা বের করা
    getAffectedKeys(existingKeys, newNode) {
        const before = new Map();
        existingKeys.forEach(key => before.set(key, this.getNode(key)));

        this.addNode(newNode);

        const toMigrate = [];
        existingKeys.forEach(key => {
            const newTarget = this.getNode(key);
            if (before.get(key) !== newTarget) {
                toMigrate.push({ key, from: before.get(key), to: newTarget });
            }
        });

        return toMigrate;
    }
}

module.exports = ConsistentHashRing;
```

---

### ২. Vitess — YouTube-এর MySQL Scaling সমাধান

Vitess হলো একটি database clustering system যা MySQL-কে horizontally scale করে। YouTube এটি ব্যবহার করে বিলিয়ন ভিউয়ের ডেটা manage করে। এটি sharding-এর জটিলতা application থেকে লুকিয়ে রাখে।

```
  Vitess Architecture
  ══════════════════

  ┌─────────────────────────────────────────────┐
  │                Application                   │
  │         (MySQL protocol ব্যবহার করে)         │
  └──────────────────┬──────────────────────────┘
                     │ MySQL Protocol
              ┌──────▼──────┐
              │    VTGate   │  ← Query Router
              │  (Stateless)│     Query parsing, planning
              └──────┬──────┘     Scatter-gather
                     │
        ┌────────────┼────────────┐
        │            │            │
  ┌─────▼─────┐┌────▼─────┐┌────▼─────┐
  │  VTTablet ││ VTTablet ││ VTTablet │  ← Shard Manager
  │ (Shard 0) ││ (Shard 1)││ (Shard 2)|     Query rewriting
  └─────┬─────┘└────┬─────┘└────┬─────┘     Connection pooling
        │           │            │
  ┌─────▼─────┐┌───▼──────┐┌───▼──────┐
  │   MySQL   ││  MySQL   ││  MySQL   │
  │ (Primary) ││ (Primary)││ (Primary)|
  └───────────┘└──────────┘└──────────┘
        │            │           │
  ┌─────▼─────┐┌───▼──────┐┌───▼──────┐
  │  Replica  ││ Replica  ││ Replica  │
  └───────────┘└──────────┘└──────────┘

  মূল সুবিধা:
  • Application কোড MySQL ভাবে — sharding logic জানতে হয় না
  • Online resharding — downtime ছাড়া shard বাড়ানো/কমানো
  • Connection pooling — হাজার হাজার app connection কমিয়ে কম MySQL conn
  • Query analysis — slow query detection
```

---

### ৩. NewSQL — CockroachDB / TiDB

NewSQL হলো নতুন প্রজন্মের ডাটাবেস যা SQL interface দেয় কিন্তু ভিতরে horizontally scalable এবং distributed। ACID + horizontal scaling — দুটোই পাওয়া যায়।

```
  Traditional SQL vs NewSQL vs NoSQL
  ══════════════════════════════════

  ┌────────────────┬───────────────┬───────────────┐
  │  Traditional   │    NewSQL     │    NoSQL      │
  │  SQL (MySQL)   │  (CockroachDB)│  (MongoDB)    │
  ├────────────────┼───────────────┼───────────────┤
  │ ✅ ACID        │ ✅ ACID       │ ❌ Limited    │
  │ ✅ SQL         │ ✅ SQL        │ ❌ Custom API │
  │ ❌ Scale-out   │ ✅ Scale-out  │ ✅ Scale-out  │
  │ ✅ Mature      │ ⚠️  Newer     │ ✅ Mature     │
  │ ❌ Geo-dist.   │ ✅ Geo-dist.  │ ⚠️  Limited   │
  │ 💰 কম খরচ     │ 💰💰 বেশি     │ 💰 মাঝামাঝি  │
  └────────────────┴───────────────┴───────────────┘

  CockroachDB/TiDB-এর মূল ধারণা:
  ──────────────────────────────

  • Raft consensus protocol — distributed nodes-এ data consistency
  • Automatic sharding — range-based, সিস্টেম নিজেই সামলায়
  • Distributed transactions — Two-Phase Commit with global ordering
  • Multi-region — data কে geographic region অনুযায়ী place করা যায়
```

---

### ৪. Multi-Region Database (Active-Active Geo-Replication)

বাংলাদেশের প্রবাসীরা (১ কোটি+) বিশ্বের বিভিন্ন দেশ থেকে bKash/Nagad ব্যবহার করেন। Multi-region database তাদের জন্য low latency নিশ্চিত করে।

```
  Multi-Region Active-Active Architecture
  ═══════════════════════════════════════

  ┌──────────────────┐          ┌──────────────────┐
  │  Dhaka Region    │◄────────►│ Singapore Region │
  │  (Primary BD)    │  Bi-dir  │ (Southeast Asia) │
  │                  │  Repl.   │                  │
  │  ┌────────────┐  │          │  ┌────────────┐  │
  │  │ DB Primary │  │          │  │ DB Primary │  │
  │  └─────┬──────┘  │          │  └─────┬──────┘  │
  │        │         │          │        │         │
  │  ┌─────▼──────┐  │          │  ┌─────▼──────┐  │
  │  │ DB Replica │  │          │  │ DB Replica │  │
  │  └────────────┘  │          │  └────────────┘  │
  └────────┬─────────┘          └────────┬─────────┘
           │                             │
           └─────────────┬───────────────┘
                         │
              ┌──────────▼─────────┐
              │  Middle East Region │
              │  (Dubai — প্রবাসী)  │
              │                    │
              │  ┌────────────┐    │
              │  │ DB Primary │    │
              │  └────────────┘    │
              └────────────────────┘

  Conflict Resolution Strategy:
  ─────────────────────────────
  • Last-Writer-Wins (LWW) — timestamp-based
  • Application-level conflict resolution
  • CRDTs (Conflict-free Replicated Data Types)
```

```javascript
// multiRegion/conflictResolver.js
// Active-Active replication-এ conflict handle করা

class ConflictResolver {
    // Last-Writer-Wins — সবচেয়ে সহজ কিন্তু data loss হতে পারে
    static lastWriterWins(localRecord, remoteRecord) {
        if (!localRecord) return remoteRecord;
        if (!remoteRecord) return localRecord;

        return localRecord.updated_at >= remoteRecord.updated_at
            ? localRecord
            : remoteRecord;
    }

    // Balance-এর জন্য CRDT-style merge — PN-Counter
    // দুই region-এ একসাথে debit/credit হলেও সঠিক balance পাওয়া যায়
    static mergeBalance(localOps, remoteOps) {
        const allOps = [...localOps, ...remoteOps];

        // Unique operation ID দিয়ে duplicate বাদ
        const uniqueOps = new Map();
        allOps.forEach(op => uniqueOps.set(op.txn_id, op));

        let balance = 0;
        for (const op of uniqueOps.values()) {
            balance += op.type === 'credit' ? op.amount : -op.amount;
        }

        return balance;
    }

    // Application-level resolution — bKash-এর ক্ষেত্রে
    // একই টাকা দুই region থেকে দুইবার পাঠানো রোধ
    static resolveTransactionConflict(localTxn, remoteTxn) {
        // একই reference ID-র দুটি version — প্রথমটি win করবে
        if (localTxn.reference_id === remoteTxn.reference_id) {
            return localTxn.created_at <= remoteTxn.created_at
                ? localTxn
                : remoteTxn;
        }

        // আলাদা transaction — দুটোই রাখা
        return [localTxn, remoteTxn];
    }
}

module.exports = ConflictResolver;
```

---

### ৫. Data Migration during Resharding

Resharding হলো চলমান সিস্টেমে shard সংখ্যা বাড়ানো বা কমানো। এটি অত্যন্ত জটিল এবং ঝুঁকিপূর্ণ — downtime ছাড়া করতে হয়।

```
  Resharding Process (4 → 8 Shards)
  ══════════════════════════════════

  Phase 1: Double-Write
  ─────────────────────
  ┌────────┐    Write to both
  │  App   │──────────────────┐
  └───┬────┘                  │
      │ Write                 │ Write
      ▼                       ▼
  ┌────────┐            ┌────────┐
  │Old Shard│           │New Shard│  ← নতুন shard-এ copy
  │  (4)    │           │  (8)    │
  └────────┘            └────────┘

  Phase 2: Backfill (পুরানো data copy)
  ─────────────────────────────────────
  Old Shard → batch by batch → New Shard
  (background job, checksum verify)

  Phase 3: Verify (data integrity check)
  ─────────────────────────────────────
  SELECT COUNT(*), SUM(amount) FROM old_shard
  SELECT COUNT(*), SUM(amount) FROM new_shard
  → মিলতে হবে!

  Phase 4: Switch Read
  ─────────────────────
  Read → New Shard (Write এখনো দুটোতে)

  Phase 5: Switch Write + Cleanup
  ─────────────────────────────────
  Write → Only New Shard
  Old Shard → Archive/Delete
```

```php
// app/Jobs/ReshardMigrationJob.php

namespace App\Jobs;

use App\Database\ShardManager;
use App\Database\ConsistentHash;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class ReshardMigrationJob implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $timeout = 3600;

    public function __construct(
        private string $tableName,
        private string $shardKeyColumn,
        private int $batchSize = 5000,
        private int $oldShardCount = 4,
        private int $newShardCount = 8,
    ) {}

    public function handle(ShardManager $shardManager): void
    {
        $oldRing = new ConsistentHash(150);
        $newRing = new ConsistentHash(150);

        for ($i = 0; $i < $this->oldShardCount; $i++) {
            $oldRing->addNode("shard_{$i}");
        }
        for ($i = 0; $i < $this->newShardCount; $i++) {
            $newRing->addNode("shard_{$i}");
        }

        // প্রতিটি পুরানো shard থেকে data পড়ে নতুন shard-এ migrate
        for ($oldShard = 0; $oldShard < $this->oldShardCount; $oldShard++) {
            $this->migrateFromShard($oldShard, $oldRing, $newRing, $shardManager);
        }
    }

    private function migrateFromShard(
        int $oldShardId,
        ConsistentHash $oldRing,
        ConsistentHash $newRing,
        ShardManager $shardManager,
    ): void {
        $offset = 0;

        while (true) {
            $rows = DB::connection("shard_{$oldShardId}")
                ->table($this->tableName)
                ->orderBy('id')
                ->offset($offset)
                ->limit($this->batchSize)
                ->get();

            if ($rows->isEmpty()) break;

            $toMigrate = [];
            foreach ($rows as $row) {
                $key = $row->{$this->shardKeyColumn};
                $oldTarget = $oldRing->getNode($key);
                $newTarget = $newRing->getNode($key);

                // Shard পরিবর্তন হলেই migrate করা
                if ($oldTarget !== $newTarget) {
                    $toMigrate[$newTarget][] = (array) $row;
                }
            }

            // Batch insert নতুন shard-এ
            foreach ($toMigrate as $targetShard => $records) {
                $shardIndex = (int) str_replace('shard_', '', $targetShard);
                DB::connection("shard_{$shardIndex}")
                    ->table($this->tableName)
                    ->upsert($records, ['id'], array_keys($records[0]));
            }

            $offset += $this->batchSize;

            Log::info("Resharding progress", [
                'old_shard' => $oldShardId,
                'offset'    => $offset,
                'migrated'  => count($toMigrate),
            ]);
        }
    }
}
```

---

### ৬. Connection Pooling at Scale

প্রতিটি ডাটাবেস connection-এর জন্য ~10MB RAM লাগে। যখন হাজার হাজার app server থাকে, connection pooling ছাড়া ডাটাবেস crash করবে।

```
  Connection Pooling Problem
  ══════════════════════════

  ছাড়া Pooling:                    সাথে Pooling:
  ──────────────                    ──────────────
  50 App Servers                    50 App Servers
  × 20 conns each                  × 20 conns each
  = 1000 DB connections             = 1000 app-side conns
  (MySQL max ~5000)                       │
                                    ┌─────▼─────┐
                                    │  PgBouncer │
                                    │  / ProxySQL│
                                    └─────┬─────┘
                                          │
                                    = 100 DB connections
                                    (10x কমানো!)
```

```javascript
// connectionPool/adaptivePool.js
// বাংলাদেশের সময় অনুযায়ী adaptive connection pool

const mysql = require('mysql2/promise');

class AdaptiveConnectionPool {
    constructor(config) {
        this.config = config;
        this.pool = null;
        this.metrics = { active: 0, waiting: 0, total: 0 };

        this.createPool(config.minConnections || 10);
        this.startAdaptiveScaling();
    }

    createPool(connectionLimit) {
        if (this.pool) this.pool.end();

        this.pool = mysql.createPool({
            ...this.config,
            connectionLimit,
            waitForConnections: true,
            queueLimit: connectionLimit * 5,
            enableKeepAlive: true,
            keepAliveInitialDelay: 30000,
        });

        this.currentLimit = connectionLimit;
    }

    // বাংলাদেশ সময় অনুযায়ী pool size adjust
    startAdaptiveScaling() {
        setInterval(() => {
            const hour = new Date().toLocaleString('en-US', {
                timeZone: 'Asia/Dhaka',
                hour: 'numeric',
                hour12: false,
            });
            const bdHour = parseInt(hour);

            let targetConnections;

            if (bdHour >= 9 && bdHour <= 11) {
                // সকালের peak — অফিস শুরু, bill payment
                targetConnections = this.config.maxConnections || 100;
            } else if (bdHour >= 12 && bdHour <= 14) {
                // দুপুরের peak — lunch break mobile usage
                targetConnections = Math.floor((this.config.maxConnections || 100) * 0.8);
            } else if (bdHour >= 20 && bdHour <= 23) {
                // রাতের peak — online shopping, recharge
                targetConnections = this.config.maxConnections || 100;
            } else if (bdHour >= 0 && bdHour <= 5) {
                // গভীর রাত — কম traffic
                targetConnections = this.config.minConnections || 10;
            } else {
                targetConnections = Math.floor((this.config.maxConnections || 100) * 0.5);
            }

            if (targetConnections !== this.currentLimit) {
                console.log(`Adjusting pool: ${this.currentLimit} → ${targetConnections} connections`);
                this.createPool(targetConnections);
            }
        }, 5 * 60 * 1000); // প্রতি ৫ মিনিটে check
    }

    async query(sql, params = []) {
        const [rows] = await this.pool.execute(sql, params);
        return rows;
    }

    async getStats() {
        const pool = this.pool.pool;
        return {
            total: pool._allConnections?.length || 0,
            free: pool._freeConnections?.length || 0,
            queued: pool._connectionQueue?.length || 0,
            limit: this.currentLimit,
        };
    }
}

module.exports = AdaptiveConnectionPool;
```

---

## ✅ Pros/Cons প্রতিটি Strategy-র জন্য

```
┌──────────────────────┬──────────────────────────────┬──────────────────────────────┐
│ Strategy             │ সুবিধা (Pros)                 │ অসুবিধা (Cons)                │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Vertical Scaling     │ • সহজ, কোড পরিবর্তন নেই     │ • হার্ডওয়্যার সীমা আছে       │
│                      │ • Consistency সহজ            │ • Single point of failure    │
│                      │ • দ্রুত implement             │ • খরচ exponential বাড়ে       │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Read Replicas        │ • Read throughput বাড়ে       │ • Replication lag            │
│                      │ • Analytics আলাদা করা যায়    │ • Write scale হয় না          │
│                      │ • Failover সহজ              │ • Stale data পড়ার risk      │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Sharding             │ • Write + Read দুটোই scale   │ • Cross-shard query জটিল     │
│                      │ • প্রায় অসীম scalability     │ • Resharding কঠিন            │
│                      │ • Data locality ভালো         │ • Application complexity বাড়ে│
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Partitioning         │ • Query pruning — দ্রুত      │ • Cross-partition query ধীর  │
│                      │ • Data archiving সহজ         │ • Partition key পরিবর্তন কঠিন│
│                      │ • একই সার্ভারে কাজ করে        │ • Index সীমাবদ্ধতা           │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Federation           │ • প্রতিটি DB স্বতন্ত্র scale │ • Cross-service JOIN নেই     │
│                      │ • Technology diversity        │ • Distributed transaction    │
│                      │ • Fault isolation ভালো        │ • Operational complexity     │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Denormalization      │ • Read অনেক দ্রুত            │ • Write ধীর ও জটিল           │
│                      │ • JOIN এড়ানো যায়             │ • Data inconsistency risk    │
│                      │ • Simple queries             │ • Storage বেশি লাগে          │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Database Proxy       │ • Connection multiplexing    │ • অতিরিক্ত network hop       │
│                      │ • Auto failover              │ • Proxy নিজেও SPOF হতে পারে │
│                      │ • Query caching              │ • Debug করা কঠিন হয়          │
└──────────────────────┴──────────────────────────────┴──────────────────────────────┘
```

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ১. অকালে Sharding করা (Premature Sharding)
```
❌ ভুল: ১০ হাজার user-এই sharding শুরু করা
✅ ঠিক: আগে vertical scaling → read replicas → caching → তারপর sharding

বাস্তব অভিজ্ঞতা: বেশিরভাগ বাংলাদেশি startup-এর জন্য
Read Replicas + Redis Cache দিয়ে ১০ লক্ষ user সামলানো সম্ভব
```

### ২. ভুল Shard Key নির্বাচন
```
❌ ভুল: created_at দিয়ে shard → সব নতুন data একটি shard-এ (hotspot)
❌ ভুল: city দিয়ে shard → ঢাকায় ৪০% user (uneven distribution)
✅ ঠিক: user_id বা phone number-এর hash দিয়ে shard
```

### ৩. Replication Lag উপেক্ষা করা
```
❌ ভুল: Write-এর ঠিক পরে Replica থেকে read করা
✅ ঠিক: Critical read-এ useWritePdo() বা sticky session ব্যবহার

সমস্যা:
  User টাকা পাঠালো → balance check → Replica-তে পুরানো balance দেখায়!
  
সমাধান:
  Laravel: 'sticky' => true (config-এ)
  Node.js: write-এর পর readFromMaster() কল
```

### ৪. Cross-Shard Join ডিজাইন করা
```
❌ ভুল: Sharded table-এ অন্য shard-এর data-সহ JOIN query
✅ ঠিক: Denormalization বা application-level join

-- এটি কাজ করবে না:
SELECT t.*, u.name FROM shard_1.transactions t
JOIN shard_3.users u ON t.user_id = u.id  -- ❌ Cross-shard!
```

### ৫. Connection Pool Exhaustion
```
❌ ভুল: প্রতিটি request-এ নতুন connection খোলা
✅ ঠিক: Connection pooling ব্যবহার + ProxySQL/PgBouncer

-- MySQL connection limit check:
SHOW VARIABLES LIKE 'max_connections';  -- default 151!
SHOW STATUS LIKE 'Threads_connected';   -- বর্তমান ব্যবহৃত
```

### ৬. Backup ও Recovery পরিকল্পনা না রাখা
```
❌ ভুল: শুধু scale করা, backup strategy না ভাবা
✅ ঠিক: প্রতিটি shard-এর আলাদা backup, point-in-time recovery test

৮টি shard = ৮টি আলাদা backup schedule + monitoring
একটি shard corrupt হলে বাকি ৭টি চলবে — partial availability
```

### ৭. Monitoring না করা
```
❌ ভুল: Scale করে ভুলে যাওয়া
✅ ঠিক: প্রতিটি shard/replica-র metrics monitor করা

অবশ্যই monitor করুন:
  • Replication lag (seconds_behind_master)
  • Query per shard (load distribution)
  • Connection pool utilization
  • Slow query count per shard
  • Disk usage per partition
```

---

## 📋 সারসংক্ষেপ — তুলনামূলক টেবিল

```
┌───────────────┬──────────┬──────────┬───────────┬──────────────┬─────────────┐
│ বৈশিষ্ট্য     │ Vertical │ Replicas │ Sharding  │ Partitioning │ Federation  │
├───────────────┼──────────┼──────────┼───────────┼──────────────┼─────────────┤
│ Read Scale    │ সীমিত   │ ✅ উচ্চ   │ ✅ উচ্চ    │ মাঝারি       │ ✅ উচ্চ     │
│ Write Scale   │ সীমিত   │ ❌ না     │ ✅ উচ্চ    │ সীমিত        │ ✅ উচ্চ     │
│ Complexity    │ ⭐       │ ⭐⭐      │ ⭐⭐⭐⭐⭐   │ ⭐⭐          │ ⭐⭐⭐       │
│ Cost          │ 💰💰💰   │ 💰💰     │ 💰💰      │ 💰           │ 💰💰       │
│ Consistency   │ ✅ Strong│ ⚠️ Lag    │ ⚠️ Complex│ ✅ Strong     │ ⚠️ Eventual│
│ Availability  │ ❌ SPOF  │ ✅ High   │ ✅ High   │ ❌ SPOF       │ ✅ High    │
│ Data Size     │ < 1TB   │ < 5TB    │ > 10TB+   │ < 5TB        │ > 10TB+    │
│ Implementation│ ১ ঘণ্টা  │ ১-২ দিন  │ ১-৩ মাস   │ ১-২ দিন      │ ২-৪ সপ্তাহ │
│ Use Case      │ MVP,    │ Read-    │ বিশাল    │ সময়-ভিত্তিক │ Micro-     │
│ উদাহরণ        │ ছোট app │ heavy    │ scale    │ data         │ services   │
└───────────────┴──────────┴──────────┴───────────┴──────────────┴─────────────┘
```

### বাংলাদেশের Scale-এ কোনটি কখন?

```
  User সংখ্যা            → প্রস্তাবিত Strategy
  ──────────────           ──────────────────────
  < ১ লক্ষ               → Vertical Scaling + Indexing
  ১-১০ লক্ষ              → Read Replicas + Redis Cache
  ১০-৫০ লক্ষ             → Partitioning + Read Replicas + ProxySQL
  ৫০ লক্ষ - ১ কোটি       → Federation (Microservices)
  ১-১০ কোটি              → Sharding + Consistent Hashing
  ১০ কোটি+ (bKash scale) → Full Architecture (সবকিছু একসাথে)

  bKash (বাস্তব):
  ───────────────
  • ৬ কোটি+ active wallet
  • দৈনিক ২ কোটি+ transaction
  • Peak: ঈদের দিনে ৫x spike
  • Strategy: Sharding + Replicas + Federation + Caching + ProxySQL
```

---

### 🔑 মূল শিক্ষা

> **১.** সবসময় সবচেয়ে সহজ সমাধান দিয়ে শুরু করুন — Vertical → Replicas → Partitioning → Sharding
>
> **২.** Shard key নির্বাচন হলো সবচেয়ে গুরুত্বপূর্ণ সিদ্ধান্ত — পরে পরিবর্তন করা অত্যন্ত ব্যয়বহুল
>
> **৩.** Monitoring ছাড়া scaling অন্ধকারে তীর ছোড়ার সমান
>
> **৪.** Consistent Hashing ব্যবহার করুন — ভবিষ্যতে resharding সহজ হবে
>
> **৫.** প্রতিটি strategy-র trade-off বুঝুন — কোনো silver bullet নেই

---

*"ডাটাবেস স্কেলিং শুধু প্রযুক্তিগত সমস্যা নয় — এটি একটি ব্যবসায়িক সিদ্ধান্ত। কখন, কীভাবে এবং কতটুকু scale করতে হবে — এটি বুঝতে পারাটাই আসল দক্ষতা।"*
