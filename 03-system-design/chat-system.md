# 💬 চ্যাট সিস্টেম ডিজাইন (কেস স্টাডি)

> **কেস:** বাংলাদেশের জন্য একটি IMO-সদৃশ মেসেজিং অ্যাপ "BDChat" তৈরি করা — যেখানে 1:1 চ্যাট, গ্রুপ চ্যাট, অনলাইন স্ট্যাটাস, রিড রিসিপ্ট, পুশ নোটিফিকেশন, ফাইল শেয়ারিং সবকিছু থাকবে। বাংলা টেক্সট সাপোর্ট এবং লো-ব্যান্ডউইথ অপটিমাইজেশন সর্বোচ্চ প্রায়োরিটি।

---

## 📌 সেকশন ১: Requirements Gathering

### Functional Requirements

| ফিচার | বিবরণ |
|--------|--------|
| **1:1 চ্যাট** | দুই ইউজারের মধ্যে রিয়েল-টাইম মেসেজিং |
| **গ্রুপ চ্যাট** | সর্বোচ্চ ৫০০ সদস্য পর্যন্ত গ্রুপ মেসেজিং |
| **অনলাইন স্ট্যাটাস** | ইউজার অনলাইন/অফলাইন/লাস্ট সিন দেখানো |
| **রিড রিসিপ্ট** | sent ✓, delivered ✓✓, read ✓✓ (নীল) স্ট্যাটাস |
| **পুশ নোটিফিকেশন** | অফলাইন ইউজারকে নোটিফিকেশন পাঠানো |
| **ফাইল/মিডিয়া শেয়ারিং** | ছবি, ভিডিও, ডকুমেন্ট পাঠানো |
| **মেসেজ সার্চ** | চ্যাট হিস্ট্রিতে কীওয়ার্ড দিয়ে সার্চ |
| **বাংলা টেক্সট সাপোর্ট** | UTF-8 এনকোডিং, বাংলা ফন্ট রেন্ডারিং |

### Non-Functional Requirements

| প্রয়োজনীয়তা | টার্গেট |
|---------------|---------|
| **Latency** | মেসেজ ডেলিভারি < ২০০ms (দেশের ভিতরে) |
| **Availability** | ৯৯.৯৯% uptime |
| **Durability** | কোনো মেসেজ হারানো যাবে না |
| **Scalability** | ১০০ মিলিয়ন DAU সাপোর্ট |
| **Bandwidth** | লো-ব্যান্ডউইথ (2G/3G) অপটিমাইজড — বাংলাদেশের গ্রামাঞ্চলের জন্য |
| **Security** | End-to-End Encryption |

### বাংলাদেশ-স্পেসিফিক বিবেচনা

```
১. বাংলা ইউনিকোড (UTF-8) — বাংলা টেক্সটের জন্য character encoding সঠিক হতে হবে
২. লো-ব্যান্ডউইথ — গ্রামাঞ্চলে 2G/Edge নেটওয়ার্কে কাজ করতে হবে
৩. মিডিয়া কম্প্রেশন — ইমেজ/ভিডিও অটো-কম্প্রেস করতে হবে
৪. অফলাইন সাপোর্ট — মেসেজ কিউতে রাখা, কানেকশন পেলে পাঠানো
৫. ডুয়াল-সিম সাপোর্ট — একই অ্যাকাউন্টে একাধিক নম্বর
```

---

## 📊 সেকশন ২: Capacity Estimation

### ইউজার ও মেসেজ হিসাব

```
DAU (Daily Active Users)     = ১০০ মিলিয়ন (১০ কোটি)
প্রতিটি ইউজার প্রতিদিন      = গড়ে ৪০টি মেসেজ পাঠায়
মোট মেসেজ/দিন               = ১০০M × ৪০ = ৪ বিলিয়ন (৪০০ কোটি)
মোট মেসেজ/সেকেন্ড (QPS)     = ৪B / ৮৬,৪০০ ≈ ৪৬,০০০ msg/sec
পিক টাইম QPS (×৩)           ≈ ১,৪০,০০০ msg/sec
```

### স্টোরেজ হিসাব

```
গড় মেসেজ সাইজ              = ১০০ bytes (বাংলা UTF-8 এ একটু বেশি)
মেটাডাটা (sender, timestamp) = ৫০ bytes
মোট প্রতি মেসেজ              = ১৫০ bytes

দৈনিক স্টোরেজ                = ৪B × ১৫০ bytes = ৬০০ GB/day
বাৎসরিক স্টোরেজ              = ৬০০ GB × ৩৬৫ ≈ ২২০ TB/year

মিডিয়া (১০% মেসেজে মিডিয়া থাকে):
  দৈনিক মিডিয়া মেসেজ         = ৪০০M
  গড় মিডিয়া সাইজ            = ২০০ KB (কম্প্রেসড)
  দৈনিক মিডিয়া স্টোরেজ       = ৪০০M × ২০০KB = ৮০ TB/day
  বাৎসরিক মিডিয়া             ≈ ২৯ PB/year
```

### ব্যান্ডউইথ হিসাব

```
ইনকামিং ব্যান্ডউইথ (মেসেজ):
  = ৪৬,০০০ msg/sec × ১৫০ bytes = ৬.৯ MB/sec ≈ ৫৫ Mbps

ইনকামিং ব্যান্ডউইথ (মিডিয়া):
  = ৪,৬০০ media/sec × ২০০KB = ৯২০ MB/sec ≈ ৭.৩ Gbps

WebSocket কানেকশন:
  সমসাময়িক কানেকশন           ≈ ১০M (DAU-এর ১০%)
  প্রতি কানেকশন মেমোরি        ≈ ১০KB
  মোট মেমোরি                  = ১০M × ১০KB = ১০০ GB
  সার্ভার প্রয়োজন (প্রতিটি ৮GB) ≈ ১৫০+ WebSocket সার্ভার
```

---

## 🏗️ সেকশন ৩: High-Level Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              LOAD BALANCER (L7)              │
                        │         (Nginx / AWS ALB + Sticky)          │
                        └──────────┬──────────────┬───────────────────┘
                                   │              │
                    ┌──────────────▼──┐    ┌──────▼──────────────┐
                    │  API Servers     │    │  WebSocket Servers   │
                    │  (PHP Laravel)   │    │  (Node.js/Socket.io) │
                    │  - Auth          │    │  - Real-time msgs    │
                    │  - User mgmt     │    │  - Presence          │
                    │  - Group mgmt    │    │  - Typing indicator  │
                    │  - File upload   │    │  - Read receipts     │
                    └───────┬─────────┘    └──────┬───────────────┘
                            │                     │
                    ┌───────▼─────────────────────▼───────────────┐
                    │           MESSAGE QUEUE (Kafka/RabbitMQ)     │
                    │   - msg.send  - msg.deliver  - msg.notify   │
                    └───┬────────────┬────────────┬───────────────┘
                        │            │            │
              ┌─────────▼──┐  ┌─────▼──────┐  ┌──▼──────────────┐
              │  Message    │  │ Notification│  │  Presence       │
              │  Service    │  │  Service    │  │  Service        │
              │  (Store &   │  │  (FCM/APNs) │  │  (Redis Pub/Sub)│
              │   Route)    │  │             │  │  (Heartbeat)    │
              └──────┬──────┘  └─────────────┘  └─────────────────┘
                     │
        ┌────────────┼────────────────┐
        │            │                │
  ┌─────▼─────┐ ┌───▼─────────┐ ┌───▼──────────┐
  │ Cassandra  │ │ Redis Cache │ │ Elasticsearch │
  │ (Messages) │ │ (Sessions,  │ │ (Message      │
  │            │ │  Presence)  │ │  Search)      │
  └────────────┘ └─────────────┘ └──────────────┘
                                        │
                               ┌────────▼────────┐
                               │   S3 / MinIO     │
                               │  (Media Storage) │
                               │       +          │
                               │   CloudFront CDN │
                               └──────────────────┘
```

### প্রতিটি কম্পোনেন্টের ভূমিকা

| কম্পোনেন্ট | দায়িত্ব | টেকনোলজি |
|------------|---------|-----------|
| **API Server** | REST API — auth, profile, group CRUD, file upload | PHP Laravel |
| **WebSocket Server** | রিয়েল-টাইম মেসেজ, প্রেজেন্স, টাইপিং ইন্ডিকেটর | Node.js + Socket.io |
| **Message Queue** | অ্যাসিংক্রোনাস মেসেজ প্রসেসিং, ডিকাপলিং | Apache Kafka |
| **Message Store** | মেসেজ পারসিস্ট্যান্ট স্টোরেজ | Cassandra |
| **Cache** | সেশন, প্রেজেন্স, সাম্প্রতিক মেসেজ ক্যাশ | Redis Cluster |
| **Notification Service** | অফলাইন পুশ নোটিফিকেশন | FCM / APNs |
| **Presence Service** | অনলাইন/অফলাইন ট্র্যাকিং | Redis Pub/Sub |
| **Search** | মেসেজ ফুল-টেক্সট সার্চ | Elasticsearch |
| **Media Storage** | ফাইল/মিডিয়া সংরক্ষণ | S3 + CloudFront CDN |

---

## 💻 সেকশন ৪: Detailed Design

### ১. Connection Protocol — কেন WebSocket?

চ্যাট অ্যাপে ক্লায়েন্ট এবং সার্ভারের মধ্যে কমিউনিকেশন হলো সবচেয়ে গুরুত্বপূর্ণ ডিজাইন সিদ্ধান্ত। তিনটি প্রধান অপশন আছে:

#### প্রোটোকল তুলনা

```
┌──────────────────┬──────────────────┬─────────────────┬──────────────────┐
│     বৈশিষ্ট্য     │   Long Polling   │      SSE        │    WebSocket     │
├──────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ দিকনির্দেশ       │ Client→Server    │ Server→Client   │ Bidirectional ⇄  │
│ কানেকশন         │ বারবার নতুন      │ একমুখী স্থায়ী    │ দ্বিমুখী স্থায়ী   │
│ Latency          │ বেশি (polling)   │ মাঝারি          │ সবচেয়ে কম ⚡    │
│ ব্যান্ডউইথ       │ অপচয় বেশি       │ মাঝারি          │ সাশ্রয়ী 💰       │
│ ব্রাউজার সাপোর্ট │ সব               │ বেশিরভাগ       │ সব আধুনিক       │
│ চ্যাটের জন্য     │ ❌ অনুপযুক্ত     │ ⚠️ আংশিক       │ ✅ আদর্শ         │
└──────────────────┴──────────────────┴─────────────────┴──────────────────┘
```

**কেন WebSocket জেতে?**

```
Long Polling সমস্যা:
  ক্লায়েন্ট → সার্ভার: "নতুন মেসেজ আছে?"     ──→ "না"
  ক্লায়েন্ট → সার্ভার: "এখন আছে?"             ──→ "না"
  ক্লায়েন্ট → সার্ভার: "এখন?"                  ──→ "হ্যাঁ! এই নাও"
  (প্রতিবার নতুন HTTP কানেকশন — ভয়ানক অপচয়!)

WebSocket সমাধান:
  ক্লায়েন্ট ←→ সার্ভার: একবার হ্যান্ডশেক ──→ স্থায়ী কানেকশন
  সার্ভার: "নতুন মেসেজ এসেছে!" ──→ তৎক্ষণাৎ ক্লায়েন্টে পুশ
  (একটিমাত্র TCP কানেকশন, দ্বিমুখী — বাংলাদেশের 2G নেটওয়ার্কেও সাশ্রয়ী)
```

বাংলাদেশ কনটেক্সটে WebSocket সবচেয়ে উপযুক্ত কারণ:
- **কম ব্যান্ডউইথ ব্যবহার**: একবার কানেকশন হলে শুধু ডাটা যায়, বারবার HTTP হেডার নয়
- **রিয়েল-টাইম**: মেসেজ তৎক্ষণাৎ পৌঁছায়
- **ব্যাটারি সাশ্রয়ী**: বারবার পোলিং করতে হয় না

---

### ২. Message Flow — সম্পূর্ণ প্রবাহ

```
 ইউজার A (প্রেরক)                                    ইউজার B (প্রাপক)
      │                                                      │
      │  ① মেসেজ পাঠায়                                       │
      │──────────────────►┌──────────────┐                    │
      │   WebSocket        │  WebSocket   │                    │
      │                    │  Server #1   │                    │
      │  ② ACK (msg_id)   │              │                    │
      │◄──────────────────│              │                    │
      │                    └──────┬───────┘                    │
      │                           │ ③ Kafka-তে পাবলিশ          │
      │                    ┌──────▼───────┐                    │
      │                    │    Kafka     │                    │
      │                    │  msg.send    │                    │
      │                    └──┬───────┬───┘                    │
      │                       │       │                        │
      │               ┌──────▼──┐  ┌─▼──────────┐             │
      │               │Cassandra│  │  Message    │             │
      │               │ (Store) │  │  Router     │             │
      │               └─────────┘  └──────┬──────┘             │
      │                                    │                    │
      │                    ④ B অনলাইন?     │                    │
      │                    ┌───────────────▼──────┐            │
      │                    │   Redis (Presence)   │            │
      │                    │   B → ws_server_3    │            │
      │                    └───────────────┬──────┘            │
      │                                    │                    │
      │                    ⑤ B-এর সার্ভারে রাউট               │
      │                    ┌───────────────▼──────┐            │
      │                    │  WebSocket Server #3 │────────────►│
      │                    └─────────────────────┘  ⑥ ডেলিভার  │
      │                                                        │
      │                    ⑦ B অফলাইন হলে:                     │
      │                    ┌─────────────────────┐             │
      │                    │  Push Notification   │─── FCM ───►│
      │                    │  Service             │            📱
      │                    └─────────────────────┘             │
```

**মেসেজ ফ্লো ধাপে ধাপে:**

```
① ইউজার A WebSocket দিয়ে মেসেজ পাঠায়
② সার্ভার তৎক্ষণাৎ msg_id সহ ACK পাঠায় (sent ✓)
③ মেসেজ Kafka কিউতে প্রকাশিত হয়
④ Message Router, Redis থেকে B-এর কানেকশন তথ্য খোঁজে
⑤ B অনলাইন → B-এর WebSocket সার্ভারে মেসেজ ফরোয়ার্ড
⑥ B-এর ক্লায়েন্টে মেসেজ ডেলিভার (delivered ✓✓)
⑦ B অফলাইন → Push Notification Service-এ পাঠায়
```

---

### ৩. 1:1 Chat — মেসেজ রাউটিং ও ডেলিভারি

1:1 চ্যাট তুলনামূলকভাবে সরল — একজন প্রেরক, একজন প্রাপক।

#### মেসেজ ডাটা মডেল

```php
<?php
// Laravel Migration — messages টেবিল (SQL রেফারেন্স)
// প্রোডাকশনে Cassandra ব্যবহার হবে, কিন্তু স্কিমা বোঝানোর জন্য SQL

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('messages', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->uuid('chat_id')->index();        // চ্যাট থ্রেড আইডি
            $table->uuid('sender_id');                // প্রেরকের আইডি
            $table->uuid('receiver_id');              // প্রাপকের আইডি
            $table->text('content');                  // মেসেজের বডি (এনক্রিপ্টেড)
            $table->string('content_type')            // text, image, video, file
                  ->default('text');
            $table->string('media_url')->nullable();  // মিডিয়া ফাইলের URL
            $table->enum('status', [
                'sent',        // ✓  সার্ভারে পৌঁছেছে
                'delivered',   // ✓✓ প্রাপকের ডিভাইসে পৌঁছেছে
                'read'         // ✓✓ প্রাপক পড়েছে (নীল)
            ])->default('sent');
            $table->timestamp('sent_at');
            $table->timestamp('delivered_at')->nullable();
            $table->timestamp('read_at')->nullable();
            $table->softDeletes();                    // মেসেজ ডিলিট
        });

        // chat_id + sent_at দিয়ে কম্পোজিট ইনডেক্স — দ্রুত চ্যাট হিস্ট্রি লোড
        Schema::table('messages', function (Blueprint $table) {
            $table->index(['chat_id', 'sent_at']);
        });
    }
};
```

#### chat_id জেনারেশন কৌশল

```php
<?php
// দুই ইউজারের মধ্যে ইউনিক chat_id তৈরি
// ছোট ID সবসময় আগে — যেন A→B এবং B→A একই chat_id পায়

namespace App\Services;

class ChatService
{
    public function generateChatId(string $userA, string $userB): string
    {
        $ids = [$userA, $userB];
        sort($ids); // সর্ট করে নির্দিষ্ট ক্রম নিশ্চিত
        return hash('sha256', implode(':', $ids));
    }

    public function sendMessage(string $senderId, string $receiverId, array $payload): array
    {
        $chatId = $this->generateChatId($senderId, $receiverId);

        $message = [
            'id'           => \Str::uuid()->toString(),
            'chat_id'      => $chatId,
            'sender_id'    => $senderId,
            'receiver_id'  => $receiverId,
            'content'      => $payload['content'],
            'content_type' => $payload['type'] ?? 'text',
            'status'       => 'sent',
            'sent_at'      => now()->toIso8601String(),
        ];

        // Kafka-তে পাবলিশ — অ্যাসিংক্রোনাস প্রসেসিং
        KafkaProducer::publish('chat.messages', $message, $chatId);

        return $message;
    }
}
```

---

### ৪. Group Chat — ফ্যান-আউট স্ট্র্যাটেজি

গ্রুপ চ্যাট ডিজাইনের সবচেয়ে বড় চ্যালেঞ্জ হলো: একটি মেসেজ সব সদস্যের কাছে কীভাবে পৌঁছাবে?

#### Fan-out on Write vs Fan-out on Read

```
┌─────────────────────────────────────────────────────────────────┐
│                    Fan-out on Write (পুশ মডেল)                   │
│                                                                   │
│  মেসেজ আসলেই প্রতিটি সদস্যের ইনবক্সে কপি করা হয়                │
│                                                                   │
│  A পাঠায় "হ্যালো" → B-এর ইনবক্সে "হ্যালো" কপি                   │
│                    → C-এর ইনবক্সে "হ্যালো" কপি                   │
│                    → D-এর ইনবক্সে "হ্যালো" কপি                   │
│                                                                   │
│  ✅ সুবিধা: পড়ার সময় দ্রুত (প্রি-কম্পিউটেড)                    │
│  ❌ অসুবিধা: বড় গ্রুপে (৫০০ সদস্য) write amplification ভয়ানক   │
│  👉 ছোট গ্রুপের জন্য (< ৫০ সদস্য) উপযুক্ত                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Fan-out on Read (পুল মডেল)                    │
│                                                                   │
│  মেসেজ একবারই স্টোর হয়, পড়ার সময় সবাই একই জায়গা থেকে পড়ে   │
│                                                                   │
│  A পাঠায় "হ্যালো" → group_messages টেবিলে একবার স্টোর           │
│  B পড়ে → group_messages থেকে ফেচ                                │
│  C পড়ে → group_messages থেকে ফেচ                                │
│                                                                   │
│  ✅ সুবিধা: স্টোরেজ সাশ্রয়ী, write সহজ                          │
│  ❌ অসুবিধা: পড়ার সময় ধীর (join query)                         │
│  👉 বড় গ্রুপের জন্য (৫০+ সদস্য) উপযুক্ত                        │
└─────────────────────────────────────────────────────────────────┘
```

#### হাইব্রিড অ্যাপ্রোচ (আমাদের সমাধান)

```
ছোট গ্রুপ (≤ ৫০ সদস্য) → Fan-out on Write
বড় গ্রুপ (> ৫০ সদস্য)  → Fan-out on Read

এটাই WhatsApp/WeChat-এর পদ্ধতি!
```

#### গ্রুপ মেসেজ টেবিল ডিজাইন

```php
<?php
// Group Messages — Cassandra-এর জন্য ডিজাইন করা

// group_messages টেবিল (Fan-out on Read — বড় গ্রুপ)
// Partition Key: group_id
// Clustering Key: sent_at (DESC)
$groupMessageSchema = [
    'group_id'   => 'UUID',    // পার্টিশন কী
    'message_id' => 'UUID',
    'sender_id'  => 'UUID',
    'content'    => 'TEXT',
    'sent_at'    => 'TIMESTAMP', // ক্লাস্টারিং কী
];

// user_inbox টেবিল (Fan-out on Write — ছোট গ্রুপ/1:1)
// Partition Key: user_id
// Clustering Key: sent_at (DESC)
$userInboxSchema = [
    'user_id'    => 'UUID',    // পার্টিশন কী
    'chat_id'    => 'UUID',
    'message_id' => 'UUID',
    'sender_id'  => 'UUID',
    'content'    => 'TEXT',    // ডিনর্মালাইজড কপি
    'sent_at'    => 'TIMESTAMP', // ক্লাস্টারিং কী
];
```

```php
<?php

namespace App\Services;

class GroupMessageService
{
    private const SMALL_GROUP_THRESHOLD = 50;

    public function sendGroupMessage(string $groupId, string $senderId, array $payload): void
    {
        $members = $this->getGroupMembers($groupId);
        $message = $this->buildMessage($groupId, $senderId, $payload);

        // সবসময় group_messages টেবিলে স্টোর (source of truth)
        $this->storeInGroupMessages($message);

        if (count($members) <= self::SMALL_GROUP_THRESHOLD) {
            // ছোট গ্রুপ: প্রতিটি সদস্যের ইনবক্সে কপি (Fan-out on Write)
            foreach ($members as $member) {
                if ($member->id === $senderId) continue;
                KafkaProducer::publish('chat.fanout', [
                    'user_id' => $member->id,
                    'message' => $message,
                ], $member->id);
            }
        }

        // অনলাইন সদস্যদের রিয়েল-টাইম নোটিফিকেশন
        $this->notifyOnlineMembers($groupId, $senderId, $message);
    }

    private function notifyOnlineMembers(string $groupId, string $senderId, array $message): void
    {
        $members = $this->getGroupMembers($groupId);

        foreach ($members as $member) {
            if ($member->id === $senderId) continue;

            $wsServer = Redis::hget("user:presence:{$member->id}", 'ws_server');

            if ($wsServer) {
                // অনলাইন — WebSocket দিয়ে পাঠাও
                Redis::publish("ws:{$wsServer}", json_encode([
                    'type'    => 'new_message',
                    'user_id' => $member->id,
                    'message' => $message,
                ]));
            } else {
                // অফলাইন — পুশ নোটিফিকেশন
                KafkaProducer::publish('notifications.push', [
                    'user_id' => $member->id,
                    'title'   => $this->getGroupName($groupId),
                    'body'    => $this->truncate($message['content'], 100),
                ]);
            }
        }
    }
}
```

---

### ৫. Message Storage — NoSQL vs SQL

#### কেন Cassandra/NoSQL?

```
চ্যাট মেসেজের ডাটা প্যাটার্ন:
  ✅ Write-heavy (প্রচুর মেসেজ লেখা হয়)
  ✅ সিকোয়েন্সিয়াল রিড (সময়ক্রম অনুযায়ী পড়া)
  ✅ কোনো জটিল JOIN নেই
  ✅ বিশাল ডাটা ভলিউম (প্রতিদিন ৪ বিলিয়ন)
  ✅ অনুভূমিক স্কেলিং দরকার

SQL সমস্যা:
  ❌ Sharding জটিল
  ❌ Write throughput সীমিত
  ❌ JOIN পারফরম্যান্স বড় টেবিলে খারাপ

Cassandra সুবিধা:
  ✅ লিনিয়ার স্কেলিং — নোড যোগ করলেই ক্ষমতা বাড়ে
  ✅ টিউনেবল কনসিস্টেন্সি (QUORUM write, ONE read)
  ✅ টাইম-সিরিজ ডাটার জন্য আদর্শ
  ✅ পার্টিশনিং বিল্ট-ইন
```

#### Cassandra স্কিমা

```sql
-- মেসেজ টেবিল
-- chat_id দিয়ে পার্টিশন, sent_at দিয়ে ক্লাস্টারিং (সাম্প্রতিক আগে)
CREATE TABLE messages (
    chat_id    UUID,
    sent_at    TIMESTAMP,
    message_id UUID,
    sender_id  UUID,
    content    TEXT,
    content_type TEXT,
    media_url  TEXT,
    status     TEXT,
    PRIMARY KEY (chat_id, sent_at)
) WITH CLUSTERING ORDER BY (sent_at DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy',
                    'compaction_window_unit': 'DAYS',
                    'compaction_window_size': 1};

-- ইউজার ইনবক্স (সাম্প্রতিক চ্যাটগুলো দ্রুত দেখানোর জন্য)
CREATE TABLE user_inbox (
    user_id      UUID,
    last_msg_at  TIMESTAMP,
    chat_id      UUID,
    chat_type    TEXT,        -- 'direct' বা 'group'
    last_message TEXT,
    unread_count INT,
    PRIMARY KEY (user_id, last_msg_at)
) WITH CLUSTERING ORDER BY (last_msg_at DESC);
```

---

### ৬. Online/Presence Service — হার্টবিট ও লাস্ট সিন

প্রেজেন্স সার্ভিস জানায় কোন ইউজার এখন অনলাইন, কবে সর্বশেষ অনলাইন ছিল।

```
┌──────────────────────────────────────────────────────────┐
│                     Presence Architecture                  │
│                                                            │
│  ক্লায়েন্ট ──(প্রতি ৩০ সেকেন্ডে হার্টবিট)──► WebSocket   │
│                                                   Server   │
│                                                     │      │
│                                              ┌──────▼────┐ │
│                                              │   Redis    │ │
│                                              │ user:X:    │ │
│                                              │  status:   │ │
│                                              │  online    │ │
│                                              │  ws:srv3   │ │
│                                              │  TTL: 60s  │ │
│                                              └──────┬─────┘ │
│                                                     │       │
│  TTL expire হলে → অটো অফলাইন                       │       │
│  status পরিবর্তন → Redis Pub/Sub → বন্ধুদের জানায়  │       │
└──────────────────────────────────────────────────────────┘
```

```javascript
// Node.js — Presence Service (Socket.io)
const Redis = require('ioredis');
const redis = new Redis({ host: process.env.REDIS_HOST });
const redisSub = new Redis({ host: process.env.REDIS_HOST });

const HEARTBEAT_INTERVAL = 30000; // ৩০ সেকেন্ড
const PRESENCE_TTL = 60;          // ৬০ সেকেন্ড (২টি হার্টবিট মিস = অফলাইন)

class PresenceService {
    constructor(io, serverId) {
        this.io = io;
        this.serverId = serverId;
        this.subscribedUsers = new Set();
    }

    async setOnline(userId, socketId) {
        const key = `presence:${userId}`;
        const wasOffline = !(await redis.exists(key));

        // Redis-এ প্রেজেন্স সেট (TTL সহ — অটো-এক্সপায়ার)
        await redis.hmset(key, {
            status: 'online',
            ws_server: this.serverId,
            socket_id: socketId,
            last_seen: Date.now(),
        });
        await redis.expire(key, PRESENCE_TTL);

        // অফলাইন থেকে অনলাইন হলে বন্ধুদের জানাও
        if (wasOffline) {
            await redis.publish('presence:changes', JSON.stringify({
                userId,
                status: 'online',
                timestamp: Date.now(),
            }));
        }
    }

    async handleHeartbeat(userId) {
        const key = `presence:${userId}`;
        await redis.expire(key, PRESENCE_TTL);
        await redis.hset(key, 'last_seen', Date.now());
    }

    async setOffline(userId) {
        const key = `presence:${userId}`;
        const lastSeen = Date.now();

        await redis.hset(key, 'status', 'offline', 'last_seen', lastSeen);
        await redis.expire(key, 86400); // last_seen ২৪ ঘণ্টা রাখো

        await redis.publish('presence:changes', JSON.stringify({
            userId,
            status: 'offline',
            lastSeen,
        }));
    }

    async getPresence(userId) {
        const data = await redis.hgetall(`presence:${userId}`);
        if (!data || !data.status) {
            return { status: 'offline', lastSeen: null };
        }
        return {
            status: data.status,
            lastSeen: parseInt(data.last_seen),
        };
    }

    async getBulkPresence(userIds) {
        const pipeline = redis.pipeline();
        userIds.forEach(id => pipeline.hgetall(`presence:${id}`));
        const results = await pipeline.exec();

        return userIds.reduce((acc, id, idx) => {
            const data = results[idx][1];
            acc[id] = data?.status
                ? { status: data.status, lastSeen: parseInt(data.last_seen) }
                : { status: 'offline', lastSeen: null };
            return acc;
        }, {});
    }

    // Redis Pub/Sub-এর মাধ্যমে প্রেজেন্স পরিবর্তন শোনা
    listenPresenceChanges() {
        redisSub.subscribe('presence:changes');
        redisSub.on('message', (channel, data) => {
            const { userId, status, lastSeen } = JSON.parse(data);

            // এই সার্ভারে কানেক্টেড বন্ধুদের জানাও
            if (this.subscribedUsers.has(userId)) {
                this.io.to(`friends:${userId}`).emit('presence:update', {
                    userId, status, lastSeen,
                });
            }
        });
    }
}

module.exports = PresenceService;
```

---

### ৭. Push Notifications — অফলাইন ইউজারদের নোটিফিকেশন

```
┌─────────────────────────────────────────────────────────┐
│                  Push Notification Flow                    │
│                                                           │
│  মেসেজ আসে → প্রাপক অফলাইন? ──YES──► Kafka Queue        │
│                    │                      │               │
│                   NO                      ▼               │
│                    │              Notification Worker      │
│                    ▼                      │               │
│             WebSocket-এ পাঠাও     ┌──────┴──────┐        │
│                               Android    iOS             │
│                                 │          │             │
│                                FCM       APNs            │
│                                 │          │             │
│                                 ▼          ▼             │
│                               📱 Push Notification 📱     │
└─────────────────────────────────────────────────────────┘
```

```php
<?php

namespace App\Services;

use Kreait\Firebase\Messaging\CloudMessage;
use Kreait\Firebase\Messaging\Notification;

class PushNotificationService
{
    public function __construct(
        private \Kreait\Firebase\Messaging $firebaseMessaging,
    ) {}

    public function sendToUser(string $userId, array $messageData): void
    {
        $tokens = $this->getUserDeviceTokens($userId);

        if (empty($tokens)) return;

        // ব্যাচে টোকেন পাঠানো (একই ইউজারের একাধিক ডিভাইস থাকতে পারে)
        $messages = array_map(function ($token) use ($messageData) {
            return CloudMessage::withTarget('token', $token)
                ->withNotification(Notification::create(
                    $messageData['sender_name'],
                    $this->truncateForNotification($messageData['content'])
                ))
                ->withData([
                    'chat_id'    => $messageData['chat_id'],
                    'message_id' => $messageData['message_id'],
                    'type'       => 'new_message',
                ])
                // Android-এ ডাটা প্রায়োরিটি HIGH — তাৎক্ষণিক ডেলিভারি
                ->withAndroidConfig([
                    'priority' => 'high',
                    'ttl'      => '86400s',
                ])
                // iOS-এ content-available — ব্যাকগ্রাউন্ড ফেচ
                ->withApnsConfig([
                    'headers' => ['apns-priority' => '10'],
                    'payload' => [
                        'aps' => [
                            'content-available' => 1,
                            'mutable-content'   => 1,
                            'sound'             => 'default',
                            'badge'             => $this->getUnreadCount($messageData['receiver_id']),
                        ],
                    ],
                ]);
        }, $tokens);

        $report = $this->firebaseMessaging->sendAll($messages);

        // ব্যর্থ টোকেন পরিষ্কার (ইউজার অ্যাপ আনইনস্টল করেছে)
        foreach ($report->failures()->getItems() as $failure) {
            if ($this->isInvalidToken($failure)) {
                $this->removeDeviceToken($failure->target()->value());
            }
        }
    }

    // বাংলা টেক্সট নোটিফিকেশনে সঠিকভাবে কাটা
    private function truncateForNotification(string $content): string
    {
        return mb_strlen($content) > 80
            ? mb_substr($content, 0, 77) . '...'
            : $content;
    }
}
```

---

### ৮. Read Receipts — sent/delivered/read স্ট্যাটাস ট্র্যাকিং

```
মেসেজ স্ট্যাটাস ট্রানজিশন:

  [SENDING] ──► [SENT ✓] ──► [DELIVERED ✓✓] ──► [READ ✓✓ নীল]
     │              │              │                  │
     │         সার্ভারে       প্রাপকের           প্রাপক চ্যাট
     │         পৌঁছেছে       ডিভাইসে           উইন্ডো ওপেন
     │                      পৌঁছেছে            করেছে
     ▼
  [FAILED ✗] ← নেটওয়ার্ক সমস্যা হলে রিট্রাই
```

```javascript
// Node.js — Read Receipt Handler
class ReadReceiptService {
    constructor(io, redis, kafkaProducer) {
        this.io = io;
        this.redis = redis;
        this.kafka = kafkaProducer;
    }

    // প্রাপকের ক্লায়েন্ট ACK পাঠালে — delivered
    async markDelivered(userId, messageIds) {
        const updates = messageIds.map(msgId => ({
            message_id: msgId,
            status: 'delivered',
            delivered_at: Date.now(),
        }));

        // ব্যাচ আপডেট Kafka-তে — Cassandra-তে লেখার জন্য
        await this.kafka.send({
            topic: 'chat.receipts',
            messages: [{ value: JSON.stringify({ type: 'delivered', updates }) }],
        });

        // প্রেরককে রিয়েল-টাইম জানাও
        for (const update of updates) {
            const senderId = await this.getSenderId(update.message_id);
            this.emitToUser(senderId, 'receipt:delivered', {
                messageId: update.message_id,
                deliveredAt: update.delivered_at,
            });
        }
    }

    // প্রাপক চ্যাট উইন্ডো ওপেন করলে — read
    async markRead(userId, chatId) {
        const unreadMsgIds = await this.getUnreadMessages(chatId, userId);

        if (unreadMsgIds.length === 0) return;

        const readAt = Date.now();

        await this.kafka.send({
            topic: 'chat.receipts',
            messages: [{
                value: JSON.stringify({
                    type: 'read',
                    chat_id: chatId,
                    reader_id: userId,
                    message_ids: unreadMsgIds,
                    read_at: readAt,
                }),
            }],
        });

        // প্রেরককে নীল টিক দেখাও
        const senderId = await this.getChatPartner(chatId, userId);
        this.emitToUser(senderId, 'receipt:read', {
            chatId,
            messageIds: unreadMsgIds,
            readAt,
        });
    }

    // গ্রুপ রিড রিসিপ্ট — প্রতিটি সদস্যের আলাদা ট্র্যাকিং
    async markGroupRead(userId, groupId) {
        const lastReadMsgId = await this.getLastMessageId(groupId);

        await this.redis.hset(
            `group:${groupId}:read_receipt`,
            userId,
            JSON.stringify({ messageId: lastReadMsgId, readAt: Date.now() })
        );

        // গ্রুপ সদস্যদের জানাও (ঐচ্ছিক — ব্যান্ডউইথ সাশ্রয়ে বন্ধ রাখা যায়)
    }

    emitToUser(userId, event, data) {
        this.io.to(`user:${userId}`).emit(event, data);
    }
}
```

---

### ৯. File/Media Sharing — S3 আপলোড ও CDN ডেলিভারি

```
মিডিয়া আপলোড ফ্লো:

  ক্লায়েন্ট                  API Server              S3/MinIO
     │                          │                       │
     │ ① pre-signed URL চাও     │                       │
     │─────────────────────────►│                       │
     │                          │ ② URL জেনারেট          │
     │   ③ pre-signed URL       │──────────────────────►│
     │◄─────────────────────────│                       │
     │                          │                       │
     │ ④ সরাসরি S3-এ আপলোড (API সার্ভার বাইপাস!)       │
     │──────────────────────────────────────────────────►│
     │                          │                       │
     │ ⑤ আপলোড সম্পন্ন          │      ⑥ Lambda/Worker │
     │─────────────────────────►│       থাম্বনেইল তৈরি  │
     │                          │──────────────────────►│
     │                          │                       │
     │ ⑦ মেসেজ + media_url      │                       │
     │─────WebSocket───────────►│                       │
     │                          │                       │
     │       প্রাপক CDN URL দিয়ে মিডিয়া ডাউনলোড করে    │
     │◄──────────CDN────────────────────────────────────│
```

```php
<?php

namespace App\Http\Controllers;

use App\Services\MediaService;
use Illuminate\Http\Request;

class MediaController extends Controller
{
    public function __construct(private MediaService $mediaService) {}

    // Pre-signed URL জেনারেট — ক্লায়েন্ট সরাসরি S3-এ আপলোড করবে
    public function getUploadUrl(Request $request)
    {
        $validated = $request->validate([
            'file_name'    => 'required|string|max:255',
            'content_type' => 'required|string|in:image/jpeg,image/png,image/webp,video/mp4,application/pdf',
            'file_size'    => 'required|integer|max:104857600', // ১০০MB সর্বোচ্চ
        ]);

        $result = $this->mediaService->generateUploadUrl($validated, $request->user()->id);

        return response()->json($result);
    }

    // আপলোড সম্পন্ন কনফার্মেশন — থাম্বনেইল জেনারেশন ট্রিগার
    public function confirmUpload(Request $request)
    {
        $validated = $request->validate([
            'media_id' => 'required|uuid',
            'chat_id'  => 'required|string',
        ]);

        $media = $this->mediaService->confirmAndProcess($validated['media_id']);

        return response()->json([
            'media_url'     => $media['cdn_url'],
            'thumbnail_url' => $media['thumbnail_url'],
        ]);
    }
}
```

```php
<?php

namespace App\Services;

use Aws\S3\S3Client;
use Illuminate\Support\Str;

class MediaService
{
    private S3Client $s3;
    private string $bucket;
    private string $cdnDomain;

    public function __construct()
    {
        $this->s3 = new S3Client([
            'region'  => env('AWS_REGION', 'ap-south-1'), // মুম্বই — বাংলাদেশের নিকটতম
            'version' => 'latest',
        ]);
        $this->bucket = env('S3_BUCKET', 'bdchat-media');
        $this->cdnDomain = env('CDN_DOMAIN', 'cdn.bdchat.com');
    }

    public function generateUploadUrl(array $fileInfo, string $userId): array
    {
        $mediaId = Str::uuid()->toString();
        $extension = $this->getExtension($fileInfo['content_type']);
        $key = "uploads/{$userId}/{$mediaId}.{$extension}";

        // Pre-signed URL — ১৫ মিনিট মেয়াদ
        $command = $this->s3->getCommand('PutObject', [
            'Bucket'      => $this->bucket,
            'Key'         => $key,
            'ContentType' => $fileInfo['content_type'],
        ]);

        $presignedUrl = $this->s3->createPresignedRequest($command, '+15 minutes');

        // মেটাডাটা ক্যাশে রাখো
        cache()->put("media:pending:{$mediaId}", [
            'user_id'      => $userId,
            's3_key'       => $key,
            'content_type' => $fileInfo['content_type'],
            'file_name'    => $fileInfo['file_name'],
        ], 900);

        return [
            'media_id'   => $mediaId,
            'upload_url' => (string) $presignedUrl->getUri(),
            'expires_in' => 900,
        ];
    }

    public function confirmAndProcess(string $mediaId): array
    {
        $meta = cache()->get("media:pending:{$mediaId}");
        if (!$meta) throw new \Exception('আপলোড সেশন মেয়াদোত্তীর্ণ');

        // থাম্বনেইল জেনারেশন জব ডিসপ্যাচ
        if ($this->isImage($meta['content_type'])) {
            dispatch(new \App\Jobs\GenerateThumbnail($mediaId, $meta));
        }

        // লো-ব্যান্ডউইথ অপটিমাইজেশন: একাধিক কোয়ালিটি ভ্যারিয়েন্ট তৈরি
        if ($this->isImage($meta['content_type'])) {
            dispatch(new \App\Jobs\CreateImageVariants($mediaId, $meta));
        }

        $cdnUrl = "https://{$this->cdnDomain}/{$meta['s3_key']}";
        $thumbnailUrl = "https://{$this->cdnDomain}/thumbnails/{$mediaId}.webp";

        return [
            'cdn_url'       => $cdnUrl,
            'thumbnail_url' => $thumbnailUrl,
        ];
    }

    private function getExtension(string $contentType): string
    {
        return match ($contentType) {
            'image/jpeg' => 'jpg',
            'image/png'  => 'png',
            'image/webp' => 'webp',
            'video/mp4'  => 'mp4',
            'application/pdf' => 'pdf',
            default => 'bin',
        };
    }

    private function isImage(string $contentType): bool
    {
        return str_starts_with($contentType, 'image/');
    }
}
```

---

### ১০. Message Search — Elasticsearch ইন্টিগ্রেশন

```php
<?php

namespace App\Services;

use Elastic\Elasticsearch\Client;

class MessageSearchService
{
    public function __construct(private Client $elasticsearch) {}

    // মেসেজ ইনডেক্স করা (Kafka Consumer থেকে কল হবে)
    public function indexMessage(array $message): void
    {
        $this->elasticsearch->index([
            'index' => 'messages',
            'id'    => $message['message_id'],
            'body'  => [
                'chat_id'      => $message['chat_id'],
                'sender_id'    => $message['sender_id'],
                'content'      => $message['content'],
                'content_type' => $message['content_type'],
                'sent_at'      => $message['sent_at'],
                // বাংলা টেক্সট অ্যানালাইজার
                'content_bn'   => $message['content'],
            ],
        ]);
    }

    // চ্যাটের মধ্যে সার্চ
    public function searchInChat(string $chatId, string $query, string $userId, int $page = 1): array
    {
        $result = $this->elasticsearch->search([
            'index' => 'messages',
            'body'  => [
                'query' => [
                    'bool' => [
                        'must' => [
                            [
                                'multi_match' => [
                                    'query'  => $query,
                                    'fields' => ['content', 'content_bn'],
                                    'type'   => 'best_fields',
                                    // বাংলা ফাজি সার্চ সাপোর্ট
                                    'fuzziness' => 'AUTO',
                                ],
                            ],
                        ],
                        'filter' => [
                            ['term' => ['chat_id' => $chatId]],
                        ],
                    ],
                ],
                'sort'      => [['sent_at' => 'desc']],
                'from'      => ($page - 1) * 20,
                'size'      => 20,
                'highlight' => [
                    'fields' => ['content' => new \stdClass()],
                    'pre_tags'  => ['<mark>'],
                    'post_tags' => ['</mark>'],
                ],
            ],
        ]);

        return [
            'total'    => $result['hits']['total']['value'],
            'messages' => array_map(fn($hit) => [
                'id'        => $hit['_id'],
                'content'   => $hit['_source']['content'],
                'highlight' => $hit['highlight']['content'][0] ?? null,
                'sent_at'   => $hit['_source']['sent_at'],
                'sender_id' => $hit['_source']['sender_id'],
            ], $result['hits']['hits']),
        ];
    }
}
```

**Elasticsearch ইনডেক্স ম্যাপিং (বাংলা সাপোর্ট):**

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "bangla_analyzer": {
          "type": "custom",
          "tokenizer": "icu_tokenizer",
          "filter": ["icu_normalizer", "icu_folding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": { "type": "text", "analyzer": "standard" },
      "content_bn": { "type": "text", "analyzer": "bangla_analyzer" },
      "chat_id": { "type": "keyword" },
      "sender_id": { "type": "keyword" },
      "sent_at": { "type": "date" }
    }
  }
}
```

---

## 🔥 সেকশন ৫: Advanced Topics

### ১. End-to-End Encryption (E2EE)

Signal Protocol-এর ধারণা ব্যবহার করে E2EE ইমপ্লিমেন্ট করা হয়:

```
Signal Protocol-এর মূল ধারণা:

┌─────────────────────────────────────────────────────────────┐
│  Double Ratchet Algorithm                                     │
│                                                               │
│  ১. Key Agreement: X3DH (Extended Triple Diffie-Hellman)     │
│     - Identity Key (দীর্ঘমেয়াদী)                              │
│     - Signed Pre-Key (মাঝারি মেয়াদী)                         │
│     - One-Time Pre-Key (একবার ব্যবহার)                        │
│                                                               │
│  ২. প্রতিটি মেসেজের জন্য নতুন এনক্রিপশন কী                   │
│     → Forward Secrecy: পুরোনো কী চুরি হলেও আগের মেসেজ নিরাপদ │
│                                                               │
│  ৩. প্রবাহ:                                                    │
│     Alice ──[AES-256-GCM encrypted]──► সার্ভার ──► Bob        │
│     সার্ভার শুধু encrypted blob দেখে, content পড়তে পারে না!  │
└─────────────────────────────────────────────────────────────┘

E2EE মেসেজ ফরম্যাট:
{
  "header": {
    "sender_identity_key": "...",
    "ephemeral_key": "...",
    "previous_chain_length": 5,
    "message_number": 42
  },
  "ciphertext": "AES-256-GCM encrypted payload...",
  "mac": "HMAC-SHA256 authentication tag"
}

সার্ভারের ভূমিকা:
  ✅ মেসেজ রিলে করা (encrypted blob হিসেবে)
  ✅ Pre-Key bundle স্টোর করা
  ❌ মেসেজ পড়া সম্ভব নয়
  ❌ মেসেজ ডিক্রিপ্ট করা সম্ভব নয়
```

---

### ২. Message Ordering — ডিস্ট্রিবিউটেড সিস্টেমে ক্রম নিশ্চিত করা

```
সমস্যা: বিভিন্ন সার্ভার থেকে আসা মেসেজের ক্রম কীভাবে ঠিক রাখা?

সমাধান ১: Lamport Timestamp
  - প্রতিটি মেসেজে একটি লজিক্যাল ক্লক
  - গ্রহণকারী: max(নিজের_ক্লক, প্রাপ্ত_ক্লক) + ১
  - সীমাবদ্ধতা: সমসাময়িক ইভেন্টের ক্রম নির্ধারণ করতে পারে না

সমাধান ২: Snowflake ID (আমাদের পছন্দ)
  ┌──────────────────────────────────────────────────┐
  │  Snowflake ID Structure (64-bit)                  │
  │                                                    │
  │  [1 bit unused][41 bit timestamp][10 bit machine][12 bit sequence] │
  │                                                    │
  │  - ৪১ বিট: ~৬৯ বছর পর্যন্ত মিলিসেকেন্ড            │
  │  - ১০ বিট: ১০২৪টি সার্ভার                          │
  │  - ১২ বিট: প্রতি মিলিসেকেন্ডে ৪০৯৬ মেসেজ          │
  │                                                    │
  │  সুবিধা:                                            │
  │  ✅ সময়ক্রম অনুযায়ী সাজানো                          │
  │  ✅ গ্লোবালি ইউনিক                                   │
  │  ✅ ইনডেক্সিং-ফ্রেন্ডলি                              │
  └──────────────────────────────────────────────────┘

সমাধান ৩: Vector Clock (গ্রুপ চ্যাটে কনফ্লিক্ট ডিটেকশন)
  মেসেজ A: {server1: 3, server2: 1}
  মেসেজ B: {server1: 2, server2: 2}
  → কনকারেন্ট! সার্ভার-সাইডে মার্জ করতে হবে
```

---

### ৩. Scaling WebSocket Servers

```
সমস্যা: ইউজার A, সার্ভার #১-এ কানেক্টেড
         ইউজার B, সার্ভার #৩-এ কানেক্টেড
         A যখন B-কে মেসেজ পাঠায়, সার্ভার #১ কীভাবে সার্ভার #৩-এ পৌঁছাবে?

সমাধান: Redis Pub/Sub দিয়ে ক্রস-সার্ভার রাউটিং

┌──────────┐     ┌──────────┐     ┌──────────┐
│ WS Srv 1 │     │ WS Srv 2 │     │ WS Srv 3 │
│  (A আছে)  │     │          │     │  (B আছে)  │
└─────┬────┘     └─────┬────┘     └─────┬────┘
      │                │                │
      │    Subscribe: ws:srv1, ws:srv2, ws:srv3
      │                │                │
      └────────────────┼────────────────┘
                       │
                ┌──────▼──────┐
                │    Redis    │
                │   Pub/Sub   │
                │   Cluster   │
                └─────────────┘

A → মেসেজ → Srv1 → Redis PUBLISH ws:srv3 → Srv3 → B
```

```javascript
// Node.js — WebSocket Server with Redis Pub/Sub Cross-Server Routing
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');
const PresenceService = require('./presence-service');

const SERVER_ID = process.env.SERVER_ID || `ws-${process.pid}`;
const PORT = process.env.PORT || 3000;

async function bootstrap() {
    const io = new Server(PORT, {
        cors: { origin: '*' },
        // লো-ব্যান্ডউইথ অপটিমাইজেশন
        perMessageDeflate: {
            threshold: 256, // ২৫৬ বাইটের বেশি হলে কম্প্রেস
        },
        pingInterval: 30000,  // ৩০ সেকেন্ড হার্টবিট
        pingTimeout: 10000,
    });

    // Redis Adapter — Socket.io-এর বিল্ট-ইন ক্রস-সার্ভার সমাধান
    const pubClient = createClient({ url: process.env.REDIS_URL });
    const subClient = pubClient.duplicate();
    await Promise.all([pubClient.connect(), subClient.connect()]);
    io.adapter(createAdapter(pubClient, subClient));

    const presence = new PresenceService(io, SERVER_ID);
    presence.listenPresenceChanges();

    // JWT দিয়ে অথেনটিকেশন
    io.use(async (socket, next) => {
        const token = socket.handshake.auth.token;
        try {
            const user = await verifyJWT(token);
            socket.userId = user.id;
            next();
        } catch (err) {
            next(new Error('অথেনটিকেশন ব্যর্থ'));
        }
    });

    io.on('connection', async (socket) => {
        const userId = socket.userId;

        // ইউজারের প্রাইভেট রুমে জয়েন
        socket.join(`user:${userId}`);
        await presence.setOnline(userId, socket.id);

        // মেসেজ পাঠানো
        socket.on('message:send', async (data, ack) => {
            try {
                const message = await processMessage(userId, data);
                // প্রেরককে ACK (sent ✓)
                ack({ status: 'sent', messageId: message.id, sentAt: message.sent_at });

                // প্রাপকের কাছে ডেলিভার (Redis Adapter স্বয়ংক্রিয়ভাবে সঠিক সার্ভারে পাঠাবে)
                io.to(`user:${data.receiverId}`).emit('message:new', message);
            } catch (err) {
                ack({ status: 'error', error: err.message });
            }
        });

        // টাইপিং ইন্ডিকেটর
        socket.on('typing:start', (data) => {
            io.to(`user:${data.receiverId}`).emit('typing:start', {
                userId, chatId: data.chatId,
            });
        });

        socket.on('typing:stop', (data) => {
            io.to(`user:${data.receiverId}`).emit('typing:stop', {
                userId, chatId: data.chatId,
            });
        });

        // রিড রিসিপ্ট
        socket.on('message:read', async (data) => {
            await markMessagesRead(userId, data.chatId, data.messageIds);
        });

        // হার্টবিট
        socket.on('heartbeat', async () => {
            await presence.handleHeartbeat(userId);
        });

        // ডিসকানেক্ট
        socket.on('disconnect', async () => {
            await presence.setOffline(userId);
        });
    });

    console.log(`🚀 WebSocket Server ${SERVER_ID} চালু হয়েছে পোর্ট ${PORT}-এ`);
}

bootstrap();
```

---

### ৪. Chat History Sync — কার্সর-ভিত্তিক পেজিনেশন

```
সমস্যা: ইউজার নতুন ডিভাইসে লগইন করলে পুরোনো চ্যাট কীভাবে লোড করবে?

অফসেট পেজিনেশনের সমস্যা:
  GET /messages?page=5&limit=20
  → নতুন মেসেজ আসলে পেজ শিফট হয়ে যায়!
  → ডুপ্লিকেট বা মিসিং মেসেজ দেখায়

কার্সর পেজিনেশন (সমাধান):
  GET /messages?chat_id=xxx&before=msg_id_123&limit=20
  → নির্দিষ্ট মেসেজের আগের ২০টি দাও
  → নতুন মেসেজ আসলেও কোনো সমস্যা নেই!
```

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class ChatHistoryController extends Controller
{
    public function getMessages(Request $request)
    {
        $validated = $request->validate([
            'chat_id' => 'required|string',
            'before'  => 'nullable|uuid',   // এই msg_id-এর আগের মেসেজ
            'after'   => 'nullable|uuid',    // এই msg_id-এর পরের মেসেজ
            'limit'   => 'integer|min:1|max:50',
        ]);

        $limit = $validated['limit'] ?? 20;
        $chatId = $validated['chat_id'];

        if (isset($validated['before'])) {
            // পুরোনো মেসেজ লোড (উপরে স্ক্রোল)
            $cursor = $this->getMessageTimestamp($validated['before']);
            $messages = $this->cassandra->execute(
                "SELECT * FROM messages WHERE chat_id = ? AND sent_at < ? ORDER BY sent_at DESC LIMIT ?",
                [$chatId, $cursor, $limit]
            );
        } elseif (isset($validated['after'])) {
            // নতুন মেসেজ সিঙ্ক (অফলাইন থেকে ফিরে আসার পর)
            $cursor = $this->getMessageTimestamp($validated['after']);
            $messages = $this->cassandra->execute(
                "SELECT * FROM messages WHERE chat_id = ? AND sent_at > ? ORDER BY sent_at ASC LIMIT ?",
                [$chatId, $cursor, $limit]
            );
        } else {
            // প্রথম লোড — সাম্প্রতিক মেসেজ
            $messages = $this->cassandra->execute(
                "SELECT * FROM messages WHERE chat_id = ? ORDER BY sent_at DESC LIMIT ?",
                [$chatId, $limit]
            );
        }

        $messageList = iterator_to_array($messages);

        return response()->json([
            'messages'    => $messageList,
            'has_more'    => count($messageList) === $limit,
            'next_cursor' => !empty($messageList)
                ? end($messageList)['message_id']
                : null,
        ]);
    }
}
```

---

### ৫. Rate Limiting & Spam Prevention

```
রেট লিমিটিং কৌশল:

┌────────────────────────────────────────────────────────┐
│  স্তর ১: প্রতি-ইউজার রেট লিমিট                        │
│  ─────────────────────────────────────                  │
│  সর্বোচ্চ ৬০ মেসেজ/মিনিট (Token Bucket)               │
│  সর্বোচ্চ ১০০০ মেসেজ/ঘণ্টা                              │
│  সর্বোচ্চ ১০টি গ্রুপ তৈরি/দিন                           │
│                                                          │
│  স্তর ২: স্প্যাম ডিটেকশন                                │
│  ─────────────────────────────────────                  │
│  একই মেসেজ বারবার → ব্লক                                │
│  অতিরিক্ত লিংক → ফ্ল্যাগ                                │
│  নতুন অ্যাকাউন্ট + বেশি মেসেজ → সন্দেহজনক              │
│                                                          │
│  স্তর ৩: IP-ভিত্তিক সুরক্ষা                             │
│  ─────────────────────────────────────                  │
│  প্রতি IP সর্বোচ্চ কানেকশন সংখ্যা সীমিত               │
│  DDoS প্রটেকশন (Cloudflare/WAF)                        │
└────────────────────────────────────────────────────────┘
```

```javascript
// Redis-ভিত্তিক Sliding Window Rate Limiter
class RateLimiter {
    constructor(redis) {
        this.redis = redis;
    }

    async checkLimit(userId, action, maxRequests, windowSeconds) {
        const key = `ratelimit:${action}:${userId}`;
        const now = Date.now();
        const windowStart = now - (windowSeconds * 1000);

        const pipe = this.redis.pipeline();
        // পুরোনো এন্ট্রি মুছো
        pipe.zremrangebyscore(key, 0, windowStart);
        // নতুন রিকোয়েস্ট যোগ করো
        pipe.zadd(key, now, `${now}:${Math.random()}`);
        // মোট সংখ্যা গণনা
        pipe.zcard(key);
        // TTL সেট
        pipe.expire(key, windowSeconds);

        const results = await pipe.exec();
        const requestCount = results[2][1];

        if (requestCount > maxRequests) {
            const oldestInWindow = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
            const retryAfter = Math.ceil(
                (parseInt(oldestInWindow[1]) + windowSeconds * 1000 - now) / 1000
            );
            return { allowed: false, retryAfter, remaining: 0 };
        }

        return {
            allowed: true,
            remaining: maxRequests - requestCount,
            retryAfter: 0,
        };
    }
}

// ব্যবহার (Socket.io middleware হিসেবে)
const limiter = new RateLimiter(redis);

io.use(async (socket, next) => {
    const result = await limiter.checkLimit(socket.userId, 'message', 60, 60);
    if (!result.allowed) {
        return next(new Error(`রেট লিমিট অতিক্রম। ${result.retryAfter} সেকেন্ড পর আবার চেষ্টা করুন।`));
    }
    next();
});
```

---

## 💻 সেকশন ৬: সম্পূর্ণ Implementation

### Node.js Socket.io Server (সম্পূর্ণ)

```javascript
// server.js — BDChat WebSocket Server
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');
const { Kafka } = require('kafkajs');
const jwt = require('jsonwebtoken');

const app = express();
const server = http.createServer(app);

const SERVER_ID = process.env.SERVER_ID || `ws-${process.pid}`;
const JWT_SECRET = process.env.JWT_SECRET;

// Kafka সেটআপ
const kafka = new Kafka({
    clientId: `bdchat-ws-${SERVER_ID}`,
    brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
});
const producer = kafka.producer();

// Redis সেটআপ
let redis, pubClient, subClient;

async function initializeConnections() {
    redis = createClient({ url: process.env.REDIS_URL || 'redis://localhost:6379' });
    pubClient = redis.duplicate();
    subClient = redis.duplicate();

    await Promise.all([
        redis.connect(),
        pubClient.connect(),
        subClient.connect(),
        producer.connect(),
    ]);
}

async function startServer() {
    await initializeConnections();

    const io = new Server(server, {
        cors: { origin: process.env.CORS_ORIGIN || '*' },
        transports: ['websocket', 'polling'], // WebSocket প্রাধান্য, polling ফলব্যাক
        perMessageDeflate: { threshold: 256 },
        pingInterval: 30000,
        pingTimeout: 10000,
        maxHttpBufferSize: 1e6, // ১MB — বড় মেসেজ ব্লক
    });

    // Redis Adapter — ক্রস-সার্ভার মেসেজিং
    io.adapter(createAdapter(pubClient, subClient));

    // অথেনটিকেশন মিডলওয়্যার
    io.use(async (socket, next) => {
        try {
            const token = socket.handshake.auth?.token;
            if (!token) throw new Error('টোকেন নেই');

            const decoded = jwt.verify(token, JWT_SECRET);
            socket.userId = decoded.userId;
            socket.userName = decoded.name;
            next();
        } catch (err) {
            next(new Error('অথেনটিকেশন ব্যর্থ'));
        }
    });

    // রেট লিমিটিং মিডলওয়্যার
    io.use(async (socket, next) => {
        const key = `conn:limit:${socket.userId}`;
        const count = await redis.incr(key);
        if (count === 1) await redis.expire(key, 60);

        if (count > 5) {
            // একই ইউজারের ৫টির বেশি সমসাময়িক কানেকশন ব্লক
            return next(new Error('অতিরিক্ত কানেকশন'));
        }
        next();
    });

    io.on('connection', async (socket) => {
        const userId = socket.userId;
        console.log(`✅ ${userId} কানেক্টেড (${SERVER_ID})`);

        // ইউজার রুমে জয়েন
        socket.join(`user:${userId}`);

        // প্রেজেন্স আপডেট
        await redis.hSet(`presence:${userId}`, {
            status: 'online',
            ws_server: SERVER_ID,
            socket_id: socket.id,
            last_seen: Date.now().toString(),
        });
        await redis.expire(`presence:${userId}`, 60);

        // বন্ধুদের জানাও
        await redis.publish('presence:changes', JSON.stringify({
            userId, status: 'online', timestamp: Date.now(),
        }));

        // ───────── মেসেজ পাঠানো ─────────
        socket.on('message:send', async (data, ack) => {
            try {
                // রেট লিমিট চেক
                const rateKey = `rate:msg:${userId}`;
                const msgCount = await redis.incr(rateKey);
                if (msgCount === 1) await redis.expire(rateKey, 60);
                if (msgCount > 60) {
                    return ack({ status: 'error', error: 'রেট লিমিট — মিনিটে ৬০টির বেশি মেসেজ পাঠানো যাবে না' });
                }

                const messageId = generateSnowflakeId();
                const sentAt = new Date().toISOString();

                const message = {
                    id: messageId,
                    chat_id: data.chatId,
                    sender_id: userId,
                    sender_name: socket.userName,
                    receiver_id: data.receiverId,
                    content: data.content,
                    content_type: data.contentType || 'text',
                    media_url: data.mediaUrl || null,
                    status: 'sent',
                    sent_at: sentAt,
                };

                // Kafka-তে পাবলিশ (পারসিস্ট্যান্ট স্টোরেজের জন্য)
                await producer.send({
                    topic: 'chat.messages',
                    messages: [{
                        key: data.chatId,
                        value: JSON.stringify(message),
                    }],
                });

                // প্রেরককে ACK
                ack({ status: 'sent', messageId, sentAt });

                // প্রাপকের কাছে রিয়েল-টাইম ডেলিভার
                io.to(`user:${data.receiverId}`).emit('message:new', message);

            } catch (err) {
                console.error('মেসেজ পাঠাতে সমস্যা:', err);
                ack({ status: 'error', error: 'মেসেজ পাঠানো যায়নি' });
            }
        });

        // ───────── গ্রুপ মেসেজ ─────────
        socket.on('group:message', async (data, ack) => {
            try {
                const messageId = generateSnowflakeId();
                const message = {
                    id: messageId,
                    group_id: data.groupId,
                    sender_id: userId,
                    sender_name: socket.userName,
                    content: data.content,
                    content_type: data.contentType || 'text',
                    sent_at: new Date().toISOString(),
                };

                await producer.send({
                    topic: 'chat.group-messages',
                    messages: [{
                        key: data.groupId,
                        value: JSON.stringify(message),
                    }],
                });

                ack({ status: 'sent', messageId });

                // গ্রুপ রুমে ব্রডকাস্ট (প্রেরক বাদে)
                socket.to(`group:${data.groupId}`).emit('message:new', message);

            } catch (err) {
                ack({ status: 'error', error: 'গ্রুপ মেসেজ পাঠানো যায়নি' });
            }
        });

        // ───────── টাইপিং ইন্ডিকেটর ─────────
        socket.on('typing:start', (data) => {
            if (data.groupId) {
                socket.to(`group:${data.groupId}`).emit('typing:start', { userId, chatId: data.groupId });
            } else {
                io.to(`user:${data.receiverId}`).emit('typing:start', { userId, chatId: data.chatId });
            }
        });

        socket.on('typing:stop', (data) => {
            if (data.groupId) {
                socket.to(`group:${data.groupId}`).emit('typing:stop', { userId });
            } else {
                io.to(`user:${data.receiverId}`).emit('typing:stop', { userId });
            }
        });

        // ───────── রিড রিসিপ্ট ─────────
        socket.on('message:delivered', async (data) => {
            await producer.send({
                topic: 'chat.receipts',
                messages: [{
                    value: JSON.stringify({
                        type: 'delivered',
                        message_ids: data.messageIds,
                        user_id: userId,
                        timestamp: Date.now(),
                    }),
                }],
            });

            // প্রেরককে delivered ✓✓ দেখাও
            io.to(`user:${data.senderId}`).emit('receipt:delivered', {
                messageIds: data.messageIds,
                deliveredAt: Date.now(),
            });
        });

        socket.on('message:read', async (data) => {
            await producer.send({
                topic: 'chat.receipts',
                messages: [{
                    value: JSON.stringify({
                        type: 'read',
                        chat_id: data.chatId,
                        message_ids: data.messageIds,
                        reader_id: userId,
                        timestamp: Date.now(),
                    }),
                }],
            });

            io.to(`user:${data.senderId}`).emit('receipt:read', {
                chatId: data.chatId,
                messageIds: data.messageIds,
                readAt: Date.now(),
            });
        });

        // ───────── গ্রুপ জয়েন/লিভ ─────────
        socket.on('group:join', (groupId) => {
            socket.join(`group:${groupId}`);
        });

        socket.on('group:leave', (groupId) => {
            socket.leave(`group:${groupId}`);
        });

        // ───────── হার্টবিট ─────────
        socket.on('heartbeat', async () => {
            await redis.expire(`presence:${userId}`, 60);
            await redis.hSet(`presence:${userId}`, 'last_seen', Date.now().toString());
        });

        // ───────── ডিসকানেক্ট ─────────
        socket.on('disconnect', async (reason) => {
            console.log(`❌ ${userId} ডিসকানেক্টেড: ${reason}`);

            await redis.hSet(`presence:${userId}`, {
                status: 'offline',
                last_seen: Date.now().toString(),
            });
            await redis.expire(`presence:${userId}`, 86400);

            await redis.publish('presence:changes', JSON.stringify({
                userId, status: 'offline', timestamp: Date.now(),
            }));

            await redis.decr(`conn:limit:${userId}`);
        });
    });

    // হেলথ চেক এন্ডপয়েন্ট
    app.get('/health', (req, res) => {
        res.json({ status: 'ok', server: SERVER_ID, connections: io.engine.clientsCount });
    });

    const PORT_NUM = process.env.PORT || 3000;
    server.listen(PORT_NUM, () => {
        console.log(`🚀 BDChat WebSocket Server ${SERVER_ID} — পোর্ট ${PORT_NUM}`);
    });
}

// Snowflake-স্টাইল ID জেনারেটর
let sequence = 0;
let lastTimestamp = 0;
const MACHINE_ID = parseInt(process.env.MACHINE_ID || '1') & 0x3FF;

function generateSnowflakeId() {
    let timestamp = Date.now();
    if (timestamp === lastTimestamp) {
        sequence = (sequence + 1) & 0xFFF;
        if (sequence === 0) {
            while (timestamp <= lastTimestamp) {
                timestamp = Date.now();
            }
        }
    } else {
        sequence = 0;
    }
    lastTimestamp = timestamp;

    const id = BigInt(timestamp) << 22n
        | BigInt(MACHINE_ID) << 12n
        | BigInt(sequence);
    return id.toString();
}

function verifyJWT(token) {
    return jwt.verify(token, JWT_SECRET);
}

startServer().catch(console.error);
```

### PHP Laravel API (সম্পূর্ণ)

```php
<?php
// routes/api.php — BDChat API Routes

use App\Http\Controllers\AuthController;
use App\Http\Controllers\ChatController;
use App\Http\Controllers\GroupController;
use App\Http\Controllers\MediaController;
use App\Http\Controllers\ChatHistoryController;
use Illuminate\Support\Facades\Route;

// পাবলিক রাউট
Route::post('/auth/register', [AuthController::class, 'register']);
Route::post('/auth/login', [AuthController::class, 'login']);
Route::post('/auth/refresh', [AuthController::class, 'refresh']);

// প্রটেক্টেড রাউট
Route::middleware('auth:sanctum')->group(function () {

    // চ্যাট
    Route::get('/chats', [ChatController::class, 'listChats']);
    Route::get('/chats/{chatId}/messages', [ChatHistoryController::class, 'getMessages']);
    Route::post('/chats/{chatId}/messages', [ChatController::class, 'sendMessage']);

    // গ্রুপ
    Route::post('/groups', [GroupController::class, 'create']);
    Route::get('/groups/{groupId}', [GroupController::class, 'show']);
    Route::post('/groups/{groupId}/members', [GroupController::class, 'addMembers']);
    Route::delete('/groups/{groupId}/members/{userId}', [GroupController::class, 'removeMember']);
    Route::get('/groups/{groupId}/messages', [ChatHistoryController::class, 'getMessages']);

    // মিডিয়া
    Route::post('/media/upload-url', [MediaController::class, 'getUploadUrl']);
    Route::post('/media/confirm', [MediaController::class, 'confirmUpload']);

    // সার্চ
    Route::get('/search/messages', [ChatController::class, 'searchMessages']);

    // প্রেজেন্স
    Route::get('/users/{userId}/presence', [ChatController::class, 'getPresence']);
    Route::post('/users/presence/bulk', [ChatController::class, 'getBulkPresence']);

    // WebSocket টোকেন
    Route::post('/ws/token', [AuthController::class, 'getWebSocketToken']);
});
```

```php
<?php
// app/Http/Controllers/ChatController.php

namespace App\Http\Controllers;

use App\Services\ChatService;
use App\Services\MessageSearchService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class ChatController extends Controller
{
    public function __construct(
        private ChatService $chatService,
        private MessageSearchService $searchService,
    ) {}

    // ইউজারের সব চ্যাট লিস্ট (সাম্প্রতিক মেসেজসহ)
    public function listChats(Request $request)
    {
        $userId = $request->user()->id;
        $chats = $this->chatService->getUserChats($userId, $request->query('limit', 20));

        // প্রতিটি চ্যাটের সাথে অপঠিত মেসেজ সংখ্যা যোগ
        $chatsWithUnread = array_map(function ($chat) use ($userId) {
            $chat['unread_count'] = $this->chatService->getUnreadCount($chat['chat_id'], $userId);
            return $chat;
        }, $chats);

        return response()->json([
            'chats' => $chatsWithUnread,
        ]);
    }

    // REST API দিয়ে মেসেজ পাঠানো (WebSocket ব্যর্থ হলে ফলব্যাক)
    public function sendMessage(Request $request, string $chatId)
    {
        $validated = $request->validate([
            'content'      => 'required_without:media_url|string|max:4096',
            'content_type' => 'string|in:text,image,video,file',
            'media_url'    => 'nullable|url',
            'receiver_id'  => 'required|uuid',
        ]);

        $message = $this->chatService->sendMessage(
            $request->user()->id,
            $validated['receiver_id'],
            $validated,
        );

        return response()->json($message, 201);
    }

    // মেসেজ সার্চ
    public function searchMessages(Request $request)
    {
        $validated = $request->validate([
            'q'       => 'required|string|min:2|max:100',
            'chat_id' => 'nullable|string',
            'page'    => 'integer|min:1',
        ]);

        $results = $this->searchService->searchInChat(
            $validated['chat_id'] ?? null,
            $validated['q'],
            $request->user()->id,
            $validated['page'] ?? 1,
        );

        return response()->json($results);
    }

    // ইউজার প্রেজেন্স
    public function getPresence(string $userId)
    {
        $data = Redis::hgetall("presence:{$userId}");

        return response()->json([
            'user_id'   => $userId,
            'status'    => $data['status'] ?? 'offline',
            'last_seen' => isset($data['last_seen']) ? (int) $data['last_seen'] : null,
        ]);
    }

    // বাল্ক প্রেজেন্স (চ্যাট লিস্ট পেজে সবার স্ট্যাটাস একসাথে)
    public function getBulkPresence(Request $request)
    {
        $validated = $request->validate([
            'user_ids'   => 'required|array|max:100',
            'user_ids.*' => 'uuid',
        ]);

        $pipeline = Redis::pipeline();
        foreach ($validated['user_ids'] as $uid) {
            $pipeline->hgetall("presence:{$uid}");
        }
        $results = $pipeline->exec() ?: [];

        $presenceMap = [];
        foreach ($validated['user_ids'] as $i => $uid) {
            $data = $results[$i] ?? [];
            $presenceMap[$uid] = [
                'status'    => $data['status'] ?? 'offline',
                'last_seen' => isset($data['last_seen']) ? (int) $data['last_seen'] : null,
            ];
        }

        return response()->json(['presence' => $presenceMap]);
    }
}
```

---

## 📋 সারসংক্ষেপ

### মূল ডিজাইন সিদ্ধান্তসমূহ

```
┌─────────────────────────────────────────────────────────────────┐
│                    BDChat Architecture Summary                    │
├─────────────────────┬───────────────────────────────────────────┤
│ কম্পোনেন্ট          │ সিদ্ধান্ত ও কারণ                           │
├─────────────────────┼───────────────────────────────────────────┤
│ প্রোটোকল            │ WebSocket — দ্বিমুখী, কম ব্যান্ডউইথ        │
│ মেসেজ কিউ           │ Kafka — উচ্চ throughput, ডিউরেবিলিটি      │
│ মেসেজ স্টোর          │ Cassandra — write-heavy, লিনিয়ার স্কেল   │
│ ক্যাশ/প্রেজেন্স       │ Redis — দ্রুত, pub/sub, TTL সাপোর্ট       │
│ ফাইল স্টোরেজ        │ S3 + CDN — সাশ্রয়ী, স্কেলেবল              │
│ সার্চ               │ Elasticsearch — ফুল-টেক্সট, বাংলা সাপোর্ট  │
│ নোটিফিকেশন          │ FCM/APNs — প্ল্যাটফর্ম-নেটিভ              │
│ API                 │ PHP Laravel — রিচ ইকোসিস্টেম              │
│ রিয়েল-টাইম          │ Node.js Socket.io — ইভেন্ট-ড্রিভেন        │
│ গ্রুপ কৌশল          │ হাইব্রিড fan-out — ছোট/বড় গ্রুপ আলাদা    │
│ পেজিনেশন            │ কার্সর-ভিত্তিক — স্থিতিশীল, দ্রুত          │
│ এনক্রিপশন           │ Signal Protocol — E2EE                    │
│ মেসেজ অর্ডার         │ Snowflake ID — সময়ক্রম + ইউনিকনেস       │
├─────────────────────┼───────────────────────────────────────────┤
│ 🇧🇩 BD অপটিমাইজেশন │                                           │
│ - ইমেজ কম্প্রেশন    │ WebP ফরম্যাটে অটো-কনভার্ট                 │
│ - মেসেজ কম্প্রেশন    │ perMessageDeflate (WebSocket)            │
│ - অফলাইন কিউ        │ লোকাল ডিভাইসে কিউ → অনলাইনে সিঙ্ক       │
│ - CDN এজ           │ ঢাকা/সিঙ্গাপুর CDN PoP                   │
│ - বাংলা সার্চ       │ ICU Tokenizer (Elasticsearch)            │
└─────────────────────┴───────────────────────────────────────────┘
```

### ইন্টারভিউতে যেসব পয়েন্ট জোর দিতে হবে

```
✅ ১. ট্রেড-অফ বোঝানো:
     - Fan-out on Write vs Read → হাইব্রিড সমাধান কেন
     - SQL vs NoSQL → চ্যাটের write-heavy প্যাটার্নে NoSQL কেন জেতে

✅ ২. স্কেলিং কৌশল:
     - WebSocket সার্ভার → Redis Adapter দিয়ে ক্রস-সার্ভার
     - Cassandra → পার্টিশন কী দিয়ে অনুভূমিক স্কেল
     - Kafka → পার্টিশন দিয়ে throughput বাড়ানো

✅ ৩. ডিউরেবিলিটি:
     - Kafka → at-least-once ডেলিভারি
     - Cassandra → Replication Factor = 3
     - ক্লায়েন্ট → ACK না পেলে রিট্রাই

✅ ৪. রিয়েল-ওয়ার্ল্ড বিবেচনা:
     - বাংলাদেশের নেটওয়ার্ক → লো-ব্যান্ডউইথ অপটিমাইজেশন
     - বাংলা ভাষা → UTF-8 + ICU Tokenizer
     - মোবাইল-ফার্স্ট → ব্যাটারি ও ডাটা সাশ্রয়
```

> **💡 মনে রাখুন:** সিস্টেম ডিজাইন ইন্টারভিউতে পারফেক্ট সমাধান নেই — ট্রেড-অফ বোঝানো এবং বাস্তবসম্মত সিদ্ধান্ত নেওয়ার ক্ষমতাই মূল্যায়ন করা হয়। BDChat-এর এই ডিজাইন একটি প্রোডাকশন-রেডি ব্লুপ্রিন্ট যা বাংলাদেশের বাজারের জন্য অপটিমাইজড।

---

*© BDChat System Design Case Study — বাংলায় সিস্টেম ডিজাইন শেখার উদ্যোগ*
