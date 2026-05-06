# 🔑 Idempotency Keys & Message Deduplication

## 📌 সংজ্ঞা ও ধারণা (Definition & Concept)

**Idempotency** মানে হলো একটি অপারেশন যতবারই execute করা হোক না কেন, ফলাফল একবার execute করার মতোই থাকবে। মেসেজিং সিস্টেমে এটি অত্যন্ত গুরুত্বপূর্ণ কারণ network failure, timeout, বা consumer crash এর কারণে একই মেসেজ একাধিকবার প্রসেস হতে পারে।

### 🤔 কেন Duplicate মেসেজ আসে?

Kafka-তে **at-least-once delivery** guarantee দেয়। এর মানে:
- Producer একটি মেসেজ পাঠালো কিন্তু ACK পেলো না (network issue)
- Producer আবার পাঠালো — এখন Kafka-তে দুইটা copy আছে
- Consumer একটি মেসেজ process করলো কিন্তু offset commit করার আগে crash হলো
- Restart এর পর আবার সেই মেসেজ process করবে

### 🔐 Idempotency Key কী?

Idempotency Key হলো একটি unique identifier যা প্রতিটি operation/message এর সাথে যুক্ত থাকে। এই key ব্যবহার করে সিস্টেম detect করতে পারে যে এই operation আগেই process হয়েছে কিনা।

**দুই ধরনের Idempotency Key:**

1. **UUID-based Key:** প্রতিটি request এ একটি random UUID generate করা হয়
2. **Business-key-based:** Business context থেকে naturally unique key তৈরি হয় (যেমন: transaction_id + event_type)

---

## 🌍 বাস্তব জীবনের উদাহরণ (Real-world Example)

### 💰 bKash Duplicate SMS Prevention

ধরুন, আপনি bKash দিয়ে ১০০০ টাকা Send Money করলেন। Transaction সফল হলে আপনার কাছে SMS আসবে। কিন্তু যদি:

1. Notification service মেসেজ পায় এবং SMS পাঠায়
2. Kafka-কে ACK পাঠানোর আগে service crash হয়
3. Restart হলে Kafka আবার সেই মেসেজ দেয়
4. **Without deduplication:** আপনি দুইবার SMS পাবেন! 😱

**bKash এর সমাধান:**
- প্রতিটি transaction এর জন্য idempotency key: `txn_id + notification_type`
- যেমন: `TXN123456_SMS_SENDMONEY_SUCCESS`
- এই key Redis-এ store করা হয় ১ ঘণ্টা TTL সহ
- দ্বিতীয়বার এই key আসলে, SMS পাঠানো skip করা হয়

### 🛵 Pathao Ride Duplicate Charge Prevention

- Rider এর trip শেষ হলো, payment event fire হলো
- Payment service টাকা কাটলো কিন্তু ACK দিতে পারলো না
- আবার event আসলো — **idempotency key ছাড়া দুইবার টাকা কাটবে!**
- Solution: `ride_id + payment_event` as idempotency key

---

## 📊 ASCII Diagram

### At-Least-Once Delivery Problem:

```
┌──────────┐         ┌─────────┐         ┌──────────────┐
│ Producer │────────▶│  Kafka  │────────▶│   Consumer   │
│ (bKash)  │         │ Broker  │         │(SMS Service) │
└──────────┘         └─────────┘         └──────────────┘
     │                                          │
     │  1. Send Message                         │
     │─────────────────────────────────────────▶│
     │                                          │
     │  2. Process (Send SMS) ✅                │
     │                                          │
     │  3. Commit Offset... ❌ CRASH!           │
     │                                          │
     │  4. Restart → Re-deliver same message    │
     │─────────────────────────────────────────▶│
     │                                          │
     │  5. Process AGAIN (Duplicate SMS!) 😱    │
     │                                          │
```

### Idempotency Deduplication Flow:

```
┌──────────────────────────────────────────────────────────────┐
│                    Deduplication Flow                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Message Received                                             │
│       │                                                       │
│       ▼                                                       │
│  ┌─────────────────────┐                                     │
│  │ Extract Idempotency │                                     │
│  │ Key from Message    │                                     │
│  └──────────┬──────────┘                                     │
│             │                                                 │
│             ▼                                                 │
│  ┌─────────────────────┐      ┌───────────────┐             │
│  │  Key exists in      │─YES─▶│ Skip/Discard  │             │
│  │  Dedup Store?       │      │ (Already Done) │             │
│  └──────────┬──────────┘      └───────────────┘             │
│             │ NO                                              │
│             ▼                                                 │
│  ┌─────────────────────┐                                     │
│  │  Process Message    │                                     │
│  │  (Business Logic)   │                                     │
│  └──────────┬──────────┘                                     │
│             │                                                 │
│             ▼                                                 │
│  ┌─────────────────────┐                                     │
│  │  Store Key in       │                                     │
│  │  Dedup Store (TTL)  │                                     │
│  └──────────┬──────────┘                                     │
│             │                                                 │
│             ▼                                                 │
│  ┌─────────────────────┐                                     │
│  │  Commit Offset      │                                     │
│  └─────────────────────┘                                     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Deduplication Strategies:

```
┌────────────────────────────────────────────────────────┐
│              Deduplication Strategies                    │
├──────────────┬──────────────────┬──────────────────────┤
│  Inbox Table │   Redis Set      │   Bloom Filter       │
│  (Database)  │   (In-Memory)    │   (Probabilistic)    │
├──────────────┼──────────────────┼──────────────────────┤
│ ✅ Durable   │ ✅ Very Fast     │ ✅ Memory Efficient  │
│ ✅ Exact     │ ✅ TTL Support   │ ✅ Extremely Fast    │
│ ❌ Slow      │ ❌ Memory Cost   │ ❌ False Positives   │
│ ❌ DB Load   │ ❌ Not Durable   │ ❌ Can't Delete      │
└──────────────┴──────────────────┴──────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Idempotency Key Generation:

```php
<?php

/**
 * bKash SMS Notification Deduplication Service
 * Duplicate SMS প্রতিরোধ করার জন্য Idempotency Key ব্যবহার
 */

class IdempotencyKeyGenerator
{
    /**
     * UUID-based idempotency key
     */
    public static function generateUUID(): string
    {
        return sprintf(
            '%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
            mt_rand(0, 0xffff), mt_rand(0, 0xffff),
            mt_rand(0, 0xffff),
            mt_rand(0, 0x0fff) | 0x4000,
            mt_rand(0, 0x3fff) | 0x8000,
            mt_rand(0, 0xffff), mt_rand(0, 0xffff), mt_rand(0, 0xffff)
        );
    }

    /**
     * Business-key-based idempotency key
     * bKash transaction context থেকে key তৈরি
     */
    public static function fromBusinessContext(
        string $transactionId,
        string $eventType,
        string $notificationType
    ): string {
        return hash('sha256', "{$transactionId}_{$eventType}_{$notificationType}");
    }
}

/**
 * Redis-based Deduplication Store
 */
class RedisDeduplicationStore
{
    private \Redis $redis;
    private int $ttlSeconds;

    public function __construct(\Redis $redis, int $ttlSeconds = 3600)
    {
        $this->redis = $redis;
        $this->ttlSeconds = $ttlSeconds;
    }

    /**
     * Check if message already processed
     */
    public function isDuplicate(string $idempotencyKey): bool
    {
        return $this->redis->exists("dedup:{$idempotencyKey}") > 0;
    }

    /**
     * Mark message as processed
     */
    public function markProcessed(string $idempotencyKey, array $metadata = []): void
    {
        $value = json_encode([
            'processed_at' => date('c'),
            'metadata' => $metadata,
        ]);

        $this->redis->setex("dedup:{$idempotencyKey}", $this->ttlSeconds, $value);
    }
}

/**
 * Database Inbox Pattern - Durable Deduplication
 */
class InboxDeduplicationStore
{
    private \PDO $pdo;

    public function __construct(\PDO $pdo)
    {
        $this->pdo = $pdo;
        $this->createTableIfNotExists();
    }

    private function createTableIfNotExists(): void
    {
        $this->pdo->exec("
            CREATE TABLE IF NOT EXISTS processed_messages (
                idempotency_key VARCHAR(255) PRIMARY KEY,
                topic VARCHAR(100) NOT NULL,
                partition_id INT NOT NULL,
                offset_id BIGINT NOT NULL,
                processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                expires_at TIMESTAMP NOT NULL,
                INDEX idx_expires_at (expires_at)
            )
        ");
    }

    public function isDuplicate(string $idempotencyKey): bool
    {
        $stmt = $this->pdo->prepare(
            "SELECT 1 FROM processed_messages 
             WHERE idempotency_key = ? AND expires_at > NOW()"
        );
        $stmt->execute([$idempotencyKey]);
        return $stmt->fetchColumn() !== false;
    }

    public function markProcessed(
        string $idempotencyKey,
        string $topic,
        int $partition,
        int $offset,
        int $ttlHours = 24
    ): void {
        $stmt = $this->pdo->prepare(
            "INSERT IGNORE INTO processed_messages 
             (idempotency_key, topic, partition_id, offset_id, expires_at)
             VALUES (?, ?, ?, ?, DATE_ADD(NOW(), INTERVAL ? HOUR))"
        );
        $stmt->execute([$idempotencyKey, $topic, $partition, $offset, $ttlHours]);
    }

    /**
     * Expired records cleanup - cron job এ চালাতে হবে
     */
    public function cleanupExpired(): int
    {
        $stmt = $this->pdo->prepare(
            "DELETE FROM processed_messages WHERE expires_at < NOW() LIMIT 10000"
        );
        $stmt->execute();
        return $stmt->rowCount();
    }
}

/**
 * bKash SMS Notification Consumer - Idempotent
 */
class BkashSmsNotificationConsumer
{
    private RedisDeduplicationStore $dedupStore;
    private SmsGateway $smsGateway;

    public function __construct(
        RedisDeduplicationStore $dedupStore,
        SmsGateway $smsGateway
    ) {
        $this->dedupStore = $dedupStore;
        $this->smsGateway = $smsGateway;
    }

    public function consume(array $message): void
    {
        $transactionId = $message['transaction_id'];
        $eventType = $message['event_type'];
        $phone = $message['phone'];

        // Business-key-based idempotency key তৈরি
        $idempotencyKey = IdempotencyKeyGenerator::fromBusinessContext(
            $transactionId,
            $eventType,
            'SMS'
        );

        // Duplicate check
        if ($this->dedupStore->isDuplicate($idempotencyKey)) {
            echo "⚠️ Duplicate detected! Skipping SMS for TXN: {$transactionId}\n";
            return;
        }

        // Process - SMS পাঠাও
        $smsBody = $this->buildSmsBody($message);
        $this->smsGateway->send($phone, $smsBody);

        // Mark as processed
        $this->dedupStore->markProcessed($idempotencyKey, [
            'transaction_id' => $transactionId,
            'phone' => $phone,
        ]);

        echo "✅ SMS sent for TXN: {$transactionId}\n";
    }

    private function buildSmsBody(array $message): string
    {
        $amount = $message['amount'];
        $type = $message['event_type'];

        return match ($type) {
            'SEND_MONEY_SUCCESS' => "আপনার bKash অ্যাকাউন্ট থেকে ৳{$amount} Send Money সফল হয়েছে। TrxID: {$message['transaction_id']}",
            'CASH_IN_SUCCESS' => "আপনার bKash অ্যাকাউন্টে ৳{$amount} Cash In হয়েছে। TrxID: {$message['transaction_id']}",
            'PAYMENT_SUCCESS' => "৳{$amount} Payment সফল। TrxID: {$message['transaction_id']}",
            default => "bKash Transaction Update: TrxID {$message['transaction_id']}",
        };
    }
}

/**
 * Kafka Idempotent Producer Configuration (PHP rdkafka)
 */
class KafkaIdempotentProducer
{
    private \RdKafka\Producer $producer;

    public function __construct()
    {
        $conf = new \RdKafka\Conf();

        // Idempotent Producer সক্রিয় করা
        $conf->set('enable.idempotence', 'true');
        $conf->set('acks', 'all'); // idempotence এর জন্য required
        $conf->set('max.in.flight.requests.per.connection', '5');
        $conf->set('retries', '2147483647'); // infinite retries

        // Transactional Producer (Exactly-Once Semantics)
        $conf->set('transactional.id', 'bkash-notification-producer-1');

        $this->producer = new \RdKafka\Producer($conf);
        $this->producer->addBrokers('kafka1:9092,kafka2:9092,kafka3:9092');
    }

    public function sendTransactional(string $topic, string $key, array $payload): void
    {
        $this->producer->initTransactions(10000);
        $this->producer->beginTransaction();

        try {
            $topicHandle = $this->producer->newTopic($topic);
            $topicHandle->producev(
                RD_KAFKA_PARTITION_UA,
                0,
                json_encode($payload),
                $key
            );
            $this->producer->flush(5000);
            $this->producer->commitTransaction(10000);
        } catch (\Exception $e) {
            $this->producer->abortTransaction(10000);
            throw $e;
        }
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * bKash SMS Notification Deduplication Service
 * Node.js implementation with KafkaJS
 */

const { Kafka } = require('kafkajs');
const Redis = require('ioredis');
const crypto = require('crypto');

// ==========================================
// Idempotency Key Generator
// ==========================================
class IdempotencyKeyGenerator {
    /**
     * UUID v4 based key
     */
    static generateUUID() {
        return crypto.randomUUID();
    }

    /**
     * Business context based key
     * bKash transaction থেকে deterministic key তৈরি
     */
    static fromBusinessContext(transactionId, eventType, notificationType) {
        const raw = `${transactionId}_${eventType}_${notificationType}`;
        return crypto.createHash('sha256').update(raw).digest('hex');
    }

    /**
     * Composite key - multiple fields combine করে
     */
    static compositeKey(...parts) {
        return crypto.createHash('sha256').update(parts.join('|')).digest('hex');
    }
}

// ==========================================
// Redis Deduplication Store
// ==========================================
class RedisDeduplicationStore {
    constructor(redis, ttlSeconds = 3600) {
        this.redis = redis;
        this.ttlSeconds = ttlSeconds;
        this.keyPrefix = 'dedup:';
    }

    async isDuplicate(idempotencyKey) {
        const exists = await this.redis.exists(`${this.keyPrefix}${idempotencyKey}`);
        return exists === 1;
    }

    async markProcessed(idempotencyKey, metadata = {}) {
        const value = JSON.stringify({
            processedAt: new Date().toISOString(),
            metadata,
        });

        await this.redis.setex(
            `${this.keyPrefix}${idempotencyKey}`,
            this.ttlSeconds,
            value
        );
    }

    /**
     * Atomic check-and-set: duplicate check + mark একসাথে
     * Race condition প্রতিরোধ করে
     */
    async checkAndMark(idempotencyKey) {
        const result = await this.redis.set(
            `${this.keyPrefix}${idempotencyKey}`,
            JSON.stringify({ processedAt: new Date().toISOString() }),
            'EX', this.ttlSeconds,
            'NX' // Only set if NOT exists
        );

        // result === 'OK' means key was NEW (not duplicate)
        // result === null means key EXISTED (duplicate!)
        return result === null; // true = duplicate
    }
}

// ==========================================
// Database Inbox Pattern
// ==========================================
class InboxDeduplicationStore {
    constructor(knex) {
        this.knex = knex;
    }

    async initialize() {
        const exists = await this.knex.schema.hasTable('processed_messages');
        if (!exists) {
            await this.knex.schema.createTable('processed_messages', (table) => {
                table.string('idempotency_key', 255).primary();
                table.string('topic', 100).notNullable();
                table.integer('partition_id').notNullable();
                table.bigInteger('offset_id').notNullable();
                table.timestamp('processed_at').defaultTo(this.knex.fn.now());
                table.timestamp('expires_at').notNullable();
                table.index('expires_at');
            });
        }
    }

    async isDuplicate(idempotencyKey) {
        const record = await this.knex('processed_messages')
            .where('idempotency_key', idempotencyKey)
            .where('expires_at', '>', new Date())
            .first();

        return !!record;
    }

    async markProcessed(idempotencyKey, topic, partition, offset, ttlHours = 24) {
        const expiresAt = new Date(Date.now() + ttlHours * 60 * 60 * 1000);

        await this.knex('processed_messages')
            .insert({
                idempotency_key: idempotencyKey,
                topic,
                partition_id: partition,
                offset_id: offset,
                expires_at: expiresAt,
            })
            .onConflict('idempotency_key')
            .ignore();
    }

    async cleanupExpired(batchSize = 10000) {
        const deleted = await this.knex('processed_messages')
            .where('expires_at', '<', new Date())
            .limit(batchSize)
            .del();

        return deleted;
    }
}

// ==========================================
// bKash SMS Consumer with Deduplication
// ==========================================
class BkashSmsNotificationConsumer {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'bkash-sms-consumer',
            brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        });

        this.redis = new Redis({
            host: 'redis-cluster.bkash.internal',
            port: 6379,
        });

        this.dedupStore = new RedisDeduplicationStore(this.redis, 3600);
        this.consumer = this.kafka.consumer({
            groupId: 'bkash-sms-notification-group',
        });
    }

    async start() {
        await this.consumer.connect();
        await this.consumer.subscribe({
            topic: 'bkash.transactions.notifications',
            fromBeginning: false,
        });

        await this.consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                await this.processMessage(topic, partition, message);
            },
        });
    }

    async processMessage(topic, partition, message) {
        const payload = JSON.parse(message.value.toString());
        const { transaction_id, event_type, phone, amount } = payload;

        // Idempotency key তৈরি
        const idempotencyKey = IdempotencyKeyGenerator.fromBusinessContext(
            transaction_id,
            event_type,
            'SMS'
        );

        // Atomic duplicate check (race-condition safe)
        const isDuplicate = await this.dedupStore.checkAndMark(idempotencyKey);

        if (isDuplicate) {
            console.log(`⚠️ Duplicate! Skipping SMS for TXN: ${transaction_id}`);
            return;
        }

        // SMS পাঠাও
        const smsBody = this.buildSmsBody(payload);
        await this.sendSms(phone, smsBody);

        console.log(`✅ SMS sent for TXN: ${transaction_id}`);
    }

    buildSmsBody({ event_type, amount, transaction_id }) {
        const templates = {
            SEND_MONEY_SUCCESS: `আপনার bKash থেকে ৳${amount} Send Money সফল। TrxID: ${transaction_id}`,
            CASH_IN_SUCCESS: `আপনার bKash এ ৳${amount} Cash In হয়েছে। TrxID: ${transaction_id}`,
            PAYMENT_SUCCESS: `৳${amount} Payment সফল। TrxID: ${transaction_id}`,
        };
        return templates[event_type] || `bKash Update: TrxID ${transaction_id}`;
    }

    async sendSms(phone, body) {
        // SMS Gateway API call
        console.log(`📱 Sending SMS to ${phone}: ${body}`);
    }
}

// ==========================================
// Kafka Idempotent Producer (Exactly-Once)
// ==========================================
class KafkaIdempotentProducer {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'bkash-txn-producer',
            brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        });

        this.producer = this.kafka.producer({
            idempotent: true,           // Idempotent producer সক্রিয়
            maxInFlightRequests: 5,     // Ordering guarantee সহ
            transactionalId: 'bkash-notification-producer-1', // EOS
        });
    }

    async sendTransactional(topic, key, payload) {
        const transaction = await this.producer.transaction();

        try {
            await transaction.send({
                topic,
                messages: [{
                    key,
                    value: JSON.stringify(payload),
                    headers: {
                        'idempotency-key': IdempotencyKeyGenerator.fromBusinessContext(
                            payload.transaction_id,
                            payload.event_type,
                            'NOTIFICATION'
                        ),
                        'produced-at': new Date().toISOString(),
                    },
                }],
            });

            await transaction.commit();
            console.log(`✅ Transaction committed for key: ${key}`);
        } catch (error) {
            await transaction.abort();
            console.error(`❌ Transaction aborted: ${error.message}`);
            throw error;
        }
    }
}

// ==========================================
// Distributed Deduplication with Bloom Filter
// ==========================================
class BloomFilterDedup {
    constructor(expectedItems = 1000000, falsePositiveRate = 0.01) {
        // Simple bloom filter implementation
        this.size = Math.ceil(
            -(expectedItems * Math.log(falsePositiveRate)) / (Math.log(2) ** 2)
        );
        this.hashCount = Math.ceil((this.size / expectedItems) * Math.log(2));
        this.bitArray = new Uint8Array(Math.ceil(this.size / 8));
    }

    _hash(key, seed) {
        let hash = seed;
        for (let i = 0; i < key.length; i++) {
            hash = ((hash << 5) - hash + key.charCodeAt(i)) | 0;
        }
        return Math.abs(hash) % this.size;
    }

    add(key) {
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this._hash(key, i + 1);
            this.bitArray[Math.floor(pos / 8)] |= (1 << (pos % 8));
        }
    }

    mightContain(key) {
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this._hash(key, i + 1);
            if (!(this.bitArray[Math.floor(pos / 8)] & (1 << (pos % 8)))) {
                return false; // Definitely NOT in set
            }
        }
        return true; // Might be in set (possible false positive)
    }
}

// ==========================================
// Usage Example
// ==========================================
async function main() {
    const consumer = new BkashSmsNotificationConsumer();
    await consumer.start();
    console.log('🚀 bKash SMS Consumer started with deduplication!');
}

main().catch(console.error);
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন দরকার |
|-----------|-----------|
| Financial transactions (bKash, Nagad) | একবারের বেশি টাকা কাটা যাবে না |
| SMS/Email notifications | Duplicate notification বিরক্তিকর |
| Order processing (Daraz) | একই order দুইবার place হওয়া যাবে না |
| Inventory update | Stock count ভুল হয়ে যাবে |
| Event-driven microservices | At-least-once delivery তে duplicate আসবেই |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন দরকার নেই |
|-----------|--------------|
| Analytics/logging events | Duplicate log তেমন ক্ষতি করে না |
| Read-only operations | কোনো state change নেই |
| Naturally idempotent ops | HTTP GET, database SELECT |
| Time-series data | Duplicate data point acceptable |

## 🎯 Idempotency Key Strategy Selection:

```
┌─────────────────────────────────────────────────────────────┐
│          Key Strategy Decision Tree                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  প্রশ্ন: Message এ natural unique ID আছে?                  │
│       │                                                      │
│       ├── হ্যাঁ → Business Key ব্যবহার করুন                │
│       │         (transaction_id, order_id, etc.)             │
│       │                                                      │
│       └── না → UUID generate করুন                           │
│              (Producer side এ generate করে header এ রাখুন)  │
│                                                              │
│  প্রশ্ন: কতক্ষণ duplicate আসতে পারে?                       │
│       │                                                      │
│       ├── মিনিট → Redis (TTL: 5-30 min)                    │
│       ├── ঘণ্টা → Redis (TTL: 1-24 hours)                  │
│       └── দিন   → Database Inbox (TTL: days/weeks)          │
│                                                              │
│  প্রশ্ন: False positive acceptable?                         │
│       │                                                      │
│       ├── না → Redis/Database (exact matching)              │
│       └── হ্যাঁ → Bloom Filter (memory efficient)           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 📚 TTL Guidelines:

| Use Case | Recommended TTL | Store |
|----------|----------------|-------|
| Payment dedup | 24 hours | Redis + DB backup |
| SMS dedup | 1 hour | Redis |
| Order creation | 7 days | Database |
| API request dedup | 5 minutes | Redis |
| Event sourcing | Forever | Database |

---

## 🔑 Key Takeaways

1. **At-least-once + Idempotency = Exactly-once (application level)**
2. **Business key > UUID** যখন possible
3. **Redis atomic SET NX** race condition prevent করে
4. **TTL** অবশ্যই set করুন — না হলে storage বাড়তেই থাকবে
5. **Kafka Idempotent Producer** broker-level duplicate prevent করে
6. **Transactional Producer** cross-topic atomicity দেয়
