# 📘 Kafka ডিজাইন প্যাটার্ন

> **Apache Kafka-তে ব্যবহৃত গুরুত্বপূর্ণ ডিজাইন প্যাটার্নসমূহ — বাস্তব বাংলাদেশি প্রেক্ষাপটে**

---

## 📖 ইভেন্ট-ড্রিভেন আর্কিটেকচারের প্যাটার্নসমূহ

ইভেন্ট-ড্রিভেন আর্কিটেকচার (EDA) হলো এমন একটি ডিজাইন পদ্ধতি যেখানে সিস্টেমের বিভিন্ন
কম্পোনেন্ট **ইভেন্ট** (ঘটনা) এর মাধ্যমে যোগাযোগ করে। **Pathao** অ্যাপে রাইড রিকোয়েস্ট হলে
একসাথে ড্রাইভার ম্যাচিং, ভাড়া হিসাব, নোটিফিকেশন — সব সমান্তরালে চলে। Kafka এই সমান্তরাল
ইভেন্ট প্রবাহের মেরুদণ্ড হিসেবে কাজ করে।

```
Traditional (সিরিয়াল):
দোকানদার → মালামাল গোছানো → বিল → প্যাকেজিং → ডেলিভারি → ধীর!

Event-Driven (সমান্তরাল):
দোকানদার → [অর্ডার-ইভেন্ট] → গোছানো টিম  ┐
                              → বিলিং টিম   ├─→ দ্রুত!
                              → ডেলিভারি টিম ┘
```

এই অধ্যায়ে আমরা Kafka-তে ব্যবহৃত **৮টি গুরুত্বপূর্ণ ডিজাইন প্যাটার্ন** শিখবো।

---

## 📌 Event Sourcing

### Event Sourcing কী?

এই প্যাটার্নে আমরা **বর্তমান অবস্থা (state) সংরক্ষণ না করে**, প্রতিটি ঘটনা (event) ক্রমানুসারে
সংরক্ষণ করি। বর্তমান অবস্থা জানতে শুরু থেকে সব ইভেন্ট রিপ্লে করি।

🏠 **বাংলাদেশি উদাহরণ:** এটা ঠিক **ব্যাংক পাসবুকের** মতো — পাসবুকে শুধু ব্যালেন্স নয়, প্রতিটি
লেনদেন রেকর্ড থাকে। যেকোনো সময় শুরু থেকে যোগ-বিয়োগ করে ব্যালেন্স বের করা যায়।

```
Traditional:                     Event Sourcing:
┌──────────────────┐             ┌────────────────────────────┐
│ Account          │             │ Event Log                  │
│ Balance:         │             │ [+500] [−200] [+1000]      │
│ 1300 BDT         │             │ [−100] [+100]              │
└──────────────────┘             │ Current = sum = 1300 BDT   │
                                 └────────────────────────────┘
❌ ভুল হলে কারণ খুঁজে             ✅ যেকোনো সময়ের ব্যালেন্স
   পাওয়া কঠিন                      রিপ্লে করে বের করা যায়
```

### ✅ PHP কোড: Event-Sourced bKash ওয়ালেট

```php
<?php
class WalletEvent {
    public string $eventId, $walletId, $type;
    public float $amount;

    public function __construct(string $walletId, string $type, float $amount) {
        $this->eventId = uniqid('evt_');
        $this->walletId = $walletId;
        $this->type = $type;
        $this->amount = $amount;
    }
}

class BkashWallet {
    private array $events = [];

    public function credit(float $amount): void {
        $this->events[] = new WalletEvent($this->walletId, 'credited', $amount);
        $this->publishToKafka(end($this->events));
    }

    public function debit(float $amount): void {
        if ($this->getBalance() < $amount) throw new Exception("অপর্যাপ্ত ব্যালেন্স!");
        $this->events[] = new WalletEvent($this->walletId, 'debited', $amount);
        $this->publishToKafka(end($this->events));
    }

    public function getBalance(): float {
        return array_reduce($this->events, fn($bal, $evt) =>
            $evt->type === 'credited' ? $bal + $evt->amount : $bal - $evt->amount, 0.0);
    }

    private function publishToKafka(WalletEvent $event): void {
        $producer = new RdKafka\Producer($this->kafkaConfig());
        $topic = $producer->newTopic('bkash-wallet-events');
        $topic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode($event), $this->walletId);
        $producer->flush(5000);
    }
}

// ব্যবহার
$wallet = new BkashWallet('01712345678');
$wallet->credit(5000);
$wallet->debit(1500);
echo $wallet->getBalance(); // 3500 BDT
```

### ✅ JS কোড: Event-Sourced অর্ডার সিস্টেম

```javascript
const { Kafka } = require('kafkajs');
const kafka = new Kafka({ brokers: ['localhost:9092'] });

class Order {
  constructor(id) { this.id = id; this.status = 'unknown'; this.events = []; }

  apply(event) {
    this.events.push(event);
    const handlers = {
      ORDER_CREATED:   () => { this.status = 'created'; this.items = event.data.items; },
      PAYMENT_RECEIVED:() => { this.status = 'paid'; },
      ORDER_SHIPPED:   () => { this.status = 'shipped'; },
      ORDER_CANCELLED: () => { this.status = 'cancelled'; },
    };
    handlers[event.type]?.();
  }

  // Time Travel — নির্দিষ্ট সময়ের অবস্থা দেখা
  getStateAt(timestamp) {
    const snapshot = new Order(this.id);
    this.events.filter(e => new Date(e.timestamp) <= new Date(timestamp))
               .forEach(e => snapshot.apply(e));
    return snapshot;
  }
}

// Kafka-তে ইভেন্ট সংরক্ষণ
async function appendEvent(orderId, type, data) {
  const producer = kafka.producer();
  await producer.connect();
  await producer.send({
    topic: 'order-events',
    messages: [{ key: orderId, value: JSON.stringify({ type, data, timestamp: new Date().toISOString() }) }],
  });
  await producer.disconnect();
}
```

### 📊 সুবিধা ও অসুবিধা

| ✅ সুবিধা | ❌ অসুবিধা |
|-----------|-----------|
| সম্পূর্ণ অডিট ট্রেইল | ইভেন্ট স্টোরের আকার দ্রুত বাড়ে |
| Time Travel — যেকোনো সময়ের অবস্থা | Event schema পরিবর্তন জটিল |
| ডিবাগিং সহজ | Eventually consistent |

---

## 📌 CQRS (Command Query Responsibility Segregation)

**ডেটা লেখা (Command)** এবং **পড়া (Query)** আলাদা মডেলে রাখার প্যাটার্ন। Write model ও
Read model সম্পূর্ণ ভিন্ন ডেটাবেস ব্যবহার করতে পারে।

🏠 **Daraz উদাহরণ:** বিক্রেতা পণ্য আপডেট করেন (Command) → Kafka → Elasticsearch (সার্চ) +
MySQL (অর্ডার হিস্ট্রি) + ClickHouse (Analytics) + Redis (ক্যাশ)

```
Command Side:                    Query Side:
┌──────────────┐    Events       ┌────────────────────────┐
│ Commands     │──→ Kafka ────→  │ Read Models            │
│ (Write API)  │    Topics       │ ├─ Elasticsearch       │
│              │                 │ ├─ MySQL               │
│  Database:   │                 │ ├─ ClickHouse          │
│  PostgreSQL  │                 │ └─ Redis               │
└──────────────┘                 └────────────────────────┘
```

### ✅ PHP কোড: CQRS

```php
<?php
class ProductCommandHandler {
    public function handleCreateProduct(PDO $db, KafkaProducer $kafka, array $cmd): string {
        $id = uniqid('prod_');
        $db->prepare("INSERT INTO products (id, name, price) VALUES (?,?,?)")
           ->execute([$id, $cmd['name'], $cmd['price']]);
        $kafka->produce('product-events', [
            'type' => 'PRODUCT_CREATED', 'product_id' => $id, 'data' => $cmd,
        ]);
        return $id;
    }
}

class SearchIndexProjector {
    public function handleEvent(ElasticsearchClient $es, array $event): void {
        match($event['type']) {
            'PRODUCT_CREATED' => $es->index([
                'index' => 'products', 'id' => $event['product_id'], 'body' => $event['data'],
            ]),
            'PRICE_UPDATED' => $es->update([
                'index' => 'products', 'id' => $event['product_id'],
                'body' => ['doc' => ['price' => $event['data']['new_price']]],
            ]),
        };
    }
}
```

### ✅ JS কোড: CQRS Consumer

```javascript
async function startCQRSConsumer() {
  const consumer = kafka.consumer({ groupId: 'search-projector' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'product-events', fromBeginning: true });
  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      await Promise.all([
        updateElasticsearch(event), invalidateRedisCache(event), insertAnalyticsRow(event),
      ]);
    },
  });
}
```

### 🎯 কখন CQRS ব্যবহার করবেন?

| পরিস্থিতি | CQRS? |
|-----------|-------|
| Read ও Write লোড অনেক ভিন্ন | ✅ হ্যাঁ |
| একই ডেটা বিভিন্নভাবে দেখাতে হয় | ✅ হ্যাঁ |
| সরল CRUD অ্যাপ | ❌ অতিরিক্ত জটিলতা |

---

## 📌 Saga Pattern

Saga হলো **distributed transaction** পরিচালনার প্যাটার্ন। একাধিক সার্ভিসে ছড়িয়ে থাকা
ব্যবসায়িক প্রক্রিয়া সমন্বয় করে। ব্যর্থ হলে **compensating transaction** দিয়ে rollback করে।

**Choreography:** প্রতিটি সার্ভিস নিজেই ইভেন্ট শোনে ও প্রকাশ করে (কেন্দ্রীয় নিয়ন্ত্রক নেই)।
**Orchestration:** একটি কেন্দ্রীয় Orchestrator প্রতিটি ধাপ নিয়ন্ত্রণ করে।

```
সফল ফ্লো:
Order Service → [order-created] → Payment Service → [payment-completed] →
Inventory Service → [inventory-reserved] → Shipping Service → [order-shipped] ✅

ব্যর্থ ফ্লো (Compensation):
Payment Service → [payment-failed] → Order Service → [order-cancelled] ❌
```

### 🏠 bKash Send Money Saga

```
ধাপ ১: প্রেরক ডেবিট  → সফল ✅
ধাপ ২: প্রাপক ক্রেডিট → ব্যর্থ ❌
Compensation: প্রেরককে ফেরত দাও (Refund) → ধাপ ১ বিপরীত ↩️
```

### ✅ PHP কোड: Saga (Choreography)

```php
<?php
class PaymentSagaHandler {
    public function handleDebitSender(array $event, KafkaProducer $kafka): void {
        try {
            $this->walletService->debit($event['sender_id'], $event['amount']);
            $kafka->produce('wallet-commands', [
                'saga_id' => $event['saga_id'], 'type' => 'CREDIT_RECEIVER',
                'receiver_id' => $event['receiver_id'], 'amount' => $event['amount'],
            ]);
        } catch (Exception $e) {
            $kafka->produce('saga-events', [
                'saga_id' => $event['saga_id'], 'type' => 'TRANSFER_FAILED',
            ]);
        }
    }

    public function handleCreditReceiver(array $event, KafkaProducer $kafka): void {
        try {
            $this->walletService->credit($event['receiver_id'], $event['amount']);
            $kafka->produce('notification-commands', [
                'saga_id' => $event['saga_id'], 'type' => 'SEND_NOTIFICATION',
            ]);
        } catch (Exception $e) {
            // Compensation — প্রেরককে ফেরত
            $kafka->produce('wallet-commands', [
                'saga_id' => $event['saga_id'], 'type' => 'REFUND_SENDER',
                'sender_id' => $event['sender_id'], 'amount' => $event['amount'],
            ]);
        }
    }
}
```

### ✅ JS কোড: Saga Orchestrator

```javascript
class TransferSagaOrchestrator {
  constructor(producer) { this.producer = producer; this.sagas = new Map(); }

  async startTransfer(senderId, receiverId, amount) {
    const sagaId = `saga_${Date.now()}`;
    this.sagas.set(sagaId, { status: 'STARTED', steps: [], senderId, receiverId, amount });
    await this.executeStep(sagaId, 'DEBIT_SENDER', { walletId: senderId, amount });
  }

  async handleStepResult(sagaId, stepName, success) {
    const saga = this.sagas.get(sagaId);
    saga.steps.push({ stepName, success });
    if (!success) { await this.compensate(sagaId); return; }

    const next = { DEBIT_SENDER: 'CREDIT_RECEIVER', CREDIT_RECEIVER: 'SEND_NOTIFICATION' };
    if (next[stepName]) await this.executeStep(sagaId, next[stepName], { amount: saga.amount });
    else saga.status = 'COMPLETED';
  }

  async compensate(sagaId) {
    const saga = this.sagas.get(sagaId);
    const compensations = { DEBIT_SENDER: 'REFUND_SENDER', CREDIT_RECEIVER: 'REVERSE_CREDIT' };
    for (const step of saga.steps.filter(s => s.success).reverse()) {
      if (compensations[step.stepName])
        await this.executeStep(sagaId, compensations[step.stepName], { amount: saga.amount });
    }
    saga.status = 'COMPENSATED';
  }

  async executeStep(sagaId, stepName, payload) {
    await this.producer.send({
      topic: 'saga-commands',
      messages: [{ key: sagaId, value: JSON.stringify({ sagaId, stepName, payload }) }],
    });
  }
}
```

---

## 📌 Outbox Pattern

### সমস্যা: Dual Write

মাইক্রোসার্ভিসে **ডেটাবেস আপডেট + Kafka ইভেন্ট** একসাথে করতে গেলে যেকোনো একটি ব্যর্থ
হতে পারে, ফলে inconsistency তৈরি হয়।

### সমাধান

ইভেন্ট **একই DB transaction-এ** `outbox_events` টেবিলে লিখুন। আলাদা Relay প্রসেস
(Debezium CDC অথবা Polling) সেখান থেকে Kafka-তে পাঠাবে।

```
┌────────────┐     ┌─────────────────┐     ┌───────────┐     ┌───────┐
│ Application│────→│   Database       │────→│ Outbox    │────→│ Kafka │
│            │     │ ┌─────────────┐ │     │ Relay     │     │       │
│ BEGIN TX   │     │ │ orders      │ │     │ (Debezium │     │       │
│ INSERT     │     │ │ outbox_events│ │     │  or Poll) │     │       │
│ COMMIT     │     │ └─────────────┘ │     │           │     │       │
└────────────┘     └─────────────────┘     └───────────┘     └───────┘
  একই Transaction!                         আলাদা প্রসেস
```

**Debezium CDC:** DB-র binlog পড়ে রিয়েল-টাইমে Kafka-তে পাঠায়।
**Polling:** ক্রন জব নিয়মিত outbox টেবিল চেক করে অপ্রেরিত ইভেন্ট পাঠায়।

### ✅ PHP কোড: Outbox Pattern

```php
<?php
class OrderService {
    public function createOrder(PDO $db, array $data): string {
        $orderId = uniqid('ord_');
        $db->beginTransaction();
        try {
            $db->prepare("INSERT INTO orders (id, customer_id, total, status) VALUES (?,?,?,'pending')")
               ->execute([$orderId, $data['customer_id'], $data['total']]);
            $db->prepare("INSERT INTO outbox_events (id, aggregate_type, aggregate_id, event_type, payload)
                          VALUES (?, 'Order', ?, 'ORDER_CREATED', ?)")
               ->execute([uniqid('evt_'), $orderId, json_encode($data)]);
            $db->commit();
            return $orderId;
        } catch (Exception $e) { $db->rollBack(); throw $e; }
    }
}

class OutboxRelay {
    public function relay(PDO $db, KafkaProducer $kafka): void {
        $db->beginTransaction();
        $events = $db->query("SELECT * FROM outbox_events WHERE sent=0 ORDER BY created_at LIMIT 100 FOR UPDATE")
                     ->fetchAll();
        foreach ($events as $evt) {
            $kafka->produce(strtolower($evt['aggregate_type']) . '-events', $evt);
            $db->prepare("UPDATE outbox_events SET sent=1 WHERE id=?")->execute([$evt['id']]);
        }
        $db->commit();
    }
}
```

### ✅ JS কোড: Outbox Relay

```javascript
class OutboxRelay {
  constructor(dbPool, kafkaProducer) { this.db = dbPool; this.producer = kafkaProducer; }

  async processOutbox() {
    const conn = await this.db.getConnection();
    try {
      await conn.beginTransaction();
      const [events] = await conn.query(
        'SELECT * FROM outbox_events WHERE sent=0 ORDER BY created_at LIMIT 50 FOR UPDATE'
      );
      if (!events.length) { await conn.commit(); return; }

      const messages = events.map(e => ({
        key: e.aggregate_id,
        value: JSON.stringify({ eventType: e.event_type, payload: JSON.parse(e.payload) }),
      }));
      await this.producer.send({ topic: `${events[0].aggregate_type.toLowerCase()}-events`, messages });
      await conn.query(`UPDATE outbox_events SET sent=1 WHERE id IN (?)`, [events.map(e => e.id)]);
      await conn.commit();
    } catch (err) { await conn.rollback(); } finally { conn.release(); }
  }

  startPolling(intervalMs = 1000) { setInterval(() => this.processOutbox(), intervalMs); }
}
```

---

## 📌 Dead Letter Queue (DLQ) Pattern

DLQ হলো **ব্যর্থ মেসেজের** জন্য বিশেষ Kafka টপিক। বারবার প্রসেস ব্যর্থ হলে মেসেজ DLQ-তে
সরিয়ে রাখা হয় যাতে মূল প্রসেসিং আটকে না যায়।

**মেসেজ ব্যর্থ হওয়ার কারণ:**
- 🧪 **Poison Pill** — ভুল ফরম্যাটের মেসেজ
- ⏳ **Transient Error** — সাময়িক নেটওয়ার্ক/DB সমস্যা
- 📋 **Business Error** — অপর্যাপ্ত ব্যালেন্স ইত্যাদি

```
main-topic → Consumer → Process → ❌ ব্যর্থ → retry-topic (retry 1-3)
                                               → ❌ এখনো ব্যর্থ → dlq-topic

dlq-topic → ম্যানুয়াল রিভিউ / অ্যালার্ট / পরে পুনরায় চেষ্টা
```

### ✅ PHP কোড: DLQ

```php
<?php
class DLQConsumer {
    private int $maxRetries = 3;

    public function consume(KafkaConsumer $consumer, KafkaProducer $producer): void {
        while (true) {
            $msg = $consumer->consume(1000);
            if ($msg->err) continue;

            $payload    = json_decode($msg->payload, true);
            $retryCount = $payload['_retry_count'] ?? 0;

            try {
                $this->processMessage($payload);
            } catch (Exception $e) {
                if ($retryCount < $this->maxRetries) {
                    $payload['_retry_count'] = $retryCount + 1;
                    $producer->produce("orders-retry-" . ($retryCount + 1), $payload);
                } else {
                    $payload['_failed_at'] = date('c');
                    $payload['_error']     = $e->getMessage();
                    $producer->produce('orders-dlq', $payload);
                    echo "☠️ DLQ: {$e->getMessage()}\n";
                }
            }
        }
    }
}
```

### ✅ JS কোড: DLQ

```javascript
const MAX_RETRIES = 3;

async function consumeWithDLQ(mainTopic) {
  const consumer = kafka.consumer({ groupId: `${mainTopic}-consumer` });
  const producer = kafka.producer();
  await consumer.connect(); await producer.connect();
  await consumer.subscribe({ topic: mainTopic });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const payload = JSON.parse(message.value.toString());
      const retryCount = payload._retryCount || 0;
      try {
        await processOrder(payload);
      } catch (error) {
        const target = retryCount < MAX_RETRIES
          ? { topic: `${mainTopic}-retry-${retryCount + 1}`,
              data: { ...payload, _retryCount: retryCount + 1, _lastError: error.message } }
          : { topic: `${mainTopic}-dlq`,
              data: { ...payload, _failedAt: new Date().toISOString(), _finalError: error.message } };
        await producer.send({ topic: target.topic,
          messages: [{ key: message.key, value: JSON.stringify(target.data) }] });
      }
    },
  });
}
```

---

## 📌 Retry Pattern

### Exponential Backoff সহ Retry

শুধু বারবার চেষ্টা নয় — প্রতিবার **আরো বেশি সময় অপেক্ষা** করতে হয়:

```
Retry #1: ১ মিনিট পর
Retry #2: ৫ মিনিট পর
Retry #3: ৩০ মিনিট পর
সর্বোচ্চ retry শেষ → DLQ

orders → Consumer → ❌ → orders-retry-1 (1m)
                         → ❌ → orders-retry-2 (5m)
                                → ❌ → orders-retry-3 (30m)
                                       → ❌ → orders-dlq
```

### ✅ PHP কোড: Retry with Backoff

```php
<?php
class RetryableConsumer {
    private array $delays = [1 => 60, 2 => 300, 3 => 1800];

    public function consumeWithRetry(string $topic): void {
        while (true) {
            $msg = $this->consumer->consume(1000);
            if ($msg->err) continue;
            $payload = json_decode($msg->payload, true);

            // টাইমস্ট্যাম্প চেক — সময় হয়েছে কি?
            if (($payload['_retry_after'] ?? 0) > time()) continue;

            try {
                $this->processMessage($payload);
            } catch (Exception $e) {
                $retry = ($payload['_retry_count'] ?? 0) + 1;
                if (isset($this->delays[$retry])) {
                    $payload['_retry_count'] = $retry;
                    $payload['_retry_after'] = time() + $this->delays[$retry];
                    $this->producer->produce("{$topic}-retry-{$retry}", $payload);
                } else {
                    $this->producer->produce("{$topic}-dlq", $payload);
                }
            }
        }
    }
}
```

### ✅ JS কোড: Retry with Backoff

```javascript
class RetryHandler {
  constructor(producer, baseTopic) {
    this.producer = producer;
    this.config = [
      { topic: `${baseTopic}-retry-1`, delayMs: 60_000 },
      { topic: `${baseTopic}-retry-2`, delayMs: 300_000 },
      { topic: `${baseTopic}-retry-3`, delayMs: 1_800_000 },
    ];
    this.dlqTopic = `${baseTopic}-dlq`;
  }

  async handleFailure(message, error) {
    const payload = JSON.parse(message.value.toString());
    const retryCount = payload._retryCount || 0;
    const isRetryable = retryCount < this.config.length;

    const target = isRetryable
      ? { topic: this.config[retryCount].topic,
          data: { ...payload, _retryCount: retryCount + 1,
                  _retryAfter: Date.now() + this.config[retryCount].delayMs } }
      : { topic: this.dlqTopic,
          data: { ...payload, _failedAt: new Date().toISOString(), _finalError: error.message } };

    await this.producer.send({
      topic: target.topic,
      messages: [{ key: message.key, value: JSON.stringify(target.data) }],
    });
  }
}
```

---

## 📌 Claim Check Pattern

### সমস্যা: Kafka-তে বড় মেসেজ (>1MB)

Kafka ডিফল্টভাবে সর্বোচ্চ **1MB** সাপোর্ট করে। Daraz-এ বড় ক্যাটালগ বা ইমেজ পাঠাতে
সরাসরি Kafka ব্যবহার অসম্ভব।

### সমাধান

বড় payload **S3/blob storage-এ** রেখে Kafka-তে শুধু **রেফারেন্স** পাঠাও।

```
Producer → ① বড় ফাইল S3-তে আপলোড → ② {s3_key: "..."} Kafka-তে পাঠাও
Consumer → ③ {s3_key} Kafka থেকে পড়ো → ④ S3 থেকে ডাউনলোড → ⑤ প্রসেস
```

### ✅ PHP কোড: Claim Check

```php
<?php
class ClaimCheckProducer {
    public function send(string $topic, string $key, array $payload): void {
        $data = json_encode($payload);

        if (strlen($data) > 500_000) {
            $s3Key = "kafka-payloads/{$topic}/{$key}/" . uniqid() . ".json";
            $this->s3->putObject(['Bucket' => 'daraz-payloads', 'Key' => $s3Key, 'Body' => $data]);

            $this->kafka->produce($topic, ['_claim_check' => true, 's3_key' => $s3Key], $key);
        } else {
            $this->kafka->produce($topic, $payload, $key);
        }
    }
}

class ClaimCheckConsumer {
    public function resolve(array $message): array {
        if (!empty($message['_claim_check'])) {
            $obj = $this->s3->getObject(['Bucket' => 'daraz-payloads', 'Key' => $message['s3_key']]);
            return json_decode($obj['Body']->getContents(), true);
        }
        return $message;
    }
}
```

### ✅ JS কোড: Claim Check

```javascript
const { S3Client, PutObjectCommand, GetObjectCommand } = require('@aws-sdk/client-s3');
const s3 = new S3Client({ region: 'ap-southeast-1' });
const BUCKET = 'daraz-payloads';

class ClaimCheckProducer {
  constructor(kafkaProducer) { this.producer = kafkaProducer; }

  async send(topic, key, payload) {
    const data = JSON.stringify(payload);
    if (Buffer.byteLength(data) > 500_000) {
      const s3Key = `kafka/${topic}/${key}/${Date.now()}.json`;
      await s3.send(new PutObjectCommand({ Bucket: BUCKET, Key: s3Key, Body: data }));
      await this.producer.send({ topic, messages: [{ key, value: JSON.stringify({ _claimCheck: true, s3Key }) }] });
    } else {
      await this.producer.send({ topic, messages: [{ key, value: data }] });
    }
  }
}
```

---

## 📌 Event-Driven Microservices

### Kafka — মাইক্রোসার্ভিসের কেন্দ্রীয় স্নায়ুতন্ত্র

সম্পূর্ণ মাইক্রোসার্ভিস আর্কিটেকচারে Kafka প্রতিটি সার্ভিসের মধ্যে **loosely coupled**
যোগাযোগ নিশ্চিত করে।

### 🏠 Pathao-এর মতো সিস্টেম

```
┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐
│  Ride          │  │  Driver        │  │  Payment       │  │  Notification   │
│  Service       │  │  Service       │  │  Service       │  │  Service        │
│ (রাইড রিকোয়েস্ট)│  │ (ড্রাইভার ম্যাচ) │  │ (bKash/Nagad)  │  │ (SMS/Push)      │
└───────┬────────┘  └───────┬────────┘  └───────┬────────┘  └────────┬────────┘
        │                   │                   │                    │
════════╪═══════════════════╪═══════════════════╪════════════════════╪═════════
        │              Kafka Cluster                                 │
        │  ride-requests │ ride-matched │ payment-events │ notifications  │
════════╪═══════════════════╪═══════════════════╪════════════════════╪═════════
        │                   │                   │                    │
┌───────┴────────┐  ┌──────┴─────────┐  ┌──────┴─────────┐  ┌──────┴────────┐
│  Pricing       │  │  Promo         │  │  Analytics     │  │  Support      │
│  Service       │  │  Service       │  │  Service       │  │  Service      │
└────────────────┘  └────────────────┘  └────────────────┘  └───────────────┘
```

### সার্ভিস কমিউনিকেশন প্যাটার্ন

| প্যাটার্ন | বিবরণ | উদাহরণ |
|-----------|-------|--------|
| **Event Notification** | ঘটনা জানানো | `ride-completed` → Payment |
| **Event-Carried State** | ইভেন্টে পুরো ডেটা | `driver-updated` ইভেন্টে সব তথ্য |
| **Async Request-Reply** | রিকোয়েস্ট/রেসপন্স আলাদা টপিকে | Pricing request → response |

### ✅ JS কোড: Event-Driven Ride Service

```javascript
class RideService {
  constructor() { this.producer = kafka.producer(); }

  async requestRide(passengerId, pickup, dropoff) {
    const rideId = `ride_${Date.now()}`;
    await this.producer.send({
      topic: 'ride-requests',
      messages: [{ key: rideId, value: JSON.stringify({
        type: 'RIDE_REQUESTED', rideId, passengerId, pickup, dropoff,
      }) }],
    });
    return rideId;
  }
}

// Driver Matching — Kafka Consumer
class DriverMatchingService {
  async start() {
    const consumer = kafka.consumer({ groupId: 'driver-matching' });
    await consumer.connect();
    await consumer.subscribe({ topic: 'ride-requests' });
    await consumer.run({
      eachMessage: async ({ message }) => {
        const event  = JSON.parse(message.value.toString());
        const driver = await this.findNearestDriver(event.pickup);
        await this.producer.send({
          topic: 'ride-matched',
          messages: [{ key: event.rideId, value: JSON.stringify({
            type: 'DRIVER_MATCHED', rideId: event.rideId, driverId: driver.id,
          }) }],
        });
      },
    });
  }
}
```

---

## 📊 প্যাটার্ন তুলনা টেবিল

| প্যাটার্ন | ব্যবহারের ক্ষেত্র | জটিলতা | কখন ব্যবহার করবেন |
|-----------|------------------|--------|-------------------|
| **Event Sourcing** | অডিট ট্রেইল, Time Travel | 🔴 উচ্চ | ফিনান্সিয়াল সিস্টেম (bKash, Nagad) |
| **CQRS** | Read/Write আলাদা করা | 🟡 মাঝারি | Read-heavy সিস্টেম (Daraz সার্চ) |
| **Saga** | Distributed Transaction | 🔴 উচ্চ | একাধিক সার্ভিসে লেনদেন |
| **Outbox** | Dual Write সমাধান | 🟢 নিম্ন | যেকোনো DB + Kafka ইন্টিগ্রেশন |
| **DLQ** | ব্যর্থ মেসেজ ব্যবস্থাপনা | 🟢 নিম্ন | সকল Kafka Consumer |
| **Retry** | সাময়িক ত্রুটিতে পুনঃচেষ্টা | 🟢 নিম্ন | Transient error সম্ভাব্য ক্ষেত্রে |
| **Claim Check** | বড় মেসেজ (>1MB) | 🟢 নিম্ন | ফাইল/ইমেজ প্রসেসিং |
| **Event-Driven** | সার্ভিস কমিউনিকেশন | 🟡 মাঝারি | মাইক্রোসার্ভিস আর্কিটেকচার |

### 📌 প্যাটার্ন নির্বাচনের সিদ্ধান্ত গাইড

```
আপনার সমস্যা কী?
├── DB ↔ Kafka ডেটা হারাচ্ছেন?        → Outbox Pattern
├── একাধিক সার্ভিসে Transaction?       → Saga Pattern
├── Read/Write লোড অনেক আলাদা?       → CQRS
├── অডিট ট্রেইল / ইতিহাস দরকার?       → Event Sourcing
├── মেসেজ প্রসেসিং বারবার ব্যর্থ?       → DLQ + Retry
├── Kafka-তে বড় ডেটা পাঠাতে হবে?     → Claim Check
└── Loosely coupled সার্ভিস?           → Event-Driven Architecture
```

---

## 🎯 সারসংক্ষেপ

| # | প্যাটার্ন | মনে রাখবেন |
|---|----------|-----------|
| ১ | **Event Sourcing** | ইভেন্ট সংরক্ষণ করুন, state নয় — ব্যাংক পাসবুকের মতো |
| ২ | **CQRS** | পড়া ও লেখা আলাদা — Daraz সার্চ ও অর্ডার আলাদা DB-তে |
| ৩ | **Saga** | Distributed transaction-এ compensating action রাখুন |
| ৪ | **Outbox** | DB + Kafka একই transaction-এ — ডেটা হারানো বন্ধ |
| ৫ | **DLQ** | ব্যর্থ মেসেজ হারাবেন না — আলাদা টপিকে রাখুন |
| ৬ | **Retry** | Exponential backoff — সার্ভার প্লাবিত করবেন না |
| ৭ | **Claim Check** | বড় payload S3-তে, Kafka-তে রেফারেন্স |
| ৮ | **Event-Driven** | Kafka দিয়ে সার্ভিসগুলো loosely couple করুন |

### 🏠 বাংলাদেশি কোম্পানিতে ব্যবহার

```
bKash / Nagad  → Event Sourcing + Saga + Outbox (ফিনান্সিয়াল লেনদেন)
Daraz          → CQRS + Claim Check + Event-Driven (ই-কমার্স)
Pathao         → Event-Driven + Saga + DLQ (রাইড শেয়ারিং)
Grameenphone   → CQRS + Retry + DLQ (টেলিকম সিস্টেম)
```

---

## 🔗 পরবর্তী টপিক

➡️ **[Kafka Performance Tuning](kafka-performance.md)** — Kafka পারফরম্যান্স অপটিমাইজেশন, থ্রুপুট বৃদ্ধি, ল্যাটেন্সি কমানো, এবং প্রোডাকশন-রেডি কনফিগারেশন।

---

> **📝 নোট:** সকল কোড উদাহরণ শিক্ষামূলক। প্রোডাকশনে error handling, logging, monitoring ও security যোগ করুন।
