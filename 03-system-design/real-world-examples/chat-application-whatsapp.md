# 💬 চ্যাট অ্যাপ্লিকেশন সিস্টেম ডিজাইন (WhatsApp)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: WhatsApp, Telegram, Signal, Messenger

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#️-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#️-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বিবরণ |
|---|---------|--------|
| 1 | 1:1 Chat | দুইজনের মধ্যে real-time messaging |
| 2 | Group Chat | সর্বোচ্চ 256 জন member (WhatsApp standard) |
| 3 | Media Sharing | ছবি, ভিডিও, voice message, document |
| 4 | Read Receipts | Sent ✓, Delivered ✓✓, Read (blue ✓✓) |
| 5 | Online Status | "Online", "Last seen" indicator |
| 6 | E2E Encryption | End-to-End encryption (Signal Protocol) |
| 7 | Push Notifications | Offline user-দের notify করা |
| 8 | Message Search | পুরানো message খুঁজে বের করা |

### Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────┐
│  Performance:  < 100ms message delivery (same region)   │
│  Availability: 99.99% uptime                            │
│  Scale:        2B+ users, 100B+ messages/day            │
│  Storage:      Efficient media storage & compression    │
│  Security:     E2E encryption, no server-side read      │
│  Offline:      Queue messages for offline users         │
│  Network:      2G/3G support (rural Bangladesh)         │
└─────────────────────────────────────────────────────────┘
```

### 🇧🇩 Bangladesh Context

বাংলাদেশে messaging app-এর landscape:

- **IMO** - সবচেয়ে জনপ্রিয় (especially গ্রামীণ এলাকায়, low data usage)
- **Messenger** - Facebook user base-এর কারণে ব্যাপক ব্যবহার
- **WhatsApp** - ব্যবসায়িক communication-এ বেশি ব্যবহৃত
- **Viber** - পুরানো ব্যবহারকারীদের মধ্যে এখনও popular
- **bKash In-app Chat** - financial transaction-এর সাথে messaging
- **Pathao/Uber Chat** - ride-sharing-এর in-app communication

**বাংলাদেশ-specific challenges:**
- গ্রামীণ এলাকায় 2G/3G network dominance
- Load shedding-এর কারণে intermittent connectivity
- Low-end Android device-এ memory constraints
- ঈদ/পূজায় massive message spike (100x normal traffic)

---

## 📊 Back-of-envelope Estimation

### User & Traffic Estimation

```
Total Users (Bangladesh focus):
├── DAU (Daily Active Users): 50M (BD context)
├── Messages/user/day: ~40
├── Total messages/day: 2B
├── Messages/second: ~23,000
├── Peak (Eid/events): ~230,000 msg/sec (10x)
│
Media Stats:
├── 10% messages have media
├── Average image size: 200KB (compressed)
├── Average video: 5MB
├── Media uploads/day: 200M
│
Storage (per day):
├── Text messages: 2B × 100 bytes = 200GB
├── Media: 200M × 300KB avg = 60TB
├── Metadata/indexes: ~50GB
└── Total/day: ~60.25TB
```

### Connection Estimation

```
Concurrent WebSocket connections:
├── 50M DAU × 30% online at peak = 15M connections
├── Each connection: ~10KB memory
├── Total memory: 15M × 10KB = 150GB
├── Servers needed: 150GB / 32GB per server = ~5 servers (minimum)
└── With redundancy: 15-20 WebSocket servers
```

### Bandwidth Estimation

```
Incoming:
├── Text: 23K msg/s × 100 bytes = 2.3 MB/s
├── Media: 2.3K media/s × 300KB = 690 MB/s
├── Total incoming: ~700 MB/s
│
Outgoing (fan-out for groups):
├── Average group size: 10 members
├── 30% messages are group messages
├── Fan-out factor: ~3x
└── Total outgoing: ~2.1 GB/s
```

---

## 🏗️ High-Level Design

### System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Android  │  │   iOS    │  │   Web    │  │ Desktop  │               │
│  └─────┬────┘  └─────┬────┘  └────┬─────┘  └────┬─────┘               │
│        │              │            │              │                       │
└────────┼──────────────┼────────────┼──────────────┼──────────────────────┘
         │              │            │              │
         ▼              ▼            ▼              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     LOAD BALANCER (HAProxy/Nginx)                         │
│                    [SSL Termination + Rate Limiting]                      │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  WebSocket GW 1  │ │  WebSocket GW 2  │ │  WebSocket GW N  │
│  (Node.js)       │ │  (Node.js)       │ │  (Node.js)       │
└────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      MESSAGE BROKER (Kafka/RabbitMQ)                      │
└──────────┬────────────────┬──────────────────┬──────────────────────────┘
           │                │                  │
           ▼                ▼                  ▼
┌───────────────┐  ┌────────────────┐  ┌─────────────────┐
│  Chat Service │  │ Presence Svc   │  │ Notification Svc│
│  (Laravel)    │  │ (Node.js)      │  │ (Node.js)       │
└───────┬───────┘  └───────┬────────┘  └────────┬────────┘
        │                  │                     │
        ▼                  ▼                     ▼
┌───────────────┐  ┌────────────────┐  ┌─────────────────┐
│  Cassandra    │  │    Redis       │  │  FCM / APNs     │
│  (Messages)   │  │  (Sessions)    │  │  (Push Notify)  │
└───────────────┘  └────────────────┘  └─────────────────┘
```

### Message Flow (Send → Receive)

```
User A sends message to User B:

   User A                    Server                    User B
     │                         │                         │
     │──── 1. Send msg ───────►│                         │
     │     (WebSocket)         │                         │
     │                         │──── 2. Store in DB ────►│ (Cassandra)
     │                         │                         │
     │◄─── 3. ACK (sent ✓) ───│                         │
     │                         │                         │
     │                         │── 4. Route to User B ──►│
     │                         │   (if online: WS)       │
     │                         │   (if offline: Queue)   │
     │                         │                         │
     │                         │◄── 5. Delivery ACK ─────│
     │                         │    (delivered ✓✓)       │
     │◄─── 6. Status update ──│                         │
     │     (delivered ✓✓)      │                         │
     │                         │                         │
     │                         │◄── 7. Read ACK ─────────│
     │◄─── 8. Status update ──│    (user opened chat)   │
     │     (read - blue ✓✓)   │                         │
```

### Connection Management Strategy

```
┌─────────────────────────────────────────────────────┐
│           CONNECTION MANAGEMENT                       │
├─────────────────────────────────────────────────────┤
│                                                       │
│  1. User connects → WebSocket Gateway assigns        │
│  2. Gateway registers: user_id → gateway_id (Redis)  │
│  3. Heartbeat every 30s to maintain connection       │
│  4. Disconnect → mark offline after 60s timeout      │
│                                                       │
│  Session Registry (Redis):                           │
│  ┌─────────────────────────────────────────┐        │
│  │ user:01234 → { gateway: "gw-3",        │        │
│  │               device: "android",         │        │
│  │               connected_at: timestamp,   │        │
│  │               last_heartbeat: timestamp }│        │
│  └─────────────────────────────────────────┘        │
│                                                       │
└─────────────────────────────────────────────────────┘
```

---

## 💻 Detailed Design

### 1. WebSocket Server (Node.js)

```javascript
// websocket-gateway/server.js
const WebSocket = require('ws');
const Redis = require('ioredis');
const { Kafka } = require('kafkajs');

const redis = new Redis(process.env.REDIS_URL);
const wss = new WebSocket.Server({ port: 8080 });

// Active connections map: userId → WebSocket
const connections = new Map();

// Kafka producer for message routing
const kafka = new Kafka({ brokers: ['kafka1:9092', 'kafka2:9092'] });
const producer = kafka.producer();
const consumer = kafka.consumer({ groupId: 'ws-gateway-1' });

async function start() {
    await producer.connect();
    await consumer.connect();

    // Subscribe to messages intended for users on this gateway
    await consumer.subscribe({ topic: 'messages.gateway-1' });

    consumer.run({
        eachMessage: async ({ message }) => {
            const data = JSON.parse(message.value.toString());
            deliverToUser(data.recipientId, data);
        }
    });
}

wss.on('connection', async (ws, req) => {
    const token = req.url.split('token=')[1];
    const user = await authenticateToken(token);

    if (!user) {
        ws.close(4001, 'Unauthorized');
        return;
    }

    // Register connection
    connections.set(user.id, ws);
    await redis.hset(`user:${user.id}`, {
        gateway: process.env.GATEWAY_ID,
        device: req.headers['x-device-type'] || 'unknown',
        connected_at: Date.now(),
        last_heartbeat: Date.now()
    });

    // Mark user online
    await redis.set(`online:${user.id}`, '1', 'EX', 90);
    broadcastPresence(user.id, 'online');

    // Deliver queued offline messages
    await deliverOfflineMessages(user.id, ws);

    ws.on('message', async (raw) => {
        const payload = JSON.parse(raw);
        await handleMessage(user.id, payload);
    });

    ws.on('close', async () => {
        connections.delete(user.id);
        await redis.del(`online:${user.id}`);
        // 60s delay before broadcasting offline (reconnect window)
        setTimeout(async () => {
            const stillOnline = await redis.exists(`online:${user.id}`);
            if (!stillOnline) {
                await redis.hset(`user:${user.id}`, 'last_seen', Date.now());
                broadcastPresence(user.id, 'offline');
            }
        }, 60000);
    });

    // Heartbeat
    ws.on('pong', () => {
        redis.hset(`user:${user.id}`, 'last_heartbeat', Date.now());
        redis.set(`online:${user.id}`, '1', 'EX', 90);
    });
});

// Message handler
async function handleMessage(senderId, payload) {
    switch (payload.type) {
        case 'chat_message':
            await routeChatMessage(senderId, payload);
            break;
        case 'typing':
            await forwardTypingIndicator(senderId, payload);
            break;
        case 'read_receipt':
            await processReadReceipt(senderId, payload);
            break;
        case 'delivery_ack':
            await processDeliveryAck(senderId, payload);
            break;
    }
}

async function routeChatMessage(senderId, payload) {
    const messageId = generateSnowflakeId();
    const message = {
        id: messageId,
        senderId,
        recipientId: payload.recipientId,
        conversationId: payload.conversationId,
        content: payload.content,
        type: payload.messageType || 'text',
        timestamp: Date.now(),
        status: 'sent'
    };

    // Send ACK to sender (message received by server)
    const senderWs = connections.get(senderId);
    if (senderWs) {
        senderWs.send(JSON.stringify({
            type: 'message_ack',
            messageId,
            clientMessageId: payload.clientMessageId,
            status: 'sent',
            timestamp: message.timestamp
        }));
    }

    // Publish to Kafka for processing
    await producer.send({
        topic: 'messages.incoming',
        messages: [{ key: payload.conversationId, value: JSON.stringify(message) }]
    });
}

async function deliverToUser(userId, message) {
    const ws = connections.get(userId);
    if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({
            type: 'new_message',
            ...message
        }));
    }
}

async function deliverOfflineMessages(userId, ws) {
    const messages = await redis.lrange(`offline:${userId}`, 0, -1);
    for (const msg of messages) {
        ws.send(msg);
    }
    await redis.del(`offline:${userId}`);
}

// Heartbeat interval (every 30 seconds)
setInterval(() => {
    wss.clients.forEach((ws) => {
        if (ws.isAlive === false) return ws.terminate();
        ws.isAlive = false;
        ws.ping();
    });
}, 30000);

start().catch(console.error);
```

### 2. Chat Service API (Laravel)

```php
<?php
// app/Http/Controllers/ChatController.php

namespace App\Http\Controllers;

use App\Models\Message;
use App\Models\Conversation;
use App\Models\ConversationParticipant;
use App\Services\MessageService;
use App\Services\MediaService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class ChatController extends Controller
{
    public function __construct(
        private MessageService $messageService,
        private MediaService $mediaService
    ) {}

    /**
     * conversation history fetch করা (pagination সহ)
     */
    public function getMessages(Request $request, string $conversationId)
    {
        $validated = $request->validate([
            'before' => 'nullable|string',  // cursor-based pagination
            'limit' => 'nullable|integer|max:50',
        ]);

        $userId = $request->user()->id;

        // Verify user is participant
        $isParticipant = ConversationParticipant::where('conversation_id', $conversationId)
            ->where('user_id', $userId)
            ->exists();

        if (!$isParticipant) {
            return response()->json(['error' => 'Forbidden'], 403);
        }

        $limit = $validated['limit'] ?? 20;

        $query = Message::where('conversation_id', $conversationId)
            ->orderBy('created_at', 'desc');

        if (!empty($validated['before'])) {
            $query->where('id', '<', $validated['before']);
        }

        $messages = $query->limit($limit)->get();

        return response()->json([
            'messages' => $messages,
            'has_more' => $messages->count() === $limit,
            'next_cursor' => $messages->last()?->id,
        ]);
    }

    /**
     * Media message upload (image, video, voice, document)
     */
    public function uploadMedia(Request $request, string $conversationId)
    {
        $validated = $request->validate([
            'file' => 'required|file|max:64000', // 64MB max
            'type' => 'required|in:image,video,voice,document',
            'thumbnail' => 'nullable|file|max:500', // thumbnail for video
        ]);

        $userId = $request->user()->id;
        $file = $request->file('file');

        // Compress & upload to object storage
        $mediaResult = $this->mediaService->processAndUpload(
            file: $file,
            type: $validated['type'],
            userId: $userId,
            conversationId: $conversationId
        );

        // Create message with media reference
        $message = $this->messageService->createMediaMessage(
            senderId: $userId,
            conversationId: $conversationId,
            mediaUrl: $mediaResult['url'],
            mediaType: $validated['type'],
            metadata: [
                'size' => $mediaResult['size'],
                'dimensions' => $mediaResult['dimensions'] ?? null,
                'duration' => $mediaResult['duration'] ?? null,
                'thumbnail_url' => $mediaResult['thumbnail_url'] ?? null,
            ]
        );

        return response()->json([
            'message_id' => $message->id,
            'media_url' => $mediaResult['url'],
            'status' => 'sent',
        ], 201);
    }
}
```

### 3. Group Chat Management (Laravel)

```php
<?php
// app/Http/Controllers/GroupController.php

namespace App\Http\Controllers;

use App\Models\Group;
use App\Models\GroupMember;
use App\Events\GroupMessageEvent;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class GroupController extends Controller
{
    /**
     * নতুন group তৈরি করা
     */
    public function createGroup(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:100',
            'description' => 'nullable|string|max:500',
            'members' => 'required|array|min:1|max:255',
            'members.*' => 'exists:users,id',
            'avatar' => 'nullable|image|max:5000',
        ]);

        $creator = $request->user();

        $group = DB::transaction(function () use ($validated, $creator) {
            $group = Group::create([
                'id' => \Str::uuid(),
                'name' => $validated['name'],
                'description' => $validated['description'] ?? null,
                'created_by' => $creator->id,
                'max_members' => 256, // WhatsApp standard
            ]);

            // Creator কে admin হিসেবে add করা
            GroupMember::create([
                'group_id' => $group->id,
                'user_id' => $creator->id,
                'role' => 'admin',
                'joined_at' => now(),
            ]);

            // বাকি members add করা
            $members = collect($validated['members'])->map(fn($userId) => [
                'group_id' => $group->id,
                'user_id' => $userId,
                'role' => 'member',
                'joined_at' => now(),
            ]);

            GroupMember::insert($members->toArray());

            return $group;
        });

        // সব member-দের notify করা
        $this->notifyGroupCreation($group);

        return response()->json([
            'group' => $group->load('members'),
            'message' => 'Group created successfully',
        ], 201);
    }

    /**
     * Group message fan-out: 256 members-এ message পাঠানো
     */
    public function sendGroupMessage(Request $request, string $groupId)
    {
        $validated = $request->validate([
            'content' => 'required|string|max:4096',
            'type' => 'in:text,image,video,voice,document',
        ]);

        $sender = $request->user();
        $group = Group::with('activeMembers')->findOrFail($groupId);

        // sender member কিনা check
        if (!$group->activeMembers->contains('user_id', $sender->id)) {
            return response()->json(['error' => 'Not a member'], 403);
        }

        $message = $this->messageService->createGroupMessage(
            senderId: $sender->id,
            groupId: $groupId,
            content: $validated['content'],
            type: $validated['type'] ?? 'text'
        );

        // Fan-out: প্রতিটি member-কে message deliver করা
        $recipients = $group->activeMembers
            ->where('user_id', '!=', $sender->id)
            ->pluck('user_id');

        // Kafka-তে fan-out publish করা (async delivery)
        event(new GroupMessageEvent($message, $recipients->toArray()));

        return response()->json([
            'message_id' => $message->id,
            'status' => 'sent',
            'recipients_count' => $recipients->count(),
        ]);
    }
}
```

### 4. Group Message Fan-out Worker (Node.js)

```javascript
// workers/group-fanout-worker.js
const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL);
const kafka = new Kafka({ brokers: ['kafka1:9092', 'kafka2:9092'] });
const consumer = kafka.consumer({ groupId: 'group-fanout-workers' });
const producer = kafka.producer();

async function start() {
    await consumer.connect();
    await producer.connect();
    await consumer.subscribe({ topic: 'messages.group' });

    consumer.run({
        eachMessage: async ({ message }) => {
            const data = JSON.parse(message.value.toString());
            await fanOutGroupMessage(data);
        }
    });
}

/**
 * Group message fan-out strategy:
 * - Small groups (≤20): Immediate parallel delivery
 * - Large groups (>20): Batched delivery to avoid thundering herd
 */
async function fanOutGroupMessage(data) {
    const { message, recipients } = data;
    const BATCH_SIZE = 50;

    for (let i = 0; i < recipients.length; i += BATCH_SIZE) {
        const batch = recipients.slice(i, i + BATCH_SIZE);

        const deliveryPromises = batch.map(async (recipientId) => {
            // Check if user is online
            const userSession = await redis.hgetall(`user:${recipientId}`);

            if (userSession && userSession.gateway) {
                // Online: route to their gateway
                await producer.send({
                    topic: `messages.${userSession.gateway}`,
                    messages: [{
                        key: recipientId,
                        value: JSON.stringify({
                            recipientId,
                            ...message,
                            isGroupMessage: true
                        })
                    }]
                });
            } else {
                // Offline: queue message + send push notification
                await redis.rpush(
                    `offline:${recipientId}`,
                    JSON.stringify(message)
                );
                await sendPushNotification(recipientId, message);
            }
        });

        await Promise.all(deliveryPromises);

        // Large groups-এ batch-এর মধ্যে ছোট delay
        if (recipients.length > 20) {
            await new Promise(resolve => setTimeout(resolve, 10));
        }
    }
}

async function sendPushNotification(userId, message) {
    const deviceToken = await redis.get(`push_token:${userId}`);
    if (!deviceToken) return;

    await producer.send({
        topic: 'notifications.push',
        messages: [{
            key: userId,
            value: JSON.stringify({
                token: deviceToken,
                title: message.senderName,
                body: truncate(message.content, 100),
                data: {
                    type: 'new_message',
                    conversationId: message.conversationId
                }
            })
        }]
    });
}

start().catch(console.error);
```

### 5. Database Schema Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PostgreSQL (Users + Groups + Metadata)                          │
│  ┌───────────────────────────────────────────────┐              │
│  │  users: id, phone, name, avatar, status,      │              │
│  │         last_seen, created_at                  │              │
│  │                                                │              │
│  │  groups: id, name, description, avatar,        │              │
│  │          created_by, max_members, created_at   │              │
│  │                                                │              │
│  │  group_members: group_id, user_id, role,       │              │
│  │                 joined_at, muted_until         │              │
│  │                                                │              │
│  │  conversations: id, type (1:1/group),          │              │
│  │                 created_at                     │              │
│  └───────────────────────────────────────────────┘              │
│                                                                   │
│  Cassandra (Messages - Write-heavy, Time-series)                 │
│  ┌───────────────────────────────────────────────┐              │
│  │  messages:                                     │              │
│  │    partition_key: conversation_id              │              │
│  │    clustering_key: message_id (TimeUUID)      │              │
│  │    columns: sender_id, content, type,          │              │
│  │             media_url, status, created_at      │              │
│  │                                                │              │
│  │  message_status:                               │              │
│  │    partition_key: message_id                   │              │
│  │    clustering_key: user_id                    │              │
│  │    columns: status, updated_at                │              │
│  └───────────────────────────────────────────────┘              │
│                                                                   │
│  Redis (Real-time State)                                         │
│  ┌───────────────────────────────────────────────┐              │
│  │  online:{user_id} → TTL 90s                   │              │
│  │  user:{user_id} → {gateway, device, ...}      │              │
│  │  offline:{user_id} → [queued messages]        │              │
│  │  typing:{conv_id}:{user_id} → TTL 5s         │              │
│  │  push_token:{user_id} → device_token          │              │
│  └───────────────────────────────────────────────┘              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 6. Message Delivery & Acknowledgment Flow

```
Message Status State Machine:

    ┌──────────┐    Server     ┌───────────┐   Recipient    ┌──────────┐
    │ PENDING  │───receives───►│   SENT    │───receives────►│DELIVERED │
    │(client)  │               │   (✓)     │                │  (✓✓)    │
    └──────────┘               └───────────┘                └────┬─────┘
                                                                  │
                                                            User opens
                                                              chat
                                                                  │
                                                                  ▼
                                                            ┌──────────┐
                                                            │   READ   │
                                                            │(blue ✓✓) │
                                                            └──────────┘
```

### 7. End-to-End Encryption Overview (Signal Protocol)

```
┌─────────────────────────────────────────────────────────────────┐
│              E2E ENCRYPTION (Signal Protocol)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Key Exchange (X3DH - Extended Triple Diffie-Hellman):           │
│                                                                   │
│  User A                   Server              User B             │
│    │                        │                    │                │
│    │                        │◄─── Upload keys ───│                │
│    │                        │    (Identity Key,  │                │
│    │                        │     Signed PreKey, │                │
│    │                        │     One-time Keys) │                │
│    │                        │                    │                │
│    │── Request B's keys ───►│                    │                │
│    │◄── B's public keys ────│                    │                │
│    │                        │                    │                │
│    │── Compute shared ──────────────────────────►│                │
│    │   secret (X3DH)        │                    │                │
│    │                        │                    │                │
│    │══ Encrypted messages (Double Ratchet) ═════►│                │
│    │                        │                    │                │
│                                                                   │
│  Double Ratchet Algorithm:                                       │
│  - প্রতিটি message-এ নতুন encryption key                        │
│  - Forward secrecy: পুরানো key compromise হলেও                  │
│    আগের messages safe                                            │
│  - Server শুধু encrypted blob দেখে, content পড়তে পারে না       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 8. Offline Message Queue & Sync

```php
<?php
// app/Services/MessageSyncService.php

namespace App\Services;

use App\Models\Message;
use Illuminate\Support\Facades\Redis;

class MessageSyncService
{
    /**
     * User reconnect করলে offline messages sync করা
     * বাংলাদেশে 2G/3G network-এ frequent disconnect হয়
     */
    public function syncOfflineMessages(string $userId, ?string $lastMessageId = null): array
    {
        // Step 1: Redis queue থেকে queued messages নেওয়া
        $queuedMessages = Redis::lrange("offline:{$userId}", 0, -1);

        // Step 2: যদি lastMessageId দেওয়া থাকে, DB থেকে missing messages fetch
        $dbMessages = [];
        if ($lastMessageId) {
            $conversations = $this->getUserConversations($userId);

            foreach ($conversations as $conversationId) {
                $messages = Message::where('conversation_id', $conversationId)
                    ->where('id', '>', $lastMessageId)
                    ->where('sender_id', '!=', $userId)
                    ->orderBy('created_at', 'asc')
                    ->limit(100) // Batch limit for low bandwidth
                    ->get();

                $dbMessages = array_merge($dbMessages, $messages->toArray());
            }
        }

        // Step 3: Queue clear করা
        Redis::del("offline:{$userId}");

        return [
            'queued_messages' => array_map(fn($m) => json_decode($m, true), $queuedMessages),
            'missed_messages' => $dbMessages,
            'sync_timestamp' => now()->timestamp,
        ];
    }

    /**
     * Low bandwidth optimization for 2G networks
     * শুধু text পাঠানো, media পরে download হবে
     */
    public function syncLowBandwidth(string $userId, string $lastMessageId): array
    {
        $messages = Message::where('recipient_id', $userId)
            ->where('id', '>', $lastMessageId)
            ->select(['id', 'sender_id', 'conversation_id', 'type', 'content', 'created_at'])
            ->orderBy('created_at', 'asc')
            ->limit(50)
            ->get()
            ->map(function ($msg) {
                // Media messages-এ শুধু thumbnail/placeholder পাঠানো
                if (in_array($msg->type, ['image', 'video', 'document'])) {
                    $msg->content = "[{$msg->type}] Tap to download";
                    $msg->media_pending = true;
                }
                return $msg;
            });

        return ['messages' => $messages, 'mode' => 'low_bandwidth'];
    }
}
```

### 9. Presence Service (Online/Last Seen)

```javascript
// services/presence-service.js
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

class PresenceService {
    /**
     * User-এর online status track করা
     * Bangladesh-এ privacy concern বেশি, তাই last seen hide option
     */
    async setOnline(userId) {
        const pipeline = redis.pipeline();
        pipeline.set(`online:${userId}`, '1', 'EX', 90);
        pipeline.hset(`user:${userId}`, 'last_activity', Date.now());
        await pipeline.exec();
    }

    async setOffline(userId) {
        await redis.del(`online:${userId}`);
        await redis.hset(`user:${userId}`, 'last_seen', Date.now());
    }

    async isOnline(userId) {
        return await redis.exists(`online:${userId}`);
    }

    /**
     * Contact list-এর সবার online status একবারে fetch
     * (একটা একটা করে query করলে slow হবে)
     */
    async getBulkPresence(userIds) {
        const pipeline = redis.pipeline();
        userIds.forEach(id => pipeline.exists(`online:${id}`));
        const results = await pipeline.exec();

        return userIds.reduce((acc, id, index) => {
            acc[id] = {
                online: results[index][1] === 1,
            };
            return acc;
        }, {});
    }

    /**
     * Typing indicator: 5 second TTL
     * "XYZ is typing..." দেখানোর জন্য
     */
    async setTyping(userId, conversationId) {
        await redis.set(
            `typing:${conversationId}:${userId}`,
            '1',
            'EX', 5
        );
    }

    async getTypingUsers(conversationId, excludeUserId) {
        const pattern = `typing:${conversationId}:*`;
        const keys = await redis.keys(pattern);

        return keys
            .map(key => key.split(':')[2])
            .filter(id => id !== excludeUserId);
    }
}

module.exports = new PresenceService();
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. WebSocket vs Long Polling vs SSE

```
┌──────────────┬──────────────────┬──────────────────┬──────────────────┐
│   Feature    │    WebSocket     │  Long Polling    │      SSE         │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Latency      │ ~10ms            │ ~100-500ms       │ ~50ms            │
│ Bidirectional│ ✅ Yes           │ ✅ (2 channels)  │ ❌ Server→Client │
│ Connection   │ Persistent       │ Reconnect often  │ Persistent       │
│ Battery      │ Efficient        │ Battery drain    │ Moderate         │
│ Proxy/FW     │ Issues possible  │ ✅ Works well    │ ✅ Works well    │
│ Scale (BD)   │ RAM heavy        │ CPU heavy        │ Moderate         │
│ 2G Network   │ May drop often   │ ✅ Reliable      │ ❌ Drops         │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ VERDICT      │ ✅ Primary       │ Fallback for     │ ❌ Not suited    │
│              │ (real-time)      │ poor networks    │ for chat         │
└──────────────┴──────────────────┴──────────────────┴──────────────────┘

বাংলাদেশ Context:
- শহরে (Dhaka, Chittagong): WebSocket ভালো কাজ করে (4G available)
- গ্রামে: Long Polling fallback দিতে হবে (2G-তে WS unstable)
- WhatsApp approach: WS primary + intelligent reconnection
```

### 2. Cassandra vs MongoDB for Messages

```
┌──────────────────┬─────────────────────┬─────────────────────┐
│   Criteria       │     Cassandra       │      MongoDB        │
├──────────────────┼─────────────────────┼─────────────────────┤
│ Write speed      │ ✅ Extremely fast   │ 🔶 Good            │
│ Read pattern     │ Sequential (by time)│ Flexible queries    │
│ Scale            │ ✅ Linear scaling   │ 🔶 Sharding complex│
│ Availability     │ ✅ No single SPOF   │ 🔶 Primary failover│
│ Message search   │ ❌ Need external    │ ✅ Text indexes     │
│ Data model       │ Denormalized        │ Document flexible   │
│ Operational cost │ 🔶 Complex tuning   │ ✅ Easier           │
├──────────────────┼─────────────────────┼─────────────────────┤
│ VERDICT          │ ✅ Chat messages    │ 🔶 Smaller scale    │
│                  │ (write-heavy, time  │ or when search is   │
│                  │ series access)      │ primary need        │
└──────────────────┴─────────────────────┴─────────────────────┘

WhatsApp's choice: Custom Mnesia (Erlang) → বড় scale-এ Cassandra-like approach
আমাদের choice: Cassandra for messages + Elasticsearch for search
```

### 3. Push vs Pull for Message Sync

```
Push Model (Server pushes to client):
├── ✅ Real-time delivery
├── ✅ Low latency
├── ❌ Server-side complexity (connection tracking)
└── ❌ Missed messages if connection drops

Pull Model (Client polls server):
├── ✅ Simple implementation
├── ✅ No missed messages
├── ❌ High latency
└── ❌ Unnecessary bandwidth usage

Hybrid Approach (WhatsApp style): ✅ BEST
├── Push: Real-time delivery when online
├── Pull: Sync missed messages on reconnect
└── Smart: Client pulls only after detecting gap
```

### 4. Client-side vs Server-side Encryption

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│                     │ Client-side (E2E)    │ Server-side          │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Privacy             │ ✅ Maximum           │ ❌ Server can read   │
│ Compliance (BD)     │ 🔶 BTRC may object  │ ✅ Lawful intercept  │
│ Search              │ ❌ Server can't index│ ✅ Full-text search  │
│ Backup              │ 🔶 Complex          │ ✅ Simple            │
│ Performance         │ 🔶 Client CPU usage │ ✅ No client overhead│
│ Key management      │ 🔶 Complex          │ ✅ Centralized       │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ VERDICT             │ ✅ Personal chat     │ ✅ Business/bKash    │
│                     │ (WhatsApp model)     │ (compliance needed)  │
└─────────────────────┴──────────────────────┴──────────────────────┘

বাংলাদেশ context:
- BTRC (Bangladesh Telecommunication Regulatory Commission) regulations
- Digital Security Act considerations
- bKash/Nagad in-app chat: server-side encryption (transaction records প্রয়োজন)
- Personal messaging: E2E preferred (user trust)
```

---

## 📈 কেস স্টাডি

### কেস ১: ঈদের শুভেচ্ছা বার্তার ঝড় (Eid Greetings Storm)

```
সমস্যা:
- ঈদের দিন রাত 12:00 AM-এ millions of messages/second
- Normal traffic-এর 50-100x spike
- 2023 Eid-ul-Fitr: estimated 500M+ messages in BD alone

Timeline:
  11:55 PM ─── Normal load (~23K msg/s)
  11:59 PM ─── Spike begins (100K msg/s)
  12:00 AM ─── PEAK (500K-1M msg/s) ← "ঈদ মুবারাক!" storm
  12:05 AM ─── Sustained high (300K msg/s)
  12:30 AM ─── Gradual decline
   1:00 AM ─── Return to elevated normal

সমাধান Strategy:
┌─────────────────────────────────────────────────────────┐
│ 1. Pre-scaling (2 hours before):                         │
│    - Auto-scale WebSocket servers: 20 → 100              │
│    - Kafka partitions increase                           │
│    - Cassandra compaction pause                          │
│                                                           │
│ 2. Message batching:                                     │
│    - Similar "Eid Mubarak" messages → deduplicate        │
│    - Template detection → single store, multiple refs    │
│                                                           │
│ 3. Priority queuing:                                     │
│    - 1:1 messages: HIGH priority                         │
│    - Group forwards: LOW priority (delay acceptable)     │
│    - Broadcast lists: BACKGROUND                         │
│                                                           │
│ 4. Graceful degradation:                                 │
│    - Read receipts temporarily disabled                  │
│    - "Last seen" updates paused                          │
│    - Media uploads throttled (text priority)             │
│                                                           │
│ 5. Bangladesh-specific:                                  │
│    - GP/Robi/Banglalink network congestion handling      │
│    - Longer retry timeouts for mobile networks           │
│    - SMS fallback for critical messages                  │
└─────────────────────────────────────────────────────────┘
```

### কেস ২: Large Group (256 Members) Message Fan-out

```
সমস্যা:
- একটি family group-এ 256 জন member
- একজন message পাঠালে 255 জনকে deliver করতে হবে
- Mixed online/offline states
- বিভিন্ন network quality (শহর vs গ্রাম)

Analysis:
┌────────────────────────────────────────┐
│ Group: "বড় পরিবার" (256 members)     │
│                                        │
│ Online (4G/WiFi): 80 members  (31%)   │
│ Online (2G/3G):  40 members  (16%)   │
│ Offline:          136 members (53%)   │
│                                        │
│ Fan-out cost per message:              │
│ - 80 WebSocket deliveries (instant)   │
│ - 40 WS deliveries (may need retry)  │
│ - 136 queue + push notifications      │
│ - Total operations: ~400+             │
│                                        │
│ If 10 messages/minute in group:        │
│ - 4000 operations/minute              │
│ - For 1000 such groups: 4M ops/min   │
└────────────────────────────────────────┘

সমাধান:
1. Tiered delivery:
   - Tier 1 (instant): Online with good connection
   - Tier 2 (batch): Online with poor connection (batch 5 msgs)
   - Tier 3 (queue): Offline (queue + single push notification)

2. Smart push notification:
   - 10 messages in 1 minute → single notification: "5 new messages in বড় পরিবার"
   - Not 10 separate notifications (battery + annoyance)

3. Read receipt optimization:
   - Group read receipts batched every 5 seconds
   - Only send to sender (not entire group)
```

### কেস ৩: Poor Network Handling (2G/3G in Rural BD)

```
সমস্যা:
- বাংলাদেশের 40%+ rural areas still on 2G/3G
- Frequent disconnections (load shedding, tower issues)
- IMO popular precisely because it handles this well
- Bandwidth: 2G = 50-100 Kbps, 3G = 1-5 Mbps

Connection Scenarios:
┌─────────────────────────────────────────────────────────┐
│                                                           │
│  Scenario 1: Intermittent connectivity                   │
│  ─────────────────────────────────────                   │
│  Connected ████░░░████░░████████░░░░████                │
│  Time      0s   5s   10s  15s  20s  25s  30s           │
│                                                           │
│  Scenario 2: Very slow connection (2G)                   │
│  ─────────────────────────────────────                   │
│  Bandwidth: ▁▂▁▁▃▂▁▁▁▂▃▅▃▂▁▁▁▁▂▁                     │
│  Max: 100Kbps, Avg: 30Kbps                              │
│                                                           │
│  Scenario 3: Load shedding recovery                      │
│  ─────────────────────────────────────                   │
│  Power:    ON████████OFF░░░░░░░░ON██████                │
│  Network:  UP████████DOWN░░░░░░░RECONNECT████           │
│  Messages:  ✓✓✓✓✓✓✓✓ QUEUED○○○○ SYNC✓✓✓✓✓             │
│                                                           │
└─────────────────────────────────────────────────────────┘

সমাধান Strategy:

1. Adaptive message delivery:
   ┌──────────────────────────────────────────┐
   │ Network    │ Strategy                     │
   ├────────────┼──────────────────────────────┤
   │ 4G/WiFi    │ Full: text + media + preview │
   │ 3G         │ Text + compressed thumbnails │
   │ 2G         │ Text only, media on-demand   │
   │ Offline    │ Queue, sync on reconnect     │
   └────────────┴──────────────────────────────┘

2. Message compression:
   - Text: gzip compression (~60% reduction)
   - Images: progressive JPEG, tiny thumbnails first
   - Voice: opus codec @ 8kbps (vs 64kbps normal)

3. Smart reconnection:
   - Exponential backoff: 1s, 2s, 4s, 8s, 16s, 30s max
   - Background sync: batch all pending in single request
   - Delta sync: only fetch messages after last known ID
```

---

## 🔧 Advanced Topics

### 1. Message Search (Elasticsearch)

```javascript
// services/search-service.js
const { Client } = require('@elastic/elasticsearch');
const client = new Client({ node: 'http://elasticsearch:9200' });

class MessageSearchService {
    /**
     * Message index করা (E2E encryption ছাড়া messages-এর জন্য)
     * Note: E2E encrypted messages server-side search করা যায় না
     * Client-side search implement করতে হবে
     */
    async indexMessage(message) {
        await client.index({
            index: 'messages',
            id: message.id,
            body: {
                conversation_id: message.conversationId,
                sender_id: message.senderId,
                content: message.content,
                type: message.type,
                timestamp: message.timestamp,
                // Bangla text analysis support
                content_bn: message.content, // Bangla analyzer field
            }
        });
    }

    /**
     * Search with Bangla support
     * বাংলা text search-এর জন্য custom analyzer
     */
    async searchMessages(userId, query, options = {}) {
        const { conversationId, from = 0, size = 20 } = options;

        const must = [
            {
                multi_match: {
                    query,
                    fields: ['content', 'content_bn'],
                    type: 'best_fields',
                    fuzziness: 'AUTO'
                }
            }
        ];

        // Specific conversation-এ search
        if (conversationId) {
            must.push({ term: { conversation_id: conversationId } });
        }

        const result = await client.search({
            index: 'messages',
            body: {
                query: { bool: { must } },
                sort: [{ timestamp: 'desc' }],
                from,
                size,
                highlight: {
                    fields: { content: {}, content_bn: {} }
                }
            }
        });

        return {
            hits: result.hits.hits.map(hit => ({
                id: hit._id,
                ...hit._source,
                highlight: hit.highlight
            })),
            total: result.hits.total.value
        };
    }
}

module.exports = new MessageSearchService();
```

### 2. Voice/Video Calling (WebRTC)

```
┌─────────────────────────────────────────────────────────────────┐
│                 VOICE/VIDEO CALL ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Caller (A)              Signaling Server           Callee (B)   │
│     │                         │                         │        │
│     │─── 1. Call initiate ───►│                         │        │
│     │    (SDP Offer)          │─── 2. Push notify ─────►│        │
│     │                         │    (incoming call)      │        │
│     │                         │                         │        │
│     │                         │◄── 3. Accept call ──────│        │
│     │◄── 4. SDP Answer ──────│    (SDP Answer)         │        │
│     │                         │                         │        │
│     │─── 5. ICE candidates ──┼──────────────────────── │        │
│     │◄───────────────────────┼── 6. ICE candidates ────│        │
│     │                         │                         │        │
│     │════════ 7. P2P Media Stream (WebRTC) ════════════│        │
│     │         (SRTP encrypted audio/video)             │        │
│     │                                                   │        │
│                                                                   │
│  TURN Server (for NAT traversal in BD networks):                 │
│  ┌────────────────────────────────────────────┐                  │
│  │  - Bangladesh ISPs often use symmetric NAT │                  │
│  │  - TURN relay needed in ~30% of calls      │                  │
│  │  - TURN server in BD (low latency)         │                  │
│  │  - Fallback: Singapore TURN server         │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Status/Stories Feature

```php
<?php
// app/Services/StatusService.php

namespace App\Services;

use App\Models\Status;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Storage;

class StatusService
{
    /**
     * Status/Story upload (24-hour expiry - WhatsApp style)
     * বাংলাদেশে status দেখা অনেক popular (especially ছবি/ভিডিও)
     */
    public function createStatus(string $userId, array $data): Status
    {
        $status = Status::create([
            'id' => \Str::uuid(),
            'user_id' => $userId,
            'type' => $data['type'], // text, image, video
            'content' => $data['content'] ?? null,
            'media_url' => $data['media_url'] ?? null,
            'background_color' => $data['background_color'] ?? null,
            'expires_at' => now()->addHours(24),
            'privacy' => $data['privacy'] ?? 'contacts', // contacts, except, only
        ]);

        // Cache-এ store করা (fast reads)
        $this->cacheUserStatus($userId, $status);

        // Contacts-দের notify করা
        $this->notifyContacts($userId, $status);

        return $status;
    }

    /**
     * Contact-দের statuses fetch করা
     * Efficient: batch query + cache
     */
    public function getContactStatuses(string $userId): array
    {
        $contacts = $this->getContacts($userId);

        $cacheKey = "statuses:feed:{$userId}";
        return Cache::remember($cacheKey, 60, function () use ($contacts) {
            return Status::whereIn('user_id', $contacts)
                ->where('expires_at', '>', now())
                ->orderBy('created_at', 'desc')
                ->get()
                ->groupBy('user_id')
                ->toArray();
        });
    }
}
```

### 4. Message Backup & Restore

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKUP & RESTORE STRATEGY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Local Backup (Device Storage):                                  │
│  ├── Daily automatic backup (2 AM)                              │
│  ├── SQLite database + media files                              │
│  └── Encrypted with user's key                                  │
│                                                                   │
│  Cloud Backup (Google Drive / iCloud):                           │
│  ├── Weekly full backup                                         │
│  ├── Daily incremental backup                                   │
│  ├── E2E encrypted (user's password)                            │
│  └── Media optional (bandwidth concern in BD)                   │
│                                                                   │
│  Restore Flow:                                                   │
│  ┌─────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐       │
│  │Phone│───►│ Verify   │───►│Download │───►│ Decrypt  │       │
│  │Setup│    │ Number   │    │ Backup  │    │& Restore │       │
│  └─────┘    └──────────┘    └─────────┘    └──────────┘       │
│                                                                   │
│  Bangladesh-specific considerations:                             │
│  - Cloud backup-এ data cost concern (limited data plans)        │
│  - Local backup preferred (SD card)                             │
│  - WiFi-only backup option (সবার WiFi নেই)                     │
│  - Compressed backup (storage concern: 32GB phones common)      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 5. Spam Detection & Prevention

```javascript
// services/spam-detection.js
class SpamDetectionService {
    constructor() {
        this.RATE_LIMITS = {
            messages_per_minute: 30,
            new_conversations_per_hour: 20,
            group_joins_per_day: 10,
            forwards_per_minute: 5  // WhatsApp's forward limit
        };
    }

    /**
     * Multi-layer spam detection
     * বাংলাদেশে common spam: fake bKash offers, job scams, political propaganda
     */
    async checkMessage(userId, message, context) {
        const checks = await Promise.all([
            this.checkRateLimit(userId),
            this.checkContentPatterns(message),
            this.checkForwardChain(message),
            this.checkNewAccountBehavior(userId),
            this.checkBulkSending(userId),
        ]);

        const flags = checks.filter(c => c.flagged);

        if (flags.length >= 2) {
            return { blocked: true, reason: flags.map(f => f.reason) };
        }

        if (flags.length === 1) {
            return { blocked: false, warning: true, reason: flags[0].reason };
        }

        return { blocked: false };
    }

    async checkContentPatterns(message) {
        // Bangladesh-specific spam patterns
        const spamPatterns = [
            /bKash.*লটারি.*জিতেছেন/i,        // fake bKash lottery
            /বিনামূল্যে.*MB.*পাচ্ছেন/i,       // free data scam
            /forward.*to.*(\d+).*people/i,   // chain messages
            /job.*opportunity.*call/i,        // job scams
            /এই link.*click করুন/i,           // phishing links
        ];

        const flagged = spamPatterns.some(pattern =>
            pattern.test(message.content)
        );

        return {
            flagged,
            reason: 'spam_content_pattern',
            confidence: flagged ? 0.8 : 0
        };
    }

    /**
     * Forward chain detection
     * WhatsApp-এর "Forwarded many times" label
     */
    async checkForwardChain(message) {
        if (!message.isForwarded) return { flagged: false };

        const forwardCount = message.forwardCount || 0;

        // 5+ times forward → label + limit further forwarding
        if (forwardCount >= 5) {
            return {
                flagged: true,
                reason: 'excessive_forward',
                action: 'limit_to_one_chat' // can only forward to 1 chat at a time
            };
        }

        return { flagged: false };
    }
}

module.exports = new SpamDetectionService();
```

---

## 🎯 সারসংক্ষেপ

### Key Architectural Decisions

```
┌─────────────────────────────────────────────────────────────────┐
│                 ARCHITECTURE SUMMARY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Component          │ Technology         │ কেন?                  │
│  ─────────────────────────────────────────────────────────────   │
│  Real-time layer    │ Node.js + WS       │ Event-driven, non-    │
│                     │                    │ blocking I/O          │
│  Business logic     │ Laravel (PHP)      │ Rapid development,    │
│                     │                    │ clean architecture    │
│  Message store      │ Cassandra          │ Write-heavy, linear   │
│                     │                    │ scaling               │
│  Session/Presence   │ Redis              │ In-memory speed,      │
│                     │                    │ TTL support           │
│  Message broker     │ Kafka              │ Durability, replay,   │
│                     │                    │ ordering              │
│  Media storage      │ S3/MinIO + CDN     │ Scalable object       │
│                     │                    │ storage               │
│  Search             │ Elasticsearch      │ Full-text + Bangla    │
│  Push notifications │ FCM + APNs         │ Platform native       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Interview Tips

```
সাক্ষাৎকারে যা মনে রাখতে হবে:

1. START with requirements clarification:
   - "1:1 or group chat? Both?"
   - "E2E encryption needed?"
   - "Target scale?"

2. Back-of-envelope MUST do:
   - Messages/second calculation
   - Storage estimation
   - Connection count

3. HIGH-LEVEL first, then drill down:
   - Client → Gateway → Service → Storage
   - Don't jump to implementation details early

4. TRADE-OFFS are key:
   - Always explain WHY you chose something
   - "Cassandra because write-heavy + time-series access pattern"

5. Bangladesh-specific (if relevant):
   - Network challenges (2G/3G)
   - Event spikes (Eid)
   - Privacy vs compliance balance
```

### Further Reading

```
📚 Resources:
├── WhatsApp Engineering Blog
├── Signal Protocol Documentation
├── "Designing Data-Intensive Applications" - Martin Kleppmann
├── Facebook's TAO (distributed data store)
├── Discord Engineering (similar scale challenges)
└── WeChat Architecture (Asian market reference)

🔗 Related System Designs:
├── Notification System → push delivery deep dive
├── Rate Limiter → spam prevention
├── CDN Design → media delivery
├── Distributed Cache → presence/session
└── Message Queue → Kafka internals
```

---

> **লেখক নোট**: এই case study বাংলাদেশের প্রেক্ষাপটে লেখা হয়েছে। IMO, Messenger, এবং WhatsApp-এর ব্যবহার pattern থেকে শেখা যায় কিভাবে low-bandwidth, high-latency environment-এ reliable messaging system build করতে হয়। Rural connectivity এবং Eid-এর মতো event spike handling বাংলাদেশী engineers-দের জন্য critical skill।
