# 🔔 নোটিফিকেশন সিস্টেম (Notification System Design)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**নোটিফিকেশন সিস্টেম** হলো এমন একটি সিস্টেম যা ইউজারকে গুরুত্বপূর্ণ ঘটনা সম্পর্কে রিয়েল-টাইমে বা নির্ধারিত সময়ে জানায়। এটি Push Notification, Email, SMS, In-App Notification ইত্যাদি চ্যানেলের মাধ্যমে কাজ করে।

### নোটিফিকেশনের ধরন

```
১. Push Notification  → মোবাইল/ব্রাউজার পুশ (FCM, APNs)
২. In-App Notification → অ্যাপের ভেতরে বেল আইকনে দেখায়
৩. SMS                → টেক্সট মেসেজ (OTP, ট্রানজেকশন)
৪. Email              → ইমেইল (মার্কেটিং, রিসিপ্ট)
```

---

## 🎯 বাস্তব উদাহরণ — bKash ট্রানজেকশন নোটিফিকেশন

```
আপনি bKash দিয়ে ১০০০ টাকা সেন্ড করলেন:

  ১. Push Notification → ফোনে "আপনার ১০০০ টাকা পাঠানো হয়েছে" 📱
  ২. SMS              → "bKash: Tk 1000 sent to 01712XXXXXX" 📩
  ৩. In-App           → bKash অ্যাপে ট্রানজেকশন হিস্ট্রিতে 📋
  ৪. Email            → রিসিপ্ট ইমেইল (যদি সেটিং চালু থাকে) ✉️

  প্রতিটি চ্যানেল আলাদা সিস্টেম দিয়ে চলে!
```

---

## 📊 সিস্টেম আর্কিটেকচার

```
          নোটিফিকেশন সিস্টেম আর্কিটেকচার
          =================================

  ┌──────────┐     ┌──────────────┐     ┌──────────────┐
  │ Service  │────→│ Notification │────→│ Message      │
  │ (Order,  │     │ Service      │     │ Queue        │
  │  Payment)│     │ (API)        │     │ (Kafka/SQS)  │
  └──────────┘     └──────────────┘     └──────┬───────┘
                                               │
                        ┌──────────────────────┼───────────────┐
                        ▼                      ▼               ▼
                  ┌───────────┐         ┌───────────┐   ┌───────────┐
                  │ Push      │         │ SMS       │   │ Email     │
                  │ Worker    │         │ Worker    │   │ Worker    │
                  │ (FCM/APNs)│         │ (Twilio)  │   │ (SES)    │
                  └───────────┘         └───────────┘   └───────────┘
```

---

## 🔄 রিয়েল-টাইম কমিউনিকেশন পদ্ধতি

### ১. Polling (পোলিং)

```
ক্লায়েন্ট বারবার সার্ভারকে জিজ্ঞেস করে: "নতুন কিছু আছে?"

Client: নতুন নোটিফিকেশন?  → Server: না
Client: নতুন নোটিফিকেশন?  → Server: না
Client: নতুন নোটিফিকেশন?  → Server: হ্যাঁ! ১টি আছে ✅

সমস্যা: অপ্রয়োজনীয় রিকোয়েস্ট, সার্ভারে লোড বাড়ে
ব্যবহার: সাধারণ সিস্টেম, কম ইউজার
```

### ২. Long Polling

```
ক্লায়েন্ট রিকোয়েস্ট পাঠায়, সার্ভার ডেটা না থাকলে ধরে রাখে

Client: নতুন কিছু আছে? → Server: (অপেক্ষা করছে... ৩০ সেকেন্ড)
                                    ↓
                          নোটিফিকেশন এলো! → Response পাঠাও

সুবিধা: অপ্রয়োজনীয় রিকোয়েস্ট কম
অসুবিধা: সার্ভারে অনেক ওপেন কানেকশন থাকে
```

### ৩. WebSocket (ওয়েবসকেট)

```
ক্লায়েন্ট ও সার্ভারের মধ্যে স্থায়ী দ্বিমুখী সংযোগ

  Client ←──────────────→ Server
          Persistent TCP
          Bi-directional

  সার্ভার যেকোনো সময় ক্লায়েন্টকে ডেটা পাঠাতে পারে
  ক্লায়েন্টও সার্ভারকে পাঠাতে পারে

সুবিধা: সত্যিকারের রিয়েল-টাইম, কম লেটেন্সি
অসুবিধা: স্টেটফুল, স্কেলিং কঠিন
ব্যবহার: চ্যাট, লাইভ নোটিফিকেশন, গেমিং
```

### ৪. SSE (Server-Sent Events)

```
সার্ভার থেকে ক্লায়েন্টে একমুখী ডেটা স্ট্রিম

  Client ←────────────── Server
          One-way stream
          (HTTP-based)

সুবিধা: সহজ, HTTP-ভিত্তিক, অটো-রিকানেক্ট
অসুবিধা: শুধু একমুখী (সার্ভার → ক্লায়েন্ট)
ব্যবহার: নোটিফিকেশন ফিড, লাইভ স্কোর, স্টক প্রাইস
```

### তুলনা

| বৈশিষ্ট্য | Polling | Long Polling | WebSocket | SSE |
|---|---|---|---|---|
| দিক | একমুখী | একমুখী | দ্বিমুখী | একমুখী |
| লেটেন্সি | বেশি | মাঝারি | খুব কম | কম |
| সার্ভার লোড | বেশি | মাঝারি | কম | কম |
| জটিলতা | সহজ | মাঝারি | কঠিন | সহজ |
| স্কেলিং | সহজ | মাঝারি | কঠিন | মাঝারি |

---

## 💻 PHP কোড উদাহরণ

```php
<?php

// নোটিফিকেশন সার্ভিস
class NotificationService
{
    private array $channels = [];
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function registerChannel(string $name, NotificationChannel $channel): void
    {
        $this->channels[$name] = $channel;
    }

    public function send(int $userId, string $type, array $data, array $channels): void
    {
        // ডেটাবেসে নোটিফিকেশন সেভ করো
        $stmt = $this->db->prepare(
            'INSERT INTO notifications (user_id, type, data, created_at) VALUES (?, ?, ?, NOW())'
        );
        $stmt->execute([$userId, $type, json_encode($data)]);
        $notificationId = $this->db->lastInsertId();

        // প্রতিটি চ্যানেলে পাঠাও
        foreach ($channels as $channelName) {
            if (isset($this->channels[$channelName])) {
                $this->channels[$channelName]->deliver($userId, $data);
            }
        }
    }

    // SSE এন্ডপয়েন্ট — রিয়েল-টাইম নোটিফিকেশন স্ট্রিম
    public function sseStream(int $userId): void
    {
        header('Content-Type: text/event-stream');
        header('Cache-Control: no-cache');
        header('Connection: keep-alive');

        $lastId = 0;
        while (true) {
            $stmt = $this->db->prepare(
                'SELECT * FROM notifications WHERE user_id = ? AND id > ? AND is_read = 0 ORDER BY id'
            );
            $stmt->execute([$userId, $lastId]);
            $notifications = $stmt->fetchAll(PDO::FETCH_ASSOC);

            foreach ($notifications as $n) {
                echo "id: {$n['id']}\n";
                echo "event: notification\n";
                echo "data: " . json_encode($n) . "\n\n";
                $lastId = $n['id'];
            }

            ob_flush();
            flush();
            sleep(2);
        }
    }
}

// চ্যানেল ইন্টারফেস
interface NotificationChannel
{
    public function deliver(int $userId, array $data): bool;
}

// পুশ নোটিফিকেশন চ্যানেল (FCM)
class PushChannel implements NotificationChannel
{
    public function deliver(int $userId, array $data): bool
    {
        $token = $this->getDeviceToken($userId);
        $payload = [
            'to' => $token,
            'notification' => [
                'title' => $data['title'],
                'body'  => $data['body'],
            ],
        ];
        // FCM API কল
        $ch = curl_init('https://fcm.googleapis.com/fcm/send');
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        $result = curl_exec($ch);
        curl_close($ch);
        return $result !== false;
    }

    private function getDeviceToken(int $userId): string
    {
        // ডেটাবেস থেকে ডিভাইস টোকেন আনো
        return 'device_token_placeholder';
    }
}

// ব্যবহার
$service = new NotificationService($pdo);
$service->registerChannel('push', new PushChannel());
$service->send(
    userId: 1001,
    type: 'transaction',
    data: ['title' => 'টাকা পাঠানো হয়েছে', 'body' => '১০০০ টাকা সেন্ড হয়েছে'],
    channels: ['push', 'sms']
);
```

---

## 💻 JavaScript কোড উদাহরণ

```javascript
// ক্লায়েন্ট সাইড — নোটিফিকেশন ম্যানেজার
class NotificationManager {
  constructor(userId) {
    this.userId = userId;
    this.listeners = new Map();
  }

  // SSE দিয়ে রিয়েল-টাইম নোটিফিকেশন শোনো
  connectSSE() {
    const eventSource = new EventSource(`/api/notifications/stream?userId=${this.userId}`);

    eventSource.addEventListener('notification', (event) => {
      const notification = JSON.parse(event.data);
      this._handleNotification(notification);
    });

    eventSource.onerror = () => {
      console.warn('SSE সংযোগ বিচ্ছিন্ন, পুনরায় সংযোগ হচ্ছে...');
    };

    return eventSource;
  }

  // WebSocket দিয়ে রিয়েল-টাইম নোটিফিকেশন
  connectWebSocket() {
    const ws = new WebSocket(`wss://api.example.com/ws?userId=${this.userId}`);

    ws.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      this._handleNotification(notification);
    };

    ws.onclose = () => {
      // অটো রিকানেক্ট
      setTimeout(() => this.connectWebSocket(), 3000);
    };

    return ws;
  }

  // Polling ফলব্যাক (পুরনো ব্রাউজারের জন্য)
  startPolling(intervalMs = 5000) {
    return setInterval(async () => {
      const res = await fetch(`/api/notifications/unread?userId=${this.userId}`);
      const notifications = await res.json();
      notifications.forEach((n) => this._handleNotification(n));
    }, intervalMs);
  }

  _handleNotification(notification) {
    // ব্রাউজার নোটিফিকেশন দেখাও
    if (Notification.permission === 'granted') {
      new Notification(notification.title, {
        body: notification.body,
        icon: '/icons/notification.png',
      });
    }
    // UI আপডেট করো
    this._emit('notification', notification);
  }

  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(callback);
  }

  _emit(event, data) {
    (this.listeners.get(event) || []).forEach((cb) => cb(data));
  }
}

// সার্ভার সাইড (Node.js + Express) — SSE এন্ডপয়েন্ট
const express = require('express');
const app = express();

app.get('/api/notifications/stream', (req, res) => {
  const userId = req.query.userId;

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    Connection: 'keep-alive',
  });

  // নতুন নোটিফিকেশন এলে পাঠাও
  const sendNotification = (notification) => {
    res.write(`event: notification\n`);
    res.write(`data: ${JSON.stringify(notification)}\n\n`);
  };

  // ইভেন্ট বাসে সাবস্ক্রাইব করো
  const channel = `user:${userId}:notifications`;
  redisSubscriber.subscribe(channel, sendNotification);

  req.on('close', () => {
    redisSubscriber.unsubscribe(channel, sendNotification);
  });
});

// ব্যবহার (ক্লায়েন্ট)
const manager = new NotificationManager(1001);
manager.on('notification', (n) => {
  document.getElementById('badge').textContent = n.unreadCount;
});
manager.connectSSE();
```

---

## ✅ কখন কোন পদ্ধতি ব্যবহার করবেন

| পরিস্থিতি | পদ্ধতি |
|---|---|
| চ্যাট অ্যাপ (দ্বিমুখী) | WebSocket |
| নোটিফিকেশন ফিড (একমুখী) | SSE |
| ইমেইল ডাইজেস্ট | Background Job + Queue |
| সাধারণ ড্যাশবোর্ড | Polling (৩০ সেকেন্ড) |
| মোবাইল পুশ | FCM (Android) / APNs (iOS) |
| উচ্চ ভলিউম (লক্ষ ইউজার) | Kafka + Worker Pool |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|---|---|
| প্রতিটি ইভেন্টে WebSocket | অনেক ইভেন্ট শুধু একমুখী — SSE সহজতর |
| রিয়েল-টাইম না হলে Polling | অপ্রয়োজনীয় সার্ভার লোড |
| সব নোটিফিকেশনে Push | ইউজার বিরক্ত হয়ে অ্যাপ ডিলিট করবে |
| Queue ছাড়া স্কেল করা | ট্রাফিক স্পাইকে সিস্টেম ক্র্যাশ করবে |

---

## 🔑 মনে রাখার পয়েন্ট

1. **WebSocket** — দ্বিমুখী রিয়েল-টাইম (চ্যাট, গেম)
2. **SSE** — একমুখী স্ট্রিম (নোটিফিকেশন ফিড, লাইভ আপডেট)
3. **Message Queue** — স্কেলেবিলিটি ও রিলায়েবিলিটির মূল চাবি
4. **Rate Limiting** — ইউজার প্রতি নোটিফিকেশন সীমা রাখুন
5. ইন্টারভিউতে — Push vs Pull, Fan-out কৌশল, ডেলিভারি গ্যারান্টি আলোচনা করুন
