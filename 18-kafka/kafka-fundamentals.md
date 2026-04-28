# 📘 Apache Kafka — মৌলিক ধারণা (Fundamentals)

> _"I thought that since Kafka was a system optimized for writing, using a writer's name would make sense. I had taken a lot of lit classes in college and liked Franz Kafka."_
> — **Jay Kreps**, Co-creator of Apache Kafka

---

## 🧭 সূচিপত্র (Table of Contents)

- [📖 Apache Kafka কী?](#-apache-kafka-কী)
- [🏠 বাস্তব জীবনের উদাহরণ](#-বাস্তব-জীবনের-উদাহরণ-real-world-analogy)
- [📊 মূল ধারণাসমূহ (Core Concepts)](#-মূল-ধারণাসমূহ-core-concepts)
- [📊 Kafka vs Traditional Message Queues](#-kafka-vs-traditional-message-queues)
- [💻 PHP কোড উদাহরণ (rdkafka)](#-php-কোড-উদাহরণ-rdkafka)
- [💻 JavaScript কোড উদাহরণ (kafkajs)](#-javascript-কোড-উদাহরণ-kafkajs)
- [✅ কখন Kafka ব্যবহার করবেন](#-কখন-kafka-ব্যবহার-করবেন)
- [❌ কখন Kafka ব্যবহার করবেন না](#-কখন-kafka-ব্যবহার-করবেন-না)
- [🎯 মূল পয়েন্ট সারসংক্ষেপ](#-মূল-পয়েন্ট-সারসংক্ষেপ)
- [🔗 পরবর্তী টপিক](#-পরবর্তী-টপিক)

---

## 📖 Apache Kafka কী?

**Apache Kafka** হলো একটি **distributed event streaming platform** যা high-throughput, fault-tolerant, এবং scalable মেসেজ প্রসেসিংয়ের জন্য ডিজাইন করা হয়েছে।

সহজ ভাষায় বললে — Kafka হচ্ছে একটি অত্যন্ত দ্রুতগতির **ডিজিটাল ডাকঘর** যেখানে লক্ষ লক্ষ চিঠি (message/event) প্রতি সেকেন্ডে আদান-প্রদান হতে পারে, কোনো চিঠি কখনো হারায় না, এবং একটি চিঠি অনেকে একসাথে পড়তে পারে।

### 🏛️ Kafka-এর জন্ম কাহিনী

**২০১০ সালে LinkedIn-এ** তৈরি হয় Kafka। সেই সময় LinkedIn-এ প্রতিদিন বিলিয়ন বিলিয়ন ইভেন্ট জেনারেট হতো — ইউজার প্রোফাইল ভিউ, জব পোস্ট, মেসেজ, নোটিফিকেশন ইত্যাদি। traditional message queue দিয়ে এত ডেটা handle করা সম্ভব হচ্ছিলো না।

**Jay Kreps**, **Neha Narkhede**, এবং **Jun Rao** — এই তিনজন LinkedIn ইঞ্জিনিয়ার মিলে Kafka তৈরি করেন। পরে ২০১১ সালে এটি **Apache Software Foundation**-এ open source করা হয়।

### 📛 নামকরণ কেন "Kafka"?

জার্মান-ভাষী চেক লেখক **Franz Kafka**-এর নামে এটির নামকরণ। Jay Kreps কলেজে সাহিত্যের ক্লাসে Franz Kafka পড়তেন। যেহেতু এই সিস্টেমটি "writing" (ডেটা লেখা)-এর জন্য optimized, তাই একজন বিখ্যাত writer-এর নাম ব্যবহার করাটা যথার্থ মনে হয়েছিল।

### 🏤 সহজ উপমা — ডাকঘরের সাথে তুলনা

```
  🏤 Traditional Message Queue           🏤 Kafka
  (সাধারণ ডাকঘর)                        (আধুনিক ডিজিটাল ডাকঘর)

  - চিঠি পড়লে মুছে যায়                - চিঠি পড়লেও থেকে যায়
  - একজনই পড়তে পারে                   - অনেকে একসাথে পড়তে পারে
  - ধীরগতি                              - অত্যন্ত দ্রুতগতি
  - সীমিত ধারণক্ষমতা                     - প্রায় সীমাহীন ধারণক্ষমতা
  - একটি ডাকঘর                          - অনেক ডাকঘর মিলে কাজ করে (Cluster)
```

---

## 🏠 বাস্তব জীবনের উদাহরণ (Real-world Analogy)

ধরুন, বাংলাদেশের **"প্রথম আলো"** পত্রিকা বিতরণ ব্যবস্থা দিয়ে Kafka বোঝার চেষ্টা করি:

```
┌─────────────────────────────────────────────────────────────────────┐
│               📰 পত্রিকা বিতরণ ব্যবস্থা = Kafka                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  📝 সাংবাদিক/সম্পাদক           ═══>    🏭 প্রিন্টিং প্রেস          │
│  (Producer)                             (Kafka Broker)              │
│                                                                     │
│  📰 পত্রিকার কপি               ═══>    📨 Message/Event             │
│  (একটি সংবাদ)                          (একটি ইভেন্ট)               │
│                                                                     │
│  📂 বিভিন্ন বিভাগ               ═══>    📁 Topics                    │
│  (খেলা, রাজনীতি, বিনোদন)               (order, payment, delivery)  │
│                                                                     │
│  🏬 আঞ্চলিক বিতরণ কেন্দ্র       ═══>    🖥️ Brokers                  │
│  (ঢাকা, চট্টগ্রাম, সিলেট)              (Server 1, Server 2, ...)   │
│                                                                     │
│  🛵 হকারদের রুট                 ═══>    📊 Partitions               │
│  (রুট ১, রুট ২, রুট ৩)                 (Partition 0, 1, 2)         │
│                                                                     │
│  🏠 গ্রাহক                      ═══>    👤 Consumer                  │
│  (যারা পত্রিকা পড়েন)                   (যারা মেসেজ পড়ে)            │
│                                                                     │
│  👨‍👩‍👧‍👦 পরিবার (একটাই কপি শেয়ার)  ═══>    👥 Consumer Group            │
│  (একটি গ্রুপ মিলে পড়ে)                 (গ্রুপ মিলে মেসেজ ভাগ করে)  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 🔍 বিস্তারিত ব্যাখ্যা

**প্রথম আলো অফিস (Producer)** প্রতিদিন সংবাদ তৈরি করে। এই সংবাদগুলো বিভিন্ন ক্যাটাগরিতে ভাগ করা হয় — খেলা, রাজনীতি, বিনোদন (Kafka-তে এগুলো **Topics**)।

তারপর সংবাদগুলো **প্রিন্টিং প্রেসে (Broker)** যায়। একটি বড় পত্রিকার একাধিক প্রিন্টিং প্রেস থাকে — ঢাকায় একটি, চট্টগ্রামে একটি (Kafka Cluster-এ একাধিক Broker)।

প্রতিটি এলাকায় **হকারদের নির্দিষ্ট রুট (Partition)** থাকে। ধানমন্ডি এলাকার হকার শুধু ধানমন্ডিতে বিলি করে, গুলশানের হকার শুধু গুলশানে। এভাবে **সমান্তরালভাবে (parallel)** বিতরণ হয়।

**গ্রাহক (Consumer)** তার নিজের সুবিধামতো পত্রিকা পড়তে পারে। একটি **পরিবার (Consumer Group)** পুরো পত্রিকাটা ভাগ করে পড়ে — বাবা রাজনীতি, মা বিনোদন, ছেলে খেলা পড়ে।

---

## 📊 মূল ধারণাসমূহ (Core Concepts)

### 📁 Topics (টপিক)

**Topic** হচ্ছে Kafka-তে মেসেজ সংরক্ষণের **লজিক্যাল চ্যানেল বা ক্যাটাগরি**। এটিকে একটি ডাটাবেসের table-এর সাথে তুলনা করা যায়, তবে আচরণ অনেকটাই আলাদা।

বাস্তব উদাহরণ: **Daraz Bangladesh**-এ বিভিন্ন টপিক থাকতে পারে:
- `order-placed` — নতুন অর্ডার এসেছে
- `payment-completed` — পেমেন্ট সফল হয়েছে
- `delivery-status` — ডেলিভারি স্ট্যাটাস আপডেট
- `user-activity` — ইউজার কী কী করছে

```
┌─────────────────────────────────────────────────┐
│                Kafka Cluster                     │
│                                                  │
│   ┌──────────────────┐                           │
│   │ Topic:           │                           │
│   │ "order-placed"   │  ← Daraz-এ নতুন অর্ডার   │
│   │  P0 P1 P2        │                           │
│   └──────────────────┘                           │
│                                                  │
│   ┌──────────────────┐                           │
│   │ Topic:           │                           │
│   │ "payment-done"   │  ← bKash/Nagad পেমেন্ট   │
│   │  P0 P1           │                           │
│   └──────────────────┘                           │
│                                                  │
│   ┌──────────────────┐                           │
│   │ Topic:           │                           │
│   │ "ride-updates"   │  ← Pathao রাইড আপডেট     │
│   │  P0 P1 P2 P3     │                           │
│   └──────────────────┘                           │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 📊 Partitions (পার্টিশন)

প্রতিটি Topic-কে এক বা একাধিক **Partition**-এ ভাগ করা হয়। Partition হচ্ছে Kafka-র **parallelism-এর মূল ভিত্তি**। প্রতিটি partition একটি ordered, immutable sequence of messages।

**কেন Partition দরকার?** ধরুন, **bKash**-এ প্রতি সেকেন্ডে ১০,০০০ ট্রানজ্যাকশন হচ্ছে। একটি মাত্র queue দিয়ে এটা handle করা অসম্ভব। কিন্তু যদি ১০টি partition থাকে, তাহলে প্রতিটি partition প্রতি সেকেন্ডে ১,০০০ ট্রানজ্যাকশন handle করবে — **সমান্তরালভাবে**!

```
Topic: "bkash-transactions" (৩টি Partition)

  Partition 0                    Partition 1                    Partition 2
  ┌────┬────┬────┬────┬────┐    ┌────┬────┬────┬────┬────┐    ┌────┬────┬────┬────┐
  │ 0  │ 1  │ 2  │ 3  │ 4  │    │ 0  │ 1  │ 2  │ 3  │ 4  │    │ 0  │ 1  │ 2  │ 3  │
  │ ৳50│৳200│৳75 │৳30 │৳500│    │৳100│৳80 │৳150│৳90 │৳300│    │ ৳60│৳400│৳120│৳70 │
  └────┴────┴────┴────┴────┘    └────┴────┴────┴────┴────┘    └────┴────┴────┴────┘
  ──────────────────────►       ──────────────────────►       ──────────────────►
       সময়ের দিকে                    সময়ের দিকে                 সময়ের দিকে

  📌 প্রতিটি ঘরের উপরের সংখ্যা = Offset (০ থেকে শুরু)
  📌 প্রতিটি partition-এ মেসেজ ক্রমানুসারে সাজানো (ordered)
  📌 কিন্তু partition গুলোর মধ্যে কোনো ক্রম-গ্যারান্টি নেই
```

### 🔢 Offsets (অফসেট)

**Offset** হলো প্রতিটি partition-এর মধ্যে প্রতিটি মেসেজের **unique sequential ID**। এটি ০ থেকে শুরু হয় এবং প্রতিটি নতুন মেসেজে ১ করে বাড়ে।

```
Partition 0:
  Offset:  0      1      2      3      4      5
           ┌──────┬──────┬──────┬──────┬──────┬──────┐
           │ Msg0 │ Msg1 │ Msg2 │ Msg3 │ Msg4 │ Msg5 │
           └──────┴──────┴──────┴──────┴──────┴──────┘
                                    ▲
                                    │
                          Consumer এখানে পড়ছে
                          (Current Offset = 3)

  📌 Consumer তার offset মনে রাখে (commit করে)
  📌 Consumer restart হলে, শেষ committed offset থেকে আবার পড়া শুরু করে
  📌 চাইলে অতীতের offset-এ গিয়ে আবার পড়া সম্ভব (replay)!
```

মনে রাখবেন — offset শুধুমাত্র **একটি partition-এর মধ্যে** unique। বিভিন্ন partition-এ একই offset থাকতে পারে।

### 🖥️ Brokers (ব্রোকার)

**Broker** হচ্ছে Kafka-র প্রতিটি **সার্ভার ইনস্ট্যান্স**। এটি মেসেজ গ্রহণ, সংরক্ষণ এবং consumer-দের কাছে পাঠানোর কাজ করে।

**বাস্তব উদাহরণ:** **Grameenphone**-এর কাস্টমার সার্ভিস সেন্টার ভাবুন — ঢাকায় একটি, চট্টগ্রামে একটি, সিলেটে একটি। প্রতিটি সেন্টার (Broker) স্বাধীনভাবে কাজ করে, কিন্তু সবাই মিলে একটি সেবা (Cluster) দেয়।

```
┌─────────────────────────────────────────────────────────────┐
│                      Kafka Cluster                           │
│                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐      │
│   │  Broker 0   │   │  Broker 1   │   │  Broker 2   │      │
│   │  (Server 1) │   │  (Server 2) │   │  (Server 3) │      │
│   │             │   │             │   │             │      │
│   │ Topic-A P0  │   │ Topic-A P1  │   │ Topic-A P2  │      │
│   │ Topic-B P0  │   │ Topic-B P1  │   │ Topic-C P0  │      │
│   │ Topic-C P1  │   │ Topic-C P2  │   │ Topic-B P2  │      │
│   └─────────────┘   └─────────────┘   └─────────────┘      │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            │                                 │
│                   ZooKeeper / KRaft                           │
│                   (Cluster Coordinator)                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘

📌 প্রতিটি Broker-এর একটি unique ID আছে (0, 1, 2, ...)
📌 Topic-এর partition গুলো বিভিন্ন broker-এ ছড়িয়ে থাকে
📌 একটি broker ডাউন হলেও বাকিরা কাজ চালিয়ে যায় (fault-tolerant)
```

### 🌐 Clusters (ক্লাস্টার)

**Cluster** হলো একাধিক Broker মিলে গঠিত একটি গ্রুপ। একটি production Kafka cluster-এ সাধারণত **৩ থেকে শত শত** broker থাকতে পারে।

Cluster-এর সুবিধা:
- **Fault Tolerance** — একটি broker মারা গেলেও সিস্টেম চালু থাকে
- **Scalability** — নতুন broker যুক্ত করে ক্ষমতা বাড়ানো যায়
- **Load Balancing** — ডেটা সমানভাবে ভাগ করা হয়

### 📨 Messages / Records (মেসেজ / রেকর্ড)

Kafka-তে প্রতিটি মেসেজকে **Record** বলা হয়। একটি Record-এর গঠন এরকম:

```
┌─────────────────────────────────────────────────────────────┐
│                     Kafka Record / Message                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🔑 Key (ঐচ্ছিক)                                      │   │
│  │    উদাহরণ: "user-12345"                               │   │
│  │    ব্যবহার: কোন partition-এ যাবে তা নির্ধারণ করে       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 📦 Value (মূল ডেটা)                                   │   │
│  │    উদাহরণ: {                                          │   │
│  │      "orderId": "ORD-98765",                          │   │
│  │      "userId": "user-12345",                          │   │
│  │      "amount": 2500,                                  │   │
│  │      "currency": "BDT",                               │   │
│  │      "items": ["T-shirt", "Mouse"]                    │   │
│  │    }                                                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ⏰ Timestamp                                          │   │
│  │    উদাহরণ: 1700000000000 (Unix milliseconds)         │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 📋 Headers (ঐচ্ছিক metadata)                         │   │
│  │    উদাহরণ: { "source": "daraz-app",                  │   │
│  │              "version": "2.1",                        │   │
│  │              "correlation-id": "abc-123" }            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 📤 Producers (প্রডিউসার)

**Producer** হলো যে অ্যাপ্লিকেশন Kafka-তে মেসেজ **পাঠায় (publish করে)**।

**বাস্তব উদাহরণ:** **Daraz** অ্যাপে কেউ যখন অর্ডার দেয়, তখন Order Service (Producer) একটি মেসেজ Kafka-তে পাঠায়:

```
Producer (Daraz Order Service)
     │
     │  "নতুন অর্ডার এসেছে!"
     │
     ▼
┌─────────────┐
│ Topic:       │
│ "orders"     │
│              │
│  P0: [..... │ ◄── key="dhaka-user" হলে P0-তে যাবে
│  P1: [..... │ ◄── key="ctg-user" হলে P1-তে যাবে
│  P2: [..... │ ◄── key="sylhet-user" হলে P2-তে যাবে
│              │
└─────────────┘

📌 Producer নির্ধারণ করে কোন partition-এ মেসেজ যাবে:
   - Key থাকলে → hash(key) % partition_count
   - Key না থাকলে → Round-robin বা Sticky partitioning
```

Producer-এর **acks (acknowledgement)** সেটিং:
- `acks=0` — কনফার্মেশন ছাড়াই পাঠায় (দ্রুততম, কিন্তু মেসেজ হারাতে পারে)
- `acks=1` — শুধু leader broker কনফার্ম করলেই চলে
- `acks=all` — সব replica কনফার্ম করলে তবে সফল (সবচেয়ে নিরাপদ)

### 📥 Consumers (কনজিউমার)

**Consumer** হলো যে অ্যাপ্লিকেশন Kafka থেকে মেসেজ **পড়ে (subscribe করে)**।

গুরুত্বপূর্ণ বিষয়:
- Consumer মেসেজ **pull** করে (Kafka push করে না)
- মেসেজ পড়ার পরেও Kafka থেকে **মুছে যায় না** (retention period পর্যন্ত থাকে)
- Consumer তার **offset** track করে — কোথায় পর্যন্ত পড়েছে

### 👥 Consumer Groups (কনজিউমার গ্রুপ)

**Consumer Group** হচ্ছে একই টপিক থেকে **সমান্তরালভাবে (parallel)** মেসেজ পড়ার কৌশল। একটি গ্রুপের প্রতিটি consumer আলাদা partition থেকে পড়ে।

```
Topic: "nagad-transactions" (4 Partitions)

  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │  P0  │  │  P1  │  │  P2  │  │  P3  │
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │         │
     ▼         ▼         ▼         ▼
  ┌─────────────────────────────────────┐
  │     Consumer Group: "fraud-check"    │
  │                                      │
  │  Consumer A ◄── P0, P1              │
  │  Consumer B ◄── P2, P3              │
  │                                      │
  └─────────────────────────────────────┘

  ┌─────────────────────────────────────┐
  │     Consumer Group: "analytics"      │
  │                                      │
  │  Consumer X ◄── P0                  │
  │  Consumer Y ◄── P1                  │
  │  Consumer Z ◄── P2, P3              │
  │                                      │
  └─────────────────────────────────────┘

📌 একটি partition কেবল একটি group-এর একটি consumer-কে assign হয়
📌 কিন্তু ভিন্ন group-এর consumer-রা একই partition পড়তে পারে
📌 Consumer > Partition হলে, অতিরিক্ত consumer idle থাকে
```

**Consumer Group-এর নিয়ম:**

| পরিস্থিতি | ফলাফল |
|---|---|
| ২টি consumer, ৪টি partition | প্রতিটি consumer ২টি করে partition পায় |
| ৪টি consumer, ৪টি partition | প্রতিটি consumer ১টি করে partition পায় (আদর্শ!) |
| ৬টি consumer, ৪টি partition | ৪টি consumer কাজ করে, ২টি idle থাকে |
| ১টি consumer, ৪টি partition | একটি consumer-ই সব partition পড়ে |

---

## 📊 Kafka vs Traditional Message Queues

**Nagad**-এর মতো একটি ফিনটেক কোম্পানি যখন message broker বাছাই করবে, তখন এই তুলনাটি কাজে আসবে:

| বৈশিষ্ট্য | Apache Kafka | RabbitMQ | ActiveMQ |
|---|---|---|---|
| **আর্কিটেকচার** | Distributed log | Message broker | Message broker |
| **ডেলিভারি মডেল** | Pull-based (Consumer টানে) | Push-based (Broker ঠেলে দেয়) | Push ও Pull উভয় |
| **মেসেজ সংরক্ষণ** | ✅ ডিস্কে সংরক্ষণ (configurable retention) | ❌ পড়লে মুছে যায় (acknowledge পরে) | ❌ পড়লে মুছে যায় |
| **Throughput** | 🚀 প্রতি সেকেন্ডে লক্ষ লক্ষ মেসেজ | 🏃 প্রতি সেকেন্ডে হাজার হাজার | 🚶 মাঝারি |
| **মেসেজ Ordering** | ✅ Partition-এর মধ্যে guaranteed | ❌ গ্যারান্টি নেই (by default) | ❌ সীমিত গ্যারান্টি |
| **মেসেজ Replay** | ✅ যেকোনো সময় পুনরায় পড়া যায় | ❌ একবার পড়লে শেষ | ❌ একবার পড়লে শেষ |
| **Scalability** | ✅ অত্যন্ত scalable (partition দিয়ে) | ⚠️ সীমিত (clustering জটিল) | ⚠️ সীমিত |
| **Routing** | ❌ সরল (key-based partitioning) | ✅ জটিল routing সম্ভব (Exchange, Binding) | ✅ ভালো routing |
| **Protocol** | নিজস্ব binary protocol | AMQP, MQTT, STOMP | JMS, AMQP, STOMP |
| **Latency** | মিলিসেকেন্ড (কিন্তু batch হলে বেশি) | মাইক্রোসেকেন্ড (খুবই কম) | মিলিসেকেন্ড |
| **মূল Use Case** | Event streaming, log aggregation | Task queues, RPC | Enterprise messaging |
| **বাংলাদেশে উদাহরণ** | bKash ট্রানজ্যাকশন লগ | ইমেইল নোটিফিকেশন কিউ | ব্যাংকিং মেসেজিং |

### 🤔 কোনটি কখন?

```
                    তোমার সিদ্ধান্ত গাইড
                    ═══════════════════

  প্রশ্ন: প্রতি সেকেন্ডে কত মেসেজ?

  ১০০০+ ──────► Kafka ✅
  <১০০  ──────► RabbitMQ ভালো হতে পারে

  প্রশ্ন: মেসেজ পরে আবার পড়তে হবে (replay)?

  হ্যাঁ ────────► Kafka ✅ (এটাই Kafka-র বড় সুবিধা)
  না ─────────► RabbitMQ/ActiveMQ চলবে

  প্রশ্ন: জটিল routing দরকার?

  হ্যাঁ ────────► RabbitMQ ✅
  না ─────────► Kafka চলবে

  প্রশ্ন: Event sourcing / CQRS দরকার?

  হ্যাঁ ────────► Kafka ✅ (এটার জন্যই Kafka তৈরি)
  না ─────────► যেকোনোটি
```

---

## 💻 PHP কোড উদাহরণ (rdkafka)

**php-rdkafka** extension ব্যবহার করে Kafka-র সাথে PHP কানেক্ট করা হয়। নিচে **Daraz-এর** মতো একটি ই-কমার্স সিস্টেমের অর্ডার ইভেন্ট পাঠানো ও গ্রহণের উদাহরণ দেওয়া হলো:

### 📤 PHP Producer — অর্ডার ইভেন্ট পাঠানো

```php
<?php
/**
 * Kafka Producer Example
 * Daraz-এর মতো সিস্টেমে নতুন অর্ডারের ইভেন্ট Kafka-তে পাঠানো
 *
 * প্রয়োজন: php-rdkafka extension
 * ইনস্টল: pecl install rdkafka
 */

$config = new RdKafka\Conf();

// Broker-এর ঠিকানা (একাধিক broker comma দিয়ে)
$config->set('metadata.broker.list', 'kafka-broker1:9092,kafka-broker2:9092');

// acks=all মানে সব replica confirm করলে তবেই সফল (সবচেয়ে নিরাপদ)
$config->set('acks', 'all');

// মেসেজ পাঠানো ব্যর্থ হলে ৩ বার retry করবে
$config->set('retries', '3');

// Delivery report callback — মেসেজ সফলভাবে পৌঁছেছে কিনা জানায়
$config->setDrMsgCb(function (RdKafka\Producer $kafka, RdKafka\Message $message) {
    if ($message->err) {
        echo "❌ মেসেজ পাঠানো ব্যর্থ: " . rd_kafka_err2str($message->err) . "\n";
    } else {
        echo "✅ মেসেজ সফল — Topic: {$message->topic_name}, "
           . "Partition: {$message->partition}, Offset: {$message->offset}\n";
    }
});

$producer = new RdKafka\Producer($config);

// 'order-events' টপিকে মেসেজ পাঠাবো
$topic = $producer->newTopic('order-events');

// ৫টি অর্ডার ইভেন্ট পাঠানো হচ্ছে
$orders = [
    ['orderId' => 'ORD-1001', 'userId' => 'USR-501', 'amount' => 2500, 'city' => 'ঢাকা'],
    ['orderId' => 'ORD-1002', 'userId' => 'USR-302', 'amount' => 1200, 'city' => 'চট্টগ্রাম'],
    ['orderId' => 'ORD-1003', 'userId' => 'USR-501', 'amount' => 800,  'city' => 'ঢাকা'],
    ['orderId' => 'ORD-1004', 'userId' => 'USR-150', 'amount' => 5600, 'city' => 'সিলেট'],
    ['orderId' => 'ORD-1005', 'userId' => 'USR-302', 'amount' => 3200, 'city' => 'চট্টগ্রাম'],
];

foreach ($orders as $order) {
    $payload = json_encode([
        'event'     => 'ORDER_PLACED',
        'data'      => $order,
        'timestamp' => date('c'),
        'source'    => 'daraz-order-service',
    ]);

    // key হিসেবে userId ব্যবহার করলে একই ইউজারের সব অর্ডার একই partition-এ যাবে
    $topic->produce(
        RD_KAFKA_PARTITION_UA,   // Kafka নিজে partition ঠিক করবে
        0,                       // flags
        $payload,                // value (মূল ডেটা)
        $order['userId']         // key (partitioning-এর জন্য)
    );

    // internal queue flush করতে poll() ডাকতে হয়
    $producer->poll(0);
}

// সব মেসেজ পাঠানো শেষ না হওয়া পর্যন্ত অপেক্ষা (সর্বোচ্চ ১০ সেকেন্ড)
$result = $producer->flush(10000);
if ($result === RD_KAFKA_RESP_ERR_NO_ERROR) {
    echo "🎉 সব অর্ডার ইভেন্ট সফলভাবে পাঠানো হয়েছে!\n";
} else {
    echo "⚠️ কিছু মেসেজ পাঠানো যায়নি।\n";
}
```

### 📥 PHP Consumer — অর্ডার ইভেন্ট প্রসেস করা

```php
<?php
/**
 * Kafka Consumer Example
 * Daraz-এর মতো সিস্টেমে অর্ডার ইভেন্ট গ্রহণ ও প্রসেস করা
 */

$config = new RdKafka\Conf();

$config->set('metadata.broker.list', 'kafka-broker1:9092,kafka-broker2:9092');

// Consumer Group ID — একই গ্রুপের consumer-রা মেসেজ ভাগাভাগি করে পড়ে
$config->set('group.id', 'order-processing-group');

// নতুন consumer হলে earliest থেকে পড়া শুরু করবে
$config->set('auto.offset.reset', 'earliest');

// offset manually commit করবো (auto commit বন্ধ)
$config->set('enable.auto.commit', 'false');

// Rebalance callback — partition assign/revoke হলে জানায়
$config->setRebalanceCb(function (RdKafka\KafkaConsumer $kafka, $err, array $partitions = null) {
    switch ($err) {
        case RD_KAFKA_RESP_ERR__ASSIGN_PARTITIONS:
            echo "📌 Partition assign হয়েছে: "
               . implode(', ', array_map(fn($p) => "P{$p->getPartition()}", $partitions))
               . "\n";
            $kafka->assign($partitions);
            break;
        case RD_KAFKA_RESP_ERR__REVOKE_PARTITIONS:
            echo "📌 Partition revoke হয়েছে\n";
            $kafka->assign(null);
            break;
        default:
            echo "❌ Rebalance error: " . rd_kafka_err2str($err) . "\n";
    }
});

$consumer = new RdKafka\KafkaConsumer($config);

// 'order-events' টপিকে subscribe
$consumer->subscribe(['order-events']);

echo "🎧 অর্ডার ইভেন্ট শোনা শুরু হচ্ছে...\n";

while (true) {
    // ২ সেকেন্ড পর্যন্ত নতুন মেসেজের জন্য অপেক্ষা
    $message = $consumer->consume(2000);

    switch ($message->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            $event = json_decode($message->payload, true);

            echo "📬 নতুন অর্ডার পেয়েছি!\n";
            echo "   Order ID: {$event['data']['orderId']}\n";
            echo "   পরিমাণ: ৳{$event['data']['amount']}\n";
            echo "   শহর: {$event['data']['city']}\n";
            echo "   Partition: {$message->partition}, Offset: {$message->offset}\n";

            // অর্ডার প্রসেসিং লজিক এখানে থাকবে
            processOrder($event['data']);

            // সফলভাবে প্রসেস হলে offset commit করো
            $consumer->commit($message);
            echo "   ✅ Offset committed!\n\n";
            break;

        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            // partition-এর শেষে পৌঁছে গেছে, নতুন মেসেজের জন্য অপেক্ষা
            break;

        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            // timeout — কোনো নতুন মেসেজ নেই, আবার চেষ্টা করবে
            break;

        default:
            echo "❌ Error: " . $message->errstr() . "\n";
            break;
    }
}

function processOrder(array $orderData): void
{
    // ইনভেন্টরি চেক, পেমেন্ট ভেরিফাই, ডেলিভারি শিডিউল ইত্যাদি
    echo "   🔄 অর্ডার প্রসেসিং চলছে...\n";
}
```

---

## 💻 JavaScript কোড উদাহরণ (kafkajs)

**kafkajs** লাইব্রেরি ব্যবহার করে Node.js-এ Kafka ব্যবহার করা হয়। নিচে **Pathao**-এর মতো একটি রাইড শেয়ারিং সিস্টেমের উদাহরণ:

### 📤 JS Producer — রাইড আপডেট পাঠানো

```javascript
/**
 * Kafka Producer — Pathao-এর মতো রাইড শেয়ারিং সিস্টেম
 * প্রতিটি রাইডের স্ট্যাটাস আপডেট Kafka-তে পাঠানো
 *
 * ইনস্টল: npm install kafkajs
 */

const { Kafka, Partitioners } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'pathao-ride-service',
  brokers: ['kafka-broker1:9092', 'kafka-broker2:9092', 'kafka-broker3:9092'],
  retry: {
    initialRetryTime: 300,
    retries: 5,
  },
});

const producer = kafka.producer({
  createPartitioner: Partitioners.DefaultPartitioner,
});

async function sendRideUpdates() {
  await producer.connect();
  console.log('✅ Kafka Producer সংযুক্ত হয়েছে');

  // বিভিন্ন রাইড আপডেট ইভেন্ট
  const rideUpdates = [
    {
      rideId: 'RIDE-5001',
      driverId: 'DRV-101',
      riderId: 'RDR-201',
      status: 'DRIVER_ASSIGNED',
      location: { lat: 23.8103, lng: 90.4125 },  // ঢাকা
      city: 'ঢাকা',
    },
    {
      rideId: 'RIDE-5001',
      driverId: 'DRV-101',
      riderId: 'RDR-201',
      status: 'RIDE_STARTED',
      location: { lat: 23.8103, lng: 90.4125 },
      city: 'ঢাকা',
    },
    {
      rideId: 'RIDE-5002',
      driverId: 'DRV-205',
      riderId: 'RDR-310',
      status: 'DRIVER_ASSIGNED',
      location: { lat: 22.3569, lng: 91.7832 },  // চট্টগ্রাম
      city: 'চট্টগ্রাম',
    },
    {
      rideId: 'RIDE-5001',
      driverId: 'DRV-101',
      riderId: 'RDR-201',
      status: 'RIDE_COMPLETED',
      location: { lat: 23.7461, lng: 90.3742 },
      city: 'ঢাকা',
      fare: 185,
      currency: 'BDT',
    },
  ];

  // Kafka-তে মেসেজ পাঠানো
  const messages = rideUpdates.map((update) => ({
    // key হিসেবে rideId ব্যবহার — একই রাইডের সব আপডেট একই partition-এ যাবে
    // ফলে একটি রাইডের ইভেন্টগুলো ক্রমানুসারে (ordered) থাকবে
    key: update.rideId,
    value: JSON.stringify({
      event: 'RIDE_STATUS_UPDATED',
      data: update,
      timestamp: new Date().toISOString(),
      source: 'pathao-ride-service',
    }),
    headers: {
      'event-type': 'RIDE_STATUS_UPDATED',
      city: update.city,
    },
  }));

  await producer.send({
    topic: 'ride-updates',
    messages,
  });

  console.log(`🎉 ${messages.length}টি রাইড আপডেট পাঠানো হয়েছে!`);
  await producer.disconnect();
}

sendRideUpdates().catch(console.error);
```

### 📥 JS Consumer — রাইড আপডেট প্রসেস করা

```javascript
/**
 * Kafka Consumer — Pathao-এর মতো সিস্টেমে রাইড আপডেট প্রসেস করা
 * দুটি ভিন্ন Consumer Group:
 *   1. "ride-tracking" — রাইডারকে লাইভ ট্র্যাকিং দেখানো
 *   2. "ride-analytics" — ব্যবসায়িক বিশ্লেষণের জন্য ডেটা সংগ্রহ
 */

const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'pathao-tracking-service',
  brokers: ['kafka-broker1:9092', 'kafka-broker2:9092', 'kafka-broker3:9092'],
});

// Consumer Group 1: রিয়েল-টাইম রাইড ট্র্যাকিং
async function startTrackingConsumer() {
  const consumer = kafka.consumer({ groupId: 'ride-tracking' });

  await consumer.connect();
  console.log('🎧 Tracking Consumer সংযুক্ত হয়েছে');

  await consumer.subscribe({ topic: 'ride-updates', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      const rideData = event.data;

      console.log(`📍 রাইড আপডেট পেয়েছি:`);
      console.log(`   Ride: ${rideData.rideId}`);
      console.log(`   Status: ${rideData.status}`);
      console.log(`   Location: ${rideData.location.lat}, ${rideData.location.lng}`);
      console.log(`   Partition: ${partition}, Offset: ${message.offset}`);

      // রাইডারের ফোনে push notification পাঠানো
      switch (rideData.status) {
        case 'DRIVER_ASSIGNED':
          await notifyRider(rideData.riderId, 'আপনার ড্রাইভার আসছেন! 🚗');
          break;
        case 'RIDE_STARTED':
          await notifyRider(rideData.riderId, 'রাইড শুরু হয়েছে! 🛣️');
          break;
        case 'RIDE_COMPLETED':
          await notifyRider(
            rideData.riderId,
            `রাইড সম্পন্ন! ভাড়া: ৳${rideData.fare} 💰`
          );
          break;
      }
    },
  });
}

// Consumer Group 2: অ্যানালিটিক্স
async function startAnalyticsConsumer() {
  const consumer = kafka.consumer({ groupId: 'ride-analytics' });

  await consumer.connect();
  console.log('📊 Analytics Consumer সংযুক্ত হয়েছে');

  await consumer.subscribe({ topic: 'ride-updates', fromBeginning: true });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      const rideData = event.data;

      // ডেটাবেসে সংরক্ষণ (analytics dashboard-এর জন্য)
      await saveToAnalyticsDB({
        rideId: rideData.rideId,
        status: rideData.status,
        city: rideData.city,
        timestamp: event.timestamp,
        fare: rideData.fare || null,
      });

      console.log(`📊 Analytics: ${rideData.city}-তে ${rideData.status}`);
    },
  });
}

async function notifyRider(riderId, message) {
  console.log(`   📱 Notification → ${riderId}: ${message}`);
}

async function saveToAnalyticsDB(data) {
  console.log(`   💾 DB-তে সংরক্ষিত: ${data.rideId} - ${data.status}`);
}

// দুটি consumer একসাথে চালু করা
Promise.all([
  startTrackingConsumer(),
  startAnalyticsConsumer(),
]).catch(console.error);
```

---

## ✅ কখন Kafka ব্যবহার করবেন

### 1. 🚀 High Throughput Event Streaming
যখন প্রতি সেকেন্ডে লক্ষ লক্ষ ইভেন্ট প্রসেস করতে হবে।

**উদাহরণ:** **bKash**-এ প্রতিদিন কোটি কোটি ট্রানজ্যাকশন হয়। প্রতিটি ট্রানজ্যাকশনের ইভেন্ট (send money, cash out, payment) Kafka-তে stream করা হলে real-time fraud detection, balance update, notification সব parallel-এ হতে পারে।

### 2. 📝 Event Sourcing
যখন সিস্টেমের সব state change ইতিহাস হিসেবে রাখতে হবে।

**উদাহরণ:** **Nagad**-এ প্রতিটি অ্যাকাউন্টের সব ট্রানজ্যাকশন event হিসেবে Kafka-তে রাখা। যেকোনো সময়ে অ্যাকাউন্টের ব্যালেন্স event replay করে বের করা সম্ভব।

### 3. 📊 Log Aggregation
বিভিন্ন সার্ভার থেকে লগ একজায়গায় জমা করা।

**উদাহরণ:** **Grameenphone**-এর শত শত সার্ভারের অ্যাপ্লিকেশন লগ Kafka-তে পাঠিয়ে Elasticsearch-এ স্টোর করা। ফলে একটি centralized dashboard থেকে সব লগ দেখা ও সার্চ করা সম্ভব।

### 4. 📈 Real-time Analytics
রিয়েল-টাইম ডেটা বিশ্লেষণ ও ড্যাশবোর্ড।

**উদাহরণ:** **Pathao**-তে রিয়েল-টাইমে কোন এলাকায় কতটি রাইড চলছে, গড় ভাড়া কত, ড্রাইভার সরবরাহ কেমন — এসব Kafka Streams বা ksqlDB দিয়ে বিশ্লেষণ করা।

### 5. 🔗 Microservice Communication
মাইক্রোসার্ভিস আর্কিটেকচারে সার্ভিসগুলোর মধ্যে asynchronous যোগাযোগ।

**উদাহরণ:** **Daraz**-এ Order Service → Kafka → Inventory Service, Payment Service, Notification Service, Delivery Service সবাই নিজে নিজে মেসেজ পড়ে কাজ করে:

```
┌──────────────┐     ┌─────────┐     ┌─────────────────────┐
│ Order Service │────►│  Kafka  │────►│ Inventory Service   │
│ (অর্ডার নেয়)  │     │         │────►│ Payment Service     │
└──────────────┘     │         │────►│ Notification Service│
                      │         │────►│ Delivery Service    │
                      └─────────┘     └─────────────────────┘

📌 Order Service-কে জানতেই হবে না কে কে মেসেজ পড়ছে (decoupled!)
```

---

## ❌ কখন Kafka ব্যবহার করবেন না

### 1. 🔄 Simple Request-Response
যখন সিস্টেমে সরল request-response দরকার, তখন Kafka অপ্রয়োজনীয় জটিলতা যোগ করবে।

**উদাহরণ:** একটি ছোট ওয়েবসাইটের contact form submission। এখানে সরাসরি API call বা সাধারণ queue (যেমন Redis Queue) যথেষ্ট। Kafka ব্যবহার করলে সেটা "মশা মারতে কামান দাগা"-র মতো হবে।

### 2. 📉 Small Scale / কম মেসেজ
যখন প্রতিদিন মাত্র কয়েক শত বা হাজার মেসেজ প্রসেস হয়।

**উদাহরণ:** একটি লোকাল রেস্টুরেন্টের অর্ডার সিস্টেম যেখানে দিনে ৫০-১০০ অর্ডার আসে। এখানে Kafka-র operational complexity ন্যায়সঙ্গত নয়। একটি সাধারণ Redis বা RabbitMQ যথেষ্ট।

### 3. 🔀 Complex Routing দরকার হলে
যখন মেসেজের content বা header দেখে জটিল routing করতে হয়।

**উদাহরণ:** বিভিন্ন ধরনের নোটিফিকেশন — কিছু SMS-এ যাবে, কিছু ইমেইলে, কিছু push notification-এ — এবং এটি মেসেজের priority, user preference, content-এর উপর নির্ভর করে। RabbitMQ-এর Exchange + Binding এই কাজে অনেক ভালো।

### 4. 🔢 সব মেসেজে Strict Global Ordering দরকার
Kafka partition-এর মধ্যে ordering গ্যারান্টি দেয়, কিন্তু **সব partition মিলে global ordering** দেয় না।

**উদাহরণ:** একটি ফিনান্সিয়াল সিস্টেমে যদি **সব** ট্রানজ্যাকশন ঠিক যে ক্রমে এসেছে সেই ক্রমেই প্রসেস করতে হয় — তাহলে হয় একটি মাত্র partition রাখতে হবে (যা throughput কমিয়ে দেবে) অথবা অন্য সিস্টেম ব্যবহার করতে হবে।

### 5. ⚡ Ultra-low Latency দরকার
Kafka সাধারণত মিলিসেকেন্ড latency দেয়, কিন্তু মাইক্রোসেকেন্ড-লেভেল latency দরকার হলে Kafka উপযুক্ত নয়।

**উদাহরণ:** High-frequency trading system যেখানে প্রতিটি মাইক্রোসেকেন্ড গুরুত্বপূর্ণ। এখানে ZeroMQ বা kernel bypass solution ভালো কাজ করে।

---

## 🎯 মূল পয়েন্ট সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────────────┐
│                  🎯 Kafka মূল পয়েন্ট সারসংক্ষেপ                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1️⃣  Kafka হলো distributed event streaming platform,             │
│      traditional message queue নয়                                │
│                                                                  │
│  2️⃣  মেসেজ পড়ার পরও Kafka থেকে মুছে যায় না                     │
│      (retention period পর্যন্ত থাকে — replay সম্ভব!)              │
│                                                                  │
│  3️⃣  Topic → Partition → Offset — এই তিনটি মূল ধারণা             │
│      - Topic = ক্যাটাগরি                                         │
│      - Partition = parallel processing unit                      │
│      - Offset = partition-এর মধ্যে মেসেজের unique ID             │
│                                                                  │
│  4️⃣  Consumer Group দিয়ে একই টপিক একাধিক consumer                │
│      সমান্তরালভাবে পড়তে পারে                                     │
│                                                                  │
│  5️⃣  Ordering গ্যারান্টি শুধু partition-এর মধ্যে,                 │
│      partition-এর মধ্যে global ordering নেই                       │
│                                                                  │
│  6️⃣  Key ব্যবহার করলে একই key-র মেসেজ সবসময়                     │
│      একই partition-এ যায়                                         │
│                                                                  │
│  7️⃣  Broker cluster-এ একটি broker ডাউন হলেও                     │
│      সিস্টেম চালু থাকে (fault-tolerant)                          │
│                                                                  │
│  8️⃣  High throughput দরকার হলে Kafka ব্যবহার করুন,               │
│      জটিল routing দরকার হলে RabbitMQ বিবেচনা করুন                │
│                                                                  │
│  9️⃣  Kafka শেখা কঠিন, কিন্তু বড় সিস্টেমে এটি অপরিহার্য         │
│                                                                  │
│  🔟  bKash, Pathao, Daraz-এর মতো বড় সিস্টেমে                    │
│      Kafka ছাড়া real-time data processing কল্পনাই করা কঠিন       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔗 পরবর্তী টপিক

পরবর্তী অধ্যায়ে আমরা Kafka-র **আর্কিটেকচার** নিয়ে বিস্তারিত আলোচনা করবো:

➡️ [Kafka Architecture — বিস্তারিত আর্কিটেকচার](./kafka-architecture.md)

আমরা জানবো:
- **Replication** — ডেটা কীভাবে কপি হয় (Leader & Follower replicas)
- **ZooKeeper vs KRaft** — Kafka কীভাবে cluster manage করে
- **Log Segments** — ডিস্কে ডেটা কীভাবে সংরক্ষিত হয়
- **ISR (In-Sync Replicas)** — ডেটা সুরক্ষার কৌশল
- **Controller Broker** — কে সবকিছু নিয়ন্ত্রণ করে
- **Consumer Offset Management** — offset কোথায় সংরক্ষিত হয়

---

> _"Kafka is not just a message queue. It's a commit log, a streaming platform, and the central nervous system of modern data architecture."_

---

**📝 শেখার টিপস:** এই ফান্ডামেন্টাল ধারণাগুলো ভালো করে বোঝা অত্যন্ত গুরুত্বপূর্ণ। প্রতিটি concept-এর সাথে বাস্তব উদাহরণ মিলিয়ে চিন্তা করুন — bKash-এ কোটি কোটি ট্রানজ্যাকশন, Pathao-তে হাজার হাজার রাইড আপডেট — এভাবে ভাবলে Kafka-র প্রয়োজনীয়তা ও কার্যপ্রণালী সহজে বোঝা যাবে। 🚀
