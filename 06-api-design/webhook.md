# 🔔 Webhook — Webhook Design ও Implementation বিস্তারিত গাইড

## 📖 সংজ্ঞা ও মূল ধারণা

Webhook হলো একটি HTTP callback mechanism যেখানে একটি সিস্টেম (provider) নির্দিষ্ট event ঘটলে অন্য একটি সিস্টেমে (consumer) HTTP POST request পাঠায়। এটি polling-এর বিপরীত — client বারবার জিজ্ঞেস করে না, বরং server নিজে থেকে জানিয়ে দেয়।

### মূল উপাদান:
- **Webhook Provider** — যে সিস্টেম event ঘটলে notification পাঠায় (যেমন Stripe, GitHub)
- **Webhook Consumer** — যে সিস্টেম notification গ্রহণ করে ও প্রক্রিয়া করে
- **Payload** — event-এর বিস্তারিত তথ্য সম্বলিত JSON data
- **Signature** — payload-এর authenticity যাচাইয়ের জন্য HMAC hash
- **Retry Strategy** — delivery ব্যর্থ হলে পুনরায় চেষ্টা করার নীতি

```
Webhook Flow:

  Provider (Stripe)              Consumer (আপনার App)
  ┌──────────────┐              ┌──────────────────┐
  │ Event ঘটলো!  │   POST      │                  │
  │ payment.done │ ──────────→  │ /webhooks/stripe │
  │              │   + HMAC     │                  │
  │              │   Signature  │ ১. Verify Sign.  │
  │              │              │ ২. Process Event │
  │              │  ← 200 OK ── │ ৩. Return 200    │
  └──────────────┘              └──────────────────┘
        │
        │ 200 না পেলে?
        ▼
  ┌──────────────┐
  │ Retry Queue  │
  │ ১ম: 1 min    │
  │ ২য়: 5 min    │
  │ ৩য়: 30 min   │
  │ ৪র্থ: 2 hour  │
  │ ৫ম: 24 hour  │
  └──────────────┘
```

---

## 🎯 বাস্তব উদাহরণ (Real-life Analogy)

**Webhook → কুরিয়ার সার্ভিস:**
আপনি অনলাইনে একটি পণ্য অর্ডার করলেন। আপনি বারবার ফোন করে জিজ্ঞেস করেন না "আমার পণ্য কোথায়?" (এটা হলো polling)। বরং কুরিয়ার কোম্পানি আপনাকে SMS পাঠায় — "আপনার পণ্য shipped হয়েছে", "আপনার পণ্য আপনার শহরে এসেছে", "আজ delivery হবে"। এই SMS-গুলো হলো webhook — event ঘটলে provider নিজে থেকে জানিয়ে দেয়।

**Polling বনাম Webhook:**

```
Polling (অদক্ষ):                    Webhook (দক্ষ):
Client → Server: "কিছু হয়েছে?"     Server → Client: "Event হয়েছে!"
Client → Server: "কিছু হয়েছে?"     (চুপচাপ অপেক্ষা)
Client → Server: "কিছু হয়েছে?"     Server → Client: "আরেকটি event!"
Client → Server: "হ্যাঁ! কী?"       (চুপচাপ অপেক্ষা)
= ৪টি request, ১টি কাজের           = ২টি request, ২টিই কাজের
```

**Signature Verification → চিঠিতে সিলমোহর:**
পুরাতন দিনে রাজা চিঠি পাঠালে সিলমোহর দিতেন। প্রাপক সিলমোহর দেখে বুঝতেন চিঠিটি আসল এবং কেউ পরিবর্তন করেনি। HMAC signature ঠিক এই কাজ করে — payload-টি আসল provider থেকে এসেছে এবং মাঝপথে পরিবর্তন হয়নি, তা নিশ্চিত করে।

---

## 🔐 Signature Verification (HMAC)

```
HMAC Signature তৈরি ও যাচাই:

Provider পক্ষ:
┌─────────────────────────────────────────────────────┐
│ Payload: {"event": "payment.success", "amount": 500}│
│ Secret:  whsec_abc123xyz                            │
│                                                     │
│ Signature = HMAC-SHA256(secret, payload)            │
│           = "sha256=a1b2c3d4e5f6..."                │
│                                                     │
│ Header: X-Webhook-Signature: sha256=a1b2c3d4e5f6...│
└─────────────────────────────────────────────────────┘

Consumer পক্ষ:
┌─────────────────────────────────────────────────────┐
│ ১. Raw body থেকে payload পড়ুন                      │
│ ২. নিজের secret দিয়ে HMAC গণনা করুন                 │
│ ৩. Header-এর signature-এর সাথে তুলনা করুন           │
│ ৪. মিললে → আসল payload ✅                           │
│    না মিললে → বাতিল করুন ❌                         │
│                                                     │
│ ⚠️ timing-safe comparison ব্যবহার করুন!             │
└─────────────────────────────────────────────────────┘
```

---

## ⏰ Retry Strategy — Exponential Backoff

```
Exponential Backoff with Jitter:

চেষ্টা | বিলম্ব        | Jitter সহ          | মোট অপেক্ষা
-------|---------------|--------------------|-----------
1st    | 0 (তাৎক্ষণিক) | —                  | 0s
2nd    | 1 min         | 45s - 1m 15s       | ~1 min
3rd    | 5 min         | 3m 45s - 6m 15s    | ~6 min
4th    | 30 min        | 22m - 37m          | ~36 min
5th    | 2 hour        | 1h 30m - 2h 30m    | ~2.5 hour
6th    | 12 hour       | 9h - 15h           | ~15 hour
7th    | 24 hour       | 18h - 30h          | ~39 hour
---    | GIVE UP       | Alert পাঠান         | —

Jitter কেন? একই সময়ে অনেক retry একসাথে না হওয়ার জন্য (thundering herd সমস্যা এড়াতে)।
```

---

## 🔄 Idempotency

Webhook-এ idempotency অত্যন্ত গুরুত্বপূর্ণ কারণ retry-র কারণে একই event একাধিকবার আসতে পারে। প্রতিটি event-এর একটি unique ID থাকা উচিত এবং consumer-কে duplicate event সনাক্ত ও উপেক্ষা করতে হবে।

```
Idempotency Flow:

Event ID: evt_12345 (প্রথমবার)
  → DB-তে আছে? → না → Process করুন → DB-তে সংরক্ষণ করুন → 200 OK

Event ID: evt_12345 (retry — আবার এলো!)
  → DB-তে আছে? → হ্যাঁ → Skip করুন → 200 OK (আবার process করবেন না!)
```

---

## 💻 PHP Code Example

```php
<?php

// === Webhook Consumer (Receiver) ===
class WebhookReceiver
{
    private string $secret;
    private PDO $db;

    public function __construct(string $secret, PDO $db)
    {
        $this->secret = $secret;
        $this->db = $db;
    }

    public function handle(): void
    {
        // ১. Raw body পড়ুন (parsed body ব্যবহার করবেন না — signature ভুল হবে)
        $payload = file_get_contents('php://input');
        $signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';

        // ২. Signature যাচাই
        if (!$this->verifySignature($payload, $signature)) {
            http_response_code(401);
            echo json_encode(['error' => 'Invalid signature']);
            return;
        }

        $event = json_decode($payload, true);

        // ৩. Idempotency check
        if ($this->isAlreadyProcessed($event['event_id'])) {
            http_response_code(200);
            echo json_encode(['status' => 'already_processed']);
            return;
        }

        // ৪. Event process করুন
        try {
            $this->processEvent($event);
            $this->markAsProcessed($event['event_id']);
            http_response_code(200);
            echo json_encode(['status' => 'success']);
        } catch (\Exception $e) {
            // 500 return করলে provider retry করবে
            http_response_code(500);
            echo json_encode(['error' => 'Processing failed']);
        }
    }

    private function verifySignature(string $payload, string $signature): bool
    {
        $expected = 'sha256=' . hash_hmac('sha256', $payload, $this->secret);
        // Timing-safe comparison — timing attack প্রতিরোধ করে
        return hash_equals($expected, $signature);
    }

    private function isAlreadyProcessed(string $eventId): bool
    {
        $stmt = $this->db->prepare(
            'SELECT COUNT(*) FROM processed_webhooks WHERE event_id = ?'
        );
        $stmt->execute([$eventId]);
        return (int) $stmt->fetchColumn() > 0;
    }

    private function markAsProcessed(string $eventId): void
    {
        $stmt = $this->db->prepare(
            'INSERT INTO processed_webhooks (event_id, processed_at) VALUES (?, NOW())'
        );
        $stmt->execute([$eventId]);
    }

    private function processEvent(array $event): void
    {
        match ($event['type']) {
            'payment.success'  => $this->handlePaymentSuccess($event['data']),
            'payment.failed'   => $this->handlePaymentFailed($event['data']),
            'refund.created'   => $this->handleRefund($event['data']),
            default            => null, // অজানা event উপেক্ষা করুন
        };
    }

    private function handlePaymentSuccess(array $data): void
    {
        // Order status update, email পাঠানো ইত্যাদি
    }

    private function handlePaymentFailed(array $data): void {}
    private function handleRefund(array $data): void {}
}

// === Webhook Provider (Sender) ===
class WebhookDispatcher
{
    private string $secret;
    private PDO $db;

    public function __construct(string $secret, PDO $db)
    {
        $this->secret = $secret;
        $this->db = $db;
    }

    public function dispatch(string $url, string $eventType, array $data): void
    {
        $payload = json_encode([
            'event_id' => $this->generateEventId(),
            'type'     => $eventType,
            'data'     => $data,
            'timestamp' => time(),
        ]);

        $signature = 'sha256=' . hash_hmac('sha256', $payload, $this->secret);

        // Queue-তে রাখুন (synchronous পাঠাবেন না — API response ধীর হবে)
        $this->queueForDelivery($url, $payload, $signature);
    }

    public function deliverWithRetry(int $webhookId): void
    {
        $webhook = $this->getWebhook($webhookId);
        $maxRetries = 7;
        $delays = [0, 60, 300, 1800, 7200, 43200, 86400]; // সেকেন্ডে

        for ($attempt = 0; $attempt < $maxRetries; $attempt++) {
            if ($attempt > 0) {
                $jitter = random_int(-$delays[$attempt] * 25 / 100, $delays[$attempt] * 25 / 100);
                sleep($delays[$attempt] + $jitter);
            }

            $success = $this->sendRequest(
                $webhook['url'],
                $webhook['payload'],
                $webhook['signature']
            );

            $this->logAttempt($webhookId, $attempt + 1, $success);

            if ($success) {
                return;
            }
        }

        // সব retry ব্যর্থ — alert পাঠান
        $this->notifyDeliveryFailure($webhookId);
    }

    private function sendRequest(string $url, string $payload, string $signature): bool
    {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => $payload,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 30,
            CURLOPT_HTTPHEADER     => [
                'Content-Type: application/json',
                "X-Webhook-Signature: {$signature}",
            ],
        ]);

        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return $httpCode >= 200 && $httpCode < 300;
    }

    private function generateEventId(): string
    {
        return 'evt_' . bin2hex(random_bytes(16));
    }

    private function queueForDelivery(string $url, string $payload, string $signature): void {}
    private function getWebhook(int $id): array { return []; }
    private function logAttempt(int $id, int $attempt, bool $success): void {}
    private function notifyDeliveryFailure(int $id): void {}
}
```

---

## 🟨 JavaScript Code Example

```javascript
const crypto = require('crypto');
const express = require('express');

// === Webhook Consumer (Receiver) ===
class WebhookReceiver {
    constructor(secret, db) {
        this.secret = secret;
        this.db = db;
    }

    verifySignature(payload, signature) {
        const expected = 'sha256=' + crypto
            .createHmac('sha256', this.secret)
            .update(payload, 'utf8')
            .digest('hex');

        // Timing-safe comparison
        return crypto.timingSafeEqual(
            Buffer.from(expected),
            Buffer.from(signature)
        );
    }

    async isAlreadyProcessed(eventId) {
        const result = await this.db('processed_webhooks')
            .where('event_id', eventId)
            .first();
        return !!result;
    }

    async processEvent(event) {
        const handlers = {
            'payment.success': (data) => this.handlePaymentSuccess(data),
            'payment.failed': (data) => this.handlePaymentFailed(data),
            'refund.created': (data) => this.handleRefund(data),
        };

        const handler = handlers[event.type];
        if (handler) {
            await handler(event.data);
        }
    }

    async handlePaymentSuccess(data) {
        console.log('Payment সফল:', data.orderId);
    }
    async handlePaymentFailed(data) {}
    async handleRefund(data) {}
}

// Express Middleware — raw body সংরক্ষণ (signature verify-র জন্য)
const app = express();
app.use('/webhooks', express.raw({ type: 'application/json' }));

const receiver = new WebhookReceiver(process.env.WEBHOOK_SECRET, db);

app.post('/webhooks/payment', async (req, res) => {
    const rawBody = req.body.toString('utf8');
    const signature = req.headers['x-webhook-signature'] || '';

    // ১. Signature যাচাই
    if (!receiver.verifySignature(rawBody, signature)) {
        return res.status(401).json({ error: 'Invalid signature' });
    }

    const event = JSON.parse(rawBody);

    // ২. Idempotency check
    if (await receiver.isAlreadyProcessed(event.event_id)) {
        return res.status(200).json({ status: 'already_processed' });
    }

    // ৩. Process (দ্রুত 200 return করুন, ভারী কাজ queue-তে দিন)
    try {
        await receiver.processEvent(event);
        await db('processed_webhooks').insert({
            event_id: event.event_id,
            processed_at: new Date(),
        });
        res.status(200).json({ status: 'success' });
    } catch (err) {
        console.error('Webhook processing failed:', err);
        res.status(500).json({ error: 'Processing failed' });
    }
});

// === Webhook Provider (Sender) ===
class WebhookDispatcher {
    constructor(secret) {
        this.secret = secret;
    }

    generateSignature(payload) {
        return 'sha256=' + crypto
            .createHmac('sha256', this.secret)
            .update(payload, 'utf8')
            .digest('hex');
    }

    async dispatch(url, eventType, data) {
        const payload = JSON.stringify({
            event_id: `evt_${crypto.randomBytes(16).toString('hex')}`,
            type: eventType,
            data,
            timestamp: Math.floor(Date.now() / 1000),
        });

        const signature = this.generateSignature(payload);
        await this.deliverWithRetry(url, payload, signature);
    }

    async deliverWithRetry(url, payload, signature, maxRetries = 7) {
        const delays = [0, 60, 300, 1800, 7200, 43200, 86400];

        for (let attempt = 0; attempt < maxRetries; attempt++) {
            if (attempt > 0) {
                const baseDelay = delays[attempt] * 1000;
                const jitter = Math.random() * baseDelay * 0.5 - baseDelay * 0.25;
                await this.sleep(baseDelay + jitter);
            }

            try {
                const response = await fetch(url, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'X-Webhook-Signature': signature,
                    },
                    body: payload,
                    signal: AbortSignal.timeout(30000),
                });

                if (response.ok) {
                    console.log(`Webhook delivered (attempt ${attempt + 1})`);
                    return;
                }
                console.warn(`Webhook failed: HTTP ${response.status} (attempt ${attempt + 1})`);
            } catch (err) {
                console.error(`Webhook error (attempt ${attempt + 1}):`, err.message);
            }
        }

        console.error('সমস্ত retry ব্যর্থ — manual intervention প্রয়োজন');
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// ব্যবহার
const dispatcher = new WebhookDispatcher(process.env.WEBHOOK_SECRET);
await dispatcher.dispatch(
    'https://example.com/webhooks/payment',
    'payment.success',
    { orderId: 'ORD-123', amount: 5000, currency: 'BDT' }
);
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কারণ |
|-----------|------|
| Payment notification (Stripe, SSLCommerz) | Real-time payment status জানা দরকার |
| CI/CD pipeline trigger (GitHub → Jenkins) | Code push হলে auto build/deploy |
| E-commerce order update | Order status পরিবর্তনে inventory ও notification update |
| Third-party integration | দুটি ভিন্ন সিস্টেমের মধ্যে event-driven communication |
| Chat/messaging notification | নতুন message-এ push notification পাঠানো |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | বিকল্প |
|-----------|--------|
| Real-time bidirectional communication | WebSocket ব্যবহার করুন |
| Consumer-এর public URL নেই (mobile app) | Push notification বা polling ব্যবহার করুন |
| Guaranteed ordering দরকার | Message queue (RabbitMQ, Kafka) ব্যবহার করুন |
| Large file/data transfer | Event-এ download link দিন, payload-এ বড় data দেবেন না |
| Internal microservice communication | Message broker ভালো কাজ করে — webhook-এ HTTP overhead আছে |

---

## 🛡️ Webhook Security Checklist

```
✅ HMAC signature verification (SHA-256 minimum)
✅ Timing-safe comparison (hash_equals / timingSafeEqual)
✅ HTTPS endpoint (HTTP কখনো নয়)
✅ Idempotency — duplicate event handle করুন
✅ Request timeout সেট করুন (30s max)
✅ IP whitelist (যদি provider IP range দেয়)
✅ Raw body থেকে signature verify করুন (parsed body নয়)
✅ Secret rotation mechanism রাখুন
✅ Webhook payload-এ sensitive data minimize করুন
✅ Replay attack prevention — timestamp check করুন (৫ মিনিটের বেশি পুরানো হলে বাতিল)
```

> 💡 **Senior Engineer Tip:** Webhook receive করলে দ্রুত 200 OK return করুন। ভারী processing (email পাঠানো, DB write, external API call) background queue-তে দিন। অনেক provider ৩০ সেকেন্ডের মধ্যে response না পেলে timeout করে retry করে — এতে duplicate processing হতে পারে। আদর্শ pattern: receive → validate → queue → return 200 → process asynchronously।
