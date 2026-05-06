# ⚠️ Dual Writes Problem & Solutions (ডুয়াল রাইট সমস্যা ও সমাধান)

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [কেন Dual Writes বিপজ্জনক](#কেন-dual-writes-বিপজ্জনক)
- [Failure Scenarios](#failure-scenarios)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [সমাধান ১: Outbox Pattern](#সমাধান-১-outbox-pattern)
- [সমাধান ২: CDC (Change Data Capture)](#সমাধান-২-cdc-change-data-capture)
- [সমাধান ৩: Listen-to-Yourself Pattern](#সমাধান-৩-listen-to-yourself-pattern)
- [সমাধান ৪: Transaction Log Tailing](#সমাধান-৪-transaction-log-tailing)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Dual Write Problem** হলো যখন একটি সার্ভিস দুটি ভিন্ন সিস্টেমে (যেমন Database ও Message Broker) আলাদাভাবে লিখে, এবং একটি সফল হলেও অন্যটি ব্যর্থ হতে পারে — যার ফলে **data inconsistency** তৈরি হয়।

### 🔑 মূল সমস্যা:
```
┌─────────────────────────────────────────────────────────┐
│                    DUAL WRITE                            │
│                                                         │
│                   ┌───────────┐                         │
│                   │  Service  │                         │
│                   └─────┬─────┘                         │
│                         │                               │
│              ┌──────────┴──────────┐                    │
│              │                     │                    │
│              ▼                     ▼                    │
│     ┌──────────────┐     ┌──────────────┐              │
│     │   Database   │     │   Message    │              │
│     │   (Write 1)  │     │   Broker     │              │
│     │              │     │   (Write 2)  │              │
│     └──────────────┘     └──────────────┘              │
│                                                         │
│     এই দুটি write ATOMIC নয়!                           │
│     একটি সফল + অন্যটি ব্যর্থ = INCONSISTENCY 💥       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### কেন এটা ঘটে:
- Database এবং Message Broker দুটি **ভিন্ন সিস্টেম**
- দুটির মধ্যে **distributed transaction** সম্ভব নয় (বা practical নয়)
- Network partition, timeout, crash যেকোনো সময় হতে পারে
- **2PC (Two-Phase Commit)** অনেক ক্ষেত্রে unavailable বা slow

---

## 💥 কেন Dual Writes বিপজ্জনক

### সাধারণ (ভুল) পদ্ধতি:

```php
// ❌ বিপজ্জনক কোড - Dual Write!
function createOrder($orderData) {
    // Write 1: Database এ সেভ করো
    $order = Order::create($orderData);        // ← সফল ✅

    // Write 2: Event publish করো
    $kafka->produce('order-created', $order);  // ← ব্যর্থ হতে পারে ❌

    return $order;
}
```

### Timing Diagram - Failure Scenario 1 (DB সফল, Broker ব্যর্থ):

```
সময়  ──────────────────────────────────────────────────────►

Service        DB Write           Broker Publish
   │              │                    │
   │──── INSERT ─►│                    │
   │              │                    │
   │◄─── OK ─────│                    │
   │              │                    │
   │──────────── PUBLISH ─────────────►│
   │              │                    │
   │◄─────────── TIMEOUT/ERROR ───────│ ❌ NETWORK FAIL
   │              │                    │
   
ফলাফল: 
  DB তে order আছে ✅
  কিন্তু event publish হয়নি ❌
  → Payment service জানে না নতুন order এসেছে
  → Inventory update হয়নি
  → Customer email পায়নি
```

### Timing Diagram - Failure Scenario 2 (Broker সফল, DB ব্যর্থ):

```
সময়  ──────────────────────────────────────────────────────►

Service        Broker Publish      DB Write
   │              │                    │
   │──── PUBLISH ►│                    │
   │              │                    │
   │◄─── ACK ────│                    │
   │              │                    │
   │──────────── INSERT ──────────────►│
   │              │                    │
   │◄─────────── DEADLOCK/CRASH ──────│ ❌ DB FAIL
   │              │                    │
   
ফলাফল:
  Event publish হয়েছে ✅ (অন্য services কাজ করছে)
  কিন্তু DB তে order নেই ❌
  → Payment নেওয়া হচ্ছে কিন্তু order নেই!
  → Ghost order সমস্যা 👻
```

### Failure Scenario 3 - Service Crash:

```
সময়  ──────────────────────────────────────────────────────►

Service        DB Write           Broker Publish
   │              │                    │
   │──── INSERT ─►│                    │
   │              │                    │
   │◄─── OK ─────│                    │
   │              │                    │
   │     💀 SERVICE CRASH              │
   │              │                    │
   │   (Broker publish হয়নি!)         │
   
ফলাফল:
  DB তে data saved ✅
  Event কখনোই publish হবে না ❌
  → Silent data inconsistency (সবচেয়ে বিপজ্জনক!)
```

---

## 🇧🇩 বাস্তব জীবনের উদাহরণ

### উদাহরণ: Daraz Order Service

```
┌─────────────────────────────────────────────────────────────┐
│                    Daraz Order Flow                          │
│                                                             │
│  Customer "অর্ডার দিলো" → Order Service                     │
│                                                             │
│  Order Service কে করতে হবে:                                 │
│  ┌────────────────────────────────────────────┐             │
│  │ 1. Orders DB তে order সেভ করো              │             │
│  │ 2. Kafka তে "OrderCreated" event publish   │             │
│  │    → Payment Service (টাকা কেটে নেবে)       │             │
│  │    → Inventory Service (stock কমাবে)        │             │
│  │    → Notification Service (email পাঠাবে)    │             │
│  │    → Analytics Service (report আপডেট)       │             │
│  └────────────────────────────────────────────┘             │
│                                                             │
│  Dual Write সমস্যা:                                         │
│  DB save হলো কিন্তু Kafka publish ব্যর্থ =                  │
│  → Customer এর order আছে কিন্তু payment হচ্ছে না           │
│  → Stock কমছে না → overselling!                             │
│  → Customer কোনো confirmation পাচ্ছে না                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### আরেকটি উদাহরণ: Pathao Ride Service

```
┌────────────────────────────────────────┐
│        Pathao Ride Assignment           │
│                                        │
│  Ride assign হলো:                      │
│  1. DB তে ride status = 'assigned'    │
│  2. Push notification driver কে        │
│  3. Kafka event → billing service      │
│                                        │
│  যদি DB update হয় কিন্তু              │
│  notification না যায়:                  │
│  → Driver জানে না ride আছে!           │
│  → Customer অপেক্ষায় থাকে 😤          │
│                                        │
└────────────────────────────────────────┘
```

---

## 🛠️ সমাধান ১: Outbox Pattern

### ধারণা:
Event publish করার বদলে, **same database transaction** এ একটি outbox table এ লিখো। আলাদা একটি process (relay/poller) outbox থেকে পড়ে broker এ publish করবে।

```
┌────────────────────────────────────────────────────────────┐
│                    OUTBOX PATTERN                           │
│                                                            │
│  ┌───────────┐     SINGLE DB TRANSACTION                   │
│  │  Service  │──────────────────────────────┐              │
│  └───────────┘                              │              │
│       │                                     │              │
│       │ BEGIN TRANSACTION                   │              │
│       │                                     │              │
│       ├─── INSERT order ───►┌──────────┐    │              │
│       │                     │ orders   │    │              │
│       │                     │ table    │    │              │
│       │                     └──────────┘    │              │
│       │                                     │              │
│       ├─── INSERT event ───►┌──────────┐    │              │
│       │                     │ outbox   │    │              │
│       │                     │ table    │    │              │
│       │                     └──────────┘    │              │
│       │                                     │              │
│       │ COMMIT ─────────────────────────────┘              │
│                                                            │
│                                                            │
│  ┌──────────────┐         ┌──────────────┐                │
│  │ Outbox Relay │────────►│    Kafka     │                │
│  │ (Poller/CDC) │         │              │                │
│  └──────────────┘         └──────────────┘                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**সুবিধা**: একটি মাত্র system (DB) এ write — ACID গ্যারান্টি পাই!

---

## 🛠️ সমাধান ২: CDC (Change Data Capture)

### ধারণা:
Database এর **transaction log** (binlog/WAL) থেকে changes ক্যাপচার করে automatically event publish করো।

```
┌────────────────────────────────────────────────────────────┐
│                       CDC Solution                          │
│                                                            │
│  ┌───────────┐                                             │
│  │  Service  │──── INSERT/UPDATE ───►┌──────────┐          │
│  └───────────┘                       │ Database │          │
│                                      └────┬─────┘          │
│                                           │                │
│                                     Transaction Log        │
│                                     (binlog / WAL)         │
│                                           │                │
│                                           ▼                │
│                                    ┌──────────────┐        │
│                                    │   Debezium   │        │
│                                    │   (CDC Tool) │        │
│                                    └──────┬───────┘        │
│                                           │                │
│                                           ▼                │
│                                    ┌──────────────┐        │
│                                    │    Kafka     │        │
│                                    └──────────────┘        │
│                                                            │
│  Service শুধু DB তে লিখে!                                  │
│  CDC tool automatically event তৈরি করে!                    │
│  → Dual write সমস্যা পুরোপুরি দূর!                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 🛠️ সমাধান ৩: Listen-to-Yourself Pattern

### ধারণা:
Service নিজে event publish করে এবং **নিজেই সেই event consume করে** DB update করে।

```
┌────────────────────────────────────────────────────────────┐
│               LISTEN-TO-YOURSELF PATTERN                    │
│                                                            │
│  ┌───────────┐                                             │
│  │  Service  │──── PUBLISH "OrderCreated" ──►┌────────┐   │
│  └─────┬─────┘                               │ Kafka  │   │
│        │                                     └────┬───┘   │
│        │                                          │       │
│        │◄──────── CONSUME "OrderCreated" ─────────┘       │
│        │                                                   │
│        │──── INSERT order ───►┌──────────┐                │
│        │                      │ Database │                │
│        │                      └──────────┘                │
│                                                            │
│  Event হলো "source of truth"                               │
│  DB write event consume করার পরে হয়                       │
│  → একটি মাত্র write path!                                  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**সুবিধা**: Dual write নেই — event-ই primary  
**অসুবিধা**: Latency বাড়ে, complexity বাড়ে, event loss হলে data loss

---

## 🛠️ সমাধান ৪: Transaction Log Tailing

### ধারণা:
Outbox Pattern এর উন্নত সংস্করণ — outbox table এ polling এর বদলে DB transaction log থেকে সরাসরি পড়ো।

```
┌────────────────────────────────────────────────────────────┐
│             TRANSACTION LOG TAILING                         │
│                                                            │
│  ┌───────────┐   BEGIN TXN                                 │
│  │  Service  │─────────────────────┐                       │
│  └───────────┘                     │                       │
│       │                            │                       │
│       ├── INSERT orders ──►        │ ┌──────────┐          │
│       │                            ├─┤ Database │          │
│       ├── INSERT outbox ──►        │ └──────┬───┘          │
│       │                            │        │              │
│       │   COMMIT ──────────────────┘   Transaction         │
│                                         Log               │
│                                          │                 │
│                                          ▼                 │
│                                   ┌──────────────┐         │
│                                   │ Log Tailer   │         │
│                                   │ (reads only  │         │
│                                   │  outbox      │         │
│                                   │  inserts)    │         │
│                                   └──────┬───────┘         │
│                                          │                 │
│                                          ▼                 │
│                                   ┌──────────────┐         │
│                                   │    Kafka     │         │
│                                   └──────────────┘         │
│                                                            │
│  Polling এর চেয়ে efficient                                 │
│  Near real-time event delivery                             │
│  No polling delay                                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### ❌ ভুল পদ্ধতি (Dual Write):

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Events\OrderCreated;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class OrderService
{
    /**
     * ❌ বিপজ্জনক! এটা Dual Write!
     * DB write এবং Kafka publish আলাদা — partial failure হতে পারে
     */
    public function createOrderDangerous(array $data): Order
    {
        // Write 1: Database
        $order = Order::create([
            'customer_id' => $data['customer_id'],
            'total'       => $data['total'],
            'status'      => 'pending',
            'items'       => json_encode($data['items']),
        ]);

        // Write 2: Message Broker
        // ⚠️ এখানে crash হলে event publish হবে না!
        // ⚠️ কিন্তু order DB তে থেকে যাবে!
        try {
            $this->kafkaProducer->produce('orders', [
                'event_type' => 'OrderCreated',
                'order_id'   => $order->id,
                'customer_id'=> $data['customer_id'],
                'total'      => $data['total'],
                'items'      => $data['items'],
            ]);
        } catch (\Exception $e) {
            // কি করবো? Order rollback করবো?
            // কিন্তু customer তো ইতিমধ্যে confirmation পেয়ে গেছে!
            Log::error("Failed to publish OrderCreated event", [
                'order_id' => $order->id,
                'error'    => $e->getMessage(),
            ]);
            // 😱 Data inconsistency!
        }

        return $order;
    }
}
```

### ✅ সঠিক পদ্ধতি (Outbox Pattern):

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\OutboxMessage;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class OrderService
{
    /**
     * ✅ নিরাপদ! Outbox Pattern ব্যবহার করে
     * একই transaction এ order ও outbox message সেভ হয়
     */
    public function createOrderSafe(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            // Business data সেভ করো
            $order = Order::create([
                'customer_id' => $data['customer_id'],
                'total'       => $data['total'],
                'status'      => 'pending',
                'items'       => json_encode($data['items']),
            ]);

            // একই transaction এ outbox এ event রাখো
            OutboxMessage::create([
                'id'             => Str::uuid(),
                'aggregate_type' => 'Order',
                'aggregate_id'   => $order->id,
                'event_type'     => 'OrderCreated',
                'payload'        => json_encode([
                    'order_id'    => $order->id,
                    'customer_id' => $data['customer_id'],
                    'total'       => $data['total'],
                    'items'       => $data['items'],
                    'created_at'  => now()->toISOString(),
                ]),
                'status'         => 'pending',
            ]);

            // দুটোই একসাথে commit হবে — অথবা দুটোই rollback!
            return $order;
        });
    }
}
```

### Outbox Relay (Publisher):

```php
<?php

namespace App\Services;

use App\Models\OutboxMessage;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class OutboxRelay
{
    private $kafkaProducer;
    private const BATCH_SIZE = 100;

    public function __construct($kafkaProducer)
    {
        $this->kafkaProducer = $kafkaProducer;
    }

    /**
     * Outbox থেকে pending messages publish করো
     * Cron বা scheduler দিয়ে চালাও (প্রতি সেকেন্ডে)
     */
    public function publishPendingMessages(): int
    {
        $published = 0;

        $messages = OutboxMessage::where('status', 'pending')
            ->orderBy('created_at')
            ->limit(self::BATCH_SIZE)
            ->lockForUpdate()
            ->get();

        foreach ($messages as $message) {
            try {
                // Kafka তে publish করো
                $this->kafkaProducer->produce(
                    topic: $this->getTopicForEvent($message->event_type),
                    payload: $message->payload,
                    key: $message->aggregate_id,
                    headers: [
                        'event_type' => $message->event_type,
                        'message_id' => $message->id,
                    ]
                );

                // Published হিসেবে mark করো
                $message->update([
                    'status'       => 'published',
                    'published_at' => now(),
                ]);

                $published++;

            } catch (\Exception $e) {
                Log::error("Outbox publish failed", [
                    'message_id' => $message->id,
                    'error'      => $e->getMessage(),
                ]);
                $message->increment('retry_count');
            }
        }

        return $published;
    }

    private function getTopicForEvent(string $eventType): string
    {
        return match ($eventType) {
            'OrderCreated'   => 'daraz.orders.created',
            'OrderCancelled' => 'daraz.orders.cancelled',
            'PaymentReceived'=> 'daraz.payments.received',
            default          => 'daraz.events.general',
        };
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### ❌ ভুল পদ্ধতি (Dual Write):

```javascript
// ❌ বিপজ্জনক Dual Write!
class OrderService {
    constructor(db, kafkaProducer) {
        this.db = db;
        this.kafkaProducer = kafkaProducer;
    }

    async createOrder(orderData) {
        // Write 1: Database
        const order = await this.db.query(
            'INSERT INTO orders (customer_id, total, status) VALUES ($1, $2, $3) RETURNING *',
            [orderData.customerId, orderData.total, 'pending']
        );

        // Write 2: Kafka
        // ⚠️ এই দুটি write atomic নয়!
        try {
            await this.kafkaProducer.send({
                topic: 'order-events',
                messages: [{
                    key: order.rows[0].id.toString(),
                    value: JSON.stringify({
                        eventType: 'OrderCreated',
                        orderId: order.rows[0].id,
                        customerId: orderData.customerId,
                        total: orderData.total,
                    }),
                }],
            });
        } catch (error) {
            // 😱 Order DB তে আছে কিন্তু event publish হয়নি!
            console.error('Kafka publish failed:', error);
            // এখন কি করবো? Order delete করবো? কিন্তু response তো চলে গেছে!
        }

        return order.rows[0];
    }
}
```

### ✅ সঠিক পদ্ধতি (Outbox Pattern):

```javascript
// ✅ নিরাপদ - Outbox Pattern
const { v4: uuidv4 } = require('uuid');

class SafeOrderService {
    constructor(pool) {
        this.pool = pool; // PostgreSQL connection pool
    }

    async createOrder(orderData) {
        const client = await this.pool.connect();

        try {
            await client.query('BEGIN');

            // Write 1: Order সেভ করো
            const orderResult = await client.query(
                `INSERT INTO orders (customer_id, total, status, items)
                 VALUES ($1, $2, $3, $4)
                 RETURNING *`,
                [orderData.customerId, orderData.total, 'pending',
                 JSON.stringify(orderData.items)]
            );
            const order = orderResult.rows[0];

            // Write 2: Outbox এ event রাখো (SAME TRANSACTION!)
            const eventPayload = {
                event_type: 'OrderCreated',
                order_id: order.id,
                customer_id: orderData.customerId,
                total: orderData.total,
                items: orderData.items,
                timestamp: new Date().toISOString(),
            };

            await client.query(
                `INSERT INTO outbox_messages
                 (id, aggregate_type, aggregate_id, event_type, payload, status)
                 VALUES ($1, $2, $3, $4, $5, $6)`,
                [
                    uuidv4(),
                    'Order',
                    order.id.toString(),
                    'OrderCreated',
                    JSON.stringify(eventPayload),
                    'pending',
                ]
            );

            await client.query('COMMIT');

            console.log(`✅ Order ${order.id} created with outbox event`);
            return order;

        } catch (error) {
            await client.query('ROLLBACK');
            console.error('❌ Order creation failed:', error.message);
            throw error;
        } finally {
            client.release();
        }
    }
}
```

### Outbox Relay (Node.js):

```javascript
// outboxRelay.js - Pending messages publish করে
const { Kafka } = require('kafkajs');
const { Pool } = require('pg');

class OutboxRelay {
    constructor(pool, kafkaProducer) {
        this.pool = pool;
        this.kafkaProducer = kafkaProducer;
        this.running = false;
    }

    async start(intervalMs = 1000) {
        this.running = true;
        console.log('🔄 Outbox Relay started');

        while (this.running) {
            try {
                const published = await this.publishBatch();
                if (published > 0) {
                    console.log(`📤 Published ${published} messages`);
                }
            } catch (error) {
                console.error('Relay error:', error.message);
            }
            await this.sleep(intervalMs);
        }
    }

    async publishBatch() {
        const client = await this.pool.connect();
        let published = 0;

        try {
            // Pending messages নাও (with lock)
            const result = await client.query(
                `SELECT * FROM outbox_messages
                 WHERE status = 'pending'
                 ORDER BY created_at ASC
                 LIMIT 100
                 FOR UPDATE SKIP LOCKED`
            );

            for (const message of result.rows) {
                try {
                    // Kafka তে publish
                    await this.kafkaProducer.send({
                        topic: this.getTopicForEvent(message.event_type),
                        messages: [{
                            key: message.aggregate_id,
                            value: message.payload,
                            headers: {
                                event_type: message.event_type,
                                message_id: message.id,
                            },
                        }],
                    });

                    // Mark as published
                    await client.query(
                        `UPDATE outbox_messages
                         SET status = 'published', published_at = NOW()
                         WHERE id = $1`,
                        [message.id]
                    );

                    published++;
                } catch (err) {
                    await client.query(
                        `UPDATE outbox_messages
                         SET retry_count = retry_count + 1
                         WHERE id = $1`,
                        [message.id]
                    );
                }
            }
        } finally {
            client.release();
        }

        return published;
    }

    getTopicForEvent(eventType) {
        const mapping = {
            'OrderCreated': 'daraz.orders.created',
            'OrderCancelled': 'daraz.orders.cancelled',
            'PaymentReceived': 'daraz.payments.received',
        };
        return mapping[eventType] || 'daraz.events.general';
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    stop() {
        this.running = false;
    }
}

module.exports = OutboxRelay;
```

---

## 📊 সমাধানগুলোর তুলনা

```
┌───────────────────────────────────────────────────────────────────┐
│              সমাধান তুলনা (Comparison)                             │
├────────────────┬───────────────┬──────────┬───────────┬──────────┤
│    সমাধান      │  Complexity   │ Latency  │ Guarantee │  Tools   │
├────────────────┼───────────────┼──────────┼───────────┼──────────┤
│ Outbox Pattern │    Medium     │  Medium  │  Strong   │ DB only  │
│ (Polling)      │               │ (1-5s)   │           │          │
├────────────────┼───────────────┼──────────┼───────────┼──────────┤
│ CDC            │    High       │   Low    │  Strong   │ Debezium │
│ (Debezium)     │               │ (<1s)    │           │ Kafka    │
├────────────────┼───────────────┼──────────┼───────────┼──────────┤
│ Listen-to-     │    Medium     │  Medium  │  Medium   │ Kafka    │
│ Yourself       │               │          │           │          │
├────────────────┼───────────────┼──────────┼───────────┼──────────┤
│ Log Tailing    │    High       │   Low    │  Strong   │ Custom   │
│                │               │ (<1s)    │           │ tooling  │
└────────────────┴───────────────┴──────────┴───────────┴──────────┘
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ সমাধান বেছে নেওয়ার গাইড:

| পরিস্থিতি | সেরা সমাধান |
|-----------|-------------|
| Simple, DB already আছে | Outbox Pattern (Polling) |
| Low latency দরকার | CDC (Debezium) |
| Event Sourcing করছেন | Listen-to-Yourself |
| High throughput + low latency | Transaction Log Tailing |
| Team ছোট, infra simple রাখতে চান | Outbox Pattern |
| Already Kafka + Debezium আছে | CDC |

### ❌ Dual Write acceptable কখন:

| পরিস্থিতি | কারণ |
|-----------|-------|
| Inconsistency tolerable (analytics) | কিছু data miss হলেও চলে |
| Retry mechanism আছে | Scheduled job পরে ঠিক করবে |
| Idempotent operations | বারবার চালালেও সমস্যা নেই |
| Non-critical secondary writes | Logging, metrics |

### 💡 Best Practices:

1. **কখনো financial data তে dual write করবেন না** — Outbox/CDC ব্যবহার করুন
2. **Outbox table cleanup করুন** — Published messages মুছুন
3. **Monitoring রাখুন** — Outbox backlog monitor করুন
4. **Ordering maintain করুন** — aggregate_id দিয়ে Kafka partition করুন
5. **Idempotent consumers লিখুন** — At-least-once delivery handle করুন

---

## 🎓 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────┐
│          Dual Writes Problem - মূল কথা             │
├───────────────────────────────────────────────────┤
│                                                    │
│  সমস্যা: দুটি system এ আলাদাভাবে write            │
│  ঝুঁকি: Partial failure → data inconsistency       │
│  সহজ সমাধান: Outbox Pattern                       │
│  উন্নত সমাধান: CDC (Debezium)                      │
│  নিয়ম: Single source of truth বজায় রাখুন         │
│                                                    │
│  মনে রাখুন:                                       │
│  "একটি transaction এ একটি system এ write করুন,    │
│   বাকিটা async handle করুন"                       │
│                                                    │
└───────────────────────────────────────────────────┘
```
