# 📘 Kafka Streams — স্ট্রিম প্রসেসিং

> Kafka Streams হলো একটি ক্লায়েন্ট লাইব্রেরি যা রিয়েল-টাইমে ডেটা প্রসেস করতে ব্যবহৃত হয়।
> এটি আলাদা কোনো ক্লাস্টার ছাড়াই আপনার অ্যাপ্লিকেশনের অংশ হিসেবে চলে।

---

## 📖 Kafka Streams কী?

### ধারণা

Kafka Streams একটি **স্ট্রিম প্রসেসিং লাইব্রেরি** — এটি কোনো ফ্রেমওয়ার্ক বা আলাদা ক্লাস্টার নয়।
এটি সরাসরি আপনার Java/Kotlin অ্যাপ্লিকেশনের ভেতরে চলে। কোনো আলাদা ইনফ্রাস্ট্রাকচার দরকার নেই।

### 🏠 বাংলা অ্যানালজি — পানি ফিল্টার সিস্টেম

> কল্পনা করুন একটি **পানি ফিল্টার সিস্টেম**।
> পানি (ডেটা) পাইপ দিয়ে ক্রমাগত আসছে। ফিল্টারের ভেতর দিয়ে যাওয়ার সময়
> ময়লা আলাদা হয়, মিনারেল যোগ হয়, পরিশোধিত পানি বের হয়।
> Kafka Streams ঠিক এইভাবে কাজ করে — ডেটা আসে, ট্রান্সফর্ম হয়, বের হয়।

### ব্যাচ vs স্ট্রিম প্রসেসিং

```
ব্যাচ প্রসেসিং (Batch):
┌─────────────────────┐
│  ডেটা জমা হয়        │ ──→ একসাথে প্রসেস ──→ রেজাল্ট
│  (ঘণ্টা/দিন ধরে)     │
└─────────────────────┘
উদাহরণ: Daraz-এ রাতে সব অর্ডারের রিপোর্ট তৈরি

স্ট্রিম প্রসেসিং (Stream):
Event ──→ প্রসেস ──→ রেজাল্ট (তৎক্ষণাৎ)
Event ──→ প্রসেস ──→ রেজাল্ট (তৎক্ষণাৎ)
Event ──→ প্রসেস ──→ রেজাল্ট (তৎক্ষণাৎ)
উদাহরণ: bKash-এ প্রতিটি ট্রানজেকশনে সাথে সাথে ফ্রড চেক
```

### Kafka Streams-এর বৈশিষ্ট্য

| বৈশিষ্ট্য                  | বিবরণ                                           |
| -------------------------- | ------------------------------------------------ |
| আলাদা ক্লাস্টার লাগে না   | আপনার অ্যাপের ভেতরেই চলে                         |
| ফল্ট টলারেন্ট              | Kafka-র রেপ্লিকেশন ও চেঞ্জলগ ব্যবহার করে         |
| স্কেলেবল                   | একাধিক ইনস্ট্যান্স চালালে পার্টিশন ভাগ হয়ে যায়  |
| Exactly-Once সাপোর্ট       | ডুপ্লিকেট ছাড়া নিশ্চিত প্রসেসিং                 |
| স্টেটফুল প্রসেসিং          | লোকাল স্টেট স্টোর (RocksDB) দিয়ে                 |

---

## 📊 KStream vs KTable vs GlobalKTable

### KStream — ইভেন্ট স্ট্রিম

KStream হলো **অসীম ইভেন্টের ধারা** — প্রতিটি রেকর্ড স্বাধীন। একই key-তে নতুন ভ্যালু এলেও
পুরোনোটা মুছে যায় না — দুটোই থাকে। এটি যেন একটি **নদীর মতো** — পানি শুধু বয়ে যায়।

### KTable — চেঞ্জলগ স্ট্রিম

KTable হলো **একটি টেবিল যা আপডেট হতে থাকে**। প্রতিটি key-তে সর্বশেষ ভ্যালুটাই থাকে।
পুরোনো ভ্যালু নতুনটি দিয়ে রিপ্লেস হয়ে যায়। এটি যেন একটি **ডেটাবেস টেবিল**।

### GlobalKTable — গ্লোবাল টেবিল

GlobalKTable হলো KTable-এর মতোই, কিন্তু এটি **প্রতিটি ইনস্ট্যান্সে সম্পূর্ণ ডেটা কপি রাখে**।
পার্টিশন নির্বিশেষে সব ডেটা পাওয়া যায়। ছোট রেফারেন্স ডেটার জন্য ব্যবহৃত হয়।

### ASCII ডায়াগ্রাম

```
KStream (ইভেন্ট স্ট্রিম):
Time ──→ [Buy A] [Buy B] [Buy A] [Sell A] [Buy C]
          ↑        ↑        ↑       ↑        ↑
     প্রতিটি ইভেন্ট স্বাধীন — কেউ কাউকে রিপ্লেস করে না

KTable (চেঞ্জলগ):
Key │ Value (সর্বশেষ)
────┼──────────────────────────────
A   │ Buy → Buy → Sell  ══▶  Sell  (শুধু শেষেরটা থাকে)
B   │ Buy              ══▶  Buy
C   │ Buy              ══▶  Buy
শুধু প্রতিটি key-এর সর্বশেষ মান সংরক্ষিত থাকে

GlobalKTable (গ্লোবাল):
┌─────────────────────────────────┐
│  Instance 1: [A=Sell, B=Buy, C=Buy]  ← সম্পূর্ণ কপি
│  Instance 2: [A=Sell, B=Buy, C=Buy]  ← সম্পূর্ণ কপি
│  Instance 3: [A=Sell, B=Buy, C=Buy]  ← সম্পূর্ণ কপি
└─────────────────────────────────┘
```

### কখন কোনটা ব্যবহার করবেন?

| টাইপ          | ব্যবহার                                     | উদাহরণ                              |
| ------------- | -------------------------------------------- | ------------------------------------ |
| KStream       | প্রতিটি ইভেন্ট আলাদাভাবে প্রসেস করতে         | bKash ট্রানজেকশন লগ                  |
| KTable        | সর্বশেষ স্টেট ট্র্যাক করতে                    | ইউজারের বর্তমান ব্যালেন্স            |
| GlobalKTable  | ছোট রেফারেন্স ডেটা সব ইনস্ট্যান্সে রাখতে     | জেলা/বিভাগের নাম ম্যাপিং             |

---

## 🔧 স্টেটলেস অপারেশন (Stateless Operations)

স্টেটলেস অপারেশনে কোনো পূর্ববর্তী ডেটার উপর নির্ভরতা নেই।
প্রতিটি রেকর্ড স্বাধীনভাবে প্রসেস হয়। কোনো স্টেট স্টোর দরকার নেই।

### filter — ফিল্টার করা

নির্দিষ্ট শর্ত পূরণ করলেই রেকর্ড এগিয়ে যায়, নাহলে বাদ পড়ে।

```javascript
// Daraz: শুধু ঢাকার অর্ডারগুলো ফিল্টার করা
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });
const consumer = kafka.consumer({ groupId: 'dhaka-order-filter' });
const producer = kafka.producer();

async function filterDhakaOrders() {
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'all-orders' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const order = JSON.parse(message.value.toString());

      // ফিল্টার: শুধু ঢাকা রিজিওনের অর্ডার
      if (order.region === 'dhaka') {
        await producer.send({
          topic: 'dhaka-orders',
          messages: [{ key: order.orderId, value: JSON.stringify(order) }],
        });
      }
    },
  });
}
```

### map — ট্রান্সফর্ম করা

প্রতিটি রেকর্ডকে নতুন ফরম্যাটে রূপান্তর করে।

```javascript
// Daraz: অর্ডারকে শিপিং ফরম্যাটে ম্যাপ করা
await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());

    // ম্যাপ: অর্ডার → শিপিং ইনস্ট্রাকশন
    const shipping = {
      trackingId: `DARAZ-${order.orderId}`,
      address: order.shippingAddress,
      weight: order.items.reduce((sum, i) => sum + i.weight, 0),
      priority: order.amount > 5000 ? 'express' : 'standard',
    };

    await producer.send({
      topic: 'shipping-instructions',
      messages: [{ key: shipping.trackingId, value: JSON.stringify(shipping) }],
    });
  },
});
```

### flatMap — একটি থেকে অনেক

একটি রেকর্ড থেকে একাধিক রেকর্ড তৈরি করে।

```javascript
// Daraz: একটি অর্ডারের প্রতিটি আইটেমকে আলাদা ইভেন্টে ভাঙা
await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());

    // flatMap: একটি অর্ডার → অনেক আইটেম ইভেন্ট
    const itemMessages = order.items.map((item) => ({
      key: item.productId,
      value: JSON.stringify({
        orderId: order.orderId,
        productId: item.productId,
        quantity: item.quantity,
        price: item.price,
      }),
    }));

    await producer.send({ topic: 'order-items', messages: itemMessages });
  },
});
```

### branch — শর্তানুযায়ী ভাগ করা

রেকর্ডকে বিভিন্ন শর্ত অনুযায়ী আলাদা টপিকে পাঠানো।

```javascript
// Daraz: অর্ডার ভ্যালু অনুযায়ী রাউটিং
await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());

    let targetTopic;
    if (order.amount > 10000) {
      targetTopic = 'high-value-orders';     // ১০,০০০+ টাকার অর্ডার
    } else if (order.amount > 2000) {
      targetTopic = 'medium-value-orders';   // ২,০০০-১০,০০০ টাকা
    } else {
      targetTopic = 'low-value-orders';      // ২,০০০ এর নিচে
    }

    await producer.send({
      topic: targetTopic,
      messages: [{ key: order.orderId, value: message.value }],
    });
  },
});
```

### peek — সাইড ইফেক্ট (ডেটা পরিবর্তন না করে পর্যবেক্ষণ)

```javascript
// লগিং বা মনিটরিং — ডেটা স্ট্রিমে কোনো পরিবর্তন হয় না
await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());

    // peek: শুধু পর্যবেক্ষণ, কোনো পরিবর্তন নেই
    console.log(`[MONITOR] অর্ডার পাওয়া গেছে: ${order.orderId}, মূল্য: ৳${order.amount}`);
    metrics.orderCount.inc(); // Prometheus মেট্রিক্স আপডেট

    // পরবর্তী প্রসেসিং-এ ডেটা অপরিবর্তিত যায়
    await producer.send({
      topic: 'processed-orders',
      messages: [{ key: order.orderId, value: message.value }],
    });
  },
});
```

### selectKey — কী পরিবর্তন

```javascript
// Daraz: অর্ডারের কী orderId থেকে customerId-তে পরিবর্তন
await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());

    // selectKey: নতুন কী = customerId
    await producer.send({
      topic: 'orders-by-customer',
      messages: [{ key: order.customerId, value: message.value }],
    });
  },
});
```

---

## 🔧 স্টেটফুল অপারেশন (Stateful Operations)

স্টেটফুল অপারেশনে পূর্ববর্তী ডেটার উপর নির্ভর করে কাজ হয়।
এজন্য **স্টেট স্টোর** ব্যবহার করা হয় (ডিফল্ট: RocksDB)।

### groupByKey / groupBy — গ্রুপ করা

```javascript
// bKash: প্রতি ইউজারের ট্রানজেকশন গ্রুপ করা
const userTransactions = new Map(); // ইন-মেমোরি স্টেট (প্রোডাকশনে Redis/DB ব্যবহার করুন)

await consumer.run({
  eachMessage: async ({ message }) => {
    const txn = JSON.parse(message.value.toString());
    const userId = txn.userId;

    // groupByKey: ইউজার আইডি অনুযায়ী গ্রুপ
    if (!userTransactions.has(userId)) {
      userTransactions.set(userId, []);
    }
    userTransactions.get(userId).push(txn);
  },
});
```

### count — গণনা

```javascript
// bKash: প্রতি ইউজারের ট্রানজেকশন সংখ্যা গণনা
const transactionCounts = new Map();

await consumer.run({
  eachMessage: async ({ message }) => {
    const txn = JSON.parse(message.value.toString());
    const userId = txn.userId;

    // count: প্রতি ইউজারে কতটি ট্রানজেকশন
    const currentCount = transactionCounts.get(userId) || 0;
    transactionCounts.set(userId, currentCount + 1);

    console.log(`ইউজার ${userId}: মোট ${currentCount + 1} টি ট্রানজেকশন`);

    // আপডেটেড কাউন্ট পাবলিশ
    await producer.send({
      topic: 'user-txn-counts',
      messages: [{
        key: userId,
        value: JSON.stringify({ userId, count: currentCount + 1 }),
      }],
    });
  },
});
```

### aggregate — সমষ্টি

```javascript
// bKash: প্রতি ইউজারের দৈনিক মোট ট্রানজেকশন অ্যামাউন্ট
const dailyTotals = new Map();

function getTodayKey(userId) {
  const today = new Date().toISOString().split('T')[0];
  return `${userId}:${today}`;
}

await consumer.run({
  eachMessage: async ({ message }) => {
    const txn = JSON.parse(message.value.toString());
    const key = getTodayKey(txn.userId);

    // aggregate: দৈনিক মোট যোগ করা
    const current = dailyTotals.get(key) || { totalAmount: 0, count: 0 };
    current.totalAmount += txn.amount;
    current.count += 1;
    dailyTotals.set(key, current);

    console.log(`ইউজার ${txn.userId} আজ: মোট ৳${current.totalAmount} (${current.count} টি)`);

    await producer.send({
      topic: 'daily-user-totals',
      messages: [{
        key: txn.userId,
        value: JSON.stringify({
          userId: txn.userId,
          date: new Date().toISOString().split('T')[0],
          totalAmount: current.totalAmount,
          transactionCount: current.count,
        }),
      }],
    });
  },
});
```

### reduce — সংক্ষেপ

```javascript
// Nagad: প্রতি ইউজারের সর্বোচ্চ একক ট্রানজেকশন ট্র্যাক
const maxTransactions = new Map();

await consumer.run({
  eachMessage: async ({ message }) => {
    const txn = JSON.parse(message.value.toString());
    const userId = txn.userId;

    // reduce: দুটি ভ্যালু থেকে একটি বেছে নেওয়া (সর্বোচ্চ)
    const currentMax = maxTransactions.get(userId) || 0;
    if (txn.amount > currentMax) {
      maxTransactions.set(userId, txn.amount);
      console.log(`ইউজার ${userId}: নতুন সর্বোচ্চ ৳${txn.amount}`);
    }
  },
});
```

---

## ⏰ উইন্ডোইং (Windowing)

উইন্ডোইং হলো সময়ের ভিত্তিতে ডেটাকে গোষ্ঠীবদ্ধ করার পদ্ধতি।
স্ট্রিম ডেটায় "কখন পর্যন্তের ডেটা?" এই প্রশ্নের উত্তর দেয় উইন্ডো।

### Tumbling Window — নির্দিষ্ট, নন-ওভারল্যাপিং

একটি নির্দিষ্ট সময়ের ব্লক। কোনো ওভারল্যাপ নেই। প্রতিটি ইভেন্ট ঠিক একটি উইন্ডোতে পড়ে।

```
Tumbling Window (৫ মিনিট):
|═══════|═══════|═══════|═══════|
0      5min   10min   15min   20min
  W1      W2      W3      W4

ইভেন্ট: ●
|●●●  ● |  ●● ● | ●     | ●●●●●|
 W1 (4)   W2 (3)  W3 (1)  W4 (5)

প্রতিটি ইভেন্ট শুধু একটি উইন্ডোতে পড়ে ✅
```

**Grameenphone উদাহরণ:** প্রতি ঘণ্টায় প্রতিটি টাওয়ারে কতগুলো কল হলো গণনা করা।

```javascript
// GP: প্রতি ঘণ্টায় কল কাউন্ট (Tumbling Window সিমুলেশন)
const WINDOW_SIZE_MS = 60 * 60 * 1000; // ১ ঘণ্টা
const tumblingCounts = new Map();

function getTumblingWindowKey(timestamp, towerId) {
  const windowStart = Math.floor(timestamp / WINDOW_SIZE_MS) * WINDOW_SIZE_MS;
  return `${towerId}:${windowStart}`;
}

await consumer.run({
  eachMessage: async ({ message }) => {
    const call = JSON.parse(message.value.toString());
    const windowKey = getTumblingWindowKey(call.timestamp, call.towerId);

    const count = (tumblingCounts.get(windowKey) || 0) + 1;
    tumblingCounts.set(windowKey, count);
  },
});
```

### Hopping Window — নির্দিষ্ট, ওভারল্যাপিং

নির্দিষ্ট সাইজের উইন্ডো, কিন্তু একটি নির্দিষ্ট ইন্টারভালে এগিয়ে যায়। উইন্ডো ওভারল্যাপ করে।

```
Hopping Window (size=10min, advance=5min):
|══════════════════|                    Window 1 (0-10)
         |══════════════════|           Window 2 (5-15)
                  |══════════════════|  Window 3 (10-20)
0       5min     10min    15min    20min

ইভেন্ট 7min-এ: Window 1 ✅ এবং Window 2 ✅ — দুটোতেই পড়ে!
```

**উদাহরণ:** গত ১০ মিনিটে bKash-এ কত ট্রানজেকশন হয়েছে — প্রতি ৫ মিনিটে আপডেট।

### Sliding Window — ইভেন্ট দ্বারা ট্রিগার

দুটি ইভেন্টের মধ্যকার সময়ের পার্থক্য নির্দিষ্ট সীমার মধ্যে থাকলে একই উইন্ডোতে পড়ে।

```
Sliding Window (difference=10min):

ইভেন্ট:   A(2min)   B(5min)   C(8min)       D(18min)   E(20min)
           ●         ●         ●               ●          ●

A-B পার্থক্য = 3min < 10min  → একই উইন্ডো ✅
A-C পার্থক্য = 6min < 10min  → একই উইন্ডো ✅
A-D পার্থক্য = 16min > 10min → আলাদা উইন্ডো ❌
D-E পার্থক্য = 2min < 10min  → একই উইন্ডো ✅
```

### Session Window — নিষ্ক্রিয়তার ভিত্তিতে

ইউজারের অ্যাক্টিভিটি গ্যাপ দিয়ে সেশন নির্ধারণ হয়। গ্যাপ পার হলে নতুন সেশন শুরু।

```
Session Window (inactivity gap = 5min):

ইভেন্ট:  ●  ● ●  ●          ● ●           ●
Time:    1  2 3  4          12 13          25 min
         |         |  gap   |    |  gap    |  |
         |=========|        |====|         |==|
         Session 1          Session 2      Session 3
         (4 events)         (2 events)     (1 event)

5 মিনিটের বেশি গ্যাপ পেলেই নতুন সেশন!
```

**Grameenphone উদাহরণ:** ইউজারের ডেটা ব্যবহারের প্যাটার্ন — কখন ব্যবহার শুরু ও শেষ হয়।

```javascript
// GP: সেশন উইন্ডো সিমুলেশন — ইউজারের ডেটা ইউসেজ সেশন
const SESSION_GAP_MS = 5 * 60 * 1000; // ৫ মিনিট গ্যাপ
const userSessions = new Map();

await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    const userId = event.userId;
    const now = event.timestamp;

    const session = userSessions.get(userId);

    if (!session || (now - session.lastActivity) > SESSION_GAP_MS) {
      // নতুন সেশন শুরু
      if (session) {
        console.log(`সেশন শেষ — ইউজার: ${userId}, ইভেন্ট: ${session.eventCount}`);
      }
      userSessions.set(userId, { start: now, lastActivity: now, eventCount: 1 });
    } else {
      // বিদ্যমান সেশনে যোগ
      session.lastActivity = now;
      session.eventCount += 1;
    }
  },
});
```

---

## 🔗 জয়েন (Joins)

Kafka Streams-এ বিভিন্ন স্ট্রিম ও টেবিলকে জয়েন করে সমৃদ্ধ ডেটা তৈরি করা যায়।

### KStream-KStream জয়েন (উইন্ডো ভিত্তিক)

দুটি ইভেন্ট স্ট্রিমকে একটি নির্দিষ্ট সময়ের উইন্ডোর মধ্যে জয়েন করা হয়।

```
KStream (রাইড রিকোয়েস্ট)  ×  KStream (পেমেন্ট)
        ↓                           ↓
   [R1, t=10:01]              [P1, t=10:03]
   [R2, t=10:05]              [P2, t=10:07]

Window = 5 min:
R1 + P1 → জয়েন (পার্থক্য 2min < 5min) ✅
R2 + P2 → জয়েন (পার্থক্য 2min < 5min) ✅
```

### KStream-KTable জয়েন (এনরিচমেন্ট)

স্ট্রিম ইভেন্টকে টেবিলের সর্বশেষ ভ্যালু দিয়ে সমৃদ্ধ করা হয়। **সবচেয়ে বেশি ব্যবহৃত জয়েন।**

```
KStream (রাইড রিকোয়েস্ট)  ×  KTable (ড্রাইভার লোকেশন)
        ↓                            ↓
   [user=U1, pickup=Gulshan]    [D1 → Banani]
                                [D2 → Gulshan]
                                [D3 → Dhanmondi]

জয়েন রেজাল্ট: U1-এর রিকোয়েস্টের সাথে কাছের ড্রাইভারের তথ্য যুক্ত
```

**Pathao উদাহরণ:**

```javascript
// Pathao: রাইড রিকোয়েস্ট + ড্রাইভার লোকেশন জয়েন
const driverLocations = new Map(); // KTable সিমুলেশন

// ড্রাইভার লোকেশন আপডেট (KTable হিসেবে কাজ করে)
const locationConsumer = kafka.consumer({ groupId: 'driver-location-table' });
await locationConsumer.subscribe({ topic: 'driver-locations' });

await locationConsumer.run({
  eachMessage: async ({ message }) => {
    const loc = JSON.parse(message.value.toString());
    driverLocations.set(loc.driverId, loc); // সর্বশেষ লোকেশন রাখে
  },
});

// রাইড রিকোয়েস্ট স্ট্রিম (KStream-KTable জয়েন)
const rideConsumer = kafka.consumer({ groupId: 'ride-matcher' });
await rideConsumer.subscribe({ topic: 'ride-requests' });

await rideConsumer.run({
  eachMessage: async ({ message }) => {
    const ride = JSON.parse(message.value.toString());

    // KStream-KTable Join: রিকোয়েস্টের সাথে ড্রাইভার লোকেশন যোগ
    const nearbyDrivers = [...driverLocations.values()]
      .filter((d) => calculateDistance(d.lat, d.lng, ride.pickupLat, ride.pickupLng) < 3);

    const enrichedRide = {
      ...ride,
      availableDrivers: nearbyDrivers.length,
      nearestDriver: nearbyDrivers[0] || null,
    };

    await producer.send({
      topic: 'matched-rides',
      messages: [{ key: ride.rideId, value: JSON.stringify(enrichedRide) }],
    });
  },
});
```

### KTable-KTable জয়েন

দুটি টেবিলের সর্বশেষ ভ্যালু দিয়ে জয়েন। যেকোনো পাশে আপডেট হলে নতুন জয়েন রেজাল্ট।

### জয়েন তুলনা টেবিল

| জয়েন টাইপ             | উইন্ডো দরকার? | ব্যবহারের ক্ষেত্র                    | Co-Partition? |
| ---------------------- | ------------- | ------------------------------------- | ------------- |
| KStream-KStream        | ✅ হ্যাঁ       | দুটি ইভেন্ট স্ট্রিম মেলানো            | ✅ হ্যাঁ       |
| KStream-KTable         | ❌ না          | স্ট্রিমকে রেফারেন্স ডেটা দিয়ে সমৃদ্ধ  | ✅ হ্যাঁ       |
| KStream-GlobalKTable   | ❌ না          | স্ট্রিমকে গ্লোবাল ডেটা দিয়ে সমৃদ্ধ   | ❌ না          |
| KTable-KTable          | ❌ না          | দুটি টেবিলের সর্বশেষ ভ্যালু জয়েন      | ✅ হ্যাঁ       |

---

## 💾 স্টেট স্টোর (State Stores)

স্টেটফুল অপারেশনের জন্য Kafka Streams লোকালি ডেটা সংরক্ষণ করে —
এগুলোকে **স্টেট স্টোর** বলে। ডিফল্ট হলো **RocksDB** (ডিস্ক-ভিত্তিক)।

### আর্কিটেকচার

```
┌──────────────────────────────────────────────────┐
│              Kafka Streams Instance               │
│                                                  │
│  ┌──────────────┐      ┌───────────────────┐     │
│  │ Processor    │ ───→ │ State Store       │     │
│  │ Topology     │ ←─── │ (RocksDB/Memory)  │     │
│  └──────┬───────┘      └────────┬──────────┘     │
│         │                       │                │
│         │              Changelog Topic           │
│         │              (ব্যাকআপ/ফল্ট টলারেন্স)   │
│         │                       │                │
└─────────┼───────────────────────┼────────────────┘
          │                       ▼
    ┌─────▼─────┐         ┌──────────────┐
    │  Input    │         │   Kafka      │
    │  Topic    │         │ (changelog)  │
    └───────────┘         └──────────────┘
```

### RocksDB — ডিফল্ট স্টেট স্টোর

- ডিস্কে সংরক্ষিত, তাই মেমোরির চেয়ে বেশি ডেটা রাখতে পারে
- LSM-tree ভিত্তিক — লেখার জন্য অপ্টিমাইজড
- Kafka changelog টপিকে ব্যাকআপ রাখে

### ইন-মেমোরি স্টোর

- দ্রুত কিন্তু সীমিত ক্যাপাসিটি
- অ্যাপ রিস্টার্ট হলে changelog থেকে রিস্টোর হয়
- ছোট স্টেটের জন্য উপযুক্ত

### চেঞ্জলগ টপিক — ফল্ট টলারেন্স

```
স্টেট স্টোর আপডেট ──→ Changelog টপিকে লেখা হয়
                        (কম্প্যাক্টেড টপিক)

ইনস্ট্যান্স ক্র্যাশ হলে:
নতুন ইনস্ট্যান্স → Changelog পড়ে → স্টেট স্টোর রিবিল্ড ✅
```

### ইন্টারেক্টিভ কোয়েরি

স্টেট স্টোরে থাকা ডেটা বাইরে থেকে কোয়েরি করা যায় — REST API-র মাধ্যমে
রিয়েল-টাইম ড্যাশবোর্ড তৈরি করা সম্ভব।

---

## 🔒 Exactly-Once প্রসেসিং

### সমস্যা

স্ট্রিম প্রসেসিং-এ তিন ধরনের গ্যারান্টি থাকতে পারে:

```
At-Most-Once:   মেসেজ হারিয়ে যেতে পারে, কিন্তু ডুপ্লিকেট হবে না
At-Least-Once:  মেসেজ হারাবে না, কিন্তু ডুপ্লিকেট হতে পারে
Exactly-Once:   মেসেজ হারাবে না এবং ডুপ্লিকেটও হবে না ✅
```

### কীভাবে কাজ করে

```
processing.guarantee = "exactly_once_v2"

অভ্যন্তরীণ মেকানিজম:
1. প্রতিটি ট্রানজেকশনে idempotent প্রডিউসার ব্যবহার
2. কনজিউমার অফসেট + আউটপুট মেসেজ একই ট্রানজেকশনে কমিট
3. ক্র্যাশ হলে uncommitted ডেটা রোলব্যাক হয়

┌──────────┐    ┌──────────┐    ┌──────────┐
│  Read    │ →  │ Process  │ →  │ Write +  │
│  Input   │    │  Data    │    │ Commit   │  ← সবকিছু একটি
│  Topic   │    │          │    │ Offset   │    ট্রানজেকশনে
└──────────┘    └──────────┘    └──────────┘
```

### পারফরম্যান্স প্রভাব

| সেটিং            | ল্যাটেন্সি | থ্রুপুট | নির্ভরযোগ্যতা |
| ----------------- | ---------- | -------- | ------------- |
| at_least_once     | কম ⚡      | বেশি 📈  | মাঝারি        |
| exactly_once_v2   | বেশি 🐢   | কম 📉   | সর্বোচ্চ ✅    |

> **bKash-এর মতো পেমেন্ট সিস্টেমে** exactly-once অপরিহার্য — ডুপ্লিকেট পেমেন্ট গ্রহণযোগ্য নয়!

---

## 📊 Kafka Streams vs অন্যান্য

| বৈশিষ্ট্য             | Kafka Streams       | Apache Flink        | Apache Spark Streaming |
| ---------------------- | ------------------- | ------------------- | ---------------------- |
| ডিপ্লয়মেন্ট           | লাইব্রেরি (JAR)     | আলাদা ক্লাস্টার     | আলাদা ক্লাস্টার        |
| ল্যাটেন্সি             | মিলিসেকেন্ড         | মিলিসেকেন্ড         | সেকেন্ড-মিনিট          |
| স্টেট ম্যানেজমেন্ট     | RocksDB (লোকাল)     | RocksDB (ম্যানেজড)  | এক্সটার্নাল             |
| Exactly-Once           | ✅ নেটিভ            | ✅ নেটিভ            | ✅ সীমিত               |
| শেখার কঠিনতা          | সহজ ⭐⭐            | মাঝারি ⭐⭐⭐        | মাঝারি ⭐⭐⭐           |
| ডেটা সোর্স             | শুধু Kafka           | যেকোনো              | যেকোনো                 |
| ব্যাচ + স্ট্রিম        | শুধু স্ট্রিম         | দুটোই ✅             | দুটোই ✅                |
| স্কেলিং                | কনজিউমার গ্রুপ      | ক্লাস্টার ম্যানেজার  | YARN/Mesos             |
| অপারেশনাল কস্ট        | কম 💰               | বেশি 💰💰💰          | বেশি 💰💰              |

### কখন কোনটা বেছে নেবেন?

- **Kafka Streams:** শুধু Kafka ডেটা প্রসেস করলে, সিম্পল ডিপ্লয়মেন্ট চাইলে
- **Apache Flink:** জটিল ইভেন্ট প্রসেসিং, মাল্টি-সোর্স, CEP দরকার হলে
- **Spark Streaming:** ব্যাচ + স্ট্রিম একসাথে, ML পাইপলাইন দরকার হলে

---

## 💻 JavaScript উদাহরণ — সম্পূর্ণ পাইপলাইন

### উদাহরণ ১: Daraz রিয়েল-টাইম অর্ডার অ্যানালিটিক্স

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'daraz-analytics',
  brokers: ['localhost:9092'],
});

const consumer = kafka.consumer({ groupId: 'order-analytics' });
const producer = kafka.producer();

// স্টেট স্টোর সিমুলেশন
const regionStats = new Map();
const categoryStats = new Map();
const WINDOW_SIZE = 5 * 60 * 1000; // ৫ মিনিট উইন্ডো

function getWindowKey(timestamp) {
  return Math.floor(timestamp / WINDOW_SIZE) * WINDOW_SIZE;
}

async function runOrderAnalytics() {
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'daraz-orders', fromBeginning: false });

  console.log('🚀 Daraz অর্ডার অ্যানালিটিক্স পাইপলাইন চালু...');

  await consumer.run({
    eachMessage: async ({ message }) => {
      const order = JSON.parse(message.value.toString());

      // ১. ফিল্টার: বাতিল অর্ডার বাদ
      if (order.status === 'cancelled') return;

      // ২. ম্যাপ: প্রয়োজনীয় ফিল্ড বের করা
      const analyticsEvent = {
        orderId: order.id,
        region: order.region,
        category: order.items[0]?.category || 'unknown',
        amount: order.totalAmount,
        timestamp: Date.now(),
      };

      // ৩. Aggregate: রিজিওন অনুযায়ী মোট
      const windowKey = getWindowKey(analyticsEvent.timestamp);
      const regionKey = `${analyticsEvent.region}:${windowKey}`;

      const regionStat = regionStats.get(regionKey) || { count: 0, total: 0 };
      regionStat.count += 1;
      regionStat.total += analyticsEvent.amount;
      regionStats.set(regionKey, regionStat);

      // ৪. Aggregate: ক্যাটাগরি অনুযায়ী মোট
      const catKey = `${analyticsEvent.category}:${windowKey}`;
      const catStat = categoryStats.get(catKey) || { count: 0, total: 0 };
      catStat.count += 1;
      catStat.total += analyticsEvent.amount;
      categoryStats.set(catKey, catStat);

      // ৫. Branch: বড় অর্ডার আলাদা টপিকে
      if (analyticsEvent.amount > 10000) {
        await producer.send({
          topic: 'high-value-alerts',
          messages: [{
            key: order.id,
            value: JSON.stringify({
              ...analyticsEvent,
              alert: 'HIGH_VALUE_ORDER',
            }),
          }],
        });
      }

      // ৬. আউটপুট: অ্যানালিটিক্স ড্যাশবোর্ডে পাঠানো
      await producer.send({
        topic: 'order-analytics-output',
        messages: [{
          key: analyticsEvent.region,
          value: JSON.stringify({
            region: analyticsEvent.region,
            windowStart: new Date(windowKey).toISOString(),
            orderCount: regionStat.count,
            totalRevenue: regionStat.total,
          }),
        }],
      });
    },
  });
}

runOrderAnalytics().catch(console.error);
```

### উদাহরণ ২: bKash রিয়েল-টাইম ফ্রড ডিটেকশন

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'bkash-fraud-detection',
  brokers: ['localhost:9092'],
});

const consumer = kafka.consumer({ groupId: 'fraud-detector' });
const producer = kafka.producer();

// স্টেট স্টোর: প্রতিটি ইউজারের রিসেন্ট ট্রানজেকশন ট্র্যাকিং
const userRecentTxns = new Map();
const userDailyTotals = new Map();
const RAPID_TXN_WINDOW = 2 * 60 * 1000;   // ২ মিনিট
const DAILY_LIMIT = 200000;                 // দৈনিক ২ লাখ টাকা সীমা
const RAPID_TXN_THRESHOLD = 5;              // ২ মিনিটে ৫টির বেশি সন্দেহজনক

async function detectFraud() {
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'bkash-transactions' });

  console.log('🔒 bKash ফ্রড ডিটেকশন পাইপলাইন চালু...');

  await consumer.run({
    eachMessage: async ({ message }) => {
      const txn = JSON.parse(message.value.toString());
      const { userId, amount, timestamp, type } = txn;
      const alerts = [];

      // ══════════════════════════════════════════════
      // রুল ১: বড় অঙ্কের ট্রানজেকশন (৳৫০,০০০+)
      // ══════════════════════════════════════════════
      if (amount > 50000) {
        alerts.push({
          rule: 'LARGE_TRANSACTION',
          severity: 'HIGH',
          message: `বড় ট্রানজেকশন: ৳${amount}`,
        });
      }

      // ══════════════════════════════════════════════
      // রুল ২: দ্রুত পরপর ট্রানজেকশন (Session Window ধারণা)
      // ══════════════════════════════════════════════
      const recentTxns = userRecentTxns.get(userId) || [];
      const now = timestamp || Date.now();

      // পুরোনো ট্রানজেকশন বাদ (উইন্ডো বাইরে)
      const activeTxns = recentTxns.filter(
        (t) => (now - t.timestamp) < RAPID_TXN_WINDOW
      );
      activeTxns.push({ timestamp: now, amount });
      userRecentTxns.set(userId, activeTxns);

      if (activeTxns.length > RAPID_TXN_THRESHOLD) {
        alerts.push({
          rule: 'RAPID_TRANSACTIONS',
          severity: 'CRITICAL',
          message: `${RAPID_TXN_WINDOW / 1000}s এ ${activeTxns.length} টি ট্রানজেকশন`,
        });
      }

      // ══════════════════════════════════════════════
      // রুল ৩: দৈনিক সীমা অতিক্রম (Tumbling Window — ১ দিন)
      // ══════════════════════════════════════════════
      const today = new Date(now).toISOString().split('T')[0];
      const dailyKey = `${userId}:${today}`;
      const dailyTotal = (userDailyTotals.get(dailyKey) || 0) + amount;
      userDailyTotals.set(dailyKey, dailyTotal);

      if (dailyTotal > DAILY_LIMIT) {
        alerts.push({
          rule: 'DAILY_LIMIT_EXCEEDED',
          severity: 'HIGH',
          message: `দৈনিক সীমা অতিক্রম: ৳${dailyTotal} / ৳${DAILY_LIMIT}`,
        });
      }

      // ══════════════════════════════════════════════
      // আউটপুট: অ্যালার্ট পাঠানো
      // ══════════════════════════════════════════════
      if (alerts.length > 0) {
        await producer.send({
          topic: 'fraud-alerts',
          messages: [{
            key: userId,
            value: JSON.stringify({
              userId,
              transactionId: txn.txnId,
              amount,
              alerts,
              timestamp: now,
            }),
          }],
        });

        console.log(`⚠️ ফ্রড অ্যালার্ট — ইউজার: ${userId}, রুল: ${alerts.map((a) => a.rule).join(', ')}`);
      }
    },
  });
}

detectFraud().catch(console.error);
```

---

## 🎯 সারসংক্ষেপ ও বেস্ট প্র্যাকটিস

### মূল বিষয়গুলো মনে রাখুন

```
✅ Kafka Streams = লাইব্রেরি, আলাদা ক্লাস্টার নয়
✅ KStream = ইভেন্ট স্ট্রিম (নদীর মতো প্রবাহ)
✅ KTable = চেঞ্জলগ (সর্বশেষ স্টেট)
✅ স্টেটলেস = filter, map, flatMap, branch
✅ স্টেটফুল = groupBy, aggregate, count, reduce
✅ উইন্ডো = Tumbling, Hopping, Sliding, Session
✅ জয়েন = Stream-Stream, Stream-Table, Table-Table
✅ স্টেট স্টোর = RocksDB + Changelog
✅ Exactly-Once = পেমেন্ট সিস্টেমে অপরিহার্য
```

### বেস্ট প্র্যাকটিস

| # | প্র্যাকটিস                                     | কারণ                                      |
| - | ----------------------------------------------- | ----------------------------------------- |
| 1 | পার্টিশন সংখ্যা পর্যাপ্ত রাখুন                  | প্যারালেলিজম পার্টিশন দ্বারা নির্ধারিত      |
| 2 | স্টেট স্টোরের সাইজ মনিটর করুন                   | RocksDB ডিস্ক পূর্ণ হতে পারে              |
| 3 | Changelog টপিকের retention ঠিক রাখুন            | স্টোর রিবিল্ডের জন্য দরকার                |
| 4 | Exactly-Once শুধু দরকার হলে ব্যবহার করুন        | পারফরম্যান্স খরচ আছে                      |
| 5 | ছোট রেফারেন্স ডেটায় GlobalKTable ব্যবহার করুন   | Co-partitioning এড়ানো যায়                |
| 6 | উইন্ডো সাইজ ব্যবসার চাহিদা অনুযায়ী ঠিক করুন    | অপ্রয়োজনীয় বড় উইন্ডো মেমোরি খরচ করে     |
| 7 | গ্রেসফুল শাটডাউন ইমপ্লিমেন্ট করুন               | স্টেট সংরক্ষণ ও রিব্যালেন্সের জন্য       |
| 8 | মেট্রিক্স ও মনিটরিং সেটআপ করুন                  | ল্যাগ ও থ্রুপুট ট্র্যাক করতে              |

### ❌ সাধারণ ভুল

```
❌ খুব বেশি স্টেট রাখা (মেমোরি/ডিস্ক শেষ)
❌ উইন্ডো retention সেট না করা (পুরোনো ডেটা জমতে থাকে)
❌ Co-partitioning না বুঝেই জয়েন করা
❌ Exactly-Once সব জায়গায় ব্যবহার করা (অপ্রয়োজনীয় ল্যাটেন্সি)
❌ Error handling না করা (poison pill মেসেজ স্ট্রিম আটকে দেয়)
❌ টপোলজি ডিবাগ না করে প্রোডাকশনে যাওয়া
```

---

## 🔗 পরবর্তী টপিক

➡️ [Kafka Connect — ডেটা ইন্টিগ্রেশন](./kafka-connect.md)

Kafka Connect দিয়ে বিভিন্ন ডেটা সোর্স (MySQL, MongoDB, S3) থেকে Kafka-তে
এবং Kafka থেকে সিঙ্কে (Elasticsearch, PostgreSQL) ডেটা স্বয়ংক্রিয়ভাবে মুভ করা শেখা যাবে।

---

> **📝 নোট:** এই ফাইলের JavaScript উদাহরণগুলো kafkajs ব্যবহার করে Kafka Streams-এর
> ধারণাগুলো সিমুলেট করে। প্রোডাকশনে Java/Kotlin-এ Kafka Streams DSL সরাসরি ব্যবহার
> করা সর্বোত্তম। তবে ধারণাগুলো (filter, map, aggregate, windowing, join)
> সব ভাষাতেই একইভাবে কাজ করে।
