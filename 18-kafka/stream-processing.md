# 🌊 Stream Processing Patterns

## 📌 সংজ্ঞা ও ধারণা (Definition & Concept)

**Stream Processing** হলো continuous, unbounded data (ডেটার ধারা) কে real-time এ process করা। Traditional batch processing এ ডেটা জমা করে পরে process করা হয়, কিন্তু stream processing এ ডেটা আসার সাথে সাথেই process হয়।

### 🤔 Stream Processing কেন দরকার?

- **Real-time insights:** ৫ মিনিট আগের fraud detect করলে টাকা ইতিমধ্যেই চলে গেছে
- **Low latency:** User experience এর জন্য millisecond-level response
- **Continuous computation:** ডেটা ক্রমাগত আসছে, batch wait করার সুযোগ নেই

### 📊 Stream Processing Frameworks Comparison:

| Feature | Kafka Streams | Apache Flink | Spark Streaming |
|---------|--------------|--------------|-----------------|
| **Deployment** | Library (JVM app) | Cluster | Cluster |
| **Processing** | Event-at-a-time | Event-at-a-time | Micro-batch |
| **Latency** | Low (ms) | Very Low (ms) | Medium (seconds) |
| **State Management** | RocksDB | Managed State | Checkpoint |
| **Exactly-Once** | ✅ | ✅ | ✅ |
| **Use Case** | Simple-Medium | Complex | Batch + Stream |

### 🔄 Stateless vs Stateful Operations:

**Stateless Operations:**
- `map()` — একটি event কে transform করা
- `filter()` — condition match না করলে discard
- `flatMap()` — একটি event থেকে multiple events

**Stateful Operations:**
- `aggregate()` — events জমা করে result বের করা
- `join()` — দুটি stream/table merge করা
- `windowing()` — time-based grouping

### 🪟 Window Types:

1. **Tumbling Window:** Fixed-size, non-overlapping (প্রতি ৫ মিনিটে)
2. **Hopping Window:** Fixed-size, overlapping (৫ মিনিটের window, ১ মিনিট পর পর)
3. **Sliding Window:** Event-driven, overlaps (event আসলেই window start)
4. **Session Window:** Inactivity gap based (user inactive হলে window close)

---

## 🌍 বাস্তব জীবনের উদাহরণ (Real-world Example)

### 💰 bKash Real-time Fraud Detection

bKash-এ প্রতিদিন লক্ষ লক্ষ transaction হয়। Fraud detect করতে real-time stream analysis দরকার:

**Fraud Rules (Stream Processing দিয়ে implement):**
1. একই account থেকে ৫ মিনিটে ৫টির বেশি transaction → সন্দেহজনক
2. রাত ২-৫টার মধ্যে বড় amount transaction → flag
3. নতুন device থেকে বড় Send Money → verification প্রয়োজন
4. একই receiver-কে multiple accounts থেকে টাকা → money laundering suspect
5. হঠাৎ করে unusual amount pattern → anomaly detection

**Stream Processing Pipeline:**
```
Transaction Events → Filter(amount > threshold)
                   → Window(5 min tumbling)
                   → Aggregate(count per account)
                   → Filter(count > 5)
                   → Alert(fraud team)
```

### 📰 Prothom Alo Real-time Trending Articles

- প্রতিটি page view event stream এ আসে
- ১ ঘণ্টার sliding window এ article view count calculate
- Top 10 articles real-time trending section এ দেখায়
- Window slide করলে পুরানো articles drop off হয়

### 🛵 Pathao Real-time Surge Pricing

- Rider requests stream → Window(5 min) → Count per area
- Driver availability stream → Window(5 min) → Count per area
- Join(requests, availability) → Demand/Supply ratio → Surge multiplier
- Real-time pricing update every minute

---

## 📊 ASCII Diagram

### Stream Processing Architecture:

```
┌──────────────────────────────────────────────────────────────┐
│              Stream Processing Architecture                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐     ┌──────────────┐     ┌────────────────┐   │
│  │  Source  │     │    Kafka     │     │   Stream       │   │
│  │  Events  │────▶│   Topics     │────▶│   Processor    │   │
│  │          │     │              │     │                │   │
│  └──────────┘     └──────────────┘     └───────┬────────┘   │
│                                                 │            │
│  Sources:                              ┌────────┼────────┐   │
│  • bKash transactions                  │        │        │   │
│  • Pathao ride events                  ▼        ▼        ▼   │
│  • Prothom Alo page views     ┌──────────┐ ┌──────┐ ┌─────┐│
│  • Daraz order events         │  Output  │ │State │ │Alert││
│  • GP usage events            │  Topic   │ │Store │ │     ││
│                               └──────────┘ └──────┘ └─────┘│
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Window Types Visualization:

```
┌──────────────────────────────────────────────────────────────┐
│                     Window Types                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Tumbling Window (5 min, non-overlapping):                │
│                                                               │
│  Time: ──────────────────────────────────────────────▶       │
│        |  Window 1  |  Window 2  |  Window 3  |             │
│        |  [0-5min]  |  [5-10min] | [10-15min] |             │
│        | E1 E2 E3   | E4 E5      | E6 E7 E8   |             │
│                                                               │
│  2. Hopping Window (5 min size, 2 min hop):                  │
│                                                               │
│  Time: ──────────────────────────────────────────────▶       │
│        |  Window 1 [0-5]   |                                 │
│            |  Window 2 [2-7]   |                             │
│                |  Window 3 [4-9]   |                         │
│        Events appear in MULTIPLE windows!                     │
│                                                               │
│  3. Sliding Window (5 min, event-triggered):                 │
│                                                               │
│  Time: ──────────────────────────────────────────────▶       │
│        E1──|  5min window from E1  |                         │
│          E2──|  5min window from E2  |                       │
│              E3──|  5min window from E3  |                   │
│        Each event starts its own window                       │
│                                                               │
│  4. Session Window (gap = 3 min inactivity):                 │
│                                                               │
│  Time: ──────────────────────────────────────────────▶       │
│        E1 E2 E3    [3min gap]    E4 E5    [3min gap]  E6    │
│        |─Session 1─|            |─Sess 2─|           |S3|   │
│        Session closes after inactivity gap                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Stream-Table Duality:

```
┌──────────────────────────────────────────────────────────────┐
│               Stream-Table Duality                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Stream (Event Log):          Table (Current State):         │
│  ┌────────────────────┐       ┌──────────────────────┐      │
│  │ time │ key │ value │       │  key  │    value     │      │
│  ├────────────────────┤       ├──────────────────────┤      │
│  │ T1   │ A   │ +100  │       │  A    │  150         │      │
│  │ T2   │ B   │ +200  │       │  B    │  200         │      │
│  │ T3   │ A   │ +50   │       │  C    │  300         │      │
│  │ T4   │ C   │ +300  │       └──────────────────────┘      │
│  │ T5   │ B   │ -50   │       ▲                             │
│  └────────────────────┘       │ Aggregate/Compact            │
│            │                   │                             │
│            └───────────────────┘                             │
│                                                               │
│  Stream → Table:  Aggregate events into current state        │
│  Table → Stream:  Watch changes (CDC) as event stream        │
│                                                               │
│  Kafka Concepts:                                             │
│  • KStream = Event Stream (all events)                       │
│  • KTable = Changelog Stream (latest per key)                │
│  • GlobalKTable = Full table replicated to all instances     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Join Types in Stream Processing:

```
┌──────────────────────────────────────────────────────────────┐
│                   Join Types                                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Stream-Stream Join (windowed):                           │
│     ┌────────────┐          ┌────────────┐                   │
│     │ Transaction│          │  Fraud     │                   │
│     │  Stream    │──JOIN───▶│  Score     │                   │
│     └────────────┘   within │  Stream    │                   │
│     ┌────────────┐   5 min  └────────────┘                   │
│     │ Device     │──────┘                                    │
│     │  Stream    │         Output: enriched events            │
│     └────────────┘                                           │
│                                                               │
│  2. Stream-Table Join (lookup enrichment):                   │
│     ┌────────────┐          ┌────────────┐                   │
│     │ Transaction│          │  Enriched  │                   │
│     │  Stream    │──JOIN───▶│  Transac-  │                   │
│     └────────────┘          │  tion      │                   │
│     ┌────────────┐          └────────────┘                   │
│     │ Customer   │──────┘                                    │
│     │  Table     │         Stream + Table info               │
│     └────────────┘                                           │
│                                                               │
│  3. Table-Table Join:                                        │
│     ┌────────────┐          ┌────────────┐                   │
│     │ Customer   │          │  Combined  │                   │
│     │  Table     │──JOIN───▶│  View      │                   │
│     └────────────┘          │  (Table)   │                   │
│     ┌────────────┐          └────────────┘                   │
│     │ Account    │──────┘                                    │
│     │  Table     │         Updated on either change          │
│     └────────────┘                                           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### State Store Architecture:

```
┌──────────────────────────────────────────────────────────────┐
│              State Store & Changelog                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Stream Processor Instance               │     │
│  │                                                      │     │
│  │  ┌────────────┐     ┌──────────────────────┐       │     │
│  │  │  Input     │     │    Processing        │       │     │
│  │  │  Topic     │────▶│    Logic             │       │     │
│  │  │            │     │                      │       │     │
│  │  └────────────┘     └──────────┬───────────┘       │     │
│  │                                │                    │     │
│  │                         Read/Write                  │     │
│  │                                │                    │     │
│  │                     ┌──────────▼───────────┐       │     │
│  │                     │   Local State Store  │       │     │
│  │                     │   (RocksDB)          │       │     │
│  │                     │                      │       │     │
│  │                     │  key → value (fast)  │       │     │
│  │                     └──────────┬───────────┘       │     │
│  │                                │                    │     │
│  └────────────────────────────────┼────────────────────┘     │
│                                   │ Changelog                 │
│                                   ▼                           │
│                     ┌──────────────────────────┐             │
│                     │   Changelog Topic        │             │
│                     │   (Kafka - for recovery) │             │
│                     │                          │             │
│                     │   Backup of state store  │             │
│                     │   Enables fault tolerance│             │
│                     └──────────────────────────┘             │
│                                                               │
│  Recovery: If instance crashes, new instance                 │
│  rebuilds state from changelog topic                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * bKash Real-time Fraud Detection Stream Processor
 * PHP implementation (conceptual - Kafka Streams is JVM-based)
 * Production এ PHP consumer + Redis state store ব্যবহার করা হয়
 */

/**
 * Stream Processing Operations - Stateless
 */
class StatelessOperations
{
    /**
     * Map: Transform transaction event
     */
    public static function enrichTransaction(array $event): array
    {
        return [
            ...$event,
            'amount_category' => self::categorizeAmount($event['amount']),
            'time_category' => self::categorizeTime($event['timestamp']),
            'risk_score_base' => self::calculateBaseRisk($event),
        ];
    }

    /**
     * Filter: Only high-value transactions
     */
    public static function isHighValue(array $event): bool
    {
        return $event['amount'] >= 10000; // ৳10,000+
    }

    /**
     * Filter: Suspicious time (late night)
     */
    public static function isSuspiciousTime(array $event): bool
    {
        $hour = (int) date('H', $event['timestamp']);
        return $hour >= 2 && $hour <= 5; // রাত ২টা - ৫টা
    }

    /**
     * FlatMap: One transaction → multiple alert events
     */
    public static function generateAlerts(array $event): array
    {
        $alerts = [];

        if ($event['amount'] > 50000) {
            $alerts[] = [
                'type' => 'HIGH_AMOUNT',
                'severity' => 'HIGH',
                'event' => $event,
            ];
        }

        if (self::isSuspiciousTime($event)) {
            $alerts[] = [
                'type' => 'SUSPICIOUS_TIME',
                'severity' => 'MEDIUM',
                'event' => $event,
            ];
        }

        if ($event['is_new_device'] ?? false) {
            $alerts[] = [
                'type' => 'NEW_DEVICE',
                'severity' => 'MEDIUM',
                'event' => $event,
            ];
        }

        return $alerts;
    }

    private static function categorizeAmount(float $amount): string
    {
        if ($amount >= 100000) return 'VERY_HIGH';
        if ($amount >= 50000) return 'HIGH';
        if ($amount >= 10000) return 'MEDIUM';
        return 'LOW';
    }

    private static function categorizeTime(int $timestamp): string
    {
        $hour = (int) date('H', $timestamp);
        if ($hour >= 2 && $hour <= 5) return 'LATE_NIGHT';
        if ($hour >= 6 && $hour <= 9) return 'MORNING';
        if ($hour >= 22 || $hour <= 1) return 'NIGHT';
        return 'NORMAL';
    }

    private static function calculateBaseRisk(array $event): float
    {
        $risk = 0.0;
        if ($event['amount'] > 50000) $risk += 0.3;
        if (self::isSuspiciousTime($event)) $risk += 0.2;
        if ($event['is_new_device'] ?? false) $risk += 0.25;
        return min($risk, 1.0);
    }
}

/**
 * Tumbling Window Aggregation
 * ৫ মিনিটের window এ transaction count aggregate করা
 */
class TumblingWindowAggregator
{
    private \Redis $redis;
    private int $windowSizeSeconds;
    private string $prefix;

    public function __construct(\Redis $redis, int $windowSizeSeconds = 300)
    {
        $this->redis = $redis;
        $this->windowSizeSeconds = $windowSizeSeconds;
        $this->prefix = 'window:tumbling:';
    }

    /**
     * Window key calculate - same window এ same key পাবে
     */
    private function getWindowKey(string $accountId, int $timestamp): string
    {
        $windowStart = floor($timestamp / $this->windowSizeSeconds) * $this->windowSizeSeconds;
        return "{$this->prefix}{$accountId}:{$windowStart}";
    }

    /**
     * Transaction count aggregate করা
     */
    public function aggregate(array $event): array
    {
        $accountId = $event['account_id'];
        $timestamp = $event['timestamp'];
        $windowKey = $this->getWindowKey($accountId, $timestamp);

        // Atomic increment
        $count = $this->redis->incr("{$windowKey}:count");
        $this->redis->incrByFloat("{$windowKey}:total_amount", $event['amount']);
        $this->redis->expire("{$windowKey}:count", $this->windowSizeSeconds * 2);
        $this->redis->expire("{$windowKey}:total_amount", $this->windowSizeSeconds * 2);

        $totalAmount = (float) $this->redis->get("{$windowKey}:total_amount");

        return [
            'account_id' => $accountId,
            'window_key' => $windowKey,
            'transaction_count' => $count,
            'total_amount' => $totalAmount,
            'window_size_seconds' => $this->windowSizeSeconds,
        ];
    }
}

/**
 * Session Window - User activity session tracking
 */
class SessionWindowAggregator
{
    private \Redis $redis;
    private int $gapSeconds; // Inactivity gap

    public function __construct(\Redis $redis, int $gapSeconds = 180)
    {
        $this->redis = $redis;
        $this->gapSeconds = $gapSeconds;
    }

    /**
     * Session update/create
     */
    public function processEvent(array $event): array
    {
        $userId = $event['user_id'];
        $sessionKey = "session:{$userId}:current";

        $session = $this->redis->hGetAll($sessionKey);

        if (empty($session)) {
            // নতুন session start
            $sessionData = [
                'session_id' => uniqid('sess_'),
                'user_id' => $userId,
                'start_time' => $event['timestamp'],
                'last_activity' => $event['timestamp'],
                'event_count' => 1,
                'total_amount' => $event['amount'] ?? 0,
            ];
            $this->redis->hMSet($sessionKey, $sessionData);
            $this->redis->expire($sessionKey, $this->gapSeconds);
            return ['action' => 'SESSION_STARTED', 'session' => $sessionData];
        }

        $lastActivity = (int) $session['last_activity'];
        $timeSinceLast = $event['timestamp'] - $lastActivity;

        if ($timeSinceLast > $this->gapSeconds) {
            // Gap exceeded - close old session, start new
            $closedSession = $session;
            $closedSession['action'] = 'SESSION_CLOSED';

            // Start new session
            $sessionData = [
                'session_id' => uniqid('sess_'),
                'user_id' => $userId,
                'start_time' => $event['timestamp'],
                'last_activity' => $event['timestamp'],
                'event_count' => 1,
                'total_amount' => $event['amount'] ?? 0,
            ];
            $this->redis->hMSet($sessionKey, $sessionData);
            $this->redis->expire($sessionKey, $this->gapSeconds);

            return ['action' => 'SESSION_ROTATED', 'closed' => $closedSession, 'new' => $sessionData];
        }

        // Same session - update
        $this->redis->hIncrBy($sessionKey, 'event_count', 1);
        $this->redis->hIncrByFloat($sessionKey, 'total_amount', $event['amount'] ?? 0);
        $this->redis->hSet($sessionKey, 'last_activity', $event['timestamp']);
        $this->redis->expire($sessionKey, $this->gapSeconds);

        return ['action' => 'SESSION_UPDATED', 'event_count' => (int) $session['event_count'] + 1];
    }
}

/**
 * Stream-Table Join: Transaction + Customer Info
 */
class StreamTableJoin
{
    private \Redis $redis; // Customer table cached in Redis
    private \PDO $db;

    public function __construct(\Redis $redis, \PDO $db)
    {
        $this->redis = $redis;
        $this->db = $db;
    }

    /**
     * Customer table cache load (KTable simulation)
     */
    public function loadCustomerTable(): void
    {
        $stmt = $this->db->query("SELECT * FROM customers");
        while ($row = $stmt->fetch(\PDO::FETCH_ASSOC)) {
            $this->redis->hMSet("customer:{$row['account_id']}", $row);
        }
    }

    /**
     * Stream-Table Join: Enrich transaction with customer data
     */
    public function joinTransactionWithCustomer(array $transaction): ?array
    {
        $accountId = $transaction['account_id'];
        $customer = $this->redis->hGetAll("customer:{$accountId}");

        if (empty($customer)) {
            // Customer not found - might be new
            return null;
        }

        return [
            ...$transaction,
            'customer_name' => $customer['name'],
            'customer_tier' => $customer['tier'],
            'account_age_days' => (time() - strtotime($customer['created_at'])) / 86400,
            'kyc_verified' => (bool) $customer['kyc_verified'],
        ];
    }
}

/**
 * bKash Fraud Detection Stream Processor (Complete Pipeline)
 */
class BkashFraudDetectionPipeline
{
    private TumblingWindowAggregator $windowAggregator;
    private StreamTableJoin $streamTableJoin;
    private \Redis $redis;
    private array $fraudRules;

    public function __construct(
        \Redis $redis,
        TumblingWindowAggregator $windowAggregator,
        StreamTableJoin $streamTableJoin
    ) {
        $this->redis = $redis;
        $this->windowAggregator = $windowAggregator;
        $this->streamTableJoin = $streamTableJoin;
        $this->initFraudRules();
    }

    private function initFraudRules(): void
    {
        $this->fraudRules = [
            'high_frequency' => fn($agg) => $agg['transaction_count'] > 5,
            'high_amount_window' => fn($agg) => $agg['total_amount'] > 100000,
            'velocity_spike' => fn($agg) => $agg['transaction_count'] > 3 && $agg['total_amount'] > 50000,
        ];
    }

    /**
     * Main processing pipeline
     */
    public function process(array $rawEvent): ?array
    {
        // Step 1: Stateless - Enrich
        $enriched = StatelessOperations::enrichTransaction($rawEvent);

        // Step 2: Stateless - Filter low-risk
        if ($enriched['risk_score_base'] < 0.1 && !StatelessOperations::isHighValue($enriched)) {
            return null; // Low risk - no further processing needed
        }

        // Step 3: Stateful - Window aggregation
        $windowResult = $this->windowAggregator->aggregate($enriched);

        // Step 4: Stream-Table Join - Customer enrichment
        $withCustomer = $this->streamTableJoin->joinTransactionWithCustomer($enriched);

        // Step 5: Fraud rule evaluation
        $fraudAlerts = [];
        foreach ($this->fraudRules as $ruleName => $ruleCheck) {
            if ($ruleCheck($windowResult)) {
                $fraudAlerts[] = [
                    'rule' => $ruleName,
                    'account_id' => $enriched['account_id'],
                    'window_data' => $windowResult,
                    'customer' => $withCustomer,
                    'timestamp' => time(),
                ];
            }
        }

        if (!empty($fraudAlerts)) {
            return [
                'type' => 'FRAUD_ALERT',
                'alerts' => $fraudAlerts,
                'risk_score' => min(1.0, $enriched['risk_score_base'] + count($fraudAlerts) * 0.2),
                'event' => $enriched,
            ];
        }

        return null;
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * bKash Real-time Fraud Detection Stream Processor
 * Node.js implementation with KafkaJS
 */

const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

// ==========================================
// Stateless Stream Operations
// ==========================================
class StatelessOperations {
    /**
     * Map: Transform/Enrich transaction
     */
    static enrichTransaction(event) {
        return {
            ...event,
            amountCategory: this.categorizeAmount(event.amount),
            timeCategory: this.categorizeTime(event.timestamp),
            baseRiskScore: this.calculateBaseRisk(event),
            processedAt: Date.now(),
        };
    }

    /**
     * Filter: High-value transactions only
     */
    static isHighValue(event) {
        return event.amount >= 10000;
    }

    /**
     * Filter: Suspicious time window
     */
    static isSuspiciousTime(event) {
        const hour = new Date(event.timestamp).getHours();
        return hour >= 2 && hour <= 5;
    }

    /**
     * FlatMap: Generate multiple alerts from one event
     */
    static generateAlerts(event) {
        const alerts = [];

        if (event.amount > 50000) {
            alerts.push({ type: 'HIGH_AMOUNT', severity: 'HIGH', event });
        }
        if (this.isSuspiciousTime(event)) {
            alerts.push({ type: 'SUSPICIOUS_TIME', severity: 'MEDIUM', event });
        }
        if (event.isNewDevice) {
            alerts.push({ type: 'NEW_DEVICE', severity: 'MEDIUM', event });
        }
        if (event.isInternational) {
            alerts.push({ type: 'INTERNATIONAL', severity: 'HIGH', event });
        }

        return alerts;
    }

    static categorizeAmount(amount) {
        if (amount >= 100000) return 'VERY_HIGH';
        if (amount >= 50000) return 'HIGH';
        if (amount >= 10000) return 'MEDIUM';
        return 'LOW';
    }

    static categorizeTime(timestamp) {
        const hour = new Date(timestamp).getHours();
        if (hour >= 2 && hour <= 5) return 'LATE_NIGHT';
        if (hour >= 6 && hour <= 9) return 'MORNING';
        if (hour >= 22 || hour <= 1) return 'NIGHT';
        return 'NORMAL';
    }

    static calculateBaseRisk(event) {
        let risk = 0.0;
        if (event.amount > 50000) risk += 0.3;
        if (this.isSuspiciousTime(event)) risk += 0.2;
        if (event.isNewDevice) risk += 0.25;
        if (event.isInternational) risk += 0.15;
        return Math.min(risk, 1.0);
    }
}

// ==========================================
// Tumbling Window Aggregator
// ==========================================
class TumblingWindowAggregator {
    constructor(redis, windowSizeMs = 300000) { // 5 minutes default
        this.redis = redis;
        this.windowSizeMs = windowSizeMs;
    }

    getWindowKey(accountId, timestamp) {
        const windowStart = Math.floor(timestamp / this.windowSizeMs) * this.windowSizeMs;
        return `window:tumbling:${accountId}:${windowStart}`;
    }

    async aggregate(event) {
        const { account_id, amount, timestamp } = event;
        const windowKey = this.getWindowKey(account_id, timestamp);
        const ttl = Math.ceil(this.windowSizeMs / 1000) * 2;

        // Atomic multi-operation
        const pipeline = this.redis.pipeline();
        pipeline.incr(`${windowKey}:count`);
        pipeline.incrbyfloat(`${windowKey}:total_amount`, amount);
        pipeline.lpush(`${windowKey}:amounts`, amount);
        pipeline.expire(`${windowKey}:count`, ttl);
        pipeline.expire(`${windowKey}:total_amount`, ttl);
        pipeline.expire(`${windowKey}:amounts`, ttl);

        const results = await pipeline.exec();

        const count = results[0][1];
        const totalAmount = parseFloat(results[1][1]);

        return {
            accountId: account_id,
            windowKey,
            transactionCount: count,
            totalAmount,
            avgAmount: totalAmount / count,
            windowSizeMs: this.windowSizeMs,
        };
    }
}

// ==========================================
// Hopping Window Aggregator
// ==========================================
class HoppingWindowAggregator {
    constructor(redis, windowSizeMs = 300000, hopSizeMs = 60000) {
        this.redis = redis;
        this.windowSizeMs = windowSizeMs; // 5 min window
        this.hopSizeMs = hopSizeMs;       // 1 min hop
    }

    /**
     * Event কোন কোন window এ পড়ে calculate
     */
    getAffectedWindows(timestamp) {
        const windows = [];
        const numWindows = Math.ceil(this.windowSizeMs / this.hopSizeMs);

        for (let i = 0; i < numWindows; i++) {
            const windowStart = Math.floor(timestamp / this.hopSizeMs) * this.hopSizeMs - (i * this.hopSizeMs);
            const windowEnd = windowStart + this.windowSizeMs;

            if (timestamp >= windowStart && timestamp < windowEnd) {
                windows.push({ start: windowStart, end: windowEnd });
            }
        }

        return windows;
    }

    async aggregate(event) {
        const { account_id, amount, timestamp } = event;
        const windows = this.getAffectedWindows(timestamp);
        const results = [];

        for (const window of windows) {
            const windowKey = `window:hopping:${account_id}:${window.start}`;
            const ttl = Math.ceil(this.windowSizeMs / 1000) * 2;

            const count = await this.redis.incr(`${windowKey}:count`);
            await this.redis.incrbyfloat(`${windowKey}:total`, amount);
            await this.redis.expire(`${windowKey}:count`, ttl);
            await this.redis.expire(`${windowKey}:total`, ttl);

            const total = parseFloat(await this.redis.get(`${windowKey}:total`));

            results.push({
                windowStart: window.start,
                windowEnd: window.end,
                count,
                totalAmount: total,
            });
        }

        return results;
    }
}

// ==========================================
// Session Window Aggregator
// ==========================================
class SessionWindowAggregator {
    constructor(redis, gapMs = 180000) { // 3 min inactivity gap
        this.redis = redis;
        this.gapMs = gapMs;
    }

    async processEvent(event) {
        const { user_id, timestamp, amount } = event;
        const sessionKey = `session:${user_id}:current`;

        const session = await this.redis.hgetall(sessionKey);

        if (!session || Object.keys(session).length === 0) {
            // New session
            const newSession = {
                sessionId: `sess_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
                userId: user_id,
                startTime: timestamp,
                lastActivity: timestamp,
                eventCount: 1,
                totalAmount: amount || 0,
                events: JSON.stringify([event]),
            };

            await this.redis.hmset(sessionKey, newSession);
            await this.redis.pexpire(sessionKey, this.gapMs);

            return { action: 'SESSION_STARTED', session: newSession };
        }

        const lastActivity = parseInt(session.lastActivity);
        const timeSinceLast = timestamp - lastActivity;

        if (timeSinceLast > this.gapMs) {
            // Session expired — close and start new
            const closedSession = { ...session, action: 'CLOSED' };

            const newSession = {
                sessionId: `sess_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
                userId: user_id,
                startTime: timestamp,
                lastActivity: timestamp,
                eventCount: 1,
                totalAmount: amount || 0,
                events: JSON.stringify([event]),
            };

            await this.redis.hmset(sessionKey, newSession);
            await this.redis.pexpire(sessionKey, this.gapMs);

            return { action: 'SESSION_ROTATED', closed: closedSession, new: newSession };
        }

        // Update existing session
        const pipeline = this.redis.pipeline();
        pipeline.hincrby(sessionKey, 'eventCount', 1);
        pipeline.hincrbyfloat(sessionKey, 'totalAmount', amount || 0);
        pipeline.hset(sessionKey, 'lastActivity', timestamp);
        pipeline.pexpire(sessionKey, this.gapMs);
        await pipeline.exec();

        return {
            action: 'SESSION_UPDATED',
            eventCount: parseInt(session.eventCount) + 1,
        };
    }
}

// ==========================================
// Stream-Stream Join (Windowed)
// ==========================================
class StreamStreamJoin {
    constructor(redis, joinWindowMs = 300000) {
        this.redis = redis;
        this.joinWindowMs = joinWindowMs;
    }

    /**
     * Transaction stream + Device login stream join
     * Same account, within 5 min window
     */
    async joinTransactionWithDevice(transactionEvent) {
        const { account_id, timestamp } = transactionEvent;
        const windowStart = timestamp - this.joinWindowMs;

        // Look for device events in window
        const deviceKey = `device_events:${account_id}`;
        const deviceEvents = await this.redis.zrangebyscore(
            deviceKey,
            windowStart,
            timestamp
        );

        if (deviceEvents.length === 0) {
            return { ...transactionEvent, deviceInfo: null, joinResult: 'NO_MATCH' };
        }

        const latestDevice = JSON.parse(deviceEvents[deviceEvents.length - 1]);

        return {
            ...transactionEvent,
            deviceInfo: latestDevice,
            joinResult: 'MATCHED',
            isNewDevice: latestDevice.isNew,
            deviceTrust: latestDevice.trustScore,
        };
    }

    /**
     * Store device event for later joins
     */
    async storeDeviceEvent(deviceEvent) {
        const { account_id, timestamp } = deviceEvent;
        const deviceKey = `device_events:${account_id}`;

        await this.redis.zadd(deviceKey, timestamp, JSON.stringify(deviceEvent));
        // Cleanup old events
        await this.redis.zremrangebyscore(deviceKey, 0, timestamp - this.joinWindowMs * 2);
        await this.redis.pexpire(deviceKey, this.joinWindowMs * 3);
    }
}

// ==========================================
// Stream-Table Join
// ==========================================
class StreamTableJoin {
    constructor(redis) {
        this.redis = redis;
    }

    /**
     * KTable simulation - customer data cached in Redis
     * CDC (Change Data Capture) events update this table
     */
    async updateCustomerTable(customerEvent) {
        const { account_id, ...customerData } = customerEvent;
        await this.redis.hmset(`ktable:customer:${account_id}`, customerData);
    }

    /**
     * Join: Transaction stream × Customer table
     */
    async enrichTransaction(transaction) {
        const customer = await this.redis.hgetall(`ktable:customer:${transaction.account_id}`);

        if (!customer || Object.keys(customer).length === 0) {
            return { ...transaction, customer: null };
        }

        return {
            ...transaction,
            customer: {
                name: customer.name,
                tier: customer.tier,
                kycVerified: customer.kycVerified === 'true',
                accountAgeDays: Math.floor(
                    (Date.now() - parseInt(customer.createdAt)) / 86400000
                ),
                riskCategory: customer.riskCategory,
            },
        };
    }
}

// ==========================================
// Complete Fraud Detection Pipeline
// ==========================================
class BkashFraudDetectionPipeline {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'bkash-fraud-detector',
            brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        });

        this.redis = new Redis({ host: 'redis.bkash.internal', port: 6379 });

        this.tumblingWindow = new TumblingWindowAggregator(this.redis, 300000); // 5 min
        this.sessionWindow = new SessionWindowAggregator(this.redis, 180000); // 3 min gap
        this.streamStreamJoin = new StreamStreamJoin(this.redis, 300000);
        this.streamTableJoin = new StreamTableJoin(this.redis);

        this.consumer = this.kafka.consumer({ groupId: 'fraud-detection-group' });
        this.producer = this.kafka.producer({ idempotent: true });

        this.fraudRules = this.initFraudRules();
    }

    initFraudRules() {
        return [
            {
                name: 'high_frequency',
                check: (ctx) => ctx.windowResult.transactionCount > 5,
                severity: 'HIGH',
            },
            {
                name: 'high_amount_window',
                check: (ctx) => ctx.windowResult.totalAmount > 100000,
                severity: 'HIGH',
            },
            {
                name: 'new_device_high_amount',
                check: (ctx) => ctx.enriched.isNewDevice && ctx.event.amount > 20000,
                severity: 'CRITICAL',
            },
            {
                name: 'unverified_large_transaction',
                check: (ctx) => !ctx.customerData?.customer?.kycVerified && ctx.event.amount > 50000,
                severity: 'CRITICAL',
            },
            {
                name: 'late_night_activity',
                check: (ctx) => ctx.enriched.timeCategory === 'LATE_NIGHT' && ctx.event.amount > 10000,
                severity: 'MEDIUM',
            },
        ];
    }

    async start() {
        await this.consumer.connect();
        await this.producer.connect();

        await this.consumer.subscribe({ topic: 'bkash.transactions.raw' });
        await this.consumer.subscribe({ topic: 'bkash.device.logins' });

        await this.consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                const event = JSON.parse(message.value.toString());

                switch (topic) {
                    case 'bkash.transactions.raw':
                        await this.processTransaction(event);
                        break;
                    case 'bkash.device.logins':
                        await this.streamStreamJoin.storeDeviceEvent(event);
                        break;
                }
            },
        });

        console.log('🚀 Fraud Detection Pipeline started!');
    }

    async processTransaction(event) {
        // Step 1: Stateless enrichment (map)
        const enriched = StatelessOperations.enrichTransaction(event);

        // Step 2: Early filter - skip very low risk
        if (enriched.baseRiskScore < 0.05 && !StatelessOperations.isHighValue(event)) {
            return; // Not worth further processing
        }

        // Step 3: Stateful - Tumbling window aggregation
        const windowResult = await this.tumblingWindow.aggregate(event);

        // Step 4: Stream-Stream join (transaction + device)
        const withDevice = await this.streamStreamJoin.joinTransactionWithDevice(enriched);

        // Step 5: Stream-Table join (transaction + customer)
        const customerData = await this.streamTableJoin.enrichTransaction(withDevice);

        // Step 6: Evaluate fraud rules
        const context = { event, enriched: withDevice, windowResult, customerData };
        const triggeredRules = this.fraudRules.filter(rule => {
            try {
                return rule.check(context);
            } catch {
                return false;
            }
        });

        // Step 7: Emit alerts if rules triggered
        if (triggeredRules.length > 0) {
            const alert = {
                alertId: `alert_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
                accountId: event.account_id,
                transactionId: event.transaction_id,
                amount: event.amount,
                triggeredRules: triggeredRules.map(r => ({ name: r.name, severity: r.severity })),
                riskScore: Math.min(1.0, enriched.baseRiskScore + triggeredRules.length * 0.2),
                context: {
                    windowTransactionCount: windowResult.transactionCount,
                    windowTotalAmount: windowResult.totalAmount,
                    isNewDevice: withDevice.isNewDevice,
                    customerTier: customerData.customer?.tier,
                },
                timestamp: Date.now(),
            };

            // Publish to alerts topic
            await this.producer.send({
                topic: 'bkash.fraud.alerts',
                messages: [{
                    key: event.account_id,
                    value: JSON.stringify(alert),
                    headers: {
                        'max-severity': triggeredRules.reduce(
                            (max, r) => this.severityRank(r.severity) > this.severityRank(max)
                                ? r.severity : max,
                            'LOW'
                        ),
                    },
                }],
            });

            console.log(
                `🚨 FRAUD ALERT for ${event.account_id}: ` +
                `${triggeredRules.map(r => r.name).join(', ')} ` +
                `(Risk: ${(alert.riskScore * 100).toFixed(0)}%)`
            );
        }
    }

    severityRank(severity) {
        const ranks = { LOW: 1, MEDIUM: 2, HIGH: 3, CRITICAL: 4 };
        return ranks[severity] || 0;
    }
}

// ==========================================
// Interactive Queries (State Store Query)
// ==========================================
class InteractiveQueryService {
    constructor(redis) {
        this.redis = redis;
    }

    /**
     * Query current window state for an account
     * Real-time dashboard এর জন্য
     */
    async getAccountWindowState(accountId) {
        const now = Date.now();
        const windowSize = 300000; // 5 min
        const windowStart = Math.floor(now / windowSize) * windowSize;
        const windowKey = `window:tumbling:${accountId}:${windowStart}`;

        const count = parseInt(await this.redis.get(`${windowKey}:count`) || '0');
        const totalAmount = parseFloat(await this.redis.get(`${windowKey}:total_amount`) || '0');

        return {
            accountId,
            currentWindow: {
                start: new Date(windowStart).toISOString(),
                end: new Date(windowStart + windowSize).toISOString(),
                transactionCount: count,
                totalAmount,
                avgAmount: count > 0 ? totalAmount / count : 0,
            },
        };
    }

    /**
     * Get active sessions
     */
    async getActiveSession(userId) {
        const sessionKey = `session:${userId}:current`;
        const session = await this.redis.hgetall(sessionKey);

        if (!session || Object.keys(session).length === 0) {
            return null;
        }

        return {
            ...session,
            durationMs: Date.now() - parseInt(session.startTime),
            eventCount: parseInt(session.eventCount),
            totalAmount: parseFloat(session.totalAmount),
        };
    }
}

// ==========================================
// Usage
// ==========================================
async function main() {
    const pipeline = new BkashFraudDetectionPipeline();
    await pipeline.start();

    // Interactive query API (for dashboards)
    const redis = new Redis();
    const queryService = new InteractiveQueryService(redis);

    // Express API for querying state
    const express = require('express');
    const app = express();

    app.get('/api/account/:accountId/risk', async (req, res) => {
        const state = await queryService.getAccountWindowState(req.params.accountId);
        res.json(state);
    });

    app.get('/api/user/:userId/session', async (req, res) => {
        const session = await queryService.getActiveSession(req.params.userId);
        res.json(session || { active: false });
    });

    app.listen(3000, () => {
        console.log('📊 Interactive Query API running on port 3000');
    });
}

main().catch(console.error);
```

---

## ✅ কখন Stream Processing ব্যবহার করবেন

| পরিস্থিতি | Framework | কেন |
|-----------|-----------|-----|
| Real-time fraud detection (bKash) | Kafka Streams / Flink | Low latency, stateful processing |
| Trending articles (Prothom Alo) | Kafka Streams | Windowed aggregation |
| Surge pricing (Pathao) | Flink | Complex event processing |
| Real-time analytics dashboard | Kafka Streams | Interactive queries |
| CDC processing | Kafka Streams | Stream-table join |
| Complex ETL with joins | Apache Flink | Advanced windowing |
| ML feature computation | Spark Streaming | Batch + stream unified |

## ❌ কখন Stream Processing লাগে না

| পরিস্থিতি | কেন লাগে না | বিকল্প |
|-----------|-------------|--------|
| Simple message routing | No computation needed | Kafka Connect |
| Hourly reports | Batch is fine | Spark Batch |
| One-time data migration | Not continuous | Batch script |
| Static data lookup | No stream involved | Database query |
| Low-volume processing | Over-engineering | Simple consumer |

## 🏗️ Framework Selection Guide:

```
┌──────────────────────────────────────────────────────────────┐
│            Framework Selection Decision Tree                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  প্রশ্ন: Deployment complexity কেমন acceptable?              │
│       │                                                       │
│       ├── Low (library, no cluster)                          │
│       │   → Kafka Streams ✅                                 │
│       │                                                       │
│       └── High (separate cluster okay)                       │
│           │                                                   │
│           ├── Need advanced windowing/CEP?                    │
│           │   → Apache Flink ✅                              │
│           │                                                   │
│           └── Need batch + stream unified?                   │
│               → Spark Structured Streaming ✅                 │
│                                                               │
│  প্রশ্ন: Latency requirement?                               │
│       │                                                       │
│       ├── Sub-millisecond → Flink                            │
│       ├── Milliseconds → Kafka Streams                       │
│       └── Seconds okay → Spark Streaming                     │
│                                                               │
│  প্রশ্ন: State size?                                        │
│       │                                                       │
│       ├── Small-Medium → Kafka Streams (RocksDB)             │
│       └── Very Large → Flink (managed state)                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔑 Key Takeaways

1. **Stream Processing = Real-time computation** — ডেটা আসার সাথে সাথে process
2. **Stateless simple, Stateful powerful** — aggregation/join এ state দরকার
3. **Window = Time-bounded aggregation** — tumbling সবচেয়ে common
4. **Stream-Table duality** — stream compact করলে table, table change দেখলে stream
5. **State stores + changelog** — fault tolerance guarantee
6. **Kafka Streams** — সবচেয়ে সহজ (just a library), বেশিরভাগ use case এর জন্য যথেষ্ট
7. **Exactly-once in streams** — transactional producer + consumer + state store
8. **Interactive queries** — state store সরাসরি query করে real-time dashboard বানানো যায়
