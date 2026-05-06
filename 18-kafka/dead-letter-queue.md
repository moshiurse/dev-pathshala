# 💀 Dead Letter Queue (DLQ) Handling

## 📌 সংজ্ঞা ও ধারণা (Definition & Concept)

**Dead Letter Queue (DLQ)** হলো একটি বিশেষ queue/topic যেখানে সেইসব মেসেজ পাঠানো হয় যেগুলো সফলভাবে process করা সম্ভব হয়নি। এটি messaging system এর "হাসপাতাল" — যেখানে "অসুস্থ" মেসেজগুলো যায় পরবর্তী analysis এবং recovery এর জন্য।

### 🤔 মেসেজ কেন ব্যর্থ হয়?

1. **Poison Pills (বিষাক্ত মেসেজ):** Malformed JSON, corrupted data
2. **Schema Mismatch:** Producer নতুন schema পাঠাচ্ছে, consumer পুরানো schema আশা করছে
3. **Business Rule Violation:** Invalid data (negative amount, non-existent user)
4. **Transient Errors:** Database down, API timeout, network issue (retry করলে ঠিক হতে পারে)
5. **Resource Exhaustion:** Memory full, disk full, connection pool exhausted

### 🔄 Retry vs DLQ Decision:

- **Transient Error → Retry** (সাময়িক সমস্যা — আবার চেষ্টা করলে কাজ হবে)
- **Permanent Error → DLQ** (data corrupt — ১০০ বার retry করলেও কাজ হবে না)

### 📬 DLQ Pattern Variations:

1. **Simple DLQ:** Fail → DLQ (no retry)
2. **Retry + DLQ:** Fail → Retry-1 → Retry-2 → Retry-3 → DLQ
3. **Exponential Backoff DLQ:** Fail → Wait 1s → Wait 10s → Wait 60s → DLQ
4. **Conditional DLQ:** Error type অনুযায়ী আলাদা DLQ

---

## 🌍 বাস্তব জীবনের উদাহরণ (Real-world Example)

### 🛒 Daraz Order Processing DLQ

Daraz-এ order placement এর পর একটি event fire হয়। Order Processing Service এই event consume করে। কিন্তু কিছু order event fail হতে পারে:

**Scenario 1: Malformed Order (Poison Pill)**
```
{
  "order_id": "ORD-98765",
  "items": null,          // ← items array নেই!
  "total": -500,          // ← negative amount!
  "customer_id": ""       // ← empty customer!
}
```
→ এই order ১০০ বার retry করলেও process হবে না → সরাসরি DLQ-তে

**Scenario 2: Payment Gateway Down (Transient)**
- Order event আসলো, payment verify করতে gateway timeout হলো
- ১ মিনিট পর retry → আবার timeout
- ৫ মিনিট পর retry → এবার সফল! ✅
- DLQ-তে যায়নি — retry-তেই solve হয়ে গেছে

**Scenario 3: Out-of-stock (Business Rule)**
- Order event আসলো, কিন্তু product out of stock
- Retry করে লাভ নেই — stock নিজে থেকে বাড়বে না
- Business DLQ-তে যাবে → human review → customer contact

### 📱 Grameenphone Recharge System

- Customer recharge করলো → event fire
- Recharge processing service event পেলো
- কিন্তু prepaid number টি invalid (postpaid number recharge করতে চাইছে)
- Retry অর্থহীন → DLQ → manual review → refund process

---

## 📊 ASCII Diagram

### Basic DLQ Flow:

```
┌──────────────────────────────────────────────────────────────┐
│                    DLQ Architecture                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐     ┌──────────────┐     ┌─────────────┐      │
│  │ Producer │────▶│  Main Topic  │────▶│  Consumer   │      │
│  │ (Daraz   │     │ orders.placed│     │  (Order     │      │
│  │  App)    │     │              │     │  Processor) │      │
│  └──────────┘     └──────────────┘     └──────┬──────┘      │
│                                                │              │
│                                         Process Result        │
│                                                │              │
│                              ┌─────────────────┼──────────┐  │
│                              │                 │           │  │
│                              ▼                 ▼           ▼  │
│                         ┌────────┐      ┌──────────┐  ┌─────┐│
│                         │Success │      │Retryable │  │Fatal││
│                         │  ✅    │      │  Error   │  │Error││
│                         └────────┘      └────┬─────┘  └──┬──┘│
│                                              │            │   │
│                                              ▼            │   │
│                                        ┌──────────┐      │   │
│                                        │  Retry   │      │   │
│                                        │  Topic   │      │   │
│                                        └────┬─────┘      │   │
│                                             │            │   │
│                                    Max retries?          │   │
│                                             │            │   │
│                                    ┌────────┴───┐        │   │
│                                    │ Yes        │ No     │   │
│                                    ▼            ▼        │   │
│                              ┌──────────┐  ┌────────┐   │   │
│                              │   DLQ    │◀─┤ Retry  │   │   │
│                              │  Topic   │  │ Again  │   │   │
│                              └──────────┘  └────────┘   │   │
│                                    ▲                     │   │
│                                    │                     │   │
│                                    └─────────────────────┘   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Multi-Level Retry Pattern:

```
┌──────────────────────────────────────────────────────────────┐
│             Multi-Level Retry with Backoff                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Main Topic          Retry-1         Retry-2         DLQ     │
│  (orders.placed)     (1 min delay)   (5 min delay)           │
│                                                               │
│  ┌─────────┐        ┌─────────┐     ┌─────────┐   ┌──────┐ │
│  │ Message │──FAIL──▶│ Retry 1 │─FAIL▶│ Retry 2 │─FAIL▶│ DLQ │ │
│  │         │        │ Wait 1m │     │ Wait 5m │   │      │ │
│  └─────────┘        └────┬────┘     └────┬────┘   └──────┘ │
│                          │                │                   │
│                     SUCCESS?          SUCCESS?                │
│                          │                │                   │
│                          ▼                ▼                   │
│                     ┌────────┐       ┌────────┐              │
│                     │  Done  │       │  Done  │              │
│                     │   ✅   │       │   ✅   │              │
│                     └────────┘       └────────┘              │
│                                                               │
│  Retry Delays:                                               │
│  ┌────────┬──────────┬──────────────┐                        │
│  │ Level  │  Delay   │  Use Case    │                        │
│  ├────────┼──────────┼──────────────┤                        │
│  │ Try 1  │ 0 sec    │ Main topic   │                        │
│  │ Retry 1│ 1 min    │ API timeout  │                        │
│  │ Retry 2│ 5 min    │ Service down │                        │
│  │ Retry 3│ 30 min   │ Dependency   │                        │
│  │ DLQ    │ Manual   │ Human review │                        │
│  └────────┴──────────┴──────────────┘                        │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### DLQ Monitoring Dashboard:

```
┌──────────────────────────────────────────────────────────────┐
│              DLQ Monitoring & Alerting                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  DLQ Depth Over Time:                                        │
│                                                               │
│  Messages                                                     │
│  │                                                            │
│  │         ╱╲        ← 🚨 Alert Threshold (100)              │
│  │        ╱  ╲                                               │
│  │       ╱    ╲    ╱╲                                        │
│  │      ╱      ╲  ╱  ╲                                      │
│  │     ╱        ╲╱    ╲                                      │
│  │    ╱                 ╲                                    │
│  │   ╱                   ╲___                                │
│  │  ╱                                                        │
│  │_╱                                                         │
│  └────────────────────────────────────────── Time            │
│                                                               │
│  Alert Rules:                                                │
│  ⚠️  DLQ depth > 10  → Slack notification                   │
│  🚨 DLQ depth > 100 → PagerDuty alert                       │
│  💀 DLQ depth > 1000 → Incident declared                    │
│  📈 DLQ growth rate > 10/min → Investigation needed         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Daraz Order Processing with DLQ Pattern
 * Failed orders গুলো DLQ-তে route করা হয়
 */

/**
 * Error Classification - কোন error retry করবো, কোনটা DLQ-তে দেবো
 */
class ErrorClassifier
{
    /**
     * Error retryable কিনা determine করা
     */
    public static function isRetryable(\Throwable $error): bool
    {
        // Transient errors - retry করলে fix হতে পারে
        $retryableErrors = [
            'ConnectionTimeoutException',
            'ServiceUnavailableException',
            'RateLimitException',
            'DatabaseConnectionException',
            'NetworkException',
        ];

        $errorClass = (new \ReflectionClass($error))->getShortName();

        if (in_array($errorClass, $retryableErrors)) {
            return true;
        }

        // HTTP status code based
        if (method_exists($error, 'getStatusCode')) {
            $code = $error->getStatusCode();
            return in_array($code, [429, 500, 502, 503, 504]);
        }

        return false;
    }

    /**
     * Error category determine করা
     */
    public static function classify(\Throwable $error): string
    {
        if ($error instanceof \JsonException) return 'POISON_PILL';
        if ($error instanceof ValidationException) return 'BUSINESS_RULE';
        if ($error instanceof SchemaException) return 'SCHEMA_MISMATCH';
        if (self::isRetryable($error)) return 'TRANSIENT';
        return 'UNKNOWN_FATAL';
    }
}

/**
 * DLQ Message Schema
 */
class DlqMessage
{
    public string $originalTopic;
    public int $originalPartition;
    public int $originalOffset;
    public string $originalKey;
    public string $originalPayload;
    public string $errorType;
    public string $errorMessage;
    public string $errorStackTrace;
    public int $retryCount;
    public string $firstFailedAt;
    public string $lastFailedAt;
    public string $consumerGroup;
    public string $consumerInstance;

    public function toArray(): array
    {
        return [
            'original_topic' => $this->originalTopic,
            'original_partition' => $this->originalPartition,
            'original_offset' => $this->originalOffset,
            'original_key' => $this->originalKey,
            'original_payload' => $this->originalPayload,
            'error' => [
                'type' => $this->errorType,
                'message' => $this->errorMessage,
                'stack_trace' => $this->errorStackTrace,
            ],
            'retry_count' => $this->retryCount,
            'first_failed_at' => $this->firstFailedAt,
            'last_failed_at' => $this->lastFailedAt,
            'consumer' => [
                'group' => $this->consumerGroup,
                'instance' => $this->consumerInstance,
            ],
        ];
    }
}

/**
 * DLQ Router - Failed messages DLQ topic-এ route করা
 */
class DlqRouter
{
    private \RdKafka\Producer $producer;
    private string $dlqTopic;
    private string $retryTopicPrefix;
    private int $maxRetries;

    public function __construct(
        \RdKafka\Producer $producer,
        string $mainTopic,
        int $maxRetries = 3
    ) {
        $this->producer = $producer;
        $this->dlqTopic = "{$mainTopic}.dlq";
        $this->retryTopicPrefix = "{$mainTopic}.retry";
        $this->maxRetries = $maxRetries;
    }

    /**
     * Failed message route করা - retry বা DLQ
     */
    public function route(array $originalMessage, \Throwable $error, int $currentRetry): void
    {
        $errorType = ErrorClassifier::classify($error);

        if ($errorType === 'TRANSIENT' && $currentRetry < $this->maxRetries) {
            $this->sendToRetry($originalMessage, $error, $currentRetry);
        } else {
            $this->sendToDlq($originalMessage, $error, $currentRetry);
        }
    }

    private function sendToRetry(array $originalMessage, \Throwable $error, int $retryCount): void
    {
        $retryTopic = "{$this->retryTopicPrefix}." . ($retryCount + 1);

        $retryMessage = $originalMessage;
        $retryMessage['_retry_metadata'] = [
            'retry_count' => $retryCount + 1,
            'error_message' => $error->getMessage(),
            'retry_at' => date('c'),
            'next_retry_delay_seconds' => $this->calculateBackoff($retryCount + 1),
        ];

        $topic = $this->producer->newTopic($retryTopic);
        $topic->producev(
            RD_KAFKA_PARTITION_UA,
            0,
            json_encode($retryMessage),
            $originalMessage['order_id'] ?? null
        );
        $this->producer->flush(5000);

        echo "🔄 Sent to retry topic: {$retryTopic} (attempt #{$retryCount + 1})\n";
    }

    private function sendToDlq(array $originalMessage, \Throwable $error, int $retryCount): void
    {
        $dlqMessage = new DlqMessage();
        $dlqMessage->originalTopic = $originalMessage['_source_topic'] ?? 'unknown';
        $dlqMessage->originalPartition = $originalMessage['_source_partition'] ?? -1;
        $dlqMessage->originalOffset = $originalMessage['_source_offset'] ?? -1;
        $dlqMessage->originalKey = $originalMessage['order_id'] ?? '';
        $dlqMessage->originalPayload = json_encode($originalMessage);
        $dlqMessage->errorType = ErrorClassifier::classify($error);
        $dlqMessage->errorMessage = $error->getMessage();
        $dlqMessage->errorStackTrace = $error->getTraceAsString();
        $dlqMessage->retryCount = $retryCount;
        $dlqMessage->firstFailedAt = $originalMessage['_first_failed_at'] ?? date('c');
        $dlqMessage->lastFailedAt = date('c');
        $dlqMessage->consumerGroup = 'daraz-order-processor';
        $dlqMessage->consumerInstance = gethostname();

        $topic = $this->producer->newTopic($this->dlqTopic);
        $topic->producev(
            RD_KAFKA_PARTITION_UA,
            0,
            json_encode($dlqMessage->toArray()),
            $dlqMessage->originalKey
        );
        $this->producer->flush(5000);

        echo "💀 Sent to DLQ: {$this->dlqTopic} (after {$retryCount} retries)\n";
    }

    /**
     * Exponential backoff calculation
     */
    private function calculateBackoff(int $retryCount): int
    {
        $baseDelay = 60; // 1 minute
        $maxDelay = 3600; // 1 hour max
        $delay = $baseDelay * pow(2, $retryCount - 1);
        return min($delay, $maxDelay);
    }
}

/**
 * Daraz Order Consumer with DLQ Support
 */
class DarazOrderConsumer
{
    private DlqRouter $dlqRouter;
    private OrderValidator $validator;
    private OrderProcessor $processor;

    public function __construct(
        DlqRouter $dlqRouter,
        OrderValidator $validator,
        OrderProcessor $processor
    ) {
        $this->dlqRouter = $dlqRouter;
        $this->validator = $validator;
        $this->processor = $processor;
    }

    public function consume(string $rawMessage, array $metadata): void
    {
        $retryCount = 0;

        try {
            // Step 1: Parse message
            $message = json_decode($rawMessage, true, 512, JSON_THROW_ON_ERROR);

            $retryCount = $message['_retry_metadata']['retry_count'] ?? 0;

            // Step 2: Validate
            $this->validator->validate($message);

            // Step 3: Process
            $this->processor->processOrder($message);

            echo "✅ Order processed: {$message['order_id']}\n";
        } catch (\JsonException $e) {
            // Poison pill - parse ই হচ্ছে না
            echo "💀 Poison pill detected! Cannot parse message.\n";
            $this->dlqRouter->route(
                ['_raw' => $rawMessage, '_source_topic' => $metadata['topic']],
                $e,
                $retryCount
            );
        } catch (ValidationException $e) {
            // Business validation failure - retry অর্থহীন
            echo "❌ Validation failed: {$e->getMessage()}\n";
            $this->dlqRouter->route($message, $e, $this->maxRetries + 1); // Force DLQ
        } catch (\Throwable $e) {
            // Other errors - classify and route
            $this->dlqRouter->route($message ?? ['_raw' => $rawMessage], $e, $retryCount);
        }
    }
}

/**
 * DLQ Replay Service - DLQ থেকে মেসেজ re-process করা
 */
class DlqReplayService
{
    private \RdKafka\KafkaConsumer $dlqConsumer;
    private \RdKafka\Producer $producer;
    private string $dlqTopic;
    private string $mainTopic;

    public function __construct(string $mainTopic)
    {
        $this->dlqTopic = "{$mainTopic}.dlq";
        $this->mainTopic = $mainTopic;
        // Configure consumer and producer...
    }

    /**
     * DLQ মেসেজ replay - filter সহ
     */
    public function replay(array $filters = [], int $maxMessages = 100): array
    {
        $replayed = [];
        $skipped = [];
        $count = 0;

        while ($count < $maxMessages) {
            $message = $this->dlqConsumer->consume(1000);

            if ($message->err === RD_KAFKA_RESP_ERR__PARTITION_EOF) break;
            if ($message->err !== RD_KAFKA_RESP_ERR_NO_ERROR) continue;

            $dlqPayload = json_decode($message->payload, true);
            $count++;

            // Filter apply করা
            if (!$this->matchesFilter($dlqPayload, $filters)) {
                $skipped[] = $dlqPayload['original_key'];
                continue;
            }

            // Original topic-এ re-publish
            $originalPayload = json_decode($dlqPayload['original_payload'], true);
            unset($originalPayload['_retry_metadata']); // Clean retry metadata

            $topic = $this->producer->newTopic($this->mainTopic);
            $topic->producev(
                RD_KAFKA_PARTITION_UA,
                0,
                json_encode($originalPayload),
                $dlqPayload['original_key']
            );

            $replayed[] = $dlqPayload['original_key'];
        }

        $this->producer->flush(10000);

        return [
            'replayed' => count($replayed),
            'skipped' => count($skipped),
            'replayed_keys' => $replayed,
        ];
    }

    private function matchesFilter(array $dlqPayload, array $filters): bool
    {
        if (isset($filters['error_type'])) {
            if ($dlqPayload['error']['type'] !== $filters['error_type']) return false;
        }
        if (isset($filters['after_date'])) {
            if ($dlqPayload['first_failed_at'] < $filters['after_date']) return false;
        }
        return true;
    }
}

/**
 * DLQ Monitoring - Alert on DLQ depth
 */
class DlqMonitor
{
    private \Redis $redis;
    private string $alertWebhook;

    public function __construct(\Redis $redis, string $alertWebhook)
    {
        $this->redis = $redis;
        $this->alertWebhook = $alertWebhook;
    }

    public function checkDlqDepth(string $dlqTopic, int $currentDepth): void
    {
        $key = "dlq_depth:{$dlqTopic}";
        $previousDepth = (int) $this->redis->get($key);
        $this->redis->set($key, $currentDepth);

        $growthRate = $currentDepth - $previousDepth;

        if ($currentDepth > 1000) {
            $this->alert('CRITICAL', $dlqTopic, $currentDepth, $growthRate);
        } elseif ($currentDepth > 100) {
            $this->alert('WARNING', $dlqTopic, $currentDepth, $growthRate);
        } elseif ($growthRate > 10) {
            $this->alert('INFO', $dlqTopic, $currentDepth, $growthRate);
        }
    }

    private function alert(string $severity, string $topic, int $depth, int $growthRate): void
    {
        $payload = [
            'severity' => $severity,
            'topic' => $topic,
            'depth' => $depth,
            'growth_rate_per_minute' => $growthRate,
            'timestamp' => date('c'),
            'message' => "DLQ Alert: {$topic} has {$depth} messages (growth: {$growthRate}/min)",
        ];

        // Send to Slack/PagerDuty
        echo "🚨 [{$severity}] DLQ {$topic}: depth={$depth}, growth={$growthRate}/min\n";
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Daraz Order Processing with Dead Letter Queue
 * Node.js implementation with KafkaJS
 */

const { Kafka } = require('kafkajs');

// ==========================================
// Error Classifier
// ==========================================
class ErrorClassifier {
    static RETRYABLE_ERRORS = [
        'ECONNREFUSED',
        'ETIMEDOUT',
        'ECONNRESET',
        'ServiceUnavailableError',
        'RateLimitError',
        'DatabaseConnectionError',
    ];

    static classify(error) {
        // Poison pill - parse error
        if (error instanceof SyntaxError) return 'POISON_PILL';
        if (error.name === 'ValidationError') return 'BUSINESS_RULE';
        if (error.name === 'SchemaError') return 'SCHEMA_MISMATCH';

        // HTTP status based
        if (error.statusCode) {
            if ([429, 500, 502, 503, 504].includes(error.statusCode)) {
                return 'TRANSIENT';
            }
            if ([400, 404, 422].includes(error.statusCode)) {
                return 'BUSINESS_RULE';
            }
        }

        // Error code based
        if (this.RETRYABLE_ERRORS.includes(error.code || error.name)) {
            return 'TRANSIENT';
        }

        return 'UNKNOWN_FATAL';
    }

    static isRetryable(error) {
        return this.classify(error) === 'TRANSIENT';
    }
}

// ==========================================
// DLQ Router
// ==========================================
class DlqRouter {
    constructor(producer, mainTopic, maxRetries = 3) {
        this.producer = producer;
        this.mainTopic = mainTopic;
        this.dlqTopic = `${mainTopic}.dlq`;
        this.retryTopicPrefix = `${mainTopic}.retry`;
        this.maxRetries = maxRetries;
    }

    async route(originalMessage, error, currentRetry = 0) {
        const errorType = ErrorClassifier.classify(error);

        if (errorType === 'TRANSIENT' && currentRetry < this.maxRetries) {
            await this.sendToRetry(originalMessage, error, currentRetry);
        } else {
            await this.sendToDlq(originalMessage, error, currentRetry);
        }
    }

    async sendToRetry(originalMessage, error, retryCount) {
        const retryLevel = retryCount + 1;
        const retryTopic = `${this.retryTopicPrefix}.${retryLevel}`;
        const delay = this.calculateBackoff(retryLevel);

        const retryMessage = {
            ...originalMessage,
            _retry_metadata: {
                retry_count: retryLevel,
                error_message: error.message,
                retry_at: new Date().toISOString(),
                next_retry_delay_ms: delay,
                original_topic: this.mainTopic,
            },
        };

        await this.producer.send({
            topic: retryTopic,
            messages: [{
                key: originalMessage.order_id || null,
                value: JSON.stringify(retryMessage),
                headers: {
                    'retry-count': String(retryLevel),
                    'error-type': ErrorClassifier.classify(error),
                    'retry-after-ms': String(delay),
                },
            }],
        });

        console.log(`🔄 Sent to ${retryTopic} (attempt #${retryLevel}, delay: ${delay}ms)`);
    }

    async sendToDlq(originalMessage, error, retryCount) {
        const dlqEnvelope = {
            original_topic: this.mainTopic,
            original_key: originalMessage.order_id || '',
            original_payload: JSON.stringify(originalMessage),
            error: {
                type: ErrorClassifier.classify(error),
                message: error.message,
                stack: error.stack,
                code: error.code || null,
            },
            retry_count: retryCount,
            first_failed_at: originalMessage._retry_metadata?.retry_at || new Date().toISOString(),
            last_failed_at: new Date().toISOString(),
            consumer: {
                group: 'daraz-order-processor',
                instance: process.env.HOSTNAME || 'unknown',
            },
        };

        await this.producer.send({
            topic: this.dlqTopic,
            messages: [{
                key: originalMessage.order_id || null,
                value: JSON.stringify(dlqEnvelope),
                headers: {
                    'error-type': dlqEnvelope.error.type,
                    'original-topic': this.mainTopic,
                    'retry-count': String(retryCount),
                    'dead-at': new Date().toISOString(),
                },
            }],
        });

        console.log(`💀 Sent to DLQ: ${this.dlqTopic} (${dlqEnvelope.error.type})`);
    }

    calculateBackoff(retryLevel) {
        const baseDelay = 60000; // 1 minute
        const maxDelay = 3600000; // 1 hour
        const delay = baseDelay * Math.pow(2, retryLevel - 1);
        // Add jitter to prevent thundering herd
        const jitter = Math.random() * delay * 0.1;
        return Math.min(delay + jitter, maxDelay);
    }
}

// ==========================================
// Daraz Order Consumer with DLQ
// ==========================================
class DarazOrderConsumer {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'daraz-order-processor',
            brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        });

        this.producer = this.kafka.producer();
        this.consumer = this.kafka.consumer({
            groupId: 'daraz-order-processor-group',
        });

        this.dlqRouter = new DlqRouter(this.producer, 'daraz.orders.placed', 3);
    }

    async start() {
        await this.producer.connect();
        await this.consumer.connect();

        // Subscribe to main topic + retry topics
        await this.consumer.subscribe({ topic: 'daraz.orders.placed' });
        await this.consumer.subscribe({ topic: 'daraz.orders.placed.retry.1' });
        await this.consumer.subscribe({ topic: 'daraz.orders.placed.retry.2' });
        await this.consumer.subscribe({ topic: 'daraz.orders.placed.retry.3' });

        await this.consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                await this.processWithDlq(topic, partition, message);
            },
        });
    }

    async processWithDlq(topic, partition, rawMessage) {
        let parsedMessage;
        let retryCount = 0;

        try {
            // Step 1: Parse
            parsedMessage = JSON.parse(rawMessage.value.toString());
            retryCount = parsedMessage._retry_metadata?.retry_count || 0;

            // Check retry delay (for retry topics)
            if (parsedMessage._retry_metadata) {
                const retryAfter = parsedMessage._retry_metadata.next_retry_delay_ms;
                const retryAt = new Date(parsedMessage._retry_metadata.retry_at);
                const elapsed = Date.now() - retryAt.getTime();

                if (elapsed < retryAfter) {
                    // Too early — delay not met yet, re-queue
                    // In production, use delayed topic or scheduler
                    await new Promise(resolve => setTimeout(resolve, Math.min(retryAfter - elapsed, 5000)));
                }
            }

            // Step 2: Validate
            this.validateOrder(parsedMessage);

            // Step 3: Process
            await this.processOrder(parsedMessage);

            console.log(`✅ Order processed: ${parsedMessage.order_id}`);
        } catch (error) {
            console.error(`❌ Error processing message: ${error.message}`);

            await this.dlqRouter.route(
                parsedMessage || { _raw: rawMessage.value.toString() },
                error,
                retryCount
            );
        }
    }

    validateOrder(order) {
        const errors = [];

        if (!order.order_id) errors.push('Missing order_id');
        if (!order.items || !Array.isArray(order.items) || order.items.length === 0) {
            errors.push('Missing or empty items array');
        }
        if (!order.customer_id) errors.push('Missing customer_id');
        if (order.total <= 0) errors.push('Invalid total amount');

        if (errors.length > 0) {
            const err = new Error(`Validation failed: ${errors.join(', ')}`);
            err.name = 'ValidationError';
            throw err;
        }
    }

    async processOrder(order) {
        // Business logic: inventory check, payment, etc.
        console.log(`📦 Processing order ${order.order_id}: ${order.items.length} items, ৳${order.total}`);

        // Simulate possible transient failure
        if (Math.random() < 0.1) {
            const err = new Error('Payment gateway timeout');
            err.code = 'ETIMEDOUT';
            throw err;
        }
    }
}

// ==========================================
// DLQ Replay Service
// ==========================================
class DlqReplayService {
    constructor(kafka, mainTopic) {
        this.kafka = kafka;
        this.mainTopic = mainTopic;
        this.dlqTopic = `${mainTopic}.dlq`;
        this.admin = kafka.admin();
    }

    /**
     * DLQ মেসেজ replay করা - filter এবং transformation সহ
     */
    async replay(options = {}) {
        const {
            errorTypeFilter = null,
            maxMessages = 100,
            dryRun = false,
            transform = null, // Optional message transformation before replay
        } = options;

        const consumer = this.kafka.consumer({
            groupId: `dlq-replay-${Date.now()}`, // Unique group per replay
        });

        const producer = this.kafka.producer();

        await consumer.connect();
        await producer.connect();
        await consumer.subscribe({ topic: this.dlqTopic, fromBeginning: true });

        const results = { replayed: 0, skipped: 0, errors: 0, messages: [] };
        let count = 0;

        await consumer.run({
            eachMessage: async ({ message }) => {
                if (count >= maxMessages) return;
                count++;

                const dlqEnvelope = JSON.parse(message.value.toString());

                // Filter by error type
                if (errorTypeFilter && dlqEnvelope.error.type !== errorTypeFilter) {
                    results.skipped++;
                    return;
                }

                // Parse original payload
                let originalPayload = JSON.parse(dlqEnvelope.original_payload);

                // Remove retry metadata
                delete originalPayload._retry_metadata;

                // Apply transformation if provided
                if (transform) {
                    originalPayload = transform(originalPayload);
                }

                if (dryRun) {
                    results.messages.push({
                        key: dlqEnvelope.original_key,
                        would_replay: true,
                    });
                    results.replayed++;
                    return;
                }

                // Re-publish to main topic
                await producer.send({
                    topic: this.mainTopic,
                    messages: [{
                        key: dlqEnvelope.original_key,
                        value: JSON.stringify(originalPayload),
                        headers: {
                            'replayed-from-dlq': 'true',
                            'original-error': dlqEnvelope.error.type,
                            'replayed-at': new Date().toISOString(),
                        },
                    }],
                });

                results.replayed++;
                console.log(`🔄 Replayed: ${dlqEnvelope.original_key}`);
            },
        });

        // Wait for processing
        await new Promise(resolve => setTimeout(resolve, 5000));

        await consumer.disconnect();
        await producer.disconnect();

        return results;
    }

    /**
     * DLQ statistics
     */
    async getStats() {
        const consumer = this.kafka.consumer({
            groupId: `dlq-stats-${Date.now()}`,
        });

        await consumer.connect();
        await consumer.subscribe({ topic: this.dlqTopic, fromBeginning: true });

        const stats = {
            total: 0,
            byErrorType: {},
            byHour: {},
            oldestMessage: null,
            newestMessage: null,
        };

        await consumer.run({
            eachMessage: async ({ message }) => {
                const envelope = JSON.parse(message.value.toString());
                stats.total++;

                // Count by error type
                const errorType = envelope.error.type;
                stats.byErrorType[errorType] = (stats.byErrorType[errorType] || 0) + 1;

                // Count by hour
                const hour = envelope.last_failed_at.substring(0, 13);
                stats.byHour[hour] = (stats.byHour[hour] || 0) + 1;

                // Track oldest/newest
                if (!stats.oldestMessage || envelope.first_failed_at < stats.oldestMessage) {
                    stats.oldestMessage = envelope.first_failed_at;
                }
                if (!stats.newestMessage || envelope.last_failed_at > stats.newestMessage) {
                    stats.newestMessage = envelope.last_failed_at;
                }
            },
        });

        await new Promise(resolve => setTimeout(resolve, 5000));
        await consumer.disconnect();

        return stats;
    }
}

// ==========================================
// DLQ Monitor with Alerting
// ==========================================
class DlqMonitor {
    constructor(kafka, redis, alertConfig) {
        this.kafka = kafka;
        this.redis = redis;
        this.alertConfig = alertConfig;
        this.admin = kafka.admin();
    }

    async checkAllDlqs(dlqTopics) {
        await this.admin.connect();

        for (const topic of dlqTopics) {
            const offsets = await this.admin.fetchTopicOffsets(topic);
            const groupOffsets = await this.admin.fetchOffsets({
                groupId: `${topic}-processor`,
                topics: [topic],
            });

            let totalLag = 0;
            for (const partition of offsets) {
                const groupOffset = groupOffsets.find(
                    g => g.topic === topic
                )?.partitions.find(
                    p => p.partition === partition.partition
                );

                const lag = parseInt(partition.offset) - parseInt(groupOffset?.offset || '0');
                totalLag += lag;
            }

            await this.evaluateAlert(topic, totalLag);
        }

        await this.admin.disconnect();
    }

    async evaluateAlert(topic, depth) {
        const prevDepth = parseInt(await this.redis.get(`dlq:depth:${topic}`) || '0');
        await this.redis.set(`dlq:depth:${topic}`, depth);

        const growthRate = depth - prevDepth;

        const { warningThreshold, criticalThreshold, growthRateThreshold } = this.alertConfig;

        if (depth >= criticalThreshold) {
            await this.sendAlert('CRITICAL', topic, depth, growthRate);
        } else if (depth >= warningThreshold) {
            await this.sendAlert('WARNING', topic, depth, growthRate);
        } else if (growthRate >= growthRateThreshold) {
            await this.sendAlert('INFO', topic, depth, growthRate);
        }
    }

    async sendAlert(severity, topic, depth, growthRate) {
        const alert = {
            severity,
            topic,
            depth,
            growthRate,
            timestamp: new Date().toISOString(),
            message: `🚨 DLQ Alert [${severity}]: ${topic} has ${depth} unprocessed messages (growth: ${growthRate}/check)`,
        };

        console.log(alert.message);

        // Send to Slack
        if (this.alertConfig.slackWebhook) {
            await fetch(this.alertConfig.slackWebhook, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ text: alert.message }),
            });
        }
    }
}

// ==========================================
// Usage
// ==========================================
async function main() {
    const consumer = new DarazOrderConsumer();
    await consumer.start();
    console.log('🚀 Daraz Order Consumer started with DLQ support!');

    // Periodic DLQ monitoring
    const kafka = new Kafka({ brokers: ['kafka1:9092'] });
    const Redis = require('ioredis');
    const redis = new Redis();

    const monitor = new DlqMonitor(kafka, redis, {
        warningThreshold: 10,
        criticalThreshold: 100,
        growthRateThreshold: 5,
        slackWebhook: process.env.SLACK_WEBHOOK,
    });

    setInterval(async () => {
        await monitor.checkAllDlqs([
            'daraz.orders.placed.dlq',
            'daraz.payments.dlq',
            'daraz.shipments.dlq',
        ]);
    }, 60000); // Check every minute
}

main().catch(console.error);
```

---

## ✅ কখন DLQ ব্যবহার করবেন

| পরিস্থিতি | কেন দরকার |
|-----------|-----------|
| Payment processing (bKash, Nagad) | Failed payments isolate করে investigate করতে |
| Order processing (Daraz) | Malformed orders block করবে না বাকি orders কে |
| SMS/Notification service | Invalid phone numbers আটকে রাখবে না queue |
| Data pipeline (ETL) | Schema mismatch detect এবং fix করতে |
| Event-driven microservices | Poison pills থেকে consumer রক্ষা করতে |

## ❌ কখন DLQ ব্যবহার করবেন না

| পরিস্থিতি | কেন দরকার নেই |
|-----------|--------------|
| All errors are transient | শুধু retry দিলেই চলবে |
| Data loss acceptable | Drop করলেই হবে |
| Real-time critical path | DLQ latency add করবে |
| Simple logging | Log and move on |

## 🏗️ DLQ Best Practices:

```
┌──────────────────────────────────────────────────────────────┐
│                DLQ Best Practices                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. 📋 Always include full context in DLQ message:           │
│     - Original message payload                               │
│     - Error details (type, message, stack)                   │
│     - Source metadata (topic, partition, offset)             │
│     - Retry count and timestamps                             │
│                                                               │
│  2. 🔔 Alert on DLQ growth:                                 │
│     - Set up monitoring dashboards                           │
│     - Threshold-based alerting                               │
│     - Track DLQ depth over time                              │
│                                                               │
│  3. 🔄 Have a replay strategy:                               │
│     - Automated replay for resolved transient issues         │
│     - Manual replay with transformation for fixed bugs       │
│     - Dry-run mode before actual replay                      │
│                                                               │
│  4. 🧹 DLQ retention policy:                                │
│     - Don't keep DLQ messages forever                        │
│     - Set appropriate retention (7-30 days)                  │
│     - Archive to cold storage if needed                      │
│                                                               │
│  5. 🏷️ Categorize errors:                                   │
│     - Separate DLQs per error category if needed             │
│     - Different handling for different error types           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔑 Key Takeaways

1. **DLQ = Safety net** — consumer কে crash হতে দেবেন না, fail মেসেজ DLQ-তে পাঠান
2. **Error classification crucial** — transient errors retry করুন, permanent errors DLQ-তে দিন
3. **Exponential backoff** — retry delay বাড়াতে থাকুন (1s → 10s → 60s → DLQ)
4. **Rich DLQ envelope** — debugging এর জন্য সব context রাখুন
5. **Monitor DLQ depth** — growing DLQ মানে কিছু একটা ভুল হচ্ছে
6. **Replay strategy** — DLQ-তে মেসেজ রেখে ভুলে যাবেন না, process করুন
7. **Jitter in backoff** — thundering herd problem prevent করুন
