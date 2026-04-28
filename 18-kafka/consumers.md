# 📘 Kafka কনজিউমার (Consumers Deep Dive)

> Kafka কনজিউমার হলো সেই কম্পোনেন্ট যেটা টপিক থেকে মেসেজ পড়ে এবং প্রসেস করে।
> এটা শুধু মেসেজ পড়া না — অফসেট ম্যানেজমেন্ট, রিব্যালান্সিং, এরর হ্যান্ডলিং সবকিছু বুঝতে হবে।

---

## 📊 কনজিউমার আর্কিটেকচার

### কনজিউমার কিভাবে কাজ করে?

Kafka কনজিউমার **pull-based** মডেল ব্যবহার করে। মানে ব্রোকার মেসেজ পুশ করে না,
বরং কনজিউমার নিজে থেকে ব্রোকারের কাছে গিয়ে মেসেজ টেনে আনে (poll)।

**কেন Pull-based?**
- কনজিউমার নিজের ক্ষমতা অনুযায়ী মেসেজ আনতে পারে
- স্লো কনজিউমার overwhelm হয় না
- ব্যাচ করে মেসেজ আনা যায় (efficiency বাড়ে)

### কনজিউমারের অভ্যন্তরীণ কম্পোনেন্ট

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kafka Consumer Instance                      │
│                                                                  │
│  ┌──────────────────┐   ┌─────────────────────┐                 │
│  │    Fetcher        │   │ ConsumerCoordinator  │                │
│  │                   │   │                      │                │
│  │ • ব্রোকার থেকে   │   │ • গ্রুপ জয়েন/লিভ    │                │
│  │   মেসেজ fetch    │   │ • পার্টিশন অ্যাসাইন  │                │
│  │ • ব্যাচ ম্যানেজ  │   │ • অফসেট কমিট        │                │
│  │ • ডিসিরিয়ালাইজ  │   │ • রিব্যালান্স ট্রিগার│                │
│  └──────────────────┘   └─────────────────────┘                 │
│                                                                  │
│  ┌──────────────────┐   ┌─────────────────────┐                 │
│  │  Heartbeat Thread │   │  SubscriptionState   │                │
│  │                   │   │                      │                │
│  │ • গ্রুপ কোঅর্ডি-  │   │ • অ্যাসাইনড          │                │
│  │   নেটরে হার্টবিট │   │   পার্টিশন ট্র্যাক   │                │
│  │ • সেশন টাইমআউট   │   │ • কারেন্ট অফসেট      │                │
│  │   ম্যানেজ        │   │ • কমিটেড অফসেট       │                │
│  └──────────────────┘   └─────────────────────┘                 │
│                                                                  │
│  ┌──────────────────────────────────────────────┐               │
│  │              NetworkClient                    │               │
│  │  • TCP কানেকশন ম্যানেজ                       │               │
│  │  • ব্রোকারদের সাথে কমিউনিকেশন                │               │
│  └──────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │ Broker 1 │        │ Broker 2 │        │ Broker 3 │
   └──────────┘        └──────────┘        └──────────┘
```

**Fetcher:** ব্রোকার থেকে মেসেজ এনে ইন্টারনাল বাফারে রাখে। `fetch.min.bytes`,
`fetch.max.wait.ms`, `max.partition.fetch.bytes` দিয়ে কনফিগার করা যায়।

**ConsumerCoordinator:** গ্রুপ কোঅর্ডিনেটরের সাথে যোগাযোগ করে। জয়েন গ্রুপ,
সিঙ্ক গ্রুপ, অফসেট কমিট — সব এর দায়িত্ব।

**Heartbeat Thread:** ব্যাকগ্রাউন্ডে আলাদা থ্রেড চলে। নির্দিষ্ট সময় পর পর
গ্রুপ কোঅর্ডিনেটরে heartbeat পাঠায়, যাতে ব্রোকার জানে কনজিউমার এখনো alive।

---

## 👥 কনজিউমার গ্রুপ (Consumer Groups) Deep Dive

### কনজিউমার গ্রুপ কী?

একই `group.id` শেয়ার করা কনজিউমারদের একটা সেট। একটা টপিকের প্রতিটা পার্টিশন
একটা গ্রুপের মধ্যে **শুধুমাত্র একটা** কনজিউমারকে অ্যাসাইন হয়।

**বাস্তব উদাহরণ — Daraz:**
- `order-events` টপিক → ৩টা পার্টিশন
- Order Service গ্রুপ → ৩ কনজিউমার (প্রতিটা ১টা পার্টিশন পায়)
- Analytics গ্রুপ → ১ কনজিউমার (সব ৩টা পার্টিশন পায়)

### একই টপিক, একাধিক কনজিউমার গ্রুপ

```
              Topic: order-events (3 Partitions)
     ┌──────────────┬──────────────┬──────────────┐
     │ Partition 0   │ Partition 1   │ Partition 2   │
     │ [0][1][2][3]  │ [0][1][2][3]  │ [0][1][2][3]  │
     └──────┬───────┴──────┬───────┴──────┬───────┘
            │              │              │
     ┌──────┼──────────────┼──────────────┼────────────────┐
     │      ▼              ▼              ▼                │
     │  Consumer Group A (group.id = "order-service")      │
     │                                                      │
     │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
     │  │Consumer 1│  │Consumer 2│  │Consumer 3│          │
     │  │  ← P0    │  │  ← P1    │  │  ← P2    │          │
     │  └──────────┘  └──────────┘  └──────────┘          │
     └──────────────────────────────────────────────────────┘

     ┌──────┬──────────────┬──────────────┬────────────────┐
     │      ▼              ▼              ▼                │
     │  Consumer Group B (group.id = "analytics-service")  │
     │                                                      │
     │  ┌──────────────────────────────────┐               │
     │  │          Consumer 1              │               │
     │  │       ← P0, P1, P2              │               │
     │  └──────────────────────────────────┘               │
     └──────────────────────────────────────────────────────┘
```

### গ্রুপ কোঅর্ডিনেটর (Group Coordinator)

প্রতিটা কনজিউমার গ্রুপের জন্য একটা ব্রোকার **গ্রুপ কোঅর্ডিনেটর** হিসেবে কাজ করে।
কোন ব্রোকার কোঅর্ডিনেটর হবে সেটা `group.id` এর hash দিয়ে নির্ধারণ হয়।

**কোঅর্ডিনেটরের দায়িত্ব:**
- নতুন কনজিউমার জয়েন হলে রিব্যালান্সিং শুরু করা
- কনজিউমার ছেড়ে গেলে (heartbeat মিস) রিব্যালান্সিং
- লিডার কনজিউমার সিলেক্ট করা (যে পার্টিশন অ্যাসাইনমেন্ট করবে)
- অফসেট কমিট সংরক্ষণ করা

### কনজিউমার জয়েন/লিভ ফ্লো

```
নতুন কনজিউমার জয়েন হলে:
═══════════════════════════

Consumer-3 (নতুন)          Group Coordinator           Consumer-1, 2 (আগের)
     │                          │                            │
     │──── JoinGroup ──────────>│                            │
     │                          │──── Rebalance Trigger ────>│
     │                          │                            │
     │                          │<─── JoinGroup ────────────│
     │<─── JoinGroup Response ──│                            │
     │     (leader নির্বাচিত)    │                            │
     │                          │                            │
     │──── SyncGroup ──────────>│  (লিডার অ্যাসাইনমেন্ট    │
     │     (অ্যাসাইনমেন্ট)      │   পাঠায়)                  │
     │                          │──── SyncGroup Response ───>│
     │<─── SyncGroup Response ──│                            │
     │     (P2 পেলাম!)          │                            │
     │                          │                            │

কনজিউমার ছেড়ে গেলে:
═══════════════════════

Consumer-2 (crash!)        Group Coordinator           Consumer-1, 3
     │                          │                            │
     ✗ (heartbeat মিস)         │                            │
                                │                            │
     session.timeout.ms পরে:    │                            │
                                │──── Rebalance Trigger ────>│
                                │                            │
                                │<─── JoinGroup ────────────│
                                │──── SyncGroup Response ───>│
                                │     (P0→C1, P1→C3, P2→C3) │
```

---

## 📦 পার্টিশন অ্যাসাইনমেন্ট কৌশল

পার্টিশন কোন কনজিউমারের কাছে যাবে সেটা **অ্যাসাইনমেন্ট স্ট্র্যাটেজি** ঠিক করে।

### 1️⃣ Range Assignor (ডিফল্ট)

প্রতিটা টপিকের পার্টিশনগুলো সাজিয়ে সমানভাবে ভাগ করে।

```
উদাহরণ: Daraz — ২টা টপিক, ৩ কনজিউমার
═══════════════════════════════════════

Topic: order-events  (P0, P1, P2)
Topic: payment-events (P0, P1, P2)

Consumers: C0, C1, C2 (বর্ণানুক্রমে সাজানো)

Range Assignor ফলাফল:
─────────────────────
order-events:    C0 ← P0  │  C1 ← P1  │  C2 ← P2
payment-events:  C0 ← P0  │  C1 ← P1  │  C2 ← P2

⚠️ সমস্যা — যখন সমান ভাগ হয় না:
Topic: order-events  (P0, P1, P2)    → 3 partitions / 2 consumers
Consumers: C0, C1

C0 ← P0, P1  (২টা!)
C1 ← P2      (১টা)

C0 এর উপর বেশি লোড পড়ে!
```

### 2️⃣ RoundRobin Assignor

সব টপিকের সব পার্টিশন একসাথে মিশিয়ে পালা করে ভাগ করে।

```
উদাহরণ: ৩ টপিক, ২ কনজিউমার
══════════════════════════════

সব পার্টিশন সাজানো:
order-P0, order-P1, order-P2, payment-P0, payment-P1, shipping-P0

RoundRobin বিতরণ:
──────────────────
C0 ← order-P0, order-P2, payment-P1
C1 ← order-P1, payment-P0, shipping-P0

✅ Range এর চেয়ে বেশি সমান বিতরণ
⚠️ রিব্যালান্সে অনেক পার্টিশন বদলায়
```

### 3️⃣ Sticky Assignor

RoundRobin এর মতো সমান ভাগ করে, কিন্তু রিব্যালান্সে **আগের অ্যাসাইনমেন্ট যতটুকু
পারা যায় ধরে রাখে**।

```
প্রথমবার (RoundRobin এর মতো):
C0 ← order-P0, payment-P0
C1 ← order-P1, payment-P1
C2 ← order-P2

C2 চলে গেলে — Sticky Assignor:
────────────────────────────────
C0 ← order-P0, payment-P0, order-P2  (শুধু order-P2 নতুন যোগ হলো)
C1 ← order-P1, payment-P1            (কোনো পরিবর্তন নেই!)

C2 চলে গেলে — RoundRobin Assignor:
─────────────────────────────────────
C0 ← order-P0, order-P2, payment-P1  (payment-P1 নতুন, payment-P0 চলে গেল!)
C1 ← order-P1, payment-P0            (payment-P0 নতুন, payment-P1 চলে গেল!)

✅ Sticky: শুধু ১টা পার্টিশন মুভ হলো
❌ RoundRobin: ৩টা পার্টিশন মুভ হলো (অপ্রয়োজনীয়!)
```

### 4️⃣ CooperativeSticky Assignor

Sticky Assignor-এর মতোই, কিন্তু **Cooperative Rebalancing** সাপোর্ট করে।
রিব্যালান্সের সময় সব পার্টিশন revoke না করে শুধু যেগুলো মুভ হবে সেগুলো revoke করে।

```
Eager Rebalancing (Range/RoundRobin/Sticky):
─────────────────────────────────────────────
Step 1: সব কনজিউমার → সব পার্টিশন ছেড়ে দাও! (STOP-THE-WORLD ⛔)
Step 2: নতুন অ্যাসাইনমেন্ট গণনা
Step 3: নতুন পার্টিশন নাও

Cooperative Rebalancing (CooperativeSticky):
────────────────────────────────────────────
Step 1: শুধু যে পার্টিশনগুলো মুভ হবে সেগুলো ছেড়ে দাও
Step 2: বাকি পার্টিশন থেকে মেসেজ পড়া চলতে থাকে! ✅
Step 3: মুভ হওয়া পার্টিশন নতুন কনজিউমারে যায়
```

### তুলনা টেবিল

| বৈশিষ্ট্য | Range | RoundRobin | Sticky | CooperativeSticky |
|---|---|---|---|---|
| সমান বিতরণ | ❌ অসমান হতে পারে | ✅ সমান | ✅ সমান | ✅ সমান |
| রিব্যালান্স মুভমেন্ট | বেশি | বেশি | কম | কম |
| Cooperative সাপোর্ট | ❌ | ❌ | ❌ | ✅ |
| ডাউনটাইম | বেশি | বেশি | বেশি | কম |
| প্রোডাকশনে সুপারিশ | ❌ | ❌ | মাঝামাঝি | ✅ সেরা |

**কখন কোনটা ব্যবহার করবেন:**
- **ছোট প্রজেক্ট / শেখার জন্য:** Range (ডিফল্ট)
- **প্রোডাকশন (Kafka ≥ 2.4):** CooperativeSticky
- **প্রোডাকশন (পুরনো Kafka):** Sticky

---

## 🔄 অফসেট ম্যানেজমেন্ট

### অফসেট কী?

প্রতিটা পার্টিশনে মেসেজের একটা ক্রমিক নম্বর থাকে — এটাই **অফসেট**।
কনজিউমার কোন মেসেজ পর্যন্ত পড়েছে সেটা ট্র্যাক করতে অফসেট কমিট করে।

### __consumer_offsets টপিক

Kafka একটা ইন্টারনাল টপিক `__consumer_offsets` এ কমিটেড অফসেট সংরক্ষণ করে।
এটা ৫০টা পার্টিশনের একটা compacted টপিক।

```
__consumer_offsets টপিকে যা সংরক্ষিত হয়:
──────────────────────────────────────────
Key:   (group.id, topic, partition)
Value: (offset, metadata, timestamp)

উদাহরণ:
Key:   ("order-service", "order-events", 0)
Value: (offset: 42, metadata: "", timestamp: 1700000000)
```

### অফসেট ফ্লো ডায়াগ্রাম

```
Consumer                    Kafka Broker               __consumer_offsets
   │                            │                            │
   │──── poll() ───────────────>│                            │
   │<─── messages (offset 5-9) ─│                            │
   │                            │                            │
   │  (মেসেজ প্রসেস করা)        │                            │
   │                            │                            │
   │──── commitSync(offset=10) ─>│                            │
   │                            │──── store(grp,topic,p,10) ─>│
   │                            │<─── ack ──────────────────│
   │<─── commit success ────────│                            │
   │                            │                            │
   │  (ক্র্যাশ ও রিস্টার্ট!)    │                            │
   │                            │                            │
   │──── poll() ───────────────>│                            │
   │                            │──── fetch offset ─────────>│
   │                            │<─── offset = 10 ──────────│
   │<─── messages (offset 10+) ─│   (১০ থেকে শুরু, ডুপ্লিকেট নেই!)
```

### ❌ খারাপ: Auto Commit (ঝুঁকিপূর্ণ)

```php
// ❌ PHP — Auto Commit চালু (ডিফল্ট)
// সমস্যা: মেসেজ প্রসেস না হলেও কমিট হয়ে যেতে পারে!

$conf = new RdKafka\Conf();
$conf->set('group.id', 'order-service');
$conf->set('enable.auto.commit', 'true');          // ❌ ঝুঁকিপূর্ণ
$conf->set('auto.commit.interval.ms', '5000');

$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['order-events']);

while (true) {
    $message = $consumer->consume(1000);

    if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
        // ⚠️ এখানে প্রসেসিং শুরু হলো
        processOrder($message->payload);
        // ⚠️ যদি এখানে ক্র্যাশ হয়?
        // Auto commit আগেই হয়ে যেতে পারে!
        // মেসেজ হারিয়ে যাবে!
    }
}
```

```javascript
// ❌ JS — Auto Commit চালু (ডিফল্ট kafkajs)
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });
const consumer = kafka.consumer({ groupId: 'order-service' });

await consumer.connect();
await consumer.subscribe({ topic: 'order-events' });

await consumer.run({
    // ❌ autoCommit ডিফল্ট true — ঝুঁকিপূর্ণ!
    eachMessage: async ({ topic, partition, message }) => {
        await processOrder(message.value.toString());
        // ক্র্যাশ হলে মেসেজ হারাবে!
    }
});
```

### ✅ ভালো: Manual Commit (নিরাপদ)

```php
// ✅ PHP — Manual Commit
$conf = new RdKafka\Conf();
$conf->set('group.id', 'order-service');
$conf->set('enable.auto.commit', 'false');  // ✅ Auto commit বন্ধ

$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['order-events']);

while (true) {
    $message = $consumer->consume(1000);

    if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
        try {
            processOrder($message->payload);

            // ✅ প্রসেসিং সফল হলেই কমিট
            $consumer->commit($message);

        } catch (Exception $e) {
            error_log("Order processing failed: " . $e->getMessage());
            // কমিট করা হলো না — পরে আবার পাবে!
        }
    }
}
```

```javascript
// ✅ JS — Manual Commit
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });
const consumer = kafka.consumer({ groupId: 'order-service' });

await consumer.connect();
await consumer.subscribe({ topic: 'order-events' });

await consumer.run({
    autoCommit: false,  // ✅ Auto commit বন্ধ
    eachMessage: async ({ topic, partition, message }) => {
        try {
            await processOrder(message.value.toString());

            // ✅ প্রসেসিং শেষ হলেই কমিট
            await consumer.commitOffsets([{
                topic,
                partition,
                offset: (Number(message.offset) + 1).toString()
            }]);

        } catch (error) {
            console.error('Processing failed:', error);
            // কমিট হবে না — পরবর্তী poll এ আবার পাবে
        }
    }
});
```

### commitSync vs commitAsync

| বৈশিষ্ট্য | commitSync | commitAsync |
|---|---|---|
| ব্লকিং | ✅ হ্যাঁ, কমিট সফল না হওয়া পর্যন্ত অপেক্ষা | ❌ না, ব্যাকগ্রাউন্ডে কমিট |
| গ্যারান্টি | ✅ শক্তিশালী | ⚠️ দুর্বল (ফেইল মিস হতে পারে) |
| পারফরম্যান্স | ধীর | দ্রুত |
| ব্যবহার | ক্রিটিকাল ডাটা (bKash পেমেন্ট) | নন-ক্রিটিকাল (লগিং, অ্যানালিটিক্স) |

---

## ⚡ রিব্যালান্সিং (Rebalancing)

### রিব্যালান্সিং কী?

কনজিউমার গ্রুপে পার্টিশন পুনর্বণ্টন করার প্রক্রিয়া। এটা হলে কনজিউমাররা
সাময়িকভাবে মেসেজ পড়া বন্ধ করে।

### কখন রিব্যালান্সিং হয়?

```
রিব্যালান্সিং ট্রিগার:
═══════════════════════

1. নতুন কনজিউমার জয়েন     → Pathao তে নতুন সার্ভার যোগ হলো
2. কনজিউমার ছেড়ে গেল       → সার্ভার ক্র্যাশ / শাটডাউন
3. Heartbeat মিস            → session.timeout.ms এর মধ্যে heartbeat আসেনি
4. poll() দেরি              → max.poll.interval.ms এর মধ্যে poll হয়নি
5. টপিকে পার্টিশন বাড়লো    → নতুন পার্টিশন ভাগ করতে হবে
6. Subscription পরিবর্তন    → কনজিউমার নতুন টপিক সাবস্ক্রাইব করলো
```

### Eager Rebalancing (Stop-the-World) ⛔

```
Eager Rebalancing (পুরনো পদ্ধতি):
═══════════════════════════════════

সময় ──────────────────────────────────────────────────────>

C1: [P0 পড়ছি][P1 পড়ছি]──STOP!──────────────[P0 পড়ছি]────>
C2: [P2 পড়ছি]────────────STOP!──────────────[P1,P2 পড়ছি]──>
C3:                     JOIN!─────────────────[ কিছু নেই ]──>
                              │             │
                              │  রিব্যালান্স │
                              │  (সব বন্ধ!) │
                              │             │
                              ▼             ▼
                        সব পার্টিশন    নতুন অ্যাসাইনমেন্ট
                        revoke          শুরু

⚠️ সমস্যা: পুরো গ্রুপ সাময়িকভাবে মেসেজ পড়া বন্ধ করে!
```

### Cooperative/Incremental Rebalancing ✅

```
Cooperative Rebalancing (নতুন পদ্ধতি):
════════════════════════════════════════

সময় ──────────────────────────────────────────────────────>

C1: [P0 পড়ছি][P1 পড়ছি]─[P0 পড়ছি]──────────[P0 পড়ছি]───>
C2: [P2 পড়ছি]───────────[P2 পড়ছি]──────────[P2 পড়ছি]───>
C3:                     JOIN!──────────────[P1 পড়ছি]──────>
                              │           │
                              │ শুধু P1    │
                              │ revoke    │
                              ▼           ▼
                         C1 থেকে P1    C3 তে P1
                         সরানো হলো     দেওয়া হলো

✅ C1 এর P0 এবং C2 এর P2 — কোনো বিরতি ছাড়াই চলতে থাকে!
```

### গুরুত্বপূর্ণ কনফিগারেশন

```
session.timeout.ms (ডিফল্ট: 45000)
────────────────────────────────────
এই সময়ের মধ্যে heartbeat না আসলে কনজিউমার মৃত ধরা হয়।
ছোট মান = দ্রুত failure detection, কিন্তু false positive বেশি।

heartbeat.interval.ms (ডিফল্ট: 3000)
─────────────────────────────────────
কত সময় পর পর heartbeat পাঠাবে।
সাধারণত session.timeout.ms এর ১/৩।

max.poll.interval.ms (ডিফল্ট: 300000 = 5 মিনিট)
──────────────────────────────────────────────────
দুটো poll() এর মধ্যে সর্বোচ্চ সময়।
এর বেশি সময় লাগলে কনজিউমারকে "stuck" ধরে রিব্যালান্স হয়।

bKash উদাহরণ: পেমেন্ট প্রসেসিং ৩০ সেকেন্ডের বেশি লাগে না,
তাই max.poll.interval.ms = 60000 (১ মিনিট) যথেষ্ট।
```

### Static Group Membership

সাধারণত কনজিউমার রিস্টার্ট হলে নতুন member হিসেবে জয়েন করে → রিব্যালান্স হয়।
`group.instance.id` সেট করলে কনজিউমার **static member** হয়।

```
Static Group Membership:
════════════════════════

group.instance.id = "order-consumer-1"

রিস্টার্ট হলে → session.timeout.ms এর মধ্যে ফিরে আসলে
              → রিব্যালান্স হবে না! ✅
              → আগের পার্টিশনগুলো ফিরে পাবে

ব্যবহারক্ষেত্র: Kubernetes এ rolling restart, Daraz এর
deploy এর সময় রিব্যালান্স এড়াতে।
```

---

## 📊 কনজিউমার ল্যাগ

### কনজিউমার ল্যাগ কী?

প্রোডিউসার যতগুলো মেসেজ পাঠিয়েছে আর কনজিউমার যতগুলো প্রসেস করেছে তার পার্থক্য।

```
Partition 0:
═══════════

Log End Offset (LEO):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
                      ↑                   ↑
                Consumer Offset        Latest
                (position: 5)        (offset: 9)

                Lag = 9 - 5 = 4 messages
                ◄──── ল্যাগ ────►

Partition 1:
┌───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │
└───┴───┴───┴───┴───┴───┴───┘
                          ↑   ↑
                    Consumer  Latest
                    (pos: 5) (off: 6)

                    Lag = 6 - 5 = 1 message ✅ ভালো!

Partition 2:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
          ↑                                       ↑
      Consumer                                 Latest
      (pos: 1)                                (off: 11)

      Lag = 11 - 1 = 10 messages ⚠️ সমস্যা!
```

### ল্যাগ মনিটরিং

```bash
# কনজিউমার গ্রুপের ল্যাগ দেখা
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group order-service --describe

# আউটপুট:
# GROUP          TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-service  order-events   0          542             546             4
# order-service  order-events   1          321             322             1
# order-service  order-events   2          198             208             10  ⚠️
```

**ল্যাগ বাড়লে কী করবেন:**
1. কনজিউমার সংখ্যা বাড়ান (পার্টিশন সংখ্যা পর্যন্ত)
2. প্রসেসিং অপটিমাইজ করুন (ব্যাচ প্রসেসিং, async I/O)
3. `max.poll.records` বাড়ান (ডিফল্ট ৫০০)
4. `fetch.min.bytes` বাড়ান (বড় ব্যাচে fetch)

---

## 🔒 ডেলিভারি সিমান্টিক্স

### At-Most-Once (সর্বাধিক একবার)

কমিট আগে, প্রসেস পরে। মেসেজ হারাতে পারে কিন্তু ডুপ্লিকেট হবে না।

```
At-Most-Once ফ্লো:
═══════════════════

Consumer          Kafka
   │                │
   │── poll() ─────>│
   │<── msg(5) ─────│
   │                │
   │── commit(6) ──>│  ← আগে কমিট! ✅
   │                │
   │  process(msg)  │  ← এখন প্রসেস
   │  ✗ CRASH! ✗    │  ← ক্র্যাশ!
   │                │
   │  (রিস্টার্ট)    │
   │── poll() ─────>│
   │<── msg(6) ─────│  ← msg(5) হারিয়ে গেল! ❌

Daraz উদাহরণ: অর্ডার নোটিফিকেশন — কিছু নোটিফিকেশন মিস হলে
বড় সমস্যা না। কিন্তু ডুপ্লিকেট নোটিফিকেশন বিরক্তিকর।
```

### At-Least-Once (কমপক্ষে একবার) — সবচেয়ে প্রচলিত

প্রসেস আগে, কমিট পরে। মেসেজ হারাবে না কিন্তু ডুপ্লিকেট হতে পারে।

```
At-Least-Once ফ্লো:
════════════════════

Consumer          Kafka
   │                │
   │── poll() ─────>│
   │<── msg(5) ─────│
   │                │
   │  process(msg)  │  ← আগে প্রসেস ✅
   │  ✗ CRASH! ✗    │  ← কমিটের আগে ক্র্যাশ!
   │                │
   │  (রিস্টার্ট)    │
   │── poll() ─────>│
   │<── msg(5) ─────│  ← আবার msg(5) পাবে! (ডুপ্লিকেট)
   │                │
   │  process(msg)  │  ← আবার প্রসেস
   │── commit(6) ──>│  ← এবার কমিট সফল ✅

Daraz উদাহরণ: অর্ডার প্রসেসিং — অর্ডার হারানোর চেয়ে
ডুপ্লিকেট অর্ডার ভালো (idempotent check দিয়ে ধরা যায়)।
```

### Exactly-Once (ঠিক একবার)

Transactional consumer + idempotent processing দিয়ে গ্যারান্টি দেওয়া হয়।

```
Exactly-Once ফ্লো:
═══════════════════

Consumer                    Kafka
   │                          │
   │── poll() ───────────────>│
   │<── msg(5) ───────────────│
   │                          │
   │── beginTransaction() ───>│
   │   process(msg)           │
   │   produce(result)        │  ← প্রসেসিং ফলাফল অন্য টপিকে
   │── commitTransaction() ──>│  ← অফসেট + প্রডিউস একসাথে কমিট
   │                          │
   │  ক্র্যাশ হলে:              │
   │  Transaction abort হবে    │
   │  অফসেট আগের জায়গায়        │
   │  প্রডিউস করা মেসেজও       │
   │  abort হবে                │

bKash উদাহরণ: টাকা ট্রান্সফার — ১০০ টাকা পাঠালে ঠিক ১০০ টাকাই
কাটবে, ডুপ্লিকেট কাটবে না, হারাবেও না। isolation.level=read_committed।
```

### তুলনা টেবিল

| সিমান্টিক | মেসেজ হারানো | ডুপ্লিকেট | পারফরম্যান্স | ব্যবহারক্ষেত্র |
|---|---|---|---|---|
| At-Most-Once | ✅ সম্ভব | ❌ না | 🚀 সেরা | লগিং, মেট্রিক্স |
| At-Least-Once | ❌ না | ✅ সম্ভব | ⚡ ভালো | অর্ডার, ইভেন্ট (idempotent সহ) |
| Exactly-Once | ❌ না | ❌ না | 🐢 ধীর | পেমেন্ট, ফাইন্যান্স |

---

## ⚠️ এরর হ্যান্ডলিং ও DLQ

### মেসেজ প্রসেসিং ফেইল হলে কী করবেন?

```
মেসেজ ফ্লো ও এরর হ্যান্ডলিং:
═══════════════════════════════

                        ┌────────────┐
Main Topic ────────────>│  Consumer   │
(order-events)          └─────┬──────┘
                              │
                         [Process]
                              │
                    ┌─────────┴──────────┐
                    │                    │
                 Success?             Failure?
                    │                    │
                    ▼                    ▼
               ┌─────────┐      ┌──────────────┐
               │  Commit  │      │ Retry Topic  │
               └─────────┘      │(order-retry-1)│
                                └──────┬───────┘
                                       │
                                  ┌────┴─────┐
                                  │  Retry   │
                                  │ Consumer │
                                  └────┬─────┘
                                       │
                              ┌────────┴────────┐
                              │                 │
                           Success?          Failure?
                              │                 │
                              ▼                 ▼
                         ┌─────────┐    ┌──────────────┐
                         │  Commit  │    │  DLQ Topic   │
                         └─────────┘    │(order-dlq)   │
                                        └──────────────┘
                                        (পরে ম্যানুয়ালি
                                         investigate করা হবে)
```

### Retry কৌশল

```
1. Immediate Retry (তাৎক্ষণিক):
   ফেইল → সাথে সাথে আবার চেষ্টা → ৩ বার পর DLQ

2. Fixed Backoff:
   ফেইল → ৫ সেকেন্ড অপেক্ষা → আবার চেষ্টা → ৫ সেকেন্ড → DLQ

3. Exponential Backoff (সেরা):
   ফেইল → ১s অপেক্ষা → ফেইল → ২s → ফেইল → ৪s → ফেইল → ৮s → DLQ

   retry-1 topic (1s delay)
   retry-2 topic (5s delay)
   retry-3 topic (30s delay)
   dlq topic (ম্যানুয়াল)
```

### Poison Pill মেসেজ

এমন মেসেজ যেটা কখনোই সফলভাবে প্রসেস হবে না (corrupt data, schema mismatch ইত্যাদি)।
Retry করেও লাভ নেই, সরাসরি DLQ তে পাঠাতে হবে।

### PHP DLQ Implementation

```php
// PHP — DLQ প্যাটার্ন (Daraz অর্ডার প্রসেসিং)
$conf = new RdKafka\Conf();
$conf->set('group.id', 'order-service');
$conf->set('enable.auto.commit', 'false');

$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['order-events']);

// DLQ প্রোডিউসার
$dlqProducer = new RdKafka\Producer($producerConf);
$dlqTopic = $dlqProducer->newTopic('order-events-dlq');

$maxRetries = 3;

while (true) {
    $message = $consumer->consume(1000);

    if ($message->err !== RD_KAFKA_RESP_ERR_NO_ERROR) {
        continue;
    }

    $retryCount = 0;
    $processed = false;

    while ($retryCount < $maxRetries && !$processed) {
        try {
            $order = json_decode($message->payload, true);
            processOrder($order);
            $processed = true;
        } catch (RetryableException $e) {
            $retryCount++;
            $delay = pow(2, $retryCount);  // Exponential backoff
            sleep($delay);
        } catch (NonRetryableException $e) {
            break;  // Poison pill — সরাসরি DLQ
        }
    }

    if ($processed) {
        $consumer->commit($message);
    } else {
        // DLQ তে পাঠাও
        $dlqPayload = json_encode([
            'original_message' => $message->payload,
            'error' => $e->getMessage(),
            'retry_count' => $retryCount,
            'failed_at' => date('c'),
            'original_topic' => $message->topic_name,
            'original_partition' => $message->partition,
            'original_offset' => $message->offset,
        ]);
        $dlqTopic->produce(RD_KAFKA_PARTITION_UA, 0, $dlqPayload);
        $dlqProducer->flush(5000);
        $consumer->commit($message);  // DLQ তে গেছে, মেইন থেকে skip
    }
}
```

### JavaScript DLQ Implementation

```javascript
// JS — DLQ প্যাটার্ন (Pathao রাইড ইভেন্ট)
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });
const consumer = kafka.consumer({ groupId: 'ride-service' });
const dlqProducer = kafka.producer();

const MAX_RETRIES = 3;

async function processWithRetry(message, topic, partition) {
    let retryCount = 0;

    while (retryCount < MAX_RETRIES) {
        try {
            const ride = JSON.parse(message.value.toString());
            await processRideEvent(ride);
            return true;
        } catch (error) {
            if (!error.retryable) {
                break;  // Poison pill
            }
            retryCount++;
            const delay = Math.pow(2, retryCount) * 1000;
            await new Promise(r => setTimeout(r, delay));
        }
    }

    // DLQ তে পাঠাও
    await dlqProducer.send({
        topic: `${topic}-dlq`,
        messages: [{
            key: message.key,
            value: JSON.stringify({
                originalMessage: message.value.toString(),
                error: 'Max retries exceeded',
                retryCount,
                failedAt: new Date().toISOString(),
                originalTopic: topic,
                originalPartition: partition,
                originalOffset: message.offset,
            }),
            headers: { 'x-original-topic': topic }
        }]
    });

    return false;
}

await consumer.connect();
await dlqProducer.connect();
await consumer.subscribe({ topic: 'ride-events' });

await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
        await processWithRetry(message, topic, partition);
        await consumer.commitOffsets([{
            topic,
            partition,
            offset: (Number(message.offset) + 1).toString()
        }]);
    }
});
```

---

## 💻 PHP কনজিউমার সম্পূর্ণ উদাহরণ

```php
<?php
/**
 * Daraz অর্ডার প্রসেসিং কনজিউমার
 * rdkafka এক্সটেনশন ব্যবহার করে
 *
 * প্রোডাকশন-রেডি কনফিগারেশন সহ
 */

$conf = new RdKafka\Conf();

// ব্রোকার কানেকশন
$conf->set('metadata.broker.list', 'kafka1:9092,kafka2:9092,kafka3:9092');

// কনজিউমার গ্রুপ
$conf->set('group.id', 'daraz-order-service');
$conf->set('group.instance.id', 'order-consumer-' . gethostname());

// অফসেট কনফিগারেশন
$conf->set('enable.auto.commit', 'false');
$conf->set('auto.offset.reset', 'earliest');

// পার্টিশন অ্যাসাইনমেন্ট
$conf->set('partition.assignment.strategy', 'cooperative-sticky');

// পারফরম্যান্স টিউনিং
$conf->set('fetch.min.bytes', '1024');
$conf->set('fetch.wait.max.ms', '500');
$conf->set('max.poll.interval.ms', '60000');
$conf->set('session.timeout.ms', '30000');
$conf->set('heartbeat.interval.ms', '10000');

// রিব্যালান্স কলব্যাক
$conf->setRebalanceCb(function (
    RdKafka\KafkaConsumer $kafka,
    $err,
    array $partitions = null
) {
    switch ($err) {
        case RD_KAFKA_RESP_ERR__ASSIGN_PARTITIONS:
            echo "পার্টিশন অ্যাসাইন হলো: ";
            foreach ($partitions as $p) {
                echo "{$p->getTopic()}-{$p->getPartition()} ";
            }
            echo "\n";
            $kafka->assign($partitions);
            break;

        case RD_KAFKA_RESP_ERR__REVOKE_PARTITIONS:
            echo "পার্টিশন revoke হলো\n";
            $kafka->assign(null);
            break;
    }
});

$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['order-events']);

echo "✅ Daraz Order Consumer চালু হলো...\n";

// Graceful shutdown
$running = true;
pcntl_signal(SIGTERM, function () use (&$running) {
    echo "🛑 Shutdown signal পাওয়া গেছে...\n";
    $running = false;
});

while ($running) {
    pcntl_signal_dispatch();
    $message = $consumer->consume(1000);

    switch ($message->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            try {
                $order = json_decode($message->payload, true);

                echo sprintf(
                    "📦 অর্ডার পাওয়া গেছে: #%s | ক্রেতা: %s | মোট: ৳%s\n",
                    $order['order_id'],
                    $order['customer_name'],
                    $order['total_amount']
                );

                // অর্ডার প্রসেস
                processOrder($order);

                // সফল — অফসেট কমিট
                $consumer->commit($message);

                echo "✅ অর্ডার #{$order['order_id']} সফলভাবে প্রসেস হয়েছে\n";

            } catch (Exception $e) {
                echo "❌ এরর: {$e->getMessage()}\n";
                // কমিট না করলে পরে আবার পাবে (at-least-once)
            }
            break;

        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            // পার্টিশনের শেষে পৌঁছেছে — স্বাভাবিক
            break;

        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            // poll timeout — স্বাভাবিক, নতুন মেসেজ নেই
            break;

        default:
            echo "⚠️ Kafka Error: {$message->errstr()}\n";
            break;
    }
}

$consumer->close();
echo "🛑 Consumer বন্ধ হয়ে গেছে।\n";
```

---

## 💻 JavaScript কনজিউমার সম্পূর্ণ উদাহরণ

```javascript
/**
 * Pathao রাইড ট্র্যাকিং কনজিউমার
 * kafkajs লাইব্রেরি ব্যবহার করে
 *
 * Manual commit, consumer group, error handling সহ
 */
const { Kafka, logLevel } = require('kafkajs');

// Kafka ক্লায়েন্ট তৈরি
const kafka = new Kafka({
    clientId: 'pathao-ride-tracker',
    brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
    logLevel: logLevel.INFO,
    retry: {
        initialRetryTime: 100,
        retries: 8
    }
});

// কনজিউমার তৈরি
const consumer = kafka.consumer({
    groupId: 'pathao-ride-service',
    sessionTimeout: 30000,
    heartbeatInterval: 10000,
    maxPollInterval: 60000,
    rebalanceTimeout: 60000,
    partitionAssigners: [
        // CooperativeSticky ব্যবহার (kafkajs এ BuiltIn)
    ]
});

// DLQ প্রোডিউসার
const dlqProducer = kafka.producer({
    idempotent: true,
    maxInFlightRequests: 1
});

// মেট্রিক্স
let processedCount = 0;
let errorCount = 0;

async function processRide(rideEvent) {
    const { ride_id, driver_id, status, location } = rideEvent;

    switch (status) {
        case 'RIDE_REQUESTED':
            console.log(`🚗 নতুন রাইড রিকুয়েস্ট: #${ride_id}`);
            await notifyNearbyDrivers(location);
            break;

        case 'DRIVER_ASSIGNED':
            console.log(`👤 ড্রাইভার অ্যাসাইন: ${driver_id} → রাইড #${ride_id}`);
            await updateRideStatus(ride_id, 'ASSIGNED');
            break;

        case 'RIDE_STARTED':
            console.log(`🏁 রাইড শুরু: #${ride_id}`);
            await startTracking(ride_id, driver_id);
            break;

        case 'LOCATION_UPDATE':
            await updateDriverLocation(driver_id, location);
            break;

        case 'RIDE_COMPLETED':
            console.log(`✅ রাইড সম্পন্ন: #${ride_id}`);
            await calculateFare(ride_id);
            await stopTracking(ride_id);
            break;

        default:
            console.warn(`⚠️ অজানা স্ট্যাটাস: ${status}`);
    }
}

async function run() {
    await consumer.connect();
    await dlqProducer.connect();

    console.log('✅ Pathao Ride Consumer চালু হলো...');

    await consumer.subscribe({
        topics: ['ride-events', 'location-updates'],
        fromBeginning: false
    });

    await consumer.run({
        autoCommit: false,
        eachBatchAutoResolve: false,

        eachBatch: async ({
            batch,
            resolveOffset,
            heartbeat,
            commitOffsetsIfNecessary,
            isRunning,
            isStale
        }) => {
            const { topic, partition, messages } = batch;

            console.log(
                `📥 ব্যাচ পাওয়া গেছে: ${topic}[${partition}] — ${messages.length} মেসেজ`
            );

            for (const message of messages) {
                if (!isRunning() || isStale()) break;

                try {
                    const rideEvent = JSON.parse(message.value.toString());
                    await processRide(rideEvent);

                    resolveOffset(message.offset);
                    processedCount++;

                    // প্রতি ১০টা মেসেজে heartbeat
                    if (processedCount % 10 === 0) {
                        await heartbeat();
                    }

                } catch (error) {
                    errorCount++;
                    console.error(`❌ প্রসেসিং ব্যর্থ: ${error.message}`);

                    // DLQ তে পাঠাও
                    await dlqProducer.send({
                        topic: `${topic}-dlq`,
                        messages: [{
                            key: message.key,
                            value: JSON.stringify({
                                originalMessage: message.value.toString(),
                                error: error.message,
                                failedAt: new Date().toISOString(),
                                partition,
                                offset: message.offset
                            })
                        }]
                    });

                    resolveOffset(message.offset);
                }
            }

            await commitOffsetsIfNecessary();
        }
    });

    // মেট্রিক্স লগিং প্রতি ৩০ সেকেন্ডে
    setInterval(() => {
        console.log(
            `📊 মেট্রিক্স — প্রসেসড: ${processedCount} | এরর: ${errorCount}`
        );
    }, 30000);
}

// Graceful shutdown
const shutdown = async () => {
    console.log('🛑 Shutdown শুরু...');
    await consumer.disconnect();
    await dlqProducer.disconnect();
    console.log('🛑 Consumer বন্ধ হয়ে গেছে।');
    process.exit(0);
};

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

run().catch(console.error);
```

---

## 🎯 কনজিউমার বেস্ট প্র্যাকটিস

### ✅ প্রোডাকশন চেকলিস্ট

```
অফসেট ম্যানেজমেন্ট:
─────────────────────
☑ enable.auto.commit = false ব্যবহার করুন
☑ প্রসেসিং সফল হলেই কমিট করুন
☑ commitSync ব্যবহার করুন ক্রিটিকাল ডাটার জন্য
☑ auto.offset.reset = earliest সেট করুন (মেসেজ মিস এড়াতে)

কনজিউমার গ্রুপ:
────────────────
☑ group.instance.id সেট করুন (static membership)
☑ CooperativeSticky অ্যাসাইনমেন্ট ব্যবহার করুন
☑ কনজিউমার সংখ্যা ≤ পার্টিশন সংখ্যা রাখুন
☑ অপ্রয়োজনে কনজিউমার সংখ্যা বাড়াবেন না

টাইমআউট কনফিগারেশন:
─────────────────────
☑ session.timeout.ms = 30000-45000 (ডিফল্ট ঠিক আছে)
☑ heartbeat.interval.ms = session.timeout.ms / 3
☑ max.poll.interval.ms আপনার সবচেয়ে ধীর প্রসেসিং-এর ২x রাখুন
☑ max.poll.records — প্রসেসিং ক্ষমতা অনুযায়ী সেট করুন

এরর হ্যান্ডলিং:
────────────────
☑ DLQ প্যাটার্ন ইমপ্লিমেন্ট করুন
☑ Exponential backoff retry ব্যবহার করুন
☑ Poison pill মেসেজ ডিটেক্ট ও স্কিপ করুন
☑ এরর মেট্রিক্স ট্র্যাক করুন

মনিটরিং:
────────
☑ কনজিউমার ল্যাগ মনিটর করুন
☑ ল্যাগ threshold এ অ্যালার্ট সেট করুন
☑ প্রসেসিং রেট ট্র্যাক করুন
☑ রিব্যালান্সিং ফ্রিকোয়েন্সি মনিটর করুন

Graceful Shutdown:
──────────────────
☑ SIGTERM হ্যান্ডল করুন
☑ consumer.close() / consumer.disconnect() কল করুন
☑ চলমান প্রসেসিং শেষ হতে দিন
☑ শেষবারের মতো অফসেট কমিট করুন

Idempotent Processing:
──────────────────────
☑ প্রতিটা মেসেজে unique ID রাখুন
☑ প্রসেসিং-এর আগে ডুপ্লিকেট চেক করুন
☑ ডাটাবেসে idempotency key সংরক্ষণ করুন
```

### ❌ সাধারণ ভুল

```
❌ কনজিউমার সংখ্যা > পার্টিশন সংখ্যা — অতিরিক্ত কনজিউমার বসে থাকবে
❌ Auto commit + slow processing — মেসেজ হারাবে
❌ poll() এর মধ্যে দীর্ঘ blocking call — রিব্যালান্স ট্রিগার হবে
❌ DLQ ছাড়া প্রোডাকশনে যাওয়া — poison pill সব আটকে দেবে
❌ Graceful shutdown না করা — অফসেট কমিট না হলে ডুপ্লিকেট হবে
❌ ল্যাগ মনিটরিং না করা — সমস্যা ধরতে দেরি হবে
```

### পারফরম্যান্স টিউনিং টিপস

```
Grameenphone ইভেন্ট প্রসেসিং উদাহরণ:
══════════════════════════════════════

প্রতি সেকেন্ডে ৫০,০০০+ CDR (Call Detail Record) ইভেন্ট

কনফিগারেশন:
├── fetch.min.bytes = 65536 (64KB — বড় ব্যাচে fetch)
├── fetch.max.wait.ms = 500 (সর্বোচ্চ ৫০০ms অপেক্ষা)
├── max.poll.records = 1000 (প্রতি poll এ ১০০০ রেকর্ড)
├── max.partition.fetch.bytes = 1048576 (1MB প্রতি পার্টিশন)
├── পার্টিশন = 12
├── কনজিউমার = 12 (প্রতি পার্টিশনে ১টা)
└── ব্যাচ প্রসেসিং + async DB write

ফলাফল: ~৫০,০০০ msg/sec throughput, < 500ms ল্যাটেন্সি
```

---

## 🔗 পরবর্তী টপিক

| টপিক | ফাইল | বিষয়বস্তু |
|---|---|---|
| ➡️ পরবর্তী | [kafka-streams.md](./kafka-streams.md) | Kafka Streams — রিয়েল-টাইম স্ট্রিম প্রসেসিং |
| ⬅️ আগের | [producers.md](./producers.md) | Kafka প্রোডিউসার — মেসেজ পাঠানো |
| 🏠 হোম | [README.md](./README.md) | Kafka সম্পূর্ণ গাইড |

---

> **📝 মনে রাখবেন:** কনজিউমার ডিজাইন প্রোডিউসারের চেয়ে জটিল। অফসেট ম্যানেজমেন্ট,
> রিব্যালান্সিং, ডেলিভারি সিমান্টিক্স — সব সঠিকভাবে করতে হবে। Nagad বা bKash এর
> মতো ফিনটেক সিস্টেমে একটা ভুল অফসেট কমিট = গ্রাহকের টাকা হারানো বা ডুপ্লিকেট
> ট্রানজেকশন। তাই **manual commit + idempotent processing + DLQ** — এই তিনটা
> সবসময় ব্যবহার করুন।
