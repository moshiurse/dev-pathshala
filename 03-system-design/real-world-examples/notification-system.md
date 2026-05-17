# 🔔 নোটিফিকেশন সিস্টেম ডিজাইন

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: WhatsApp Notifications, Slack, bKash Alerts, Daraz Order Updates

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

| # | Feature | বর্ণনা |
|---|---------|--------|
| 1 | Push Notification | iOS (APNs) ও Android (FCM) এ push পাঠানো |
| 2 | SMS | Grameenphone, Robi, Banglalink সব operator এ SMS delivery |
| 3 | Email | Transactional ও promotional email পাঠানো |
| 4 | In-App | App এর ভেতর notification bell এ দেখানো |
| 5 | WebSocket Real-time | Live update — চ্যাট, ride tracking, payment confirmation |
| 6 | User Preferences | ইউজার কোন channel এ notification চায় সেটা manage করা |
| 7 | Template Management | Dynamic template দিয়ে personalized message তৈরি |
| 8 | Scheduling | নির্দিষ্ট সময়ে notification পাঠানো (DND respect করা) |

### Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────────────┐
│  Non-Functional Requirements                                     │
├─────────────────────────────────────────────────────────────────┤
│  • Delivery Guarantee : At-least-once delivery                   │
│  • Latency           : < 5 seconds end-to-end (P99)             │
│  • Throughput        : 10B+ notifications/day                    │
│  • Availability      : 99.99% uptime                            │
│  • Scalability       : Horizontal scaling for spike handling     │
│  • Deduplication     : Same notification দুবার না পাঠানো         │
│  • Rate Limiting     : Per-user rate limit (spam prevention)     │
│  • Observability     : Delivery tracking, open rate analytics    │
└─────────────────────────────────────────────────────────────────┘
```

### 🇧🇩 বাংলাদেশ Context

- **bKash**: প্রতিটা transaction এর পর instant SMS + push — ঈদের দিন 50M+ alerts
- **Daraz**: Order placed → shipped → out for delivery → delivered — multi-step tracking
- **Pathao**: Ride accept → driver location → arrival → trip complete — real-time WebSocket
- **Grameenphone**: Balance deduction SMS, offer notifications, recharge confirmation
- **Nagad**: Payment confirmation, cashback notification, merchant payment alert

---

## 📊 Back-of-envelope Estimation

### ধারণা ও সংখ্যা

```
ইউজার সংখ্যা:
  - Total registered users    : 100M (bKash scale)
  - Daily Active Users (DAU)  : 40M
  - Average notifications/user: 5/day

Daily Notifications:
  - Total                     : 40M × 5 = 200M/day
  - Peak (Eid/Flash Sale)     : 10x = 2B/day
  - Per second (average)      : 200M / 86400 ≈ 2,300 notifications/sec
  - Per second (peak)         : 2B / 86400 ≈ 23,000 notifications/sec

Channel Breakdown:
  - Push (60%)                : 120M/day
  - SMS (20%)                 : 40M/day
  - Email (10%)               : 20M/day
  - In-App (10%)              : 20M/day

Storage:
  - Per notification record   : ~500 bytes
  - Daily storage             : 200M × 500B = 100GB/day
  - Monthly storage           : 100GB × 30 = 3TB/month
  - With 90-day retention     : ~9TB active storage

Bandwidth:
  - Outgoing (avg)            : 200M × 1KB = 200GB/day
  - Peak bandwidth            : ~20 Gbps burst
```

### Infrastructure প্রয়োজন

```
┌──────────────────────────────────────────────────┐
│  Component          │ Instances │ Spec            │
├──────────────────────────────────────────────────┤
│  API Gateway        │ 10        │ 8 CPU, 16GB RAM │
│  Notification Router│ 20        │ 4 CPU, 8GB RAM  │
│  Push Workers       │ 50        │ 4 CPU, 8GB RAM  │
│  SMS Workers        │ 30        │ 2 CPU, 4GB RAM  │
│  Email Workers      │ 20        │ 4 CPU, 8GB RAM  │
│  WebSocket Servers  │ 40        │ 8 CPU, 32GB RAM │
│  Redis Cluster      │ 6         │ 16GB RAM each   │
│  Kafka Cluster      │ 12        │ 8 CPU, 32GB RAM │
│  PostgreSQL (Write) │ 3         │ 16 CPU, 64GB    │
│  PostgreSQL (Read)  │ 9         │ 16 CPU, 64GB    │
└──────────────────────────────────────────────────┘
```

---

## 🏗️ High-Level Design

### System Architecture (ASCII Diagram)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        NOTIFICATION SYSTEM ARCHITECTURE                        │
└──────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  bKash App  │  │  Daraz App  │  │ Pathao App  │  │  Admin Panel│
  │  (Payment)  │  │  (E-comm)   │  │  (Ride)     │  │  (Campaign) │
  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
         │                 │                 │                 │
         └────────────────┬┴─────────────────┴─────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │      API Gateway      │
              │  (Rate Limit, Auth)   │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Notification Service │
              │  (Validate, Enrich)   │
              └───────────┬───────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │      MESSAGE QUEUE (Kafka)      │
         │                                  │
         │  ┌──────┐ ┌──────┐ ┌──────────┐│
         │  │ HIGH │ │ MED  │ │   LOW    ││
         │  │PRIOR.│ │PRIOR.│ │ PRIORITY ││
         │  └──┬───┘ └──┬───┘ └────┬─────┘│
         └─────┼─────────┼──────────┼──────┘
               │         │          │
               ▼         ▼          ▼
      ┌────────────────────────────────────┐
      │       NOTIFICATION ROUTER          │
      │  (Channel Selection + Preference)  │
      └──┬──────────┬──────────┬───────┬───┘
         │          │          │       │
         ▼          ▼          ▼       ▼
   ┌──────────┐ ┌────────┐ ┌──────┐ ┌──────────┐
   │  PUSH    │ │  SMS   │ │EMAIL │ │ WEBSOCKET│
   │ WORKER   │ │ WORKER │ │WORKER│ │  SERVER  │
   │          │ │        │ │      │ │          │
   │ ┌──────┐ │ │┌──────┐│ │┌────┐│ │┌────────┐│
   │ │ FCM  │ │ ││ GP   ││ ││SES ││ ││Socket. ││
   │ │ APNs │ │ ││ Robi ││ ││Send││ ││   io   ││
   │ └──────┘ │ ││ BL   ││ ││Grid││ ││        ││
   └──────────┘ │└──────┘│ │└────┘│ │└────────┘│
                └────────┘ └──────┘ └──────────┘
                     │          │         │
                     ▼          ▼         ▼
              ┌─────────────────────────────────┐
              │     DELIVERY STATUS TRACKER      │
              │  (Success/Fail/Retry Tracking)   │
              └─────────────────────────────────┘
                              │
                              ▼
              ┌─────────────────────────────────┐
              │      ANALYTICS & MONITORING     │
              │  (Open Rate, CTR, Delivery %)   │
              └─────────────────────────────────┘
```

### Priority Queue System

```
Priority Levels:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  P0 (CRITICAL) ──→ OTP, Payment Confirmation, Emergency Alert
       │              Process immediately, no batching
       │              Example: bKash OTP, Nagad payment confirm
       │
  P1 (HIGH) ──────→ Order Updates, Ride Status, Chat Messages
       │              Process within 2 seconds
       │              Example: Pathao driver assigned, Daraz shipped
       │
  P2 (MEDIUM) ────→ Promotional, Reminders, Social
       │              Process within 30 seconds, batchable
       │              Example: Daraz flash sale, GP offer
       │
  P3 (LOW) ───────→ Analytics, Weekly Digest, Newsletters
                      Process within minutes, heavily batched
                      Example: Weekly spending summary
```

---

## 💻 Detailed Design

### 1. Database Schema

```sql
-- ইউজার ও তাদের device/preference সংরক্ষণ
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    phone           VARCHAR(15) UNIQUE NOT NULL,  -- +8801XXXXXXXXX
    email           VARCHAR(255),
    created_at      TIMESTAMP DEFAULT NOW()
);

-- ইউজারের device tokens (multiple devices support)
CREATE TABLE user_devices (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    device_token    VARCHAR(512) NOT NULL,
    platform        VARCHAR(10) NOT NULL,  -- 'ios', 'android', 'web'
    is_active       BOOLEAN DEFAULT TRUE,
    last_active_at  TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- ইউজারের notification preference
CREATE TABLE notification_preferences (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    channel         VARCHAR(20) NOT NULL,  -- 'push', 'sms', 'email', 'in_app'
    category        VARCHAR(50) NOT NULL,  -- 'transaction', 'promotion', 'social'
    enabled         BOOLEAN DEFAULT TRUE,
    quiet_hours_start TIME,               -- DND start (e.g., 23:00)
    quiet_hours_end   TIME,               -- DND end (e.g., 07:00)
    UNIQUE(user_id, channel, category)
);

-- Notification records
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         BIGINT REFERENCES users(id),
    template_id     VARCHAR(100),
    channel         VARCHAR(20) NOT NULL,
    priority        SMALLINT DEFAULT 2,   -- 0=critical, 1=high, 2=medium, 3=low
    title           VARCHAR(255),
    body            TEXT,
    data            JSONB,                -- extra payload
    status          VARCHAR(20) DEFAULT 'pending',
    -- status: pending → queued → sent → delivered → read → failed
    scheduled_at    TIMESTAMP,
    sent_at         TIMESTAMP,
    delivered_at    TIMESTAMP,
    read_at         TIMESTAMP,
    retry_count     SMALLINT DEFAULT 0,
    idempotency_key VARCHAR(100) UNIQUE,  -- deduplication
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Notification templates
CREATE TABLE notification_templates (
    id              VARCHAR(100) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    channel         VARCHAR(20) NOT NULL,
    subject         VARCHAR(255),          -- email subject
    body_template   TEXT NOT NULL,          -- "আপনার bKash অ্যাকাউন্টে {{amount}} টাকা জমা হয়েছে"
    variables       JSONB,                 -- required variables list
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Delivery tracking ও analytics
CREATE TABLE delivery_logs (
    id              BIGSERIAL PRIMARY KEY,
    notification_id UUID REFERENCES notifications(id),
    event_type      VARCHAR(30),  -- 'sent', 'delivered', 'opened', 'clicked', 'bounced'
    provider        VARCHAR(50),  -- 'fcm', 'apns', 'gp_sms', 'ses'
    response_code   VARCHAR(20),
    error_message   TEXT,
    metadata        JSONB,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Index for fast queries
CREATE INDEX idx_notifications_user_status ON notifications(user_id, status);
CREATE INDEX idx_notifications_scheduled ON notifications(scheduled_at) WHERE status = 'pending';
CREATE INDEX idx_notifications_created ON notifications(created_at);
CREATE INDEX idx_delivery_logs_notification ON delivery_logs(notification_id);
```

### 2. PHP (Laravel) — Notification Dispatcher

```php
<?php
// app/Services/NotificationDispatcher.php

namespace App\Services;

use App\Models\Notification;
use App\Models\NotificationPreference;
use App\Jobs\SendPushNotification;
use App\Jobs\SendSmsNotification;
use App\Jobs\SendEmailNotification;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Str;

class NotificationDispatcher
{
    private const PRIORITY_QUEUES = [
        0 => 'notifications:critical',  // OTP, Payment
        1 => 'notifications:high',      // Order, Ride
        2 => 'notifications:medium',    // Promo, Social
        3 => 'notifications:low',       // Digest, Newsletter
    ];

    private const RATE_LIMITS = [
        'transaction' => ['limit' => 100, 'window' => 3600],
        'promotion'   => ['limit' => 5,   'window' => 86400],
        'social'      => ['limit' => 50,  'window' => 3600],
    ];

    /**
     * মূল dispatch method — সব notification এখান দিয়ে যায়
     */
    public function dispatch(array $payload): ?Notification
    {
        // Step 1: Idempotency check — duplicate notification আটকানো
        $idempotencyKey = $payload['idempotency_key'] ?? $this->generateIdempotencyKey($payload);
        if ($this->isDuplicate($idempotencyKey)) {
            return null;
        }

        // Step 2: User preference check
        $userId = $payload['user_id'];
        $category = $payload['category'] ?? 'general';
        $channels = $this->getEnabledChannels($userId, $category);

        if (empty($channels)) {
            return null;
        }

        // Step 3: Rate limit check
        if (!$this->checkRateLimit($userId, $category)) {
            return null;
        }

        // Step 4: DND (Do Not Disturb) check
        $channels = $this->filterByQuietHours($userId, $channels);

        // Step 5: Template rendering
        $rendered = $this->renderTemplate(
            $payload['template_id'],
            $payload['variables'] ?? []
        );

        // Step 6: Notification record তৈরি ও queue এ পাঠানো
        $notification = Notification::create([
            'user_id'         => $userId,
            'template_id'     => $payload['template_id'],
            'priority'        => $payload['priority'] ?? 2,
            'title'           => $rendered['title'],
            'body'            => $rendered['body'],
            'data'            => $payload['data'] ?? [],
            'status'          => 'queued',
            'idempotency_key' => $idempotencyKey,
            'scheduled_at'    => $payload['scheduled_at'] ?? now(),
        ]);

        // Step 7: প্রতিটা channel এ dispatch
        foreach ($channels as $channel) {
            $this->dispatchToChannel($notification, $channel);
        }

        return $notification;
    }

    /**
     * bKash-style transaction alert — high priority, multi-channel
     */
    public function sendTransactionAlert(int $userId, array $txnData): void
    {
        $this->dispatch([
            'user_id'     => $userId,
            'template_id' => 'bkash_transaction_alert',
            'category'    => 'transaction',
            'priority'    => 0, // CRITICAL
            'variables'   => [
                'amount'     => $txnData['amount'],
                'type'       => $txnData['type'], // 'send_money', 'cash_out', 'payment'
                'sender'     => $txnData['sender_name'],
                'balance'    => $txnData['new_balance'],
                'txn_id'     => $txnData['transaction_id'],
                'time'       => now()->format('d M Y, h:i A'),
            ],
            'idempotency_key' => "txn_{$txnData['transaction_id']}",
        ]);
    }

    /**
     * Daraz-style order status update
     */
    public function sendOrderUpdate(int $userId, array $orderData): void
    {
        $this->dispatch([
            'user_id'     => $userId,
            'template_id' => "order_status_{$orderData['status']}",
            'category'    => 'transaction',
            'priority'    => 1, // HIGH
            'variables'   => [
                'order_id'    => $orderData['order_id'],
                'item_name'   => $orderData['item_name'],
                'status'      => $orderData['status'],
                'eta'         => $orderData['estimated_delivery'] ?? null,
            ],
        ]);
    }

    private function dispatchToChannel(Notification $notification, string $channel): void
    {
        $queue = self::PRIORITY_QUEUES[$notification->priority];

        match ($channel) {
            'push'    => SendPushNotification::dispatch($notification)->onQueue($queue),
            'sms'     => SendSmsNotification::dispatch($notification)->onQueue($queue),
            'email'   => SendEmailNotification::dispatch($notification)->onQueue($queue),
            'in_app'  => $this->storeInApp($notification),
            'websocket' => $this->pushToWebSocket($notification),
        };
    }

    private function isDuplicate(string $key): bool
    {
        // Redis SETNX দিয়ে idempotency — 24 ঘণ্টা TTL
        return !Redis::set("idempotency:{$key}", 1, 'EX', 86400, 'NX');
    }

    private function checkRateLimit(int $userId, string $category): bool
    {
        $config = self::RATE_LIMITS[$category] ?? ['limit' => 50, 'window' => 3600];
        $key = "rate_limit:{$userId}:{$category}";

        $current = Redis::incr($key);
        if ($current === 1) {
            Redis::expire($key, $config['window']);
        }

        return $current <= $config['limit'];
    }

    private function getEnabledChannels(int $userId, string $category): array
    {
        return NotificationPreference::where('user_id', $userId)
            ->where('category', $category)
            ->where('enabled', true)
            ->pluck('channel')
            ->toArray();
    }

    private function filterByQuietHours(int $userId, array $channels): array
    {
        $now = now()->format('H:i:s');
        $preferences = NotificationPreference::where('user_id', $userId)
            ->whereIn('channel', $channels)
            ->get();

        return $preferences->filter(function ($pref) use ($now) {
            if (!$pref->quiet_hours_start || !$pref->quiet_hours_end) {
                return true;
            }
            // Quiet hours এর মধ্যে থাকলে filter out
            return !($now >= $pref->quiet_hours_start && $now <= $pref->quiet_hours_end);
        })->pluck('channel')->toArray();
    }

    private function renderTemplate(string $templateId, array $variables): array
    {
        $template = \Cache::remember(
            "template:{$templateId}",
            3600,
            fn() => \App\Models\NotificationTemplate::find($templateId)
        );

        $body = $template->body_template;
        foreach ($variables as $key => $value) {
            $body = str_replace("{{{$key}}}", $value, $body);
        }

        return ['title' => $template->subject ?? '', 'body' => $body];
    }

    private function pushToWebSocket(Notification $notification): void
    {
        Redis::publish("ws:user:{$notification->user_id}", json_encode([
            'type'  => 'notification',
            'id'    => $notification->id,
            'title' => $notification->title,
            'body'  => $notification->body,
            'data'  => $notification->data,
        ]));
    }

    private function storeInApp(Notification $notification): void
    {
        $notification->update(['channel' => 'in_app', 'status' => 'delivered']);
    }

    private function generateIdempotencyKey(array $payload): string
    {
        return md5(json_encode([
            $payload['user_id'],
            $payload['template_id'],
            $payload['variables'] ?? [],
            now()->format('Y-m-d-H'),
        ]));
    }
}
```

### 3. Laravel Job — Push Notification Worker

```php
<?php
// app/Jobs/SendPushNotification.php

namespace App\Jobs;

use App\Models\Notification;
use App\Models\UserDevice;
use App\Models\DeliveryLog;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SendPushNotification implements ShouldQueue
{
    use InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public array $backoff = [30, 120, 600]; // Exponential backoff

    public function __construct(private Notification $notification) {}

    public function handle(): void
    {
        $devices = UserDevice::where('user_id', $this->notification->user_id)
            ->where('is_active', true)
            ->get();

        foreach ($devices as $device) {
            try {
                $response = match ($device->platform) {
                    'android' => $this->sendViaFCM($device, $this->notification),
                    'ios'     => $this->sendViaAPNs($device, $this->notification),
                    'web'     => $this->sendViaWebPush($device, $this->notification),
                };

                DeliveryLog::create([
                    'notification_id' => $this->notification->id,
                    'event_type'      => 'sent',
                    'provider'        => $device->platform === 'ios' ? 'apns' : 'fcm',
                    'response_code'   => $response['status'],
                    'metadata'        => $response,
                ]);

            } catch (\Exception $e) {
                // Token invalid হলে device deactivate করো
                if ($this->isInvalidToken($e)) {
                    $device->update(['is_active' => false]);
                }

                DeliveryLog::create([
                    'notification_id' => $this->notification->id,
                    'event_type'      => 'failed',
                    'provider'        => $device->platform,
                    'error_message'   => $e->getMessage(),
                ]);

                throw $e; // Retry trigger
            }
        }

        $this->notification->update([
            'status'  => 'sent',
            'sent_at' => now(),
        ]);
    }

    private function sendViaFCM(UserDevice $device, Notification $notification): array
    {
        // Firebase Cloud Messaging API call
        $payload = [
            'message' => [
                'token' => $device->device_token,
                'notification' => [
                    'title' => $notification->title,
                    'body'  => $notification->body,
                ],
                'data'  => $notification->data ?? [],
                'android' => [
                    'priority' => $notification->priority <= 1 ? 'high' : 'normal',
                ],
            ],
        ];

        // HTTP call to FCM API
        $response = \Http::withToken($this->getFCMToken())
            ->post('https://fcm.googleapis.com/v1/projects/myapp/messages:send', $payload);

        return $response->json();
    }

    private function sendViaAPNs(UserDevice $device, Notification $notification): array
    {
        // Apple Push Notification service call
        // Implementation with JWT auth
        return ['status' => 'success'];
    }

    private function sendViaWebPush(UserDevice $device, Notification $notification): array
    {
        return ['status' => 'success'];
    }

    private function isInvalidToken(\Exception $e): bool
    {
        return str_contains($e->getMessage(), 'InvalidRegistration')
            || str_contains($e->getMessage(), 'NotRegistered');
    }

    private function getFCMToken(): string
    {
        return \Cache::remember('fcm_access_token', 3500, function () {
            // Google OAuth2 service account token generation
            return 'generated_token';
        });
    }
}
```

### 4. Node.js (Express) — WebSocket Real-time Delivery

```javascript
// src/websocket-server.js

const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const Redis = require('ioredis');
const jwt = require('jsonwebtoken');

const app = express();
const server = http.createServer(app);

// Socket.IO setup — sticky session এর জন্য Redis adapter
const io = new Server(server, {
  cors: { origin: '*' },
  transports: ['websocket', 'polling'],
  pingTimeout: 30000,
  pingInterval: 10000,
});

// Redis subscriber — Laravel থেকে notification receive করতে
const redisSub = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
  maxRetriesPerRequest: 3,
});

const redisClient = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
});

// Connected users tracking
const connectedUsers = new Map(); // userId -> Set<socketId>

/**
 * Authentication middleware
 * JWT verify করে user identify করা
 */
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token
      || socket.handshake.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return next(new Error('Authentication token required'));
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.user_id;
    socket.deviceId = socket.handshake.query.device_id || 'unknown';

    next();
  } catch (err) {
    next(new Error('Invalid authentication token'));
  }
});

/**
 * Connection handler
 * প্রতিটা connected user কে track করা
 */
io.on('connection', (socket) => {
  const userId = socket.userId;
  console.log(`User connected: ${userId}, Socket: ${socket.id}`);

  // User কে connected list এ add করো
  if (!connectedUsers.has(userId)) {
    connectedUsers.set(userId, new Set());
  }
  connectedUsers.get(userId).add(socket.id);

  // Redis এ online status update
  redisClient.sadd('online_users', userId.toString());
  redisClient.hset(`user:${userId}:sockets`, socket.id, Date.now());

  // User subscribe করলে Redis channel subscribe
  redisSub.subscribe(`ws:user:${userId}`);

  // Unread notifications পাঠাও (reconnection এ)
  sendUnreadNotifications(socket, userId);

  // Client থেকে notification read acknowledgment
  socket.on('notification:read', async (data) => {
    const { notificationId } = data;
    await markAsRead(userId, notificationId);
    socket.emit('notification:read:ack', { notificationId, success: true });
  });

  // Client typing indicator (chat feature — Pathao style)
  socket.on('typing:start', (data) => {
    socket.to(`chat:${data.chatId}`).emit('typing:update', {
      userId,
      isTyping: true,
    });
  });

  // Disconnect handling
  socket.on('disconnect', (reason) => {
    console.log(`User disconnected: ${userId}, Reason: ${reason}`);

    const userSockets = connectedUsers.get(userId);
    if (userSockets) {
      userSockets.delete(socket.id);
      if (userSockets.size === 0) {
        connectedUsers.delete(userId);
        redisClient.srem('online_users', userId.toString());
        redisSub.unsubscribe(`ws:user:${userId}`);
      }
    }

    redisClient.hdel(`user:${userId}:sockets`, socket.id);
  });
});

/**
 * Redis message handler
 * Laravel service থেকে published notification receive ও deliver করা
 */
redisSub.on('message', (channel, message) => {
  // channel format: "ws:user:{userId}"
  const userId = channel.split(':')[2];
  const notification = JSON.parse(message);

  deliverToUser(userId, notification);
});

/**
 * নির্দিষ্ট user এর সব connected socket এ notification পাঠানো
 */
function deliverToUser(userId, notification) {
  const userSockets = connectedUsers.get(userId);

  if (userSockets && userSockets.size > 0) {
    // User online — সব device এ পাঠাও
    userSockets.forEach((socketId) => {
      io.to(socketId).emit('notification:new', {
        id: notification.id,
        title: notification.title,
        body: notification.body,
        data: notification.data,
        timestamp: new Date().toISOString(),
      });
    });

    // Delivery status update
    redisClient.publish('notification:delivered', JSON.stringify({
      notificationId: notification.id,
      userId,
      deliveredAt: new Date().toISOString(),
    }));
  } else {
    // User offline — queue for later delivery
    redisClient.lpush(
      `offline_queue:${userId}`,
      JSON.stringify(notification)
    );
    // 7 দিন পর expire
    redisClient.expire(`offline_queue:${userId}`, 7 * 24 * 3600);
  }
}

/**
 * Reconnect এ unread notifications পাঠানো
 */
async function sendUnreadNotifications(socket, userId) {
  const offlineQueue = await redisClient.lrange(`offline_queue:${userId}`, 0, 49);

  if (offlineQueue.length > 0) {
    const notifications = offlineQueue.map((item) => JSON.parse(item));
    socket.emit('notification:batch', { notifications });

    // Queue clear করো
    await redisClient.del(`offline_queue:${userId}`);
  }
}

/**
 * Notification read mark করা
 */
async function markAsRead(userId, notificationId) {
  await redisClient.publish('notification:read', JSON.stringify({
    notificationId,
    userId,
    readAt: new Date().toISOString(),
  }));
}

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    connectedUsers: connectedUsers.size,
    uptime: process.uptime(),
  });
});

// Metrics endpoint
app.get('/metrics', (req, res) => {
  res.json({
    totalConnections: connectedUsers.size,
    totalSockets: [...connectedUsers.values()].reduce((acc, s) => acc + s.size, 0),
    memoryUsage: process.memoryUsage(),
  });
});

const PORT = process.env.WS_PORT || 3001;
server.listen(PORT, () => {
  console.log(`WebSocket server running on port ${PORT}`);
});
```

### 5. Retry & Deduplication Strategy

```
┌──────────────────────────────────────────────────────────────────┐
│                    RETRY STRATEGY (Exponential Backoff)            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Attempt 1 ──→ Fail ──→ Wait 30s ──→ Retry                      │
│   Attempt 2 ──→ Fail ──→ Wait 2min ──→ Retry                     │
│   Attempt 3 ──→ Fail ──→ Wait 10min ──→ Retry                    │
│   Attempt 4 ──→ Fail ──→ Dead Letter Queue (DLQ)                  │
│                                                                    │
│   Retry শর্ত:                                                      │
│   ✓ Network timeout → Retry                                       │
│   ✓ 5xx server error → Retry                                     │
│   ✗ 4xx client error → Don't retry (invalid token etc.)           │
│   ✗ Duplicate detected → Don't retry                              │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│                    DEDUPLICATION STRATEGY                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌────────────┐     ┌─────────────────┐     ┌──────────────┐   │
│   │ Incoming   │────→│ Check Redis     │────→│ Process if   │   │
│   │ Notification│     │ idempotency_key │     │ not exists   │   │
│   └────────────┘     └─────────────────┘     └──────────────┘   │
│                              │                                     │
│                              │ Key exists?                         │
│                              ▼                                     │
│                       ┌─────────────┐                             │
│                       │ DROP (dup)  │                             │
│                       └─────────────┘                             │
│                                                                    │
│   Idempotency Key = hash(user_id + template_id + variables + hr) │
│   TTL = 24 hours                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 6. Rate Limiting Per User

```
┌───────────────────────────────────────────────────────────────┐
│              RATE LIMITING CONFIGURATION                        │
├───────────────────────────────────────────────────────────────┤
│                                                                 │
│  Category       │ Limit/Window  │ উদাহরণ                       │
│  ───────────────┼───────────────┼──────────────────────────     │
│  Transaction    │ 100/hour      │ bKash payment alerts          │
│  OTP           │ 5/10min       │ Login OTP, verify OTP          │
│  Promotional   │ 5/day         │ Daraz sale, GP offers          │
│  Social        │ 50/hour       │ Like, comment, follow          │
│  System        │ 20/hour       │ App update, maintenance        │
│                                                                 │
│  Algorithm: Sliding Window Counter (Redis ZSET)                 │
│                                                                 │
│  Implementation:                                                │
│  ZADD rate:{userId}:{category} {timestamp} {notificationId}    │
│  ZREMRANGEBYSCORE rate:{userId}:{category} 0 {windowStart}     │
│  ZCARD rate:{userId}:{category} → current count                │
│                                                                 │
└───────────────────────────────────────────────────────────────┘
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. Push vs Pull (Mobile Notifications)

```
┌─────────────────────────────────────────────────────────────────┐
│  PUSH MODEL                    │  PULL MODEL                     │
├────────────────────────────────┼─────────────────────────────────┤
│  Server → Client instantly     │  Client polls server            │
│  Low latency (< 1s)           │  High latency (polling interval)│
│  Battery efficient (FCM/APNs) │  Battery drain (constant HTTP)  │
│  Complex infrastructure        │  Simple implementation          │
│  Delivery uncertain (offline) │  Guaranteed on poll             │
│                                │                                  │
│  ✅ Best for: Real-time alerts │  ✅ Best for: Non-urgent feeds  │
│  🇧🇩 bKash OTP, Pathao ride    │  🇧🇩 Daraz wishlist updates     │
├────────────────────────────────┴─────────────────────────────────┤
│  আমাদের সিদ্ধান্ত: PUSH (FCM/APNs) — real-time essential       │
│  Fallback: Pull for offline queue sync on app open              │
└─────────────────────────────────────────────────────────────────┘
```

### 2. At-least-once vs Exactly-once Delivery

```
┌─────────────────────────────────────────────────────────────────┐
│  AT-LEAST-ONCE                 │  EXACTLY-ONCE                   │
├────────────────────────────────┼─────────────────────────────────┤
│  Simple implementation         │  Complex (2PC, idempotency)     │
│  May send duplicates          │  No duplicates                   │
│  Higher throughput             │  Lower throughput (overhead)     │
│  Retry-friendly               │  Coordination required           │
│                                │                                  │
│  ✅ ভালো: সব notification      │  ✅ ভালো: Financial txn only    │
│  💰 সস্তা implementation       │  💰 ব্যয়বহুল infrastructure     │
├────────────────────────────────┴─────────────────────────────────┤
│  আমাদের সিদ্ধান্ত: At-least-once + Client-side deduplication    │
│  কারণ: Exactly-once অনেক complex, duplicate notification        │
│  user experience এ minor impact (vs missing notification)       │
│  bKash OTP: At-least-once — দুবার পেলেও সমস্যা নেই             │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Real-time Delivery: WebSocket vs SSE vs Long Polling

```
┌──────────────┬───────────────────┬──────────────┬────────────────┐
│  Feature     │  WebSocket        │  SSE         │  Long Polling  │
├──────────────┼───────────────────┼──────────────┼────────────────┤
│  Direction   │  Bidirectional    │  Server→Only │  Simulated     │
│  Connection  │  Persistent       │  Persistent  │  Repeated HTTP │
│  Latency     │  Very Low (~ms)   │  Low (~ms)   │  Medium (~s)   │
│  Scaling     │  Complex (sticky) │  Moderate    │  Simple        │
│  Battery     │  Moderate         │  Good        │  Poor          │
│  Proxy/FW    │  Sometimes blocked│  HTTP-based  │  Always works  │
│  Use Case    │  Chat, Location   │  Feed, Alert │  Fallback      │
├──────────────┴───────────────────┴──────────────┴────────────────┤
│  আমাদের সিদ্ধান্ত:                                                │
│  Primary: WebSocket (Pathao ride tracking, chat)                  │
│  Fallback: SSE (notification feed)                                │
│  Last resort: Long Polling (restricted networks)                  │
└──────────────────────────────────────────────────────────────────┘
```

### 4. Single Queue vs Priority Queues

```
┌─────────────────────────────────────────────────────────────────┐
│  SINGLE QUEUE                  │  PRIORITY QUEUES                │
├────────────────────────────────┼─────────────────────────────────┤
│  Simple to manage              │  Complex but flexible            │
│  FIFO — fair ordering          │  Critical msgs get priority      │
│  Head-of-line blocking         │  No blocking for high priority   │
│  One consumer group            │  Multiple consumer groups        │
│                                │                                  │
│  সমস্যা: ঈদের দিন promo       │  সমাধান: OTP/payment আগে যায়,  │
│  blast করলে OTP আটকে যায়      │  promo পরে process হয়            │
├────────────────────────────────┴─────────────────────────────────┤
│  আমাদের সিদ্ধান্ত: PRIORITY QUEUES (4 levels)                    │
│  কারণ: bKash OTP কখনোই promotional notification এর পিছনে       │
│  আটকে থাকতে পারবে না — ইউজার experience critical                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📈 কেস স্টাডি

### কেস ১: bKash — ঈদের দিন 50M Transaction Alerts

```
সমস্যা:
━━━━━━━
ঈদুল ফিতরের দিন সকাল ৬টা থেকে রাত ১২টা পর্যন্ত bKash এ
স্বাভাবিকের ১০ গুণ transaction হয়। প্রতিটা transaction এর
সাথে sender ও receiver দুজনকেই instant SMS + push পাঠাতে হয়।

সংখ্যায়:
  - স্বাভাবিক দিন: 5M transactions → 10M notifications
  - ঈদের দিন: 25M transactions → 50M notifications
  - Peak hour (সন্ধ্যা ৬-৮): 5M notifications/hour = 1,400/sec

চ্যালেঞ্জ:
  1. SMS Gateway saturation (GP/Robi capacity limit)
  2. Push notification queue backlog
  3. Database write throughput
  4. OTP delivery must not be affected

সমাধান Architecture:
━━━━━━━━━━━━━━━━━━━━━

  [Transaction Service]
         │
         ▼
  [Priority Queue P0: OTP/Confirmation]──→ [Dedicated SMS Pool]
         │                                        │
         │                                   [GP Gateway]
         ▼                                   [Robi Gateway]
  [Priority Queue P1: Alerts]──→ [Scaled Workers (5x)]
         │                              │
         ▼                         [FCM Batch API]
  [Priority Queue P3: Promo]──→ [Delayed/Throttled]
         │
    (Eid promo paused)

মূল কৌশল:
  ✓ Auto-scaling: Workers ৫ গুণ বাড়ানো (20 → 100 instances)
  ✓ SMS Gateway failover: GP full হলে Robi, তারপর Banglalink
  ✓ Batch FCM: একবারে 500 device token পাঠানো (vs 1-by-1)
  ✓ Promotional notifications pause during peak
  ✓ Pre-warm: ঈদের ২ ঘণ্টা আগেই extra capacity provision

ফলাফল:
  - 99.7% delivery success rate
  - P99 latency: 3.2 seconds (OTP < 2s)
  - Zero OTP delivery failure
  - Cost: SMS gateway bill 3x normal (~15 lakh BDT)
```

### কেস ২: Daraz Flash Sale Notifications

```
সমস্যা:
━━━━━━━
Daraz 11.11 sale শুরুর ১ ঘণ্টা আগে 20M user কে একসাথে
push notification পাঠাতে হবে। সবাই একই সময়ে app open
করলে server crash হতে পারে।

চ্যালেঞ্জ:
  1. Thundering herd problem — সবাই একসাথে আসলে backend crash
  2. 20M push একসাথে পাঠানো FCM rate limit hit করবে
  3. Notification পাওয়ার পর app open — API server overload

সমাধান:
━━━━━━━━

  Time-staggered Delivery:
  ┌─────────────────────────────────────────────────────┐
  │  T-60min │ Segment A (5M): "Sale শুরু হতে ১ ঘণ্টা"  │
  │  T-30min │ Segment B (5M): "Sale শুরু হতে ৩০ মিনিট" │
  │  T-10min │ Segment C (5M): "Sale শুরু হতে ১০ মিনিট"│
  │  T-0     │ Segment D (5M): "Sale শুরু হয়েছে! 🎉"   │
  └─────────────────────────────────────────────────────┘

  FCM Topic Messaging:
  - User দের segment অনুযায়ী FCM topic এ subscribe করানো
  - Topic message = single API call → millions delivery
  - Individual token push এর বদলে topic push

  Backend Protection:
  - CDN pre-warming for sale page
  - API rate limiting per user
  - Graceful degradation — heavy load এ simplified response

ফলাফল:
  - 18.5M successfully delivered (92.5% delivery rate)
  - App open rate: 34% (6.8M users)
  - Server load evenly distributed
  - Zero downtime during sale
```

### কেস ৩: Emergency Alert System (Cyclone Warning)

```
সমস্যা:
━━━━━━━
বাংলাদেশে ঘূর্ণিঝড় সতর্কতা — উপকূলীয় ৫ কোটি মানুষকে
১৫ মিনিটের মধ্যে alert পৌঁছাতে হবে। Network infrastructure
ক্ষতিগ্রস্ত হতে পারে।

Requirements:
  - 50M people in 15 minutes
  - Multiple channels simultaneously
  - Works even with degraded network
  - Bengali language support mandatory

সমাধান:
━━━━━━━━

  ┌─────────────────────────────────────────────┐
  │           EMERGENCY ALERT FLOW               │
  ├─────────────────────────────────────────────┤
  │                                              │
  │  [Disaster Management Authority]             │
  │              │                               │
  │              ▼                               │
  │  [Emergency API — P0 Queue, bypass all]     │
  │              │                               │
  │    ┌─────────┼──────────┬─────────┐         │
  │    ▼         ▼          ▼         ▼         │
  │  [Cell     [Push     [SMS      [TV/Radio   │
  │  Broadcast] Notif]   Blast]    Alert]      │
  │    │         │          │         │         │
  │    ▼         ▼          ▼         ▼         │
  │  All phones  App users  All SIMs  Broadcast │
  │  in area    (online)   (telco)   stations   │
  │                                              │
  └─────────────────────────────────────────────┘

  বিশেষ ব্যবস্থা:
  ✓ Cell Broadcast: No internet needed, reaches all phones in area
  ✓ Geo-fencing: শুধু উপকূলীয় এলাকার users কে target
  ✓ All rate limits bypassed for emergency
  ✓ Multi-language: বাংলা + English simultaneously
  ✓ Retry aggressive: 10 attempts in 5 minutes
  ✓ Telco partnership: Dedicated SMS gateway capacity
```

---

## 🔧 Advanced Topics

### 1. Notification Aggregation/Batching

```
সমস্যা: একজন user ১ ঘণ্টায় ২০টা like notification পেলে বিরক্ত হয়

সমাধান: Intelligent Batching

  ┌──────────────────────────────────────────────────────────┐
  │  BEFORE (Annoying):                                       │
  │    "রহিম আপনার পোস্ট লাইক করেছে"                          │
  │    "করিম আপনার পোস্ট লাইক করেছে"                          │
  │    "সালমা আপনার পোস্ট লাইক করেছে"                         │
  │    ... (20 more)                                           │
  │                                                            │
  │  AFTER (Aggregated):                                       │
  │    "রহিম, করিম ও আরও ২০ জন আপনার পোস্ট লাইক করেছে"      │
  └──────────────────────────────────────────────────────────┘

  Algorithm:
  1. Notification আসলে buffer এ রাখো (Redis sorted set)
  2. Buffer window: 5 minutes (configurable per category)
  3. Window শেষে aggregate করে single notification পাঠাও
  4. Exception: P0/P1 — কখনো batch করা হবে না
```

```php
<?php
// app/Services/NotificationAggregator.php

class NotificationAggregator
{
    private const BATCH_WINDOWS = [
        'social_like'    => 300,  // 5 min
        'social_comment' => 180,  // 3 min
        'social_follow'  => 600,  // 10 min
    ];

    public function addToBatch(int $userId, string $category, array $data): void
    {
        $key = "batch:{$userId}:{$category}";
        $window = self::BATCH_WINDOWS[$category] ?? 300;

        Redis::zadd($key, time(), json_encode($data));
        Redis::expire($key, $window + 60);

        // প্রথম item হলে delayed job schedule করো
        $count = Redis::zcard($key);
        if ($count === 1) {
            ProcessBatchNotification::dispatch($userId, $category)
                ->delay(now()->addSeconds($window));
        }
    }

    public function processBatch(int $userId, string $category): void
    {
        $key = "batch:{$userId}:{$category}";
        $items = Redis::zrange($key, 0, -1);

        if (empty($items)) return;

        $count = count($items);
        $items = array_map(fn($item) => json_decode($item, true), $items);

        // Aggregate message তৈরি
        $message = $this->buildAggregateMessage($category, $items, $count);

        // Single notification dispatch
        app(NotificationDispatcher::class)->dispatch([
            'user_id'     => $userId,
            'template_id' => "aggregated_{$category}",
            'category'    => 'social',
            'priority'    => 2,
            'variables'   => [
                'names' => $this->formatNames($items),
                'count' => $count,
                'extra' => $count > 3 ? $count - 3 : 0,
            ],
        ]);

        Redis::del($key);
    }

    private function formatNames(array $items): string
    {
        $names = array_column(array_slice($items, 0, 2), 'actor_name');
        $extra = count($items) - 2;

        if ($extra > 0) {
            return implode(', ', $names) . " ও আরও {$extra} জন";
        }
        return implode(' ও ', $names);
    }
}
```

### 2. Do Not Disturb (DND) Scheduling

```
┌──────────────────────────────────────────────────────────────────┐
│                   DND SCHEDULING SYSTEM                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  User Settings:                                                    │
│    ┌─────────────────────────────────────┐                        │
│    │  🌙 Quiet Hours: 11:00 PM - 7:00 AM │                        │
│    │  📱 Allow: OTP, Emergency only       │                        │
│    │  🔕 Mute: Promotional, Social        │                        │
│    └─────────────────────────────────────┘                        │
│                                                                    │
│  Flow:                                                             │
│    Notification arrives                                            │
│         │                                                          │
│         ▼                                                          │
│    Is it DND time? ──No──→ Deliver immediately                    │
│         │                                                          │
│        Yes                                                         │
│         │                                                          │
│         ▼                                                          │
│    Is it P0 (OTP/Emergency)? ──Yes──→ Deliver immediately         │
│         │                                                          │
│        No                                                          │
│         │                                                          │
│         ▼                                                          │
│    Queue for DND end time                                          │
│    (Schedule at 7:00 AM)                                          │
│         │                                                          │
│         ▼                                                          │
│    At 7:00 AM: Deliver aggregated summary                         │
│    "রাতে ৫টি notification এসেছে"                                   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### 3. A/B Testing Notifications

```javascript
// src/ab-testing/notification-experiment.js

class NotificationExperiment {
  constructor(redisClient, analyticsService) {
    this.redis = redisClient;
    this.analytics = analyticsService;
  }

  /**
   * Daraz style: কোন notification copy বেশি effective?
   * Variant A: "Flash Sale! ৭০% ছাড়! ⚡"
   * Variant B: "আপনার Wishlist এ থাকা Samsung Phone এখন ৫,০০০৳ কমে!"
   */
  async getVariant(userId, experimentId) {
    // Consistent hashing — same user always gets same variant
    const bucket = this.hashUser(userId, experimentId) % 100;

    const experiment = await this.redis.hgetall(`experiment:${experimentId}`);
    const variants = JSON.parse(experiment.variants);

    // Traffic split: A=50%, B=50%
    let cumulative = 0;
    for (const variant of variants) {
      cumulative += variant.trafficPercent;
      if (bucket < cumulative) {
        // Track assignment
        await this.analytics.track('experiment_assigned', {
          experimentId,
          userId,
          variantId: variant.id,
        });
        return variant;
      }
    }

    return variants[0]; // fallback
  }

  /**
   * Track notification outcomes
   */
  async trackOutcome(experimentId, variantId, event) {
    const key = `experiment:${experimentId}:${variantId}:${event}`;
    await this.redis.incr(key);
  }

  /**
   * Statistical significance check
   */
  async getResults(experimentId) {
    const variants = ['A', 'B'];
    const results = {};

    for (const v of variants) {
      const sent = await this.redis.get(`experiment:${experimentId}:${v}:sent`) || 0;
      const opened = await this.redis.get(`experiment:${experimentId}:${v}:opened`) || 0;
      const clicked = await this.redis.get(`experiment:${experimentId}:${v}:clicked`) || 0;

      results[v] = {
        sent: parseInt(sent),
        opened: parseInt(opened),
        clicked: parseInt(clicked),
        openRate: sent > 0 ? (opened / sent * 100).toFixed(2) : 0,
        clickRate: sent > 0 ? (clicked / sent * 100).toFixed(2) : 0,
      };
    }

    return results;
  }

  hashUser(userId, experimentId) {
    const str = `${userId}:${experimentId}`;
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit int
    }
    return Math.abs(hash);
  }
}

module.exports = NotificationExperiment;
```

### 4. Analytics & Tracking

```
┌──────────────────────────────────────────────────────────────────┐
│              NOTIFICATION ANALYTICS PIPELINE                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Events Tracked:                                                   │
│    📤 sent       → Notification server থেকে পাঠানো হয়েছে         │
│    📬 delivered  → Device/provider confirm করেছে                   │
│    👁️ opened     → User notification দেখেছে                       │
│    👆 clicked    → User action নিয়েছে (CTA click)                 │
│    🚫 dismissed  → User dismiss করেছে                              │
│    ❌ bounced    → Delivery fail (invalid token, etc.)             │
│    🔕 unsubscribed → User notification off করেছে                   │
│                                                                    │
│  Metrics Dashboard:                                                │
│  ┌─────────────────────────────────────────────┐                  │
│  │  Channel  │ Sent  │ Delivered │ Open% │ CTR% │                  │
│  │  ─────────┼───────┼──────────┼───────┼───── │                  │
│  │  Push     │ 120M  │ 108M     │ 12%   │ 4.2% │                  │
│  │  SMS      │ 40M   │ 38M      │ N/A   │ N/A  │                  │
│  │  Email    │ 20M   │ 18M      │ 22%   │ 3.8% │                  │
│  │  In-App   │ 20M   │ 20M      │ 45%   │ 15%  │                  │
│  └─────────────────────────────────────────────┘                  │
│                                                                    │
│  Data Flow:                                                        │
│    Event → Kafka → Stream Processing → ClickHouse → Dashboard     │
│                         │                                          │
│                         ▼                                          │
│                   Real-time Alerting                                │
│                   (delivery rate < 90% → alert)                    │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

```php
<?php
// app/Services/NotificationAnalytics.php

class NotificationAnalytics
{
    public function trackEvent(string $notificationId, string $event, array $metadata = []): void
    {
        $payload = [
            'notification_id' => $notificationId,
            'event'           => $event,
            'timestamp'       => now()->toISOString(),
            'metadata'        => $metadata,
        ];

        // Kafka এ publish — async processing
        \Kafka::produce('notification_events', $payload);

        // Real-time counter update (Redis)
        $date = now()->format('Y-m-d');
        $channel = $metadata['channel'] ?? 'unknown';

        Redis::hincrby("analytics:{$date}:{$channel}", $event, 1);
    }

    public function getDailyStats(string $date, string $channel): array
    {
        $stats = Redis::hgetall("analytics:{$date}:{$channel}");

        $sent = (int) ($stats['sent'] ?? 0);
        $delivered = (int) ($stats['delivered'] ?? 0);
        $opened = (int) ($stats['opened'] ?? 0);
        $clicked = (int) ($stats['clicked'] ?? 0);

        return [
            'sent'          => $sent,
            'delivered'     => $delivered,
            'opened'        => $opened,
            'clicked'       => $clicked,
            'delivery_rate' => $sent > 0 ? round($delivered / $sent * 100, 2) : 0,
            'open_rate'     => $delivered > 0 ? round($opened / $delivered * 100, 2) : 0,
            'click_rate'    => $opened > 0 ? round($clicked / $opened * 100, 2) : 0,
        ];
    }

    /**
     * Alert trigger — delivery rate হঠাৎ কমে গেলে on-call কে notify
     */
    public function checkHealthAlerts(): void
    {
        $stats = $this->getDailyStats(now()->format('Y-m-d'), 'push');

        if ($stats['delivery_rate'] < 90) {
            // PagerDuty/Slack alert trigger
            event(new DeliveryRateDropped($stats));
        }
    }
}
```

---

## 🎯 সারসংক্ষেপ

### Key Design Decisions

```
┌──────────────────────────────────────────────────────────────────┐
│                    DESIGN DECISIONS SUMMARY                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. Architecture: Event-driven, microservice-based                │
│     কারণ: Independent scaling per channel                        │
│                                                                    │
│  2. Queue: Kafka with priority partitions                         │
│     কারণ: High throughput + ordering guarantee per partition     │
│                                                                    │
│  3. Delivery: At-least-once + client deduplication                │
│     কারণ: Missing notification > duplicate notification          │
│                                                                    │
│  4. Real-time: WebSocket (primary) + SSE (fallback)              │
│     কারণ: Bidirectional needed for chat/ride tracking            │
│                                                                    │
│  5. Storage: PostgreSQL (metadata) + Redis (state/queue)          │
│     কারণ: ACID for records, speed for real-time ops             │
│                                                                    │
│  6. Rate Limiting: Sliding window per user per category           │
│     কারণ: Fair usage + spam prevention                           │
│                                                                    │
│  7. Retry: Exponential backoff with DLQ                           │
│     কারণ: Graceful failure handling without infinite loops       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### বাংলাদেশ Context এ গুরুত্বপূর্ণ বিবেচনা

```
🇧🇩 Bangladesh-specific Considerations:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. SMS Gateway Diversity
     - একাধিক telecom provider (GP, Robi, BL, Teletalk)
     - Each has different API, rate limits, capacity
     - Failover strategy essential

  2. Internet Connectivity
     - 4G coverage ভালো কিন্তু stable না সবসময়
     - Push notification delivery uncertain in rural areas
     - SMS fallback critical for financial notifications

  3. Dual Language Support
     - বাংলা ও English উভয়ই support করতে হবে
     - Unicode SMS costs more (70 chars vs 160 chars)
     - User preference based language selection

  4. Peak Traffic Patterns
     - ঈদ (Eid): Payment apps spike 10x
     - 11.11/12.12: E-commerce notification flood
     - Cricket match: Social notification burst
     - Ramadan: Food delivery app surge at Iftar

  5. Cost Optimization
     - SMS cost: ০.২৫-০.৫০ টাকা per SMS
     - Push notification: Free (FCM/APNs)
     - Strategy: Push first, SMS only for critical/offline users

  6. Regulatory Compliance
     - BTRC regulations on bulk SMS
     - Opt-in requirement for promotional messages
     - Data privacy (user consent for notifications)
```

### Interview Tips

```
📝 System Design Interview এ যা মনে রাখতে হবে:

  ✓ Requirements clarify করো — "কতজন user? কত notification/day?"
  ✓ Back-of-envelope calculation দেখাও
  ✓ High-level architecture আগে, তারপর deep dive
  ✓ Trade-offs discuss করো — কেন এটা choose করলে
  ✓ Failure scenarios handle করো — "যদি SMS gateway down থাকে?"
  ✓ Scalability plan বলো — "10x traffic হলে কী করবে?"
  ✓ Real examples দাও — bKash, Pathao, Daraz experiences

  ❌ যা করবে না:
  ✗ Perfect system design করতে যেও না — trade-offs acknowledge করো
  ✗ একটা technology এ locked থেকো না — alternatives discuss করো
  ✗ Security/privacy ভুলে যেও না
  ✗ Monitoring/observability skip করো না
```

---

> **📚 আরও পড়ুন**: [Message Queue Design](./message-queue.md) | [Real-time System](./real-time-system.md)
>
> **🔗 References**: Firebase Cloud Messaging Docs, AWS SNS Architecture, Kafka Streams Documentation
