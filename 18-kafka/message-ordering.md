# 📊 Message Ordering Guarantees

## 📌 সংজ্ঞা ও ধারণা (Definition & Concept)

**Message Ordering** মানে হলো মেসেজগুলো যে ক্রমে (order) পাঠানো হয়েছে, ঠিক সেই ক্রমেই consume/process করা হবে। Kafka শুধুমাত্র **partition level** এ ordering guarantee দেয় — topic level এ নয়।

### 🤔 কেন Ordering গুরুত্বপূর্ণ?

ধরুন bKash-এ একটি অ্যাকাউন্টে এই events ঘটলো:
1. Balance: ৳5000 → Debit ৳3000 (Event 1)
2. Balance: ৳2000 → Credit ৳1000 (Event 2)
3. Balance: ৳3000 → Debit ৳2500 (Event 3)

যদি Event 3 আগে process হয় Event 1 এর আগে, তাহলে:
- System ভাববে balance ৳5000 আছে, ৳2500 কাটা যাবে ✅
- কিন্তু আসলে Event 1 প্রথমে হওয়া উচিত ছিল!
- **Wrong balance, wrong decisions, financial disaster!** 💀

### 🎯 Kafka-র Ordering Model:

- **Partition-level ordering:** একটি partition এর মধ্যে মেসেজ FIFO order এ থাকে
- **Topic-level ordering:** কোনো guarantee নেই (partitions parallel process হয়)
- **Partition Key:** same key = same partition = ordered

---

## 🌍 বাস্তব জীবনের উদাহরণ (Real-world Example)

### 💰 bKash Transaction History Ordering

bKash-এ প্রতিটি account এর transaction history সঠিক ক্রমে দেখাতে হবে:

**সমস্যা:**
- User A তার bKash থেকে ৳1000 Send Money করলো (Debit Event)
- তারপর ৳500 Cash In করলো (Credit Event)
- Consumer যদি Credit আগে process করে, Debit পরে — তাহলে balance calculation ভুল!

**সমাধান:**
- Partition Key = `account_number` (e.g., `01712345678`)
- একই account এর সব events একই partition এ যাবে
- সেই partition এর মধ্যে FIFO order maintain হবে

### 📰 Prothom Alo Article Publishing Pipeline

- Editor article লিখলেন → `CREATED` event
- Review হলো → `REVIEWED` event
- Publish হলো → `PUBLISHED` event
- যদি `PUBLISHED` আগে আসে `CREATED` এর আগে — system crash! 
- Partition Key = `article_id`

### 🛒 Daraz Order State Machine

Order lifecycle events must be in order:
```
PLACED → CONFIRMED → SHIPPED → DELIVERED
```

যদি `SHIPPED` আগে আসে `CONFIRMED` এর আগে — invalid state transition!

---

## 📊 ASCII Diagram

### Kafka Partition-Level Ordering:

```
┌─────────────────────────────────────────────────────────────┐
│                    Topic: bkash.transactions                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Partition 0 (Account: 01711111111)                         │
│  ┌─────┬─────┬─────┬─────┬─────┐                          │
│  │ E1  │ E2  │ E3  │ E4  │ E5  │ ──▶ FIFO Order ✅        │
│  │Debit│Crdt │Debit│Crdt │Debit│                          │
│  └─────┴─────┴─────┴─────┴─────┘                          │
│                                                              │
│  Partition 1 (Account: 01722222222)                         │
│  ┌─────┬─────┬─────┬─────┐                                │
│  │ E1  │ E2  │ E3  │ E4  │ ──▶ FIFO Order ✅              │
│  │Crdt │Debit│Debit│Crdt │                                │
│  └─────┴─────┴─────┴─────┘                                │
│                                                              │
│  Partition 2 (Account: 01733333333)                         │
│  ┌─────┬─────┬─────┐                                      │
│  │ E1  │ E2  │ E3  │ ──▶ FIFO Order ✅                    │
│  │Debit│Debit│Crdt │                                      │
│  └─────┴─────┴─────┘                                      │
│                                                              │
│  ⚠️ Cross-partition ordering NOT guaranteed!                 │
│  Partition 0 এর E1 আর Partition 1 এর E1 এর মধ্যে          │
│  কোনটি আগে process হবে — guaranteed নয়!                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Partition Key Selection Strategy:

```
┌──────────────────────────────────────────────────────────────┐
│              Partition Key Selection                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐     hash(key) % partitions     ┌────────┐  │
│  │   Message   │ ─────────────────────────────▶  │Partition│  │
│  │ key="01711" │     = partition number          │   0    │  │
│  └─────────────┘                                 └────────┘  │
│                                                               │
│  Good Keys:                    Bad Keys:                      │
│  ✅ account_number             ❌ timestamp                   │
│  ✅ user_id                    ❌ random UUID                  │
│  ✅ order_id                   ❌ null (round-robin)           │
│  ✅ device_id                  ❌ country_code (skewed)        │
│                                                               │
│  Key Selection Rules:                                         │
│  1. Same entity's events → same key                          │
│  2. Good cardinality (না খুব কম, না খুব বেশি)              │
│  3. Even distribution across partitions                       │
│  4. Stable (change হবে না)                                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Ordering vs Parallelism Trade-off:

```
┌──────────────────────────────────────────────────────────────┐
│         Ordering vs Parallelism Spectrum                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Strong Ordering ◄──────────────────────▶ Max Parallelism    │
│                                                               │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │ 1 Parti- │     │ N Parti- │     │ N Parti- │            │
│  │ tion     │     │ tions +  │     │ tions +  │            │
│  │ (Total   │     │ Key-based│     │ No Key   │            │
│  │  Order)  │     │ Order    │     │ (No      │            │
│  │          │     │          │     │  Order)  │            │
│  └──────────┘     └──────────┘     └──────────┘            │
│  Throughput:      Throughput:      Throughput:               │
│  ❌ Low           ✅ Balanced      ✅✅ Maximum              │
│  Ordering:        Ordering:        Ordering:                 │
│  ✅✅ Total       ✅ Per-entity    ❌ None                   │
│                                                               │
│  Use: Audit log   Use: bKash       Use: Analytics            │
│       Ledger          Accounts         Logs                  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Rebalancing Impact on Ordering:

```
┌──────────────────────────────────────────────────────────────┐
│            Consumer Rebalance Impact                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  BEFORE Rebalance:                                           │
│  Consumer A: [P0, P1] ── processing msg offset 100          │
│  Consumer B: [P2, P3]                                        │
│                                                               │
│  Consumer A CRASHES! → Rebalance triggered                   │
│                                                               │
│  AFTER Rebalance:                                            │
│  Consumer B: [P0, P1, P2, P3]                                │
│              │                                                │
│              ▼                                                │
│  P0 last committed offset: 95                                │
│  Messages 96-100 may be RE-PROCESSED! ⚠️                     │
│  (They were processed but offset not committed)              │
│                                                               │
│  Solution:                                                    │
│  1. Idempotent processing (duplicate safe)                   │
│  2. Cooperative sticky assignor (minimize reassignment)      │
│  3. Static group membership (no unnecessary rebalance)       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * bKash Transaction Event Producer - Ordering Guaranteed
 * Account number কে partition key হিসেবে ব্যবহার করে ordering নিশ্চিত করা
 */

class OrderedTransactionProducer
{
    private \RdKafka\Producer $producer;
    private string $topic = 'bkash.account.transactions';

    public function __construct()
    {
        $conf = new \RdKafka\Conf();
        $conf->set('bootstrap.servers', 'kafka1:9092,kafka2:9092');
        $conf->set('enable.idempotence', 'true');
        $conf->set('acks', 'all');
        // max.in.flight = 1 ensures strict ordering even during retries
        $conf->set('max.in.flight.requests.per.connection', '1');

        $this->producer = new \RdKafka\Producer($conf);
    }

    /**
     * Transaction event পাঠানো - account number দিয়ে partition select
     */
    public function publishTransactionEvent(array $event): void
    {
        $accountNumber = $event['account_number']; // Partition Key
        $sequenceNumber = $event['sequence_number'];

        $message = [
            'event_id' => uniqid('evt_', true),
            'account_number' => $accountNumber,
            'event_type' => $event['type'],
            'amount' => $event['amount'],
            'balance_after' => $event['balance_after'],
            'sequence_number' => $sequenceNumber,
            'timestamp' => microtime(true),
        ];

        $topic = $this->producer->newTopic($this->topic);
        $topic->producev(
            RD_KAFKA_PARTITION_UA,  // Kafka will select partition based on key
            0,
            json_encode($message),
            $accountNumber          // KEY: same account → same partition
        );

        $this->producer->flush(5000);
    }

    /**
     * Bulk events publish - same account এর events ordered থাকবে
     */
    public function publishBatch(array $events): void
    {
        $topic = $this->producer->newTopic($this->topic);

        foreach ($events as $event) {
            $topic->producev(
                RD_KAFKA_PARTITION_UA,
                0,
                json_encode($event),
                $event['account_number'] // Same key = same partition = ordered
            );
        }

        $this->producer->flush(10000);
    }
}

/**
 * Ordered Consumer with Sequence Number Validation
 * Out-of-order detection এবং buffering
 */
class OrderedTransactionConsumer
{
    private array $expectedSequence = []; // account → next expected seq
    private array $outOfOrderBuffer = []; // account → buffered messages
    private int $maxBufferSize = 1000;

    /**
     * Message process করা - ordering validate সহ
     */
    public function processMessage(array $message): void
    {
        $account = $message['account_number'];
        $sequence = $message['sequence_number'];

        // প্রথম message হলে sequence initialize
        if (!isset($this->expectedSequence[$account])) {
            $this->expectedSequence[$account] = $sequence;
        }

        $expected = $this->expectedSequence[$account];

        if ($sequence === $expected) {
            // ✅ Expected order - process করো
            $this->executeBusinessLogic($message);
            $this->expectedSequence[$account] = $expected + 1;

            // Buffer এ পরবর্তী messages আছে কিনা check
            $this->drainBuffer($account);
        } elseif ($sequence > $expected) {
            // ⚠️ Gap detected - buffer করো
            $this->bufferMessage($account, $sequence, $message);
            echo "⚠️ Out-of-order! Expected seq {$expected}, got {$sequence} for {$account}\n";
        } else {
            // 🔄 Duplicate (already processed) - skip
            echo "🔄 Duplicate seq {$sequence} for {$account}, skipping\n";
        }
    }

    private function bufferMessage(string $account, int $sequence, array $message): void
    {
        if (!isset($this->outOfOrderBuffer[$account])) {
            $this->outOfOrderBuffer[$account] = [];
        }

        if (count($this->outOfOrderBuffer[$account]) >= $this->maxBufferSize) {
            throw new \RuntimeException(
                "Buffer overflow for account {$account}! Possible data loss."
            );
        }

        $this->outOfOrderBuffer[$account][$sequence] = $message;
    }

    /**
     * Buffer থেকে sequential messages drain করা
     */
    private function drainBuffer(string $account): void
    {
        if (!isset($this->outOfOrderBuffer[$account])) {
            return;
        }

        while (true) {
            $expected = $this->expectedSequence[$account];

            if (!isset($this->outOfOrderBuffer[$account][$expected])) {
                break;
            }

            $message = $this->outOfOrderBuffer[$account][$expected];
            unset($this->outOfOrderBuffer[$account][$expected]);

            $this->executeBusinessLogic($message);
            $this->expectedSequence[$account] = $expected + 1;
        }

        // Buffer empty হলে cleanup
        if (empty($this->outOfOrderBuffer[$account])) {
            unset($this->outOfOrderBuffer[$account]);
        }
    }

    private function executeBusinessLogic(array $message): void
    {
        $type = $message['event_type'];
        $amount = $message['amount'];
        $account = $message['account_number'];
        $balance = $message['balance_after'];

        echo "✅ Processing: {$type} ৳{$amount} for {$account} (Balance: ৳{$balance})\n";

        // Update account state, send notifications, etc.
    }
}

/**
 * Single-Writer Pattern
 * একটি entity-র জন্য শুধুমাত্র একটি writer থাকবে
 */
class SingleWriterPartitionAssignor
{
    private array $entityOwnership = [];
    private string $lockPrefix = 'writer_lock:';
    private \Redis $redis;

    public function __construct(\Redis $redis)
    {
        $this->redis = $redis;
    }

    /**
     * Entity-র exclusive ownership নেওয়া
     */
    public function acquireOwnership(string $entityId, string $writerId, int $ttl = 30): bool
    {
        $lockKey = $this->lockPrefix . $entityId;

        // SET NX - only if not exists
        $acquired = $this->redis->set($lockKey, $writerId, ['NX', 'EX' => $ttl]);

        if ($acquired) {
            $this->entityOwnership[$entityId] = $writerId;
            return true;
        }

        // Already owned — check if it's us
        $currentOwner = $this->redis->get($lockKey);
        return $currentOwner === $writerId;
    }

    /**
     * Heartbeat - ownership renew করা
     */
    public function renewOwnership(string $entityId, string $writerId, int $ttl = 30): bool
    {
        $lockKey = $this->lockPrefix . $entityId;
        $currentOwner = $this->redis->get($lockKey);

        if ($currentOwner === $writerId) {
            $this->redis->expire($lockKey, $ttl);
            return true;
        }

        return false;
    }
}

/**
 * Partition Key Strategy - Hot Partition Prevention
 */
class PartitionKeyStrategy
{
    /**
     * Simple hash-based key (default Kafka behavior)
     */
    public static function simpleKey(string $accountNumber): string
    {
        return $accountNumber;
    }

    /**
     * Compound key - entity + sub-entity ordering
     * একই account এর মধ্যে transaction type অনুযায়ী আলাদা ordering
     */
    public static function compoundKey(string $accountNumber, string $transactionType): string
    {
        return "{$accountNumber}:{$transactionType}";
    }

    /**
     * Time-bucketed key - prevents hot partition for very active accounts
     * (Ordering guarantee weakened to per-time-bucket)
     */
    public static function timeBucketedKey(string $accountNumber, int $bucketMinutes = 5): string
    {
        $bucket = floor(time() / ($bucketMinutes * 60));
        return "{$accountNumber}:{$bucket}";
    }

    /**
     * Salted key - distributes load but loses ordering
     * শুধুমাত্র ordering জরুরি না হলে ব্যবহার করুন
     */
    public static function saltedKey(string $baseKey, int $saltRange = 10): string
    {
        $salt = crc32($baseKey) % $saltRange;
        return "{$baseKey}:{$salt}";
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * bKash Transaction Ordering System
 * Partition-level ordering guarantee with KafkaJS
 */

const { Kafka, Partitioners } = require('kafkajs');

// ==========================================
// Ordered Transaction Producer
// ==========================================
class OrderedTransactionProducer {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'bkash-transaction-producer',
            brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        });

        this.producer = this.kafka.producer({
            idempotent: true,
            maxInFlightRequests: 1, // Strict ordering during retries
            createPartitioner: Partitioners.DefaultPartitioner,
        });

        this.sequenceCounters = new Map(); // account → sequence number
    }

    async connect() {
        await this.producer.connect();
    }

    /**
     * Transaction event publish - account number as partition key
     */
    async publishTransactionEvent(accountNumber, eventType, amount, balanceAfter) {
        // Sequence number maintain
        const currentSeq = this.sequenceCounters.get(accountNumber) || 0;
        const nextSeq = currentSeq + 1;
        this.sequenceCounters.set(accountNumber, nextSeq);

        const message = {
            key: accountNumber, // Partition key - same account → same partition
            value: JSON.stringify({
                eventId: `evt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
                accountNumber,
                eventType,
                amount,
                balanceAfter,
                sequenceNumber: nextSeq,
                timestamp: Date.now(),
            }),
            headers: {
                'event-type': eventType,
                'sequence-number': String(nextSeq),
            },
        };

        await this.producer.send({
            topic: 'bkash.account.transactions',
            messages: [message],
        });

        console.log(`📤 Published: ${eventType} ৳${amount} for ${accountNumber} [seq: ${nextSeq}]`);
    }

    /**
     * Batch publish - same account এর events ordered থাকবে
     */
    async publishBatch(events) {
        const messages = events.map(event => ({
            key: event.accountNumber,
            value: JSON.stringify(event),
            headers: {
                'event-type': event.eventType,
                'sequence-number': String(event.sequenceNumber),
            },
        }));

        await this.producer.send({
            topic: 'bkash.account.transactions',
            messages,
        });
    }
}

// ==========================================
// Ordered Consumer with Out-of-Order Detection
// ==========================================
class OrderedTransactionConsumer {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'bkash-transaction-consumer',
            brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        });

        this.consumer = this.kafka.consumer({
            groupId: 'bkash-balance-updater',
            sessionTimeout: 30000,
            // Cooperative sticky assignor - rebalancing impact কমায়
            partitionAssigners: [
                // CooperativeStickyAssigner
            ],
        });

        this.expectedSequence = new Map(); // account → next expected seq
        this.outOfOrderBuffer = new Map(); // account → Map(seq → message)
        this.MAX_BUFFER_SIZE = 1000;
        this.MAX_WAIT_MS = 30000; // 30 seconds max wait for missing message
    }

    async start() {
        await this.consumer.connect();
        await this.consumer.subscribe({
            topic: 'bkash.account.transactions',
            fromBeginning: false,
        });

        await this.consumer.run({
            // Per-partition processing - ordering maintain করার জন্য
            partitionsConsumedConcurrently: 1, // Process one partition at a time
            eachMessage: async ({ topic, partition, message }) => {
                await this.processMessage(partition, message);
            },
        });
    }

    async processMessage(partition, rawMessage) {
        const message = JSON.parse(rawMessage.value.toString());
        const { accountNumber, sequenceNumber } = message;

        // প্রথমবার হলে initialize
        if (!this.expectedSequence.has(accountNumber)) {
            this.expectedSequence.set(accountNumber, sequenceNumber);
        }

        const expected = this.expectedSequence.get(accountNumber);

        if (sequenceNumber === expected) {
            // ✅ Correct order - process
            await this.executeBusinessLogic(message);
            this.expectedSequence.set(accountNumber, expected + 1);

            // Buffer drain
            await this.drainBuffer(accountNumber);
        } else if (sequenceNumber > expected) {
            // ⚠️ Gap - buffer and wait
            this.bufferMessage(accountNumber, sequenceNumber, message);
            console.warn(
                `⚠️ Out-of-order for ${accountNumber}: expected ${expected}, got ${sequenceNumber}`
            );
        } else {
            // 🔄 Already processed (duplicate)
            console.log(`🔄 Duplicate seq ${sequenceNumber} for ${accountNumber}`);
        }
    }

    bufferMessage(accountNumber, sequenceNumber, message) {
        if (!this.outOfOrderBuffer.has(accountNumber)) {
            this.outOfOrderBuffer.set(accountNumber, new Map());
        }

        const buffer = this.outOfOrderBuffer.get(accountNumber);

        if (buffer.size >= this.MAX_BUFFER_SIZE) {
            throw new Error(
                `Buffer overflow for ${accountNumber}! ${buffer.size} messages buffered.`
            );
        }

        buffer.set(sequenceNumber, {
            message,
            bufferedAt: Date.now(),
        });
    }

    async drainBuffer(accountNumber) {
        const buffer = this.outOfOrderBuffer.get(accountNumber);
        if (!buffer) return;

        while (true) {
            const expected = this.expectedSequence.get(accountNumber);
            const entry = buffer.get(expected);

            if (!entry) break;

            buffer.delete(expected);
            await this.executeBusinessLogic(entry.message);
            this.expectedSequence.set(accountNumber, expected + 1);
        }

        if (buffer.size === 0) {
            this.outOfOrderBuffer.delete(accountNumber);
        }
    }

    async executeBusinessLogic(message) {
        const { accountNumber, eventType, amount, balanceAfter, sequenceNumber } = message;
        console.log(
            `✅ [seq:${sequenceNumber}] ${eventType} ৳${amount} → ${accountNumber} (Balance: ৳${balanceAfter})`
        );
        // Update database, trigger notifications, etc.
    }

    /**
     * Stale buffer cleanup - timeout হলে skip/alert
     */
    cleanupStaleBuffers() {
        const now = Date.now();

        for (const [account, buffer] of this.outOfOrderBuffer.entries()) {
            for (const [seq, entry] of buffer.entries()) {
                if (now - entry.bufferedAt > this.MAX_WAIT_MS) {
                    console.error(
                        `❌ Timeout! Message seq ${seq} for ${account} waited too long. Possible gap.`
                    );
                    // Alert ops team, or force-process with gap handling
                }
            }
        }
    }
}

// ==========================================
// Multi-Region Ordering with Vector Clocks
// ==========================================
class MultiRegionOrderingService {
    constructor(regionId) {
        this.regionId = regionId;
        this.vectorClocks = new Map(); // entity → {region: counter}
    }

    /**
     * Vector clock update - multi-region causality tracking
     */
    incrementClock(entityId) {
        if (!this.vectorClocks.has(entityId)) {
            this.vectorClocks.set(entityId, {});
        }

        const clock = this.vectorClocks.get(entityId);
        clock[this.regionId] = (clock[this.regionId] || 0) + 1;

        return { ...clock };
    }

    /**
     * Merge vector clocks from remote event
     */
    mergeClock(entityId, remoteClock) {
        if (!this.vectorClocks.has(entityId)) {
            this.vectorClocks.set(entityId, {});
        }

        const localClock = this.vectorClocks.get(entityId);

        for (const [region, counter] of Object.entries(remoteClock)) {
            localClock[region] = Math.max(localClock[region] || 0, counter);
        }

        return { ...localClock };
    }

    /**
     * Check if event A happened before event B (causality)
     */
    happenedBefore(clockA, clockB) {
        let atLeastOneLess = false;

        const allRegions = new Set([
            ...Object.keys(clockA),
            ...Object.keys(clockB),
        ]);

        for (const region of allRegions) {
            const a = clockA[region] || 0;
            const b = clockB[region] || 0;

            if (a > b) return false; // A is NOT before B
            if (a < b) atLeastOneLess = true;
        }

        return atLeastOneLess;
    }

    /**
     * Concurrent events detection
     */
    areConcurrent(clockA, clockB) {
        return !this.happenedBefore(clockA, clockB) &&
               !this.happenedBefore(clockB, clockA);
    }
}

// ==========================================
// Partition Key Strategies
// ==========================================
class PartitionKeyStrategy {
    /**
     * Simple entity-based key
     */
    static entityKey(accountNumber) {
        return accountNumber;
    }

    /**
     * Compound key for sub-entity ordering
     */
    static compoundKey(accountNumber, transactionType) {
        return `${accountNumber}:${transactionType}`;
    }

    /**
     * Consistent hashing - partition count change এও stable
     */
    static consistentHash(key, partitionCount) {
        const hash = this._murmur2(key);
        return Math.abs(hash) % partitionCount;
    }

    static _murmur2(key) {
        let h = 0;
        for (let i = 0; i < key.length; i++) {
            h = Math.imul(h ^ key.charCodeAt(i), 0x5bd1e995);
            h ^= h >>> 13;
        }
        return h;
    }

    /**
     * Hot partition detection and mitigation
     */
    static async detectHotPartitions(admin, topic) {
        const metadata = await admin.fetchTopicOffsets(topic);
        const offsets = metadata.map(p => ({
            partition: p.partition,
            offset: parseInt(p.offset),
        }));

        const avgOffset = offsets.reduce((s, p) => s + p.offset, 0) / offsets.length;

        const hotPartitions = offsets.filter(p => p.offset > avgOffset * 2);

        if (hotPartitions.length > 0) {
            console.warn('🔥 Hot partitions detected:', hotPartitions);
        }

        return hotPartitions;
    }
}

// ==========================================
// Usage Example - bKash Transaction Flow
// ==========================================
async function main() {
    const producer = new OrderedTransactionProducer();
    await producer.connect();

    // Same account - events will be in order
    const account = '01712345678';

    await producer.publishTransactionEvent(account, 'DEBIT', 3000, 2000);
    await producer.publishTransactionEvent(account, 'CREDIT', 1000, 3000);
    await producer.publishTransactionEvent(account, 'DEBIT', 2500, 500);

    // These will go to SAME partition → ordered ✅
    console.log('All events published in order for account:', account);

    // Start consumer
    const consumer = new OrderedTransactionConsumer();
    await consumer.start();
}

main().catch(console.error);
```

---

## ✅ কখন Ordering দরকার

| Use Case | Partition Key | কেন |
|----------|--------------|-----|
| bKash balance update | account_number | Balance calculation correct রাখতে |
| Order state machine | order_id | Invalid state transitions এড়াতে |
| Chat messages | conversation_id | মেসেজ সঠিক ক্রমে দেখাতে |
| Event sourcing | aggregate_id | Event replay correct হতে |
| CDC (Change Data Capture) | table_name + primary_key | Row-level consistency |

## ❌ কখন Ordering লাগে না

| Use Case | কেন লাগে না |
|----------|-------------|
| Analytics events | Aggregation এ order matter করে না |
| Log shipping | Logs out-of-order হলেও timestamp থেকে sort করা যায় |
| Email sending | কোন email আগে যাবে — important না |
| Image processing | Independent operations |
| Metrics collection | Counters are commutative |

## ⚖️ Ordering vs Parallelism Trade-offs:

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Decision Matrix:                                            │
│                                                               │
│  ┌────────────────────┬───────────┬──────────┬────────────┐ │
│  │ Strategy           │ Ordering  │ Throughput│ Complexity │ │
│  ├────────────────────┼───────────┼──────────┼────────────┤ │
│  │ 1 partition        │ Total     │ Low      │ Low        │ │
│  │ N partitions + key │ Per-key   │ High     │ Medium     │ │
│  │ Sequence numbers   │ Detected  │ High     │ High       │ │
│  │ Vector clocks      │ Causal    │ High     │ Very High  │ │
│  │ No ordering        │ None      │ Maximum  │ Low        │ │
│  └────────────────────┴───────────┴──────────┴────────────┘ │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔑 Key Takeaways

1. **Kafka ordering = partition-level only** — topic level ordering নেই
2. **Partition Key = Ordering Unit** — same key → same partition → ordered
3. **max.in.flight = 1** strict ordering চাইলে (retries এ reordering prevent)
4. **Sequence numbers** দিয়ে out-of-order detect করুন
5. **Rebalancing** ordering ভাঙতে পারে — cooperative sticky assignor ব্যবহার করুন
6. **Multi-region** ordering এর জন্য vector clocks বা centralized sequencer দরকার
7. **Trade-off বুঝুন:** ordering বাড়ালে parallelism/throughput কমে
