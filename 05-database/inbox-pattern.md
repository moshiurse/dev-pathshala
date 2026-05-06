# 📥 Inbox Pattern (Transactional Inbox for Reliable Message Consumption)

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [কেন Inbox Pattern দরকার](#কেন-inbox-pattern-দরকার)
- [Outbox Pattern এর সাথে সম্পর্ক](#outbox-pattern-এর-সাথে-সম্পর্ক)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Cleanup ও Dead Message Handling](#cleanup-ও-dead-message-handling)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Inbox Pattern** হলো একটি মেসেজিং প্যাটার্ন যা incoming মেসেজের **idempotent processing** নিশ্চিত করে। এটি Outbox Pattern এর বিপরীত (inverse) — Outbox যেখানে মেসেজ পাঠানোর reliability নিশ্চিত করে, Inbox মেসেজ গ্রহণের reliability নিশ্চিত করে।

### 🔑 মূল ধারণা:
- **Deduplication**: একই মেসেজ একাধিকবার আসলেও একবারই প্রসেস হবে
- **Idempotency**: একই অপারেশন বারবার চালালেও ফলাফল একই থাকবে
- **At-least-once delivery handling**: মেসেজ ব্রোকার অন্তত একবার ডেলিভারি গ্যারান্টি দেয়, কিন্তু duplicate আসতে পারে
- **Transactional guarantee**: মেসেজ প্রসেসিং এবং inbox এ রেকর্ড করা একই transaction এ হয়

### 💡 কীভাবে কাজ করে:
```
মেসেজ আসে → Inbox টেবিলে message_id চেক করো → 
  যদি আগে প্রসেস না হয়ে থাকে → প্রসেস করো + একই transaction এ inbox এ mark করো
  যদি আগে প্রসেস হয়ে থাকে → মেসেজ skip/acknowledge করো
```

---

## ❓ কেন Inbox Pattern দরকার

### সমস্যা: At-Least-Once Delivery
বেশিরভাগ মেসেজ ব্রোকার (Kafka, RabbitMQ) **at-least-once delivery** গ্যারান্টি দেয়। এর মানে:

```
+------------------------------------------+
|        At-Least-Once Delivery            |
+------------------------------------------+
| ১. মেসেজ পাঠানো হলো                     |
| ২. Consumer প্রসেস করলো                  |
| ৩. ACK পাঠাতে গিয়ে network fail         |
| ৪. Broker মনে করে ডেলিভার হয়নি          |
| ৫. আবার একই মেসেজ পাঠায়                 |
| ৬. Consumer দ্বিতীয়বার পায় (DUPLICATE!) |
+------------------------------------------+
```

### Duplicate ছাড়া কি হতে পারে?
- 💸 bKash এ একই পেমেন্ট দুইবার credit হয়ে যেতে পারে
- 📦 Daraz এ একই অর্ডার দুইবার প্লেস হতে পারে
- 📧 একই SMS দুইবার পাঠানো হতে পারে

---

## 🔄 Outbox Pattern এর সাথে সম্পর্ক

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Service A                          Service B              │
│   ┌──────────┐                      ┌──────────┐           │
│   │          │    Message Broker     │          │           │
│   │  OUTBOX  │ ──────────────────► │  INBOX   │           │
│   │  Pattern │    (Kafka/RabbitMQ)   │  Pattern │           │
│   │          │                      │          │           │
│   └──────────┘                      └──────────┘           │
│                                                             │
│   "আমি নিশ্চিত করি                  "আমি নিশ্চিত করি      │
│    মেসেজ অবশ্যই                      মেসেজ একবারই          │
│    পাঠানো হবে"                        প্রসেস হবে"           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| বিষয় | Outbox Pattern | Inbox Pattern |
|--------|---------------|---------------|
| উদ্দেশ্য | মেসেজ পাঠানো নিশ্চিত করা | মেসেজ একবার প্রসেস নিশ্চিত করা |
| টেবিল | outbox_messages | inbox_messages |
| Guarantee | At-least-once sending | Exactly-once processing |
| সমাধান | Partial failure in publishing | Duplicate consumption |

---

## 🏗️ আর্কিটেকচার ডায়াগ্রাম

### মূল Flow:

```
                    ┌──────────────────┐
                    │  Message Broker  │
                    │  (Kafka/RabbitMQ)│
                    └────────┬─────────┘
                             │
                             │ মেসেজ আসে
                             ▼
                    ┌──────────────────┐
                    │    Consumer      │
                    │    Service       │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ message_id চেক   │
                    │ (inbox table)    │
                    └────────┬─────────┘
                             │
                    ┌────────┴────────┐
                    │                  │
              নতুন মেসেজ          আগেই আছে
                    │                  │
                    ▼                  ▼
         ┌──────────────────┐  ┌──────────────┐
         │ BEGIN TRANSACTION│  │   ACK করো    │
         │                  │  │   Skip করো   │
         │ 1. Business      │  └──────────────┘
         │    Logic চালাও   │
         │                  │
         │ 2. Inbox টেবিলে  │
         │    INSERT করো    │
         │                  │
         │ COMMIT           │
         └──────────────────┘
                    │
                    ▼
         ┌──────────────────┐
         │    ACK to Broker │
         └──────────────────┘
```

### Inbox Database Schema:

```
┌─────────────────────────────────────────────┐
│              inbox_messages                   │
├─────────────────────────────────────────────┤
│ message_id      VARCHAR(255) PRIMARY KEY     │
│ source          VARCHAR(100)                 │
│ event_type      VARCHAR(100)                 │
│ payload         JSON                         │
│ received_at     TIMESTAMP                    │
│ processed_at    TIMESTAMP                    │
│ status          ENUM('processing','done',    │
│                      'failed')               │
│ retry_count     INT DEFAULT 0               │
│ error_message   TEXT NULL                    │
└─────────────────────────────────────────────┘
```

---

## 🇧🇩 বাস্তব জীবনের উদাহরণ

### উদাহরণ: bKash Payment Confirmation

ধরুন bKash এর Payment Service ব্যাংক গেটওয়ে থেকে পেমেন্ট কনফার্মেশন ইভেন্ট receive করে:

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Bank       │         │    Kafka     │         │   bKash      │
│   Gateway    │────────►│   Broker     │────────►│   Payment    │
│              │         │              │         │   Service    │
└──────────────┘         └──────────────┘         └──────┬───────┘
                                                         │
                                                         ▼
                                                  ┌──────────────┐
                                                  │   Database   │
                                                  │              │
                                                  │ - inbox      │
                                                  │ - wallets    │
                                                  │ - txn_log    │
                                                  └──────────────┘
```

**সমস্যা**: ব্যাংক গেটওয়ে retry করে, Kafka re-deliver করে — একই payment confirmation বারবার আসতে পারে। Inbox Pattern ছাড়া একই ১০০০ টাকা দুইবার credit হয়ে যাবে!

**সমাধান**: Inbox Pattern দিয়ে `payment_confirmation_id` track করি। একই ID দ্বিতীয়বার আসলে skip করি।

---

## 💻 PHP কোড উদাহরণ

### Inbox Service Implementation:

```php
<?php

namespace App\Services;

use App\Models\InboxMessage;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class InboxService
{
    /**
     * মেসেজ প্রসেস করার মূল মেথড
     * Idempotent - একই message_id বারবার দিলেও একবারই প্রসেস হবে
     */
    public function processMessage(string $messageId, string $eventType, array $payload): bool
    {
        // ১. আগে প্রসেস হয়েছে কিনা চেক করো
        if ($this->isAlreadyProcessed($messageId)) {
            Log::info("Duplicate message skipped", ['message_id' => $messageId]);
            return true; // ACK করো, আবার আসবে না
        }

        // ২. Transaction এ প্রসেস করো
        return DB::transaction(function () use ($messageId, $eventType, $payload) {
            // Double-check lock (race condition প্রতিরোধ)
            $exists = InboxMessage::where('message_id', $messageId)
                ->lockForUpdate()
                ->exists();

            if ($exists) {
                return true; // অন্য thread আগেই প্রসেস করেছে
            }

            // Inbox এ record করো
            $inboxMessage = InboxMessage::create([
                'message_id'   => $messageId,
                'event_type'   => $eventType,
                'payload'      => json_encode($payload),
                'status'       => 'processing',
                'received_at'  => now(),
            ]);

            // Business logic চালাও
            $this->handleEvent($eventType, $payload);

            // Mark as done
            $inboxMessage->update([
                'status'       => 'done',
                'processed_at' => now(),
            ]);

            return true;
        });
    }

    private function isAlreadyProcessed(string $messageId): bool
    {
        return InboxMessage::where('message_id', $messageId)
            ->where('status', 'done')
            ->exists();
    }

    private function handleEvent(string $eventType, array $payload): void
    {
        match ($eventType) {
            'payment.confirmed'    => $this->handlePaymentConfirmed($payload),
            'payment.failed'       => $this->handlePaymentFailed($payload),
            'refund.initiated'     => $this->handleRefundInitiated($payload),
            default                => throw new \RuntimeException("Unknown event: {$eventType}"),
        };
    }

    /**
     * bKash: ব্যাংক থেকে পেমেন্ট কনফার্মেশন আসলে wallet credit করো
     */
    private function handlePaymentConfirmed(array $payload): void
    {
        $walletId = $payload['wallet_id'];
        $amount = $payload['amount'];
        $transactionRef = $payload['bank_transaction_ref'];

        // Wallet balance update (same transaction এ)
        DB::table('wallets')
            ->where('id', $walletId)
            ->increment('balance', $amount);

        // Transaction log
        DB::table('transaction_logs')->insert([
            'wallet_id'     => $walletId,
            'amount'        => $amount,
            'type'          => 'credit',
            'reference'     => $transactionRef,
            'created_at'    => now(),
        ]);

        Log::info("Payment credited", [
            'wallet_id' => $walletId,
            'amount'    => $amount,
        ]);
    }

    private function handlePaymentFailed(array $payload): void
    {
        // Failed payment handling
        DB::table('failed_payments')->insert([
            'wallet_id'  => $payload['wallet_id'],
            'amount'     => $payload['amount'],
            'reason'     => $payload['failure_reason'],
            'created_at' => now(),
        ]);
    }

    private function handleRefundInitiated(array $payload): void
    {
        DB::table('wallets')
            ->where('id', $payload['wallet_id'])
            ->increment('balance', $payload['refund_amount']);
    }
}
```

### Kafka Consumer (Laravel):

```php
<?php

namespace App\Consumers;

use App\Services\InboxService;
use Junges\Kafka\Contracts\KafkaConsumerMessage;
use Illuminate\Support\Facades\Log;

class PaymentEventConsumer
{
    public function __construct(
        private InboxService $inboxService
    ) {}

    public function handle(KafkaConsumerMessage $message): void
    {
        $payload = $message->getBody();
        $headers = $message->getHeaders();

        // message_id হেডার থেকে নাও (অথবা payload থেকে)
        $messageId = $headers['message_id'] ?? $payload['event_id'];
        $eventType = $payload['event_type'];

        try {
            $processed = $this->inboxService->processMessage(
                $messageId,
                $eventType,
                $payload
            );

            if ($processed) {
                Log::info("Message processed successfully", [
                    'message_id' => $messageId
                ]);
            }
        } catch (\Exception $e) {
            Log::error("Message processing failed", [
                'message_id' => $messageId,
                'error'      => $e->getMessage(),
            ]);

            // Retry logic বা Dead Letter Queue তে পাঠাও
            throw $e;
        }
    }
}
```

### Migration:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('inbox_messages', function (Blueprint $table) {
            $table->string('message_id')->primary();
            $table->string('source')->nullable();
            $table->string('event_type');
            $table->json('payload');
            $table->enum('status', ['processing', 'done', 'failed'])
                  ->default('processing');
            $table->integer('retry_count')->default(0);
            $table->text('error_message')->nullable();
            $table->timestamp('received_at');
            $table->timestamp('processed_at')->nullable();
            $table->timestamps();

            $table->index(['event_type', 'status']);
            $table->index('received_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('inbox_messages');
    }
};
```

---

## 🟨 JavaScript কোড উদাহরণ

### Node.js Inbox Service:

```javascript
// services/InboxService.js
const { Sequelize, DataTypes, Op } = require('sequelize');
const sequelize = require('../config/database');

// Inbox Message Model
const InboxMessage = sequelize.define('InboxMessage', {
    message_id: {
        type: DataTypes.STRING,
        primaryKey: true,
    },
    source: DataTypes.STRING,
    event_type: DataTypes.STRING,
    payload: DataTypes.JSON,
    status: {
        type: DataTypes.ENUM('processing', 'done', 'failed'),
        defaultValue: 'processing',
    },
    retry_count: {
        type: DataTypes.INTEGER,
        defaultValue: 0,
    },
    error_message: DataTypes.TEXT,
    received_at: DataTypes.DATE,
    processed_at: DataTypes.DATE,
}, {
    tableName: 'inbox_messages',
    timestamps: true,
});

class InboxService {
    constructor(eventHandlers) {
        this.eventHandlers = eventHandlers;
    }

    /**
     * মেসেজ প্রসেস করো - Idempotent
     * একই message_id বারবার আসলেও সমস্যা নেই
     */
    async processMessage(messageId, eventType, payload) {
        // ১. দ্রুত চেক - আগেই প্রসেস হয়েছে?
        const existing = await InboxMessage.findByPk(messageId);
        if (existing && existing.status === 'done') {
            console.log(`⏭️ Duplicate message skipped: ${messageId}`);
            return { status: 'skipped', reason: 'already_processed' };
        }

        // ২. Transaction এ প্রসেস করো
        const transaction = await sequelize.transaction({
            isolationLevel: Sequelize.Transaction.ISOLATION_LEVELS.SERIALIZABLE,
        });

        try {
            // Double-check with lock
            const [inboxRecord, created] = await InboxMessage.findOrCreate({
                where: { message_id: messageId },
                defaults: {
                    event_type: eventType,
                    payload: payload,
                    status: 'processing',
                    received_at: new Date(),
                },
                transaction,
                lock: transaction.LOCK.UPDATE,
            });

            if (!created && inboxRecord.status === 'done') {
                await transaction.rollback();
                return { status: 'skipped', reason: 'already_processed' };
            }

            // Business logic চালাও
            const handler = this.eventHandlers[eventType];
            if (!handler) {
                throw new Error(`Unknown event type: ${eventType}`);
            }
            await handler(payload, transaction);

            // Mark as processed
            await inboxRecord.update({
                status: 'done',
                processed_at: new Date(),
            }, { transaction });

            await transaction.commit();

            console.log(`✅ Message processed: ${messageId}`);
            return { status: 'processed' };

        } catch (error) {
            await transaction.rollback();

            // Failed হিসেবে mark করো
            await InboxMessage.upsert({
                message_id: messageId,
                event_type: eventType,
                payload: payload,
                status: 'failed',
                retry_count: sequelize.literal('retry_count + 1'),
                error_message: error.message,
                received_at: new Date(),
            });

            console.error(`❌ Message failed: ${messageId}`, error.message);
            throw error;
        }
    }
}

module.exports = { InboxService, InboxMessage };
```

### Kafka Consumer (Node.js):

```javascript
// consumers/paymentConsumer.js
const { Kafka } = require('kafkajs');
const { InboxService } = require('../services/InboxService');
const { Wallet, TransactionLog } = require('../models');
const sequelize = require('../config/database');

// Event Handlers define করো
const eventHandlers = {
    'payment.confirmed': async (payload, transaction) => {
        const { wallet_id, amount, bank_transaction_ref } = payload;

        // bKash wallet এ টাকা credit করো
        await Wallet.increment('balance', {
            by: amount,
            where: { id: wallet_id },
            transaction,
        });

        // Transaction log রাখো
        await TransactionLog.create({
            wallet_id,
            amount,
            type: 'credit',
            reference: bank_transaction_ref,
            description: `Bank payment confirmed: ${bank_transaction_ref}`,
        }, { transaction });
    },

    'payment.failed': async (payload, transaction) => {
        const { wallet_id, amount, failure_reason } = payload;

        await sequelize.models.FailedPayment.create({
            wallet_id,
            amount,
            reason: failure_reason,
        }, { transaction });
    },
};

// Consumer সেটআপ
const kafka = new Kafka({
    clientId: 'bkash-payment-service',
    brokers: ['kafka-1:9092', 'kafka-2:9092'],
});

const consumer = kafka.consumer({ groupId: 'payment-processor' });
const inboxService = new InboxService(eventHandlers);

async function startConsumer() {
    await consumer.connect();
    await consumer.subscribe({
        topic: 'bank-payment-events',
        fromBeginning: false,
    });

    await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
            const messageId = message.headers['message_id']?.toString()
                || message.key?.toString();
            const payload = JSON.parse(message.value.toString());

            try {
                await inboxService.processMessage(
                    messageId,
                    payload.event_type,
                    payload
                );
            } catch (error) {
                console.error('Processing failed, will retry:', error.message);
                // Kafka auto-retry করবে (commit না করলে)
            }
        },
    });

    console.log('🚀 Payment consumer started');
}

startConsumer().catch(console.error);
```

---

## 🧹 Cleanup ও Dead Message Handling

### Cleanup Strategy:

```
┌─────────────────────────────────────────────────────────┐
│                  Inbox Cleanup Strategy                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────┐   7 দিন পর    ┌──────────┐               │
│  │  done   │ ─────────────► │  DELETE  │               │
│  └─────────┘                └──────────┘               │
│                                                          │
│  ┌─────────┐   3 retry পর   ┌──────────┐              │
│  │ failed  │ ─────────────► │   DLQ    │              │
│  └─────────┘                └──────────┘              │
│                                                          │
│  ┌──────────────┐  24 ঘণ্টা  ┌──────────┐             │
│  │ processing   │ ─────────► │  ALERT   │             │
│  │ (stuck)      │            └──────────┘             │
│  └──────────────┘                                      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### PHP Cleanup Command:

```php
<?php
// app/Console/Commands/CleanupInbox.php

namespace App\Console\Commands;

use App\Models\InboxMessage;
use Illuminate\Console\Command;

class CleanupInbox extends Command
{
    protected $signature = 'inbox:cleanup';
    protected $description = 'পুরানো inbox messages পরিষ্কার করো';

    public function handle(): void
    {
        // ৭ দিনের পুরানো সফল মেসেজ মুছে ফেলো
        $deleted = InboxMessage::where('status', 'done')
            ->where('processed_at', '<', now()->subDays(7))
            ->delete();

        $this->info("🗑️ {$deleted} পুরানো মেসেজ মুছে ফেলা হয়েছে");

        // Stuck মেসেজ alert
        $stuck = InboxMessage::where('status', 'processing')
            ->where('received_at', '<', now()->subHours(24))
            ->get();

        if ($stuck->isNotEmpty()) {
            $this->warn("⚠️ {$stuck->count()} মেসেজ stuck অবস্থায় আছে!");
            // Alert পাঠাও
        }

        // Max retry পার হওয়া মেসেজ DLQ তে পাঠাও
        $failed = InboxMessage::where('status', 'failed')
            ->where('retry_count', '>=', 3)
            ->get();

        foreach ($failed as $message) {
            $this->sendToDeadLetterQueue($message);
            $message->update(['status' => 'dead_lettered']);
        }
    }

    private function sendToDeadLetterQueue(InboxMessage $message): void
    {
        // DLQ implementation
    }
}
```

---

## 📋 Ordering Guarantees

### Ordering সমস্যা:

```
সময়  ──────────────────────────────────────────►

Message A (order: 1) ──────► প্রসেস শুরু ─── ─ ── ─ ── done (দেরি)
Message B (order: 2) ──────► প্রসেস শুরু ── done (আগে শেষ)

ফলাফল: B আগে শেষ হলো A এর আগে! 😱
```

### সমাধান - Sequence Number Tracking:

```javascript
// Ordered inbox processing
class OrderedInboxService {
    async processMessage(messageId, sequenceNumber, entityId, payload) {
        const lastProcessed = await this.getLastSequence(entityId);

        if (sequenceNumber <= lastProcessed) {
            // পুরানো মেসেজ, skip করো
            return { status: 'skipped', reason: 'out_of_order' };
        }

        if (sequenceNumber > lastProcessed + 1) {
            // Gap আছে, পরে প্রসেস হবে
            await this.storeForLater(messageId, sequenceNumber, entityId, payload);
            return { status: 'deferred', reason: 'gap_in_sequence' };
        }

        // সঠিক sequence, প্রসেস করো
        await this.processAndUpdateSequence(entityId, sequenceNumber, payload);

        // Deferred মেসেজ চেক করো
        await this.processDeferredMessages(entityId, sequenceNumber + 1);

        return { status: 'processed' };
    }
}
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন:

| পরিস্থিতি | উদাহরণ |
|-----------|---------|
| Financial transactions | bKash পেমেন্ট credit/debit |
| At-least-once delivery ব্যবহার করলে | Kafka, RabbitMQ consumers |
| Duplicate processing ক্ষতিকর হলে | SMS পাঠানো, Email পাঠানো |
| External system থেকে event আসলে | Bank gateway callbacks |
| Microservice event consumption | Order → Payment → Shipping events |

### ❌ কখন ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|-------|
| Naturally idempotent operations | SET balance = 500 (বারবার চালালেও একই) |
| Read-only operations | শুধু query করলে duplicate সমস্যা নেই |
| Low-value notifications | একটা extra notification এ ক্ষতি নেই |
| High throughput + low risk | Inbox table bottleneck হতে পারে |
| Exactly-once broker ব্যবহার করলে | Kafka Streams EOS mode |

### 💡 Best Practices:

1. **message_id সবসময় source system থেকে নিন** — নিজে generate করবেন না
2. **TTL রাখুন** — পুরানো inbox records মুছুন (7-30 দিন)
3. **Index রাখুন** — message_id তে unique index অবশ্যই
4. **Monitoring** — stuck messages alert সেটআপ করুন
5. **Dead Letter Queue** — max retry পর DLQ তে পাঠান
6. **Partition-level ordering** — Kafka তে same partition key ব্যবহার করুন

---

## 🎓 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────┐
│           Inbox Pattern Summary                   │
├─────────────────────────────────────────────────┤
│                                                  │
│  সমস্যা: Duplicate message consumption          │
│  সমাধান: message_id দিয়ে deduplication          │
│  গ্যারান্টি: Exactly-once processing             │
│  খরচ: Extra DB lookup প্রতি মেসেজে              │
│  জোড়া: Outbox (sender) + Inbox (receiver)       │
│                                                  │
└─────────────────────────────────────────────────┘
```
