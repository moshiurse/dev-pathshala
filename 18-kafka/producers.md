# 📘 Kafka প্রডিউসার (Producers Deep Dive)

> **Kafka প্রডিউসার** হলো সেই কম্পোনেন্ট যা Kafka টপিকে মেসেজ পাঠায়।
> এটা শুনতে সহজ মনে হলেও, একটি প্রোডাকশন-গ্রেড প্রডিউসার বানাতে গেলে
> অনেক কিছু বুঝতে হয় — acks, batching, idempotency, transactions, error handling ইত্যাদি।

---

## 📑 সূচিপত্র

| ক্রম | টপিক |
|------|-------|
| ১ | [প্রডিউসার আর্কিটেকচার](#-প্রডিউসার-আর্কিটেকচার) |
| ২ | [Acks কনফিগারেশন](#-acks-কনফিগারেশন) |
| ৩ | [পার্টিশনিং কৌশল](#-পার্টিশনিং-কৌশল) |
| ৪ | [সিরিয়ালাইজেশন](#-সিরিয়ালাইজেশন) |
| ৫ | [ব্যাচিং ও কম্প্রেশন](#-ব্যাচিং-ও-কম্প্রেশন) |
| ৬ | [আইডেমপোটেন্ট প্রডিউসার](#-আইডেমপোটেন্ট-প্রডিউসার) |
| ৭ | [ট্রানজ্যাকশনাল প্রডিউসার](#-ট্রানজ্যাকশনাল-প্রডিউসার) |
| ৮ | [এরর হ্যান্ডলিং ও রিট্রাই](#-এরর-হ্যান্ডলিং-ও-রিট্রাই) |
| ৯ | [PHP প্রডিউসার উদাহরণ](#-php-প্রডিউসার-সম্পূর্ণ-উদাহরণ) |
| ১০ | [JS প্রডিউসার উদাহরণ](#-javascript-প্রডিউসার-সম্পূর্ণ-উদাহরণ) |
| ১১ | [বেস্ট প্র্যাকটিস](#-প্রডিউসার-বেস্ট-প্র্যাকটিস) |

---

## 📊 প্রডিউসার আর্কিটেকচার

প্রডিউসার শুধু `send()` কল করলেই কাজ শেষ না। ভেতরে অনেকগুলো ধাপ আছে।
প্রতিটি মেসেজ নিচের পাইপলাইনের মধ্য দিয়ে যায়:

### 🔄 প্রডিউসার ইন্টারনাল পাইপলাইন

```
┌─────────────────────────────────────────────────┐
│                   Producer                       │
│                                                  │
│  ┌──────────┐  ┌─────────────┐  ┌────────────┐  │
│  │  Record   │→│ Serializer  │→│ Partitioner │  │
│  │ (key,val) │  │ (key+value) │  │(partition#) │  │
│  └──────────┘  └─────────────┘  └─────┬──────┘  │
│                                       │          │
│                          ┌────────────▼───────┐  │
│                          │ Record Accumulator  │  │
│                          │ (batch per partition)│  │
│                          │                     │  │
│                          │ P0: [m1, m2, m3]   │  │
│                          │ P1: [m4, m5]       │  │
│                          │ P2: [m6]           │  │
│                          └────────────┬───────┘  │
│                                       │          │
│                          ┌────────────▼───────┐  │
│                          │   Sender Thread     │  │
│                          │  (background I/O)   │  │
│                          └────────────┬───────┘  │
└───────────────────────────────────────┼──────────┘
                                        │
                            ┌───────────▼────────┐
                            │    Kafka Broker     │
                            │  (Leader Partition) │
                            └────────────────────┘
```

### 📖 প্রতিটি ধাপের ব্যাখ্যা

| ধাপ | কম্পোনেন্ট | কাজ |
|-----|------------|------|
| ১ | **Record** | আপনার মেসেজ — key, value, headers, timestamp |
| ২ | **Serializer** | key ও value কে bytes এ রূপান্তর করে (JSON, Avro, Protobuf) |
| ৩ | **Partitioner** | কোন partition এ যাবে তা ঠিক করে (key hash বা round-robin) |
| ৪ | **Record Accumulator** | partition অনুযায়ী মেসেজ batch আকারে জমা করে |
| ৫ | **Sender Thread** | ব্যাকগ্রাউন্ডে batch গুলো broker এ পাঠায় |
| ৬ | **Kafka Broker** | মেসেজ receive করে, partition leader এ লেখে |

> 🏠 **বাস্তব উদাহরণ:** Daraz এ অর্ডার প্লেস হলে, অর্ডার ডেটা (Record) → JSON
> serialize হয় → order_id দিয়ে partition ঠিক হয় → batch এ জমা হয় → broker এ যায়।

---

## 🔧 Acks কনফিগারেশন

**acks** (acknowledgments) হলো সবচেয়ে গুরুত্বপূর্ণ প্রডিউসার কনফিগ।
এটা ঠিক করে — broker থেকে কতটুকু নিশ্চয়তা (confirmation) পেলে
প্রডিউসার মেসেজটাকে "সফল" বলে ধরবে।

### acks=0 (Fire and Forget)

```
Producer                    Broker
   │                          │
   │──── Send Message ───────→│
   │                          │  (কোনো response এর
   │  (অপেক্ষা করে না)        │   জন্য wait করে না)
   │                          │
   ▼                          ▼
 "সফল" ধরে নেয়           মেসেজ হারিয়ে
                           যেতে পারে!
```

- ব্রোকার রেসপন্স না পাঠালেও প্রডিউসার এগিয়ে যায়
- সবচেয়ে দ্রুত, কিন্তু মেসেজ হারানোর ঝুঁকি আছে
- **ব্যবহার:** লগিং, মেট্রিক্স — যেখানে কিছু ডেটা হারালে সমস্যা নেই

### acks=1 (Leader Acknowledgment)

```
Producer                    Leader Broker         Follower
   │                          │                     │
   │──── Send Message ───────→│                     │
   │                          │── Write to disk     │
   │                          │                     │
   │←── ACK (success) ───────│                     │
   │                          │── Replicate ───────→│
   ▼                          ▼      (পরে হবে)      ▼
 "সফল"                   লেখা হয়েছে         এখনো হয়নি
```

- Leader broker ডিস্কে লিখলেই ACK পাঠায়
- Follower এ replicate না হলেও ACK আসবে
- **ঝুঁকি:** Leader crash হলে, replicate না হওয়া ডেটা হারাতে পারে

### acks=all (-1) (All ISR Acknowledgment)

```
Producer                    Leader Broker         Followers (ISR)
   │                          │                     │
   │──── Send Message ───────→│                     │
   │                          │── Write to disk     │
   │                          │── Replicate ───────→│
   │                          │                     │── Write to disk
   │                          │←── ACK ────────────│
   │←── ACK (success) ───────│                     │
   ▼                          ▼                     ▼
 "সফল"                   লেখা হয়েছে          সবাই লিখেছে ✅
```

- সব In-Sync Replica (ISR) লেখা শেষ করলে তবেই ACK আসে
- সবচেয়ে নিরাপদ, কিন্তু সবচেয়ে ধীর
- **ব্যবহার:** পেমেন্ট, ফাইনান্সিয়াল ট্রানজ্যাকশন

### 📊 Acks তুলনা টেবিল

| acks | Latency | Durability | Throughput | Use Case (বাংলাদেশ উদাহরণ) |
|------|---------|------------|------------|----------------------------|
| `0` | 🟢 সবচেয়ে কম | 🔴 কোনো গ্যারান্টি নেই | 🟢 সর্বোচ্চ | Grameenphone app analytics লগ |
| `1` | 🟡 মাঝামাঝি | 🟡 Leader পর্যন্ত | 🟡 ভালো | Daraz প্রোডাক্ট ভিউ ইভেন্ট |
| `all` | 🔴 সবচেয়ে বেশি | 🟢 সম্পূর্ণ নিরাপদ | 🔴 সবচেয়ে কম | bKash/Nagad পেমেন্ট ইভেন্ট |

### ❌ খারাপ প্র্যাকটিস

```php
// ❌ bKash পেমেন্ট ইভেন্টে acks=0 ব্যবহার করা — টাকা হারিয়ে যেতে পারে!
$conf->set('acks', '0');
$producer->produce('bkash-payments', $paymentEvent);
// মেসেজ হারালে ১০০০ টাকার ট্রানজ্যাকশন হারিয়ে যাবে
```

### ✅ ভালো প্র্যাকটিস

```php
// ✅ bKash পেমেন্ট ইভেন্টে acks=all ব্যবহার — নিরাপদ!
$conf->set('acks', 'all');
$conf->set('min.insync.replicas', '2');
$producer->produce('bkash-payments', $paymentEvent);
// সব replica তে লেখা নিশ্চিত হলেই সফল
```

---

## 📦 পার্টিশনিং কৌশল

মেসেজ কোন partition এ যাবে সেটা Partitioner ঠিক করে।
সঠিক পার্টিশনিং কৌশল ব্যবহার করলে throughput, ordering এবং load balance ভালো হয়।

### কৌশল ১: Round-Robin (কোনো key নেই)

```
Messages: [m1] [m2] [m3] [m4] [m5] [m6]
              │
              ▼
     ┌─── Round Robin ───┐
     │                    │
     ▼        ▼        ▼
 ┌──────┐ ┌──────┐ ┌──────┐
 │ P-0  │ │ P-1  │ │ P-2  │
 │ m1   │ │ m2   │ │ m3   │
 │ m4   │ │ m5   │ │ m6   │
 └──────┘ └──────┘ └──────┘
```

- key না দিলে মেসেজ সমানভাবে সব partition এ ছড়িয়ে যায়
- **সুবিধা:** লোড ব্যালেন্সিং
- **অসুবিধা:** একই entity র মেসেজ আলাদা partition এ যায়, ordering নষ্ট হয়

### কৌশল ২: Key-Based Partitioning

```
Key: "user-101"  ──→ hash("user-101") % 3 = 1 ──→ Partition 1
Key: "user-202"  ──→ hash("user-202") % 3 = 0 ──→ Partition 0
Key: "user-101"  ──→ hash("user-101") % 3 = 1 ──→ Partition 1  (আবার P1!)

 ┌──────────┐  ┌──────────┐  ┌──────────┐
 │   P-0    │  │   P-1    │  │   P-2    │
 │ user-202 │  │ user-101 │  │ user-303 │
 │ user-404 │  │ user-101 │  │ user-505 │
 └──────────┘  └──────────┘  └──────────┘
```

- একই key সবসময় একই partition এ যায়
- **সুবিধা:** key-level ordering গ্যারান্টি
- **ব্যবহার:** Pathao তে `rider_id` দিয়ে key সেট করলে একই rider এর সব ইভেন্ট ordered থাকবে

### কৌশল ৩: Custom Partitioner

কখনো কখনো default partitioner যথেষ্ট না। নিজের partitioner লিখতে হয়।

**PHP Custom Partitioner:**

```php
<?php
class RegionBasedPartitioner
{
    private array $regionMap = [
        'dhaka'      => 0,
        'chittagong' => 1,
        'sylhet'     => 2,
        'rajshahi'   => 3,
        'khulna'     => 4,
    ];

    /**
     * বাংলাদেশের বিভাগ অনুযায়ী partition ঠিক করে।
     * Pathao রাইড সিস্টেমে — ঢাকার রাইড P0, চট্টগ্রামের P1 তে যাবে।
     */
    public function partition(string $region, int $totalPartitions): int
    {
        $region = strtolower($region);

        if (isset($this->regionMap[$region])) {
            return $this->regionMap[$region] % $totalPartitions;
        }

        // অচেনা region হলে hash ব্যবহার করো
        return crc32($region) % $totalPartitions;
    }
}

// ব্যবহার
$partitioner = new RegionBasedPartitioner();
$partition = $partitioner->partition('dhaka', 5); // 0
$partition = $partitioner->partition('chittagong', 5); // 1
```

**JavaScript Custom Partitioner (kafkajs):**

```javascript
const { Kafka } = require("kafkajs");

const kafka = new Kafka({ brokers: ["localhost:9092"] });

// বাংলাদেশের বিভাগ ভিত্তিক কাস্টম পার্টিশনার
const RegionPartitioner = () => {
  const regionMap = {
    dhaka: 0,
    chittagong: 1,
    sylhet: 2,
    rajshahi: 3,
    khulna: 4,
  };

  return ({ topic, partitionMetadata, message }) => {
    const region = message.key?.toString().toLowerCase();
    const numPartitions = partitionMetadata.length;

    if (region && regionMap[region] !== undefined) {
      return regionMap[region] % numPartitions;
    }

    // fallback: key এর hash
    const hash = message.key
      ? Buffer.from(message.key).reduce((a, b) => a + b, 0)
      : Math.floor(Math.random() * numPartitions);

    return hash % numPartitions;
  };
};

const producer = kafka.producer({
  createPartitioner: RegionPartitioner,
});
```

### 📊 কৌশল তুলনা

| কৌশল | Ordering | Load Balance | Use Case |
|-------|----------|-------------|----------|
| Round-Robin | ❌ নেই | ✅ সেরা | Grameenphone অ্যাপ লগ |
| Key-Based | ✅ key-level | 🟡 key distribution এর উপর নির্ভর | Daraz অর্ডার (order_id key) |
| Custom | ✅ নিয়ন্ত্রণযোগ্য | ✅ নিয়ন্ত্রণযোগ্য | Pathao বিভাগভিত্তিক রাউটিং |

---

## 🔄 সিরিয়ালাইজেশন

Kafka শুধু **bytes** বোঝে। তাই key ও value কে bytes এ রূপান্তর করতে হয়।
এই রূপান্তর প্রক্রিয়াকেই সিরিয়ালাইজেশন বলে।

### JSON সিরিয়ালাইজেশন

সবচেয়ে সহজ ও জনপ্রিয়। কিন্তু schema enforcement নেই।

```php
// PHP — JSON সিরিয়ালাইজেশন
$order = [
    'order_id'  => 'ORD-2024-001',
    'customer'  => 'রহিম উদ্দিন',
    'amount'    => 2500.00,
    'items'     => ['ল্যাপটপ ব্যাগ', 'মাউস'],
    'region'    => 'dhaka',
    'timestamp' => time(),
];

$serialized = json_encode($order, JSON_UNESCAPED_UNICODE);
$producer->produce('daraz-orders', 0, $serialized, 'ORD-2024-001');
```

```javascript
// JS — JSON সিরিয়ালাইজেশন
const order = {
  orderId: "ORD-2024-001",
  customer: "রহিম উদ্দিন",
  amount: 2500.0,
  items: ["ল্যাপটপ ব্যাগ", "মাউস"],
  region: "dhaka",
  timestamp: Date.now(),
};

await producer.send({
  topic: "daraz-orders",
  messages: [{ key: order.orderId, value: JSON.stringify(order) }],
});
```

### Avro সিরিয়ালাইজেশন (Schema Registry সহ)

বড় প্রজেক্টে Avro ব্যবহার করা হয় কারণ schema evolution সাপোর্ট করে।

```javascript
// JS — Avro সিরিয়ালাইজেশন (kafkajs + @kafkajs/confluent-schema-registry)
const { SchemaRegistry } = require("@kafkajs/confluent-schema-registry");

const registry = new SchemaRegistry({ host: "http://localhost:8081" });

const schema = {
  type: "record",
  name: "BkashPayment",
  fields: [
    { name: "txn_id", type: "string" },
    { name: "sender", type: "string" },
    { name: "receiver", type: "string" },
    { name: "amount", type: "double" },
    { name: "currency", type: { type: "enum", name: "Currency", symbols: ["BDT", "USD"] } },
  ],
};

const { id: schemaId } = await registry.register({
  type: "AVRO",
  schema: JSON.stringify(schema),
});

const encodedPayment = await registry.encode(schemaId, {
  txn_id: "TXN-8834923",
  sender: "01712345678",
  receiver: "01898765432",
  amount: 1500.0,
  currency: "BDT",
});

await producer.send({
  topic: "bkash-payments",
  messages: [{ key: "TXN-8834923", value: encodedPayment }],
});
```

### 📊 সিরিয়ালাইজেশন তুলনা

| বৈশিষ্ট্য | JSON | Avro | Protobuf |
|-----------|------|------|----------|
| Schema | ❌ নেই | ✅ Schema Registry | ✅ .proto ফাইল |
| সাইজ | 🔴 বড় (text) | 🟢 ছোট (binary) | 🟢 ছোট (binary) |
| পড়া সহজ | ✅ human-readable | ❌ binary | ❌ binary |
| Evolution | 🔴 কঠিন | ✅ backward/forward | ✅ backward/forward |
| Speed | 🟡 মাঝারি | 🟢 দ্রুত | 🟢 সবচেয়ে দ্রুত |
| ব্যবহার | ছোট প্রজেক্ট | বড় Kafka সিস্টেম | gRPC + Kafka |

---

## ⚡ ব্যাচিং ও কম্প্রেশন

প্রডিউসার প্রতিটি মেসেজ আলাদাভাবে পাঠায় না।
**ব্যাচিং** — একসাথে অনেক মেসেজ জড়ো করে একবারে পাঠায়।
**কম্প্রেশন** — ব্যাচ কম্প্রেস করে সাইজ কমায়।

### 📦 ব্যাচিং কনফিগারেশন

```
Messages: [m1] [m2] [m3] [m4] [m5] [m6] [m7] [m8]
                         │
                ┌────────▼────────┐
                │   Batch Buffer   │
                │  linger.ms = 5   │  ← ৫ms অপেক্ষা করে আরো মেসেজ আসলে
                │  batch.size=16KB │  ← ১৬KB পূর্ণ হলে আর অপেক্ষা না করে
                │                  │
                │ [m1,m2,m3,m4,m5] │  ← একটি ব্যাচ
                └────────┬────────┘
                         │
                ┌────────▼────────┐
                │   Compression    │
                │  type = snappy   │  ← ব্যাচ কম্প্রেস করে
                │  16KB → 4KB      │
                └────────┬────────┘
                         │
                ┌────────▼────────┐
                │  Send to Broker  │  ← একটি নেটওয়ার্ক কলে পুরো ব্যাচ
                └─────────────────┘
```

### 🔑 গুরুত্বপূর্ণ কনফিগ

| কনফিগ | ডিফল্ট | ব্যাখ্যা |
|--------|---------|----------|
| `batch.size` | 16384 (16KB) | এক ব্যাচের সর্বোচ্চ সাইজ (bytes) |
| `linger.ms` | 0 | মেসেজ জমার জন্য কতক্ষণ অপেক্ষা (milliseconds) |
| `buffer.memory` | 33554432 (32MB) | মোট buffer memory |
| `compression.type` | none | কম্প্রেশন — none, gzip, snappy, lz4, zstd |

### 🗜️ কম্প্রেশন তুলনা

| টাইপ | কম্প্রেশন রেশিও | CPU ব্যবহার | স্পিড | প্রস্তাবিত ক্ষেত্র |
|------|-----------------|------------|-------|---------------------|
| `none` | 1x | ❌ নেই | 🟢 সর্বোচ্চ | কম ডেটা, low latency |
| `gzip` | 🟢 সেরা (5-7x) | 🔴 বেশি | 🔴 ধীর | স্টোরেজ সাশ্রয় দরকার |
| `snappy` | 🟡 ভালো (3-4x) | 🟢 কম | 🟢 দ্রুত | ✅ সাধারণ ব্যবহারে প্রস্তাবিত |
| `lz4` | 🟡 ভালো (3-4x) | 🟢 কম | 🟢 সবচেয়ে দ্রুত | real-time সিস্টেম |
| `zstd` | 🟢 অনেক ভালো (5-6x) | 🟡 মাঝারি | 🟡 ভালো | বড় মেসেজ |

> 🏠 **Daraz উদাহরণ:** প্রতিদিন কোটি কোটি প্রোডাক্ট ভিউ ইভেন্ট হয়। `snappy`
> কম্প্রেশন ব্যবহার করলে নেটওয়ার্ক ব্যান্ডউইথ ৬০-৭০% কমে যায়।

---

## 🔒 আইডেমপোটেন্ট প্রডিউসার

### সমস্যা: ডুপ্লিকেট মেসেজ

নেটওয়ার্ক সমস্যার কারণে প্রডিউসার একই মেসেজ একাধিকবার পাঠাতে পারে:

```
Producer                         Broker
   │                               │
   │── Send msg (seq=1) ──────────→│  ✅ Broker লিখেছে
   │                               │
   │   (ACK হারিয়ে গেছে!!)        │── ACK ─── ✗ (network loss)
   │                               │
   │── Retry msg (seq=1) ─────────→│  😱 আবার লিখবে? DUPLICATE!
   │                               │
```

### সমাধান: Idempotent Producer

```
enable.idempotence = true

Producer (PID=7)                  Broker
   │                               │
   │── Send msg (PID=7, seq=1) ───→│  ✅ Broker লিখেছে
   │                               │      PID=7, seq=1 মনে রাখে
   │   (ACK হারিয়ে গেছে!!)        │
   │                               │
   │── Retry msg (PID=7, seq=1) ──→│  🛡️ "PID=7, seq=1 আগেই
   │                               │      আছে — SKIP করছি"
   │←── ACK (success) ────────────│
   ▼                               ▼
 একবারই লেখা হয়েছে ✅           ডুপ্লিকেট নেই ✅
```

### কিভাবে কাজ করে?

| কম্পোনেন্ট | ব্যাখ্যা |
|------------|----------|
| **Producer ID (PID)** | প্রতিটি প্রডিউসার ইনস্ট্যান্সের ইউনিক আইডি |
| **Sequence Number** | প্রতিটি partition এর জন্য ক্রমবর্ধমান নম্বর |
| **Broker Dedup** | Broker PID+seq দেখে ডুপ্লিকেট চেনে |

### কনফিগ

```php
// PHP
$conf->set('enable.idempotence', 'true');
// নিচেরগুলো automatically সেট হয়:
// acks = all
// retries = MAX_INT
// max.in.flight.requests.per.connection = 5
```

```javascript
// JS (kafkajs)
const producer = kafka.producer({
  idempotent: true,
  maxInFlightRequests: 5,
});
```

> 🏠 **Nagad উদাহরণ:** মোবাইল নেটওয়ার্ক অস্থিতিশীল হলে retry হয়।
> Idempotent producer ছাড়া একই ক্যাশ-আউট দুইবার হতে পারে!

---

## 💰 ট্রানজ্যাকশনাল প্রডিউসার

### সমস্যা: Atomic Multi-Partition Write

ধরুন bKash মানি ট্রান্সফারে দুটি ইভেন্ট পাঠাতে হবে:
1. Sender এর account থেকে **debit**
2. Receiver এর account এ **credit**

দুটো **একসাথে সফল** হতে হবে — একটা হয়ে আরেকটা না হলে সিস্টেম inconsistent!

```
                    ┌─── Transaction ───┐
                    │                   │
         ┌──────────▼──┐    ┌──────────▼──┐
         │ Topic:       │    │ Topic:       │
         │ account-debit│    │ account-credit│
         │ P0: -1500 BDT│    │ P2: +1500 BDT│
         └──────────────┘    └──────────────┘
                    │                   │
                    └─── COMMIT ────────┘
                    
            দুটোই সফল = COMMIT ✅
            যেকোনো একটি ব্যর্থ = ABORT ❌ (কিছুই হবে না)
```

### Transaction API Flow

```
Producer                              Kafka Broker
   │                                     │
   │── initTransactions() ──────────────→│  PID বরাদ্দ
   │                                     │
   │── beginTransaction() ─────────────→│  ট্রানজ্যাকশন শুরু
   │                                     │
   │── send(debit event) ─────────────→│  মেসেজ ১ (uncommitted)
   │── send(credit event) ────────────→│  মেসেজ ২ (uncommitted)
   │                                     │
   │  সব ঠিক থাকলে:                     │
   │── commitTransaction() ───────────→│  ✅ দুটোই COMMIT
   │                                     │
   │  সমস্যা হলে:                       │
   │── abortTransaction() ────────────→│  ❌ দুটোই ROLLBACK
   │                                     │
```

### PHP ট্রানজ্যাকশনাল প্রডিউসার

```php
<?php
class BkashTransferProducer
{
    private \RdKafka\Producer $producer;

    public function __construct()
    {
        $conf = new \RdKafka\Conf();
        $conf->set('bootstrap.servers', 'kafka1:9092,kafka2:9092');
        $conf->set('transactional.id', 'bkash-transfer-producer-01');
        $conf->set('enable.idempotence', 'true');
        $conf->set('acks', 'all');

        $this->producer = new \RdKafka\Producer($conf);
        $this->producer->initTransactions(10000); // 10s timeout
    }

    public function transfer(string $sender, string $receiver, float $amount): bool
    {
        try {
            $this->producer->beginTransaction();

            // Debit ইভেন্ট
            $debitTopic = $this->producer->newTopic('account-debits');
            $debitTopic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode([
                'account'   => $sender,
                'amount'    => -$amount,
                'currency'  => 'BDT',
                'txn_type'  => 'DEBIT',
                'timestamp' => microtime(true),
            ]), $sender);

            // Credit ইভেন্ট
            $creditTopic = $this->producer->newTopic('account-credits');
            $creditTopic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode([
                'account'   => $receiver,
                'amount'    => $amount,
                'currency'  => 'BDT',
                'txn_type'  => 'CREDIT',
                'timestamp' => microtime(true),
            ]), $receiver);

            $this->producer->flush(5000);
            $this->producer->commitTransaction(10000);

            return true;
        } catch (\Exception $e) {
            $this->producer->abortTransaction(10000);
            error_log("Transfer ব্যর্থ: " . $e->getMessage());
            return false;
        }
    }
}

// ব্যবহার
$bkash = new BkashTransferProducer();
$bkash->transfer('01712345678', '01898765432', 1500.00);
```

### JavaScript ট্রানজ্যাকশনাল প্রডিউসার

```javascript
const { Kafka } = require("kafkajs");

const kafka = new Kafka({ brokers: ["kafka1:9092", "kafka2:9092"] });

const producer = kafka.producer({
  transactionalId: "bkash-transfer-producer-01",
  idempotent: true,
  maxInFlightRequests: 1,
});

async function transfer(sender, receiver, amount) {
  const transaction = await producer.transaction();

  try {
    // Debit ইভেন্ট
    await transaction.send({
      topic: "account-debits",
      messages: [
        {
          key: sender,
          value: JSON.stringify({
            account: sender,
            amount: -amount,
            currency: "BDT",
            txnType: "DEBIT",
            timestamp: Date.now(),
          }),
        },
      ],
    });

    // Credit ইভেন্ট
    await transaction.send({
      topic: "account-credits",
      messages: [
        {
          key: receiver,
          value: JSON.stringify({
            account: receiver,
            amount: amount,
            currency: "BDT",
            txnType: "CREDIT",
            timestamp: Date.now(),
          }),
        },
      ],
    });

    await transaction.commit();
    console.log(`✅ ট্রান্সফার সফল: ${sender} → ${receiver}, ${amount} BDT`);
  } catch (error) {
    await transaction.abort();
    console.error(`❌ ট্রান্সফার ব্যর্থ: ${error.message}`);
    throw error;
  }
}

// ব্যবহার
await producer.connect();
await transfer("01712345678", "01898765432", 1500.0);
```

---

## ⚠️ এরর হ্যান্ডলিং ও রিট্রাই

প্রোডাকশনে এরর হওয়া স্বাভাবিক। গুরুত্বপূর্ণ হলো কিভাবে এরর হ্যান্ডেল করা হচ্ছে।

### Retriable vs Non-Retriable এরর

```
          ┌─────────────────────┐
          │    Error হয়েছে!    │
          └──────────┬──────────┘
                     │
            ┌────────▼────────┐
            │  Retriable?     │
            └───┬─────────┬───┘
               YES        NO
                │          │
      ┌─────────▼──┐  ┌───▼──────────────┐
      │   Retry     │  │  Dead Letter     │
      │ (backoff সহ)│  │  Queue (DLQ) তে  │
      └─────────┬──┘  │  পাঠাও           │
                │      └──────────────────┘
        সফল?   │
       ┌───┴───┐
      YES     NO (max retries)
       │       │
   ┌───▼──┐ ┌─▼──────────────┐
   │ Done ✅│ │ DLQ তে পাঠাও │
   └──────┘ └────────────────┘
```

| এরর টাইপ | Retriable? | উদাহরণ |
|-----------|-----------|--------|
| `LEADER_NOT_AVAILABLE` | ✅ হ্যাঁ | নতুন leader নির্বাচন হচ্ছে |
| `NOT_ENOUGH_REPLICAS` | ✅ হ্যাঁ | কিছু replica সাময়িকভাবে down |
| `NETWORK_EXCEPTION` | ✅ হ্যাঁ | নেটওয়ার্ক সাময়িকভাবে বিচ্ছিন্ন |
| `MESSAGE_TOO_LARGE` | ❌ না | মেসেজ সাইজ লিমিটের বেশি |
| `INVALID_TOPIC` | ❌ না | ভুল টপিক নাম |
| `AUTHORIZATION_FAILED` | ❌ না | permission নেই |

### 🔑 গুরুত্বপূর্ণ রিট্রাই কনফিগ

| কনফিগ | ডিফল্ট | ব্যাখ্যা |
|--------|---------|----------|
| `retries` | 2147483647 | সর্বোচ্চ retry সংখ্যা |
| `retry.backoff.ms` | 100 | retry এর মাঝে অপেক্ষা (ms) |
| `delivery.timeout.ms` | 120000 (2min) | মোট সময় সীমা |
| `max.in.flight.requests.per.connection` | 5 | একসাথে কতটি request পাঠানো যাবে |

### PHP এরর হ্যান্ডলিং উদাহরণ

```php
<?php
class ResilientProducer
{
    private \RdKafka\Producer $producer;
    private \RdKafka\Producer $dlqProducer;

    public function __construct()
    {
        $conf = new \RdKafka\Conf();
        $conf->set('bootstrap.servers', 'kafka1:9092,kafka2:9092');
        $conf->set('acks', 'all');
        $conf->set('retries', '5');
        $conf->set('retry.backoff.ms', '200');
        $conf->set('enable.idempotence', 'true');

        $conf->setDrMsgCb(function ($kafka, $message) {
            if ($message->err) {
                error_log("ডেলিভারি ব্যর্থ: " . rd_kafka_err2str($message->err));
                $this->sendToDLQ($message);
            }
        });

        $this->producer = new \RdKafka\Producer($conf);
        $this->dlqProducer = new \RdKafka\Producer($conf);
    }

    public function send(string $topic, string $key, array $data): void
    {
        $topicObj = $this->producer->newTopic($topic);
        $payload = json_encode($data, JSON_UNESCAPED_UNICODE);

        $topicObj->produce(RD_KAFKA_PARTITION_UA, 0, $payload, $key);
        $this->producer->poll(0);
    }

    private function sendToDLQ(\RdKafka\Message $failedMessage): void
    {
        $dlqTopic = $this->dlqProducer->newTopic('dead-letter-queue');
        $envelope = json_encode([
            'original_topic' => $failedMessage->topic_name,
            'original_key'   => $failedMessage->key,
            'payload'        => $failedMessage->payload,
            'error'          => rd_kafka_err2str($failedMessage->err),
            'failed_at'      => date('c'),
        ]);
        $dlqTopic->produce(RD_KAFKA_PARTITION_UA, 0, $envelope);
    }

    public function flush(): void
    {
        $result = $this->producer->flush(10000);
        if ($result !== RD_KAFKA_RESP_ERR_NO_ERROR) {
            throw new \RuntimeException('সব মেসেজ ফ্লাশ করা যায়নি');
        }
    }
}
```

### JavaScript এরর হ্যান্ডলিং উদাহরণ

```javascript
const { Kafka } = require("kafkajs");

const kafka = new Kafka({
  brokers: ["kafka1:9092", "kafka2:9092"],
  retry: {
    initialRetryTime: 200,
    retries: 5,
    maxRetryTime: 30000,
    factor: 2, // exponential backoff
  },
});

const producer = kafka.producer({ idempotent: true });
const dlqProducer = kafka.producer();

async function sendWithErrorHandling(topic, key, data) {
  try {
    await producer.send({
      topic,
      messages: [{ key, value: JSON.stringify(data) }],
    });
    console.log(`✅ মেসেজ পাঠানো হয়েছে: ${topic}/${key}`);
  } catch (error) {
    console.error(`❌ মেসেজ পাঠানো ব্যর্থ: ${error.message}`);

    // Dead Letter Queue তে পাঠাও
    try {
      await dlqProducer.send({
        topic: "dead-letter-queue",
        messages: [
          {
            key,
            value: JSON.stringify({
              originalTopic: topic,
              originalKey: key,
              payload: data,
              error: error.message,
              failedAt: new Date().toISOString(),
            }),
          },
        ],
      });
      console.log("📬 DLQ তে পাঠানো হয়েছে");
    } catch (dlqError) {
      console.error("🚨 DLQ তেও পাঠানো ব্যর্থ:", dlqError.message);
    }
  }
}
```

---

## 💻 PHP প্রডিউসার সম্পূর্ণ উদাহরণ

**Daraz-স্টাইল অর্ডার সিস্টেম** — php-rdkafka ব্যবহার করে পূর্ণাঙ্গ প্রডিউসার।

```php
<?php

declare(strict_types=1);

class DarazOrderProducer
{
    private \RdKafka\Producer $producer;
    private array $topicCache = [];

    private const TOPIC_ORDER_CREATED  = 'daraz.orders.created';
    private const TOPIC_ORDER_PAID     = 'daraz.orders.paid';
    private const TOPIC_ORDER_SHIPPED  = 'daraz.orders.shipped';

    public function __construct(array $brokers, string $transactionalId = null)
    {
        $conf = new \RdKafka\Conf();
        $conf->set('bootstrap.servers', implode(',', $brokers));
        $conf->set('acks', 'all');
        $conf->set('enable.idempotence', 'true');
        $conf->set('compression.type', 'snappy');
        $conf->set('linger.ms', '5');
        $conf->set('batch.size', '32768');
        $conf->set('retries', '5');

        if ($transactionalId) {
            $conf->set('transactional.id', $transactionalId);
        }

        $conf->setDrMsgCb(function ($kafka, $message) {
            if ($message->err) {
                error_log("[DarazProducer] ব্যর্থ: " . rd_kafka_err2str($message->err));
            }
        });

        $this->producer = new \RdKafka\Producer($conf);

        if ($transactionalId) {
            $this->producer->initTransactions(10000);
        }
    }

    public function publishOrderCreated(array $order): void
    {
        $event = [
            'event_type' => 'ORDER_CREATED',
            'order_id'   => $order['id'],
            'customer'   => $order['customer'],
            'items'      => $order['items'],
            'total'      => $order['total'],
            'currency'   => 'BDT',
            'region'     => $order['region'],
            'created_at' => date('c'),
        ];

        $this->produce(
            self::TOPIC_ORDER_CREATED,
            $order['id'],
            $event
        );
    }

    public function publishOrderPaid(string $orderId, string $paymentMethod, float $amount): void
    {
        $event = [
            'event_type'     => 'ORDER_PAID',
            'order_id'       => $orderId,
            'payment_method' => $paymentMethod,
            'amount'         => $amount,
            'currency'       => 'BDT',
            'paid_at'        => date('c'),
        ];

        $this->produce(self::TOPIC_ORDER_PAID, $orderId, $event);
    }

    public function publishOrderShipped(string $orderId, string $trackingId, string $courier): void
    {
        $event = [
            'event_type'  => 'ORDER_SHIPPED',
            'order_id'    => $orderId,
            'tracking_id' => $trackingId,
            'courier'     => $courier,
            'shipped_at'  => date('c'),
        ];

        $this->produce(self::TOPIC_ORDER_SHIPPED, $orderId, $event);
    }

    private function produce(string $topicName, string $key, array $data): void
    {
        if (!isset($this->topicCache[$topicName])) {
            $this->topicCache[$topicName] = $this->producer->newTopic($topicName);
        }

        $payload = json_encode($data, JSON_UNESCAPED_UNICODE | JSON_THROW_ON_ERROR);

        $this->topicCache[$topicName]->produce(
            RD_KAFKA_PARTITION_UA,
            0,
            $payload,
            $key
        );

        $this->producer->poll(0);
    }

    public function flush(int $timeoutMs = 10000): void
    {
        $result = $this->producer->flush($timeoutMs);
        if ($result !== RD_KAFKA_RESP_ERR_NO_ERROR) {
            throw new \RuntimeException('সব মেসেজ flush করা যায়নি');
        }
    }
}

// ========== ব্যবহারের উদাহরণ ==========
$producer = new DarazOrderProducer(
    brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
    transactionalId: 'daraz-order-producer-01'
);

$producer->publishOrderCreated([
    'id'       => 'ORD-2024-78923',
    'customer' => ['name' => 'করিম সাহেব', 'phone' => '01712345678'],
    'items'    => [
        ['sku' => 'LAPTOP-001', 'name' => 'Lenovo ThinkPad', 'qty' => 1, 'price' => 85000],
        ['sku' => 'MOUSE-042', 'name' => 'Logitech Mouse', 'qty' => 1, 'price' => 1500],
    ],
    'total'    => 86500.00,
    'region'   => 'dhaka',
]);

$producer->publishOrderPaid('ORD-2024-78923', 'bKash', 86500.00);
$producer->publishOrderShipped('ORD-2024-78923', 'TRK-99812', 'Pathao Courier');
$producer->flush();
```

---

## 💻 JavaScript প্রডিউসার সম্পূর্ণ উদাহরণ

**Pathao-স্টাইল রাইড সিস্টেম** — kafkajs ব্যবহার করে পূর্ণাঙ্গ প্রডিউসার।

```javascript
const { Kafka, Partitioners, CompressionTypes, logLevel } = require("kafkajs");

const TOPICS = {
  RIDE_REQUESTED: "pathao.rides.requested",
  RIDE_ACCEPTED: "pathao.rides.accepted",
  RIDE_COMPLETED: "pathao.rides.completed",
  RIDE_LOCATION: "pathao.rides.location",
};

class PathaoRideProducer {
  constructor(brokers, clientId = "pathao-ride-service") {
    this.kafka = new Kafka({
      clientId,
      brokers,
      logLevel: logLevel.WARN,
      retry: {
        initialRetryTime: 200,
        retries: 5,
        maxRetryTime: 30000,
      },
    });

    this.producer = this.kafka.producer({
      idempotent: true,
      maxInFlightRequests: 5,
      transactionalId: "pathao-ride-producer-01",
      createPartitioner: Partitioners.DefaultPartitioner,
    });
  }

  async connect() {
    await this.producer.connect();
    console.log("✅ Pathao Ride Producer সংযুক্ত হয়েছে");
  }

  async publishRideRequested(ride) {
    const event = {
      eventType: "RIDE_REQUESTED",
      rideId: ride.rideId,
      passenger: ride.passenger,
      pickup: ride.pickup,
      dropoff: ride.dropoff,
      vehicleType: ride.vehicleType,
      estimatedFare: ride.estimatedFare,
      currency: "BDT",
      requestedAt: new Date().toISOString(),
    };

    await this._send(TOPICS.RIDE_REQUESTED, ride.rideId, event);
  }

  async publishRideAccepted(rideId, driver) {
    const event = {
      eventType: "RIDE_ACCEPTED",
      rideId,
      driver: {
        id: driver.id,
        name: driver.name,
        phone: driver.phone,
        vehicleNumber: driver.vehicleNumber,
        rating: driver.rating,
      },
      acceptedAt: new Date().toISOString(),
    };

    await this._send(TOPICS.RIDE_ACCEPTED, rideId, event);
  }

  async publishRideCompleted(rideId, fare, distance) {
    const event = {
      eventType: "RIDE_COMPLETED",
      rideId,
      fare,
      distance,
      currency: "BDT",
      completedAt: new Date().toISOString(),
    };

    await this._send(TOPICS.RIDE_COMPLETED, rideId, event);
  }

  async publishLocationUpdate(rideId, lat, lng) {
    const event = {
      eventType: "LOCATION_UPDATE",
      rideId,
      location: { lat, lng },
      timestamp: Date.now(),
    };

    // লোকেশন আপডেটে acks=1 কারণ কিছু হারালে সমস্যা নেই
    await this.producer.send({
      topic: TOPICS.RIDE_LOCATION,
      compression: CompressionTypes.Snappy,
      messages: [{ key: rideId, value: JSON.stringify(event) }],
    });
  }

  async _send(topic, key, event) {
    await this.producer.send({
      topic,
      compression: CompressionTypes.Snappy,
      messages: [
        {
          key,
          value: JSON.stringify(event),
          headers: {
            "event-type": event.eventType,
            source: "pathao-ride-service",
          },
        },
      ],
    });
    console.log(`📤 [${topic}] ইভেন্ট পাঠানো হয়েছে: ${key}`);
  }

  async disconnect() {
    await this.producer.disconnect();
    console.log("🔌 Producer বিচ্ছিন্ন হয়েছে");
  }
}

// ========== ব্যবহারের উদাহরণ ==========
async function main() {
  const producer = new PathaoRideProducer([
    "kafka1:9092",
    "kafka2:9092",
    "kafka3:9092",
  ]);

  await producer.connect();

  // রাইড রিকোয়েস্ট
  await producer.publishRideRequested({
    rideId: "RIDE-2024-45678",
    passenger: { id: "USR-101", name: "আরিফ হোসেন", phone: "01712345678" },
    pickup: { lat: 23.8103, lng: 90.4125, address: "গুলশান ২, ঢাকা" },
    dropoff: { lat: 23.7461, lng: 90.3742, address: "ধানমন্ডি ২৭, ঢাকা" },
    vehicleType: "BIKE",
    estimatedFare: 120,
  });

  // ড্রাইভার অ্যাকসেপ্ট
  await producer.publishRideAccepted("RIDE-2024-45678", {
    id: "DRV-501",
    name: "জাহিদ হাসান",
    phone: "01898765432",
    vehicleNumber: "ঢাকা মেট্রো হ ১২-৩৪৫৬",
    rating: 4.8,
  });

  // রাইড সম্পন্ন
  await producer.publishRideCompleted("RIDE-2024-45678", 135, 8.5);

  await producer.disconnect();
}

main().catch(console.error);
```

---

## 🎯 প্রডিউসার বেস্ট প্র্যাকটিস

### ✅ করণীয় (DO)

| # | প্র্যাকটিস | কারণ |
|---|-----------|------|
| ১ | `acks=all` ব্যবহার করুন গুরুত্বপূর্ণ ডেটায় | ডেটা হারাবে না |
| ২ | `enable.idempotence=true` সেট করুন | ডুপ্লিকেট রোধ করবে |
| ৩ | ব্যাচিং ব্যবহার করুন (`linger.ms=5`) | throughput বাড়বে |
| ৪ | কম্প্রেশন ব্যবহার করুন (snappy/lz4) | নেটওয়ার্ক ব্যান্ডউইথ কমবে |
| ৫ | key ব্যবহার করুন ordering দরকার হলে | একই entity র মেসেজ ordered থাকবে |
| ৬ | Dead Letter Queue রাখুন | ব্যর্থ মেসেজ হারাবে না |
| ৭ | Delivery callback implement করুন | ব্যর্থতা ধরতে পারবেন |
| ৮ | `min.insync.replicas=2` সেট করুন | acks=all এর সাথে শক্তিশালী durability |
| ৯ | মেসেজে schema version রাখুন | ভবিষ্যতে schema পরিবর্তন সহজ হবে |
| ১০ | `flush()` কল করুন shutdown এ | buffer এ থাকা মেসেজ হারাবে না |

### ❌ বর্জনীয় (DON'T)

| # | প্র্যাকটিস | ঝুঁকি |
|---|-----------|------|
| ১ | পেমেন্টে `acks=0` ব্যবহার | টাকা হারিয়ে যেতে পারে |
| ২ | খুব বড় মেসেজ পাঠানো (>1MB) | broker performance খারাপ হবে |
| ৩ | sync produce ব্যবহার করা লুপে | throughput মারাত্মক কমবে |
| ৪ | error callback ছাড়া produce | নীরবে মেসেজ হারাবে |
| ৫ | Partition সংখ্যা ঘন ঘন পরিবর্তন | key-based ordering ভেঙে যাবে |
| ৬ | একই `transactional.id` একাধিক instance এ | fencing error হবে |
| ৭ | `flush()` না করে process exit | buffer এর মেসেজ হারাবে |

### 📋 প্রোডাকশন চেকলিস্ট

```
☐ acks কনফিগ ডেটার গুরুত্ব অনুযায়ী সেট করা হয়েছে
☐ Idempotence enable করা হয়েছে
☐ Retry ও backoff কনফিগ করা হয়েছে
☐ Dead Letter Queue সেটআপ করা হয়েছে
☐ Compression enable করা হয়েছে (snappy/lz4)
☐ Batching টিউন করা হয়েছে (linger.ms, batch.size)
☐ Delivery callback implement করা হয়েছে
☐ Key strategy ঠিক করা হয়েছে
☐ Monitoring সেটআপ করা হয়েছে (produce rate, error rate, latency)
☐ Graceful shutdown এ flush() কল হচ্ছে
```

---

## 🔗 পরবর্তী টপিক

| টপিক | ফাইল | বিষয় |
|-------|------|-------|
| ➡️ পরবর্তী | [consumers.md](./consumers.md) | কনজিউমার গ্রুপ, অফসেট ম্যানেজমেন্ট, রিব্যালান্সিং |
| 🔙 আগের | [README.md](./README.md) | Kafka ফান্ডামেন্টালস |

---

> 📝 **লেখক নোট:** এই ডকুমেন্টে PHP (rdkafka) এবং JavaScript (kafkajs)
> দুটো ইকোসিস্টেমেই উদাহরণ দেওয়া হয়েছে। আপনার প্রজেক্টে যেটি ব্যবহার করেন সেই
> অংশ ফলো করুন। কনসেপ্ট একই — শুধু syntax আলাদা।
