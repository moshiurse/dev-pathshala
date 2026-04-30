# 🛡️ ফল্ট টলারেন্স ও রেজিলিয়েন্স প্যাটার্ন (Fault Tolerance & Resilience Patterns)

## 📌 সংজ্ঞা ও পরিচিতি

**ফল্ট টলারেন্স** হলো একটি সিস্টেমের সেই ক্ষমতা যেখানে একটি বা একাধিক কম্পোনেন্ট ব্যর্থ (fail) হলেও সমগ্র সিস্টেম সচল থাকে এবং ব্যবহারকারীদের সেবা দিতে পারে। **রেজিলিয়েন্স** হলো সিস্টেমের সেই বৈশিষ্ট্য যা ব্যর্থতার পরে দ্রুত পুনরুদ্ধার (recover) হতে পারে।

ডিস্ট্রিবিউটেড সিস্টেমে — যেমন বাংলাদেশের bKash, Pathao, বা Daraz — ব্যর্থতা অনিবার্য। প্রশ্ন হলো: "ব্যর্থতা কখন হবে" নয়, বরং "ব্যর্থতা হলে সিস্টেম কীভাবে সামলাবে?"

---

## 📊 সবকিছু ব্যর্থ হয় — Everything Fails Eventually

### কেন ফল্ট টলারেন্স গুরুত্বপূর্ণ?

প্রোডাকশন সিস্টেমে নিম্নলিখিত জিনিসগুলো যেকোনো সময় ব্যর্থ হতে পারে:

```
┌─────────────────────────────────────────────────────────────────┐
│              যেসব জিনিস ব্যর্থ হতে পারে                          │
│                                                                 │
│  ⚡ নেটওয়ার্ক        → প্যাকেট লস, টাইমআউট, DNS ব্যর্থতা       │
│  💾 ডাটাবেস          → কানেকশন পুল পূর্ণ, স্লো কোয়েরি, ক্র্যাশ  │
│  🖥️  সার্ভার          → মেমোরি লিক, CPU 100%, ডিস্ক ফুল          │
│  🔌 থার্ড-পার্টি API  → bKash API ডাউন, SMS গেটওয়ে স্লো          │
│  📦 ক্যাশ সার্ভার     → Redis ক্র্যাশ, মেমোরি OOM                 │
│  🌐 CDN              → এজ সার্ভার ব্যর্থতা                        │
│  ⚙️  কনফিগারেশন      → ভুল ডিপ্লয়, ফিচার ফ্ল্যাগ ত্রুটি         │
└─────────────────────────────────────────────────────────────────┘
```

### 🏠 বাস্তব উদাহরণ: Pathao-তে ক্যাসকেডিং ফেইলিউর

কল্পনা করুন **Pathao**-তে রাইড সার্ভিস ব্যবহার করছেন। পেমেন্ট সার্ভিস ব্যর্থ হলে কীভাবে সবকিছু ভেঙে পড়তে পারে:

```
             ক্যাসকেডিং ফেইলিউর — Pathao উদাহরণ
             ====================================

  1️⃣ পেমেন্ট সার্ভিস ধীর হয়ে গেল (DB কানেকশন লিক)
     │
     ▼
  2️⃣ রাইড সার্ভিস পেমেন্ট কল করে → timeout অপেক্ষা করছে
     │         (থ্রেড ব্লক হয়ে গেল)
     ▼
  3️⃣ রাইড সার্ভিসের থ্রেড পুল পূর্ণ → নতুন রিকোয়েস্ট reject
     │
     ▼
  4️⃣ ইউজার সার্ভিস → রাইড সার্ভিস কল → timeout
     │
     ▼
  5️⃣ নোটিফিকেশন সার্ভিস → ইউজার সার্ভিস কল → timeout
     │
     ▼
  6️⃣ 💀 সম্পূর্ণ সিস্টেম ডাউন — কোনো রাইড বুক হচ্ছে না!

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Payment  │◄───│  Ride    │◄───│  User    │◄───│  Notif   │
  │ Service  │    │ Service  │    │ Service  │    │ Service  │
  │   💀     │    │   💀     │    │   💀     │    │   💀     │
  │ DB Leak  │    │ Blocked  │    │ Timeout  │    │ Timeout  │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
       ↑
    মূল সমস্যা         ──────→ প্রভাব ছড়িয়ে পড়ছে ──────→
```

**ফল্ট টলারেন্স প্যাটার্ন ব্যবহার করলে:** পেমেন্ট সার্ভিস ব্যর্থ হলেও রাইড বুকিং চালু থাকবে — পেমেন্ট পরে প্রসেস হবে।

---

## 📖 সার্কিট ব্রেকার প্যাটার্ন (Circuit Breaker Pattern)

### মূল ধারণা

সার্কিট ব্রেকার বাড়ির ইলেকট্রিক্যাল সার্কিট ব্রেকারের মতোই কাজ করে। যখন কোনো সার্ভিসে বারবার ব্যর্থতা হয়, সার্কিট ব্রেকার "ওপেন" হয়ে যায় এবং সেই সার্ভিসে আর কল পাঠায় না — বরং সাথে সাথে ব্যর্থতা রিটার্ন করে। এতে:

- ব্যর্থ সার্ভিসকে পুনরুদ্ধারের সময় দেওয়া হয়
- ক্যাসকেডিং ফেইলিউর রোধ হয়
- রিসোর্স (থ্রেড, কানেকশন) অপচয় হয় না

### তিনটি স্টেট

```
        সার্কিট ব্রেকার স্টেট ডায়াগ্রাম
        ===================================

  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │    ┌──────────┐   ব্যর্থতা > threshold   ┌────────┐  │
  │    │          │ ─────────────────────→ │        │  │
  │    │ CLOSED   │                       │  OPEN  │  │
  │    │ (স্বাভাবিক)│ ←─────────────────── │(ব্লক)  │  │
  │    │          │   সফল কল (Half-Open   │        │  │
  │    └────┬─────┘    থেকে ফিরে)         └───┬────┘  │
  │         │                                  │      │
  │    সফল কল →                           timeout    │
  │    কাউন্ট রিসেট                        শেষ হলে    │
  │         │                                  │      │
  │         │         ┌────────────┐           │      │
  │         │         │            │           │      │
  │         └─────── │ HALF-OPEN  │ ←─────────┘      │
  │                   │ (পরীক্ষামূলক)│                  │
  │                   │            │                   │
  │                   └─────┬──────┘                   │
  │                         │                          │
  │              পরীক্ষামূলক কল ব্যর্থ?                 │
  │              হ্যাঁ → OPEN-এ ফিরে যাও               │
  │              না → CLOSED-এ যাও                     │
  └─────────────────────────────────────────────────────┘

  CLOSED:    সব কল পাস হয়, ব্যর্থতা গণনা হয়
  OPEN:      সব কল ব্লক, সাথে সাথে ব্যর্থ রিটার্ন
  HALF-OPEN: সীমিত কল পাস হয় পরীক্ষার জন্য
```

### 🏠 bKash উদাহরণ

```
bKash অ্যাপ থেকে ব্যাংক ট্রান্সফার — সার্কিট ব্রেকার সহ:

  ইউজার → bKash App → [Circuit Breaker] → Bank API
                             │
                    ┌────────┴────────┐
                    │                 │
              CLOSED:            OPEN:
              Bank API-তে       "ব্যাংক সেবা সাময়িক
              কল পাঠাও          অনুপলব্ধ, পরে চেষ্টা
                                করুন" — দেখাও
```

### PHP ইমপ্লিমেন্টেশন

```php
<?php

class CircuitBreaker
{
    private string $service;
    private int $failureThreshold;
    private int $recoveryTimeout;    // সেকেন্ডে
    private int $successThreshold;   // Half-Open থেকে Closed যেতে কতটি সফল কল লাগবে

    private string $state = 'CLOSED';
    private int $failureCount = 0;
    private int $successCount = 0;
    private ?int $lastFailureTime = null;

    public function __construct(
        string $service,
        int $failureThreshold = 5,
        int $recoveryTimeout = 30,
        int $successThreshold = 3
    ) {
        $this->service = $service;
        $this->failureThreshold = $failureThreshold;
        $this->recoveryTimeout = $recoveryTimeout;
        $this->successThreshold = $successThreshold;
    }

    public function call(callable $action, callable $fallback = null): mixed
    {
        // OPEN স্টেটে — timeout চেক
        if ($this->state === 'OPEN') {
            if (time() - $this->lastFailureTime >= $this->recoveryTimeout) {
                $this->state = 'HALF_OPEN';
                $this->successCount = 0;
            } else {
                // সাথে সাথে ব্যর্থ রিটার্ন — সার্ভিসে কল পাঠানো হবে না
                return $fallback ? $fallback() : throw new CircuitOpenException(
                    "{$this->service} সার্কিট ওপেন আছে"
                );
            }
        }

        try {
            $result = $action();
            $this->onSuccess();
            return $result;
        } catch (\Throwable $e) {
            $this->onFailure();
            if ($fallback) {
                return $fallback();
            }
            throw $e;
        }
    }

    private function onSuccess(): void
    {
        if ($this->state === 'HALF_OPEN') {
            $this->successCount++;
            if ($this->successCount >= $this->successThreshold) {
                $this->state = 'CLOSED';
                $this->failureCount = 0;
            }
        } else {
            $this->failureCount = 0;
        }
    }

    private function onFailure(): void
    {
        $this->failureCount++;
        $this->lastFailureTime = time();

        if ($this->state === 'HALF_OPEN' || $this->failureCount >= $this->failureThreshold) {
            $this->state = 'OPEN';
        }
    }

    public function getState(): string
    {
        return $this->state;
    }
}

// --- ব্যবহার: bKash → Bank API ---
$bankCircuit = new CircuitBreaker(
    service: 'bank-api',
    failureThreshold: 5,
    recoveryTimeout: 30,
    successThreshold: 3
);

$result = $bankCircuit->call(
    action: fn() => $bankApiClient->transfer($amount, $account),
    fallback: fn() => ['status' => 'queued', 'message' => 'ব্যাংক ট্রান্সফার পরে প্রসেস হবে']
);
```

### JavaScript ইমপ্লিমেন্টেশন (opossum লাইব্রেরি সহ)

```javascript
// npm install opossum
const CircuitBreaker = require('opossum');

// Pathao রাইড সার্ভিস → পেমেন্ট সার্ভিস কল
async function callPaymentService(rideId, amount) {
    const response = await fetch('https://payment.pathao.internal/charge', {
        method: 'POST',
        body: JSON.stringify({ rideId, amount }),
        signal: AbortSignal.timeout(5000)
    });
    if (!response.ok) throw new Error(`Payment failed: ${response.status}`);
    return response.json();
}

const paymentBreaker = new CircuitBreaker(callPaymentService, {
    timeout: 5000,            // ৫ সেকেন্ড timeout
    errorThresholdPercentage: 50,  // ৫০% ব্যর্থ হলে ওপেন
    resetTimeout: 30000,      // ৩০ সেকেন্ড পর Half-Open
    volumeThreshold: 10       // ন্যূনতম ১০টি কল আসার পর থ্রেশহোল্ড গণনা
});

// ফলব্যাক: পেমেন্ট সার্ভিস ডাউন হলে রাইড কিউতে রাখো
paymentBreaker.fallback((rideId, amount) => ({
    status: 'deferred',
    message: 'পেমেন্ট পরে প্রসেস হবে',
    rideId
}));

// ইভেন্ট মনিটরিং
paymentBreaker.on('open', () => console.log('⚠️ পেমেন্ট সার্কিট ওপেন!'));
paymentBreaker.on('halfOpen', () => console.log('🔄 পেমেন্ট সার্কিট পরীক্ষা করা হচ্ছে...'));
paymentBreaker.on('close', () => console.log('✅ পেমেন্ট সার্কিট স্বাভাবিক'));

// কল
const result = await paymentBreaker.fire(rideId, amount);
```

### কাস্টম JavaScript সার্কিট ব্রেকার

```javascript
class CircuitBreaker {
    constructor(options = {}) {
        this.failureThreshold = options.failureThreshold || 5;
        this.recoveryTimeout = options.recoveryTimeout || 30000;
        this.successThreshold = options.successThreshold || 3;

        this.state = 'CLOSED';
        this.failureCount = 0;
        this.successCount = 0;
        this.lastFailureTime = null;
    }

    async execute(action, fallback = null) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime >= this.recoveryTimeout) {
                this.state = 'HALF_OPEN';
                this.successCount = 0;
            } else {
                if (fallback) return fallback();
                throw new Error('Circuit is OPEN');
            }
        }

        try {
            const result = await action();
            this._onSuccess();
            return result;
        } catch (error) {
            this._onFailure();
            if (fallback) return fallback();
            throw error;
        }
    }

    _onSuccess() {
        if (this.state === 'HALF_OPEN') {
            this.successCount++;
            if (this.successCount >= this.successThreshold) {
                this.state = 'CLOSED';
                this.failureCount = 0;
            }
        } else {
            this.failureCount = 0;
        }
    }

    _onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        if (this.state === 'HALF_OPEN' || this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}
```

### Netflix Hystrix ও resilience4j ইতিহাস

Netflix সর্বপ্রথম মাইক্রোসার্ভিসে সার্কিট ব্রেকার প্যাটার্ন জনপ্রিয় করে **Hystrix** লাইব্রেরি দিয়ে। পরে এটি maintenance mode-এ চলে যায় এবং **resilience4j** এর উত্তরসূরি হিসেবে আসে, যা Java-তে লাইটওয়েট ও ফাংশনাল প্রোগ্রামিং বান্ধব।

---

## 🔄 রিট্রাই প্যাটার্ন (Retry Pattern)

### মূল ধারণা

সাময়িক ব্যর্থতা (transient failure) — যেমন নেটওয়ার্ক গ্লিচ বা সাময়িক সার্ভার ওভারলোড — স্বয়ংক্রিয়ভাবে পুনরায় চেষ্টা করে সামলানো যায়। তবে রিট্রাই অবশ্যই সঠিকভাবে করতে হবে, নাহলে সমস্যা আরও বাড়তে পারে।

### রিট্রাই কৌশলসমূহ

```
  ┌─────────────────────────────────────────────────────────────┐
  │              রিট্রাই কৌশল তুলনা                               │
  │                                                             │
  │  1. Simple Retry (সরল পুনরায় চেষ্টা)                        │
  │     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                              │
  │     Attempt 1 → [1s] → Attempt 2 → [1s] → Attempt 3       │
  │     সমস্যা: সবাই একসাথে retry করলে thundering herd          │
  │                                                             │
  │  2. Exponential Backoff (ক্রমবর্ধমান বিলম্ব)                 │
  │     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                            │
  │     Attempt 1 → [1s] → Attempt 2 → [2s] → Attempt 3       │
  │                → [4s] → Attempt 4 → [8s] → Attempt 5       │
  │     সুবিধা: সার্ভারকে রিকভারির সময় দেয়                      │
  │                                                             │
  │  3. Exponential Backoff + Jitter (এলোমেলো বিলম্ব)           │
  │     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                    │
  │     Attempt 1 → [1.3s] → Attempt 2 → [2.7s] → Attempt 3   │
  │                → [3.1s] → Attempt 4 → [9.4s] → Attempt 5   │
  │     সুবিধা: thundering herd সমস্যা এড়ায়                     │
  └─────────────────────────────────────────────────────────────┘
```

### Thundering Herd সমস্যা

```
   jitter ছাড়া — সবাই একসাথে retry করে:
   ─────────────────────────────────────
   Client A:  ──X──[2s]──X──[4s]──✓
   Client B:  ──X──[2s]──X──[4s]──✓     ← সবাই একই সময়ে!
   Client C:  ──X──[2s]──X──[4s]──✓
   Server:         💥          💥💥💥

   jitter সহ — retry ছড়িয়ে পড়ে:
   ─────────────────────────────────
   Client A:  ──X──[1.7s]──X──[3.2s]──✓
   Client B:  ──X──[2.4s]──X──[5.1s]──────✓
   Client C:  ──X──[1.9s]──✓
   Server:         ✓    ✓      ✓   ✓     ← লোড সমানভাবে ছড়িয়ে
```

### PHP রিট্রাই ইমপ্লিমেন্টেশন

```php
<?php

class RetryHandler
{
    public static function execute(
        callable $action,
        int $maxRetries = 3,
        int $baseDelayMs = 1000,
        bool $useJitter = true
    ): mixed {
        $attempt = 0;

        while (true) {
            try {
                return $action();
            } catch (\Throwable $e) {
                $attempt++;

                if ($attempt >= $maxRetries) {
                    throw new \RuntimeException(
                        "সর্বোচ্চ {$maxRetries} বার চেষ্টার পরেও ব্যর্থ: {$e->getMessage()}",
                        previous: $e
                    );
                }

                $delay = $baseDelayMs * (2 ** ($attempt - 1)); // exponential
                if ($useJitter) {
                    $delay = $delay / 2 + random_int(0, $delay / 2); // jitter
                }

                usleep($delay * 1000);
            }
        }
    }
}

// --- ব্যবহার: Daraz → Delivery Partner API ---
$trackingInfo = RetryHandler::execute(
    action: function () use ($orderId) {
        $response = $httpClient->get("https://delivery-api.redx.com.bd/track/{$orderId}");
        if ($response->getStatusCode() !== 200) {
            throw new \RuntimeException("Delivery API ব্যর্থ: " . $response->getStatusCode());
        }
        return json_decode($response->getBody(), true);
    },
    maxRetries: 3,
    baseDelayMs: 1000,
    useJitter: true
);
```

### JavaScript রিট্রাই ইমপ্লিমেন্টেশন

```javascript
async function retryWithBackoff(fn, options = {}) {
    const {
        maxRetries = 3,
        baseDelay = 1000,
        useJitter = true,
        retryableErrors = null    // নির্দিষ্ট error-এ retry করতে চাইলে
    } = options;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            return await fn();
        } catch (error) {
            // শেষ চেষ্টা হলে throw করো
            if (attempt === maxRetries) {
                throw new Error(`${maxRetries} বার চেষ্টার পরেও ব্যর্থ: ${error.message}`);
            }

            // নির্দিষ্ট error ছাড়া retry করবে না
            if (retryableErrors && !retryableErrors.some(e => error instanceof e)) {
                throw error;
            }

            let delay = baseDelay * Math.pow(2, attempt);
            if (useJitter) {
                delay = delay / 2 + Math.random() * (delay / 2);
            }

            console.log(`⚠️ চেষ্টা ${attempt + 1} ব্যর্থ, ${Math.round(delay)}ms পর retry...`);
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}

// --- ব্যবহার: Nagad → SMS গেটওয়ে ---
const smsResult = await retryWithBackoff(
    () => fetch('https://sms-gateway.example.com/send', {
        method: 'POST',
        body: JSON.stringify({ phone: '+8801XXXXXXXXX', message: 'OTP: 123456' })
    }).then(res => {
        if (!res.ok) throw new Error(`SMS API error: ${res.status}`);
        return res.json();
    }),
    { maxRetries: 3, baseDelay: 500, useJitter: true }
);
```

### ⚠️ Idempotency — রিট্রাই করার পূর্বশর্ত

রিট্রাই তখনই নিরাপদ যখন অপারেশনটি **idempotent** — অর্থাৎ একই অপারেশন একাধিকবার চালালে ফলাফল একই থাকে।

```
  ❌ Non-Idempotent (রিট্রাই বিপজ্জনক):
     POST /transfer { amount: 500 }
     → ১ম কল সফল কিন্তু response হারিয়ে গেল
     → Retry: আবার ৫০০ টাকা কেটে গেল! 💸

  ✅ Idempotent (রিট্রাই নিরাপদ):
     POST /transfer { amount: 500, idempotencyKey: "txn-abc-123" }
     → ১ম কল সফল কিন্তু response হারিয়ে গেল
     → Retry: সার্ভার দেখলো এই key আগে এসেছে, আগের result রিটার্ন ✓
```

---

## ⏱️ টাইমআউট প্যাটার্ন (Timeout Pattern)

### Connection Timeout বনাম Read Timeout

```
  ┌────────────────────────────────────────────────────────────┐
  │                  Timeout প্রকারভেদ                          │
  │                                                            │
  │  Connection Timeout (কানেকশন টাইমআউট):                     │
  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                     │
  │  সার্ভারের সাথে TCP কানেকশন স্থাপনের সর্বোচ্চ সময়          │
  │  সাধারণত: 2-5 সেকেন্ড                                      │
  │                                                            │
  │  Client ──[TCP SYN]──→ Server                              │
  │          ⏱️ এই সময়ে কানেকশন না হলে timeout                  │
  │                                                            │
  │  Read Timeout (রিড টাইমআউট):                                │
  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                               │
  │  কানেকশনের পর response আসার সর্বোচ্চ অপেক্ষার সময়          │
  │  সাধারণত: 5-30 সেকেন্ড (API অনুযায়ী)                       │
  │                                                            │
  │  Client ──[Request]──→ Server                              │
  │          ⏱️ Response-এর জন্য অপেক্ষা                        │
  └────────────────────────────────────────────────────────────┘
```

### ক্যাসকেডিং টাইমআউট বাজেট

মাইক্রোসার্ভিসে প্রতিটি সার্ভিসের timeout এমনভাবে সেট করতে হবে যেন মোট সময় ব্যবহারকারীর ধৈর্যের সীমার মধ্যে থাকে:

```
  ক্যাসকেডিং টাইমআউট বাজেট — Daraz অর্ডার প্লেস
  =================================================

  ইউজার (সর্বোচ্চ ১০ সেকেন্ড অপেক্ষা করবে)
    │
    ▼ timeout: 10s
  API Gateway
    │
    ├──▶ Order Service (timeout: 8s)
    │       │
    │       ├──▶ Inventory Service (timeout: 3s)
    │       │
    │       ├──▶ Payment Service (timeout: 4s)
    │       │       │
    │       │       └──▶ bKash API (timeout: 3s)
    │       │
    │       └──▶ Notification (timeout: 2s, async — অপেক্ষা করে না)
    │
    └──▶ Response ← মোট সময় ≤ 10s হতে হবে

  ⚠️ নিয়ম: Inner timeout < Outer timeout
     bKash (3s) < Payment (4s) < Order (8s) < Gateway (10s)
```

### PHP Timeout (Guzzle)

```php
<?php

use GuzzleHttp\Client;
use GuzzleHttp\Exception\ConnectException;
use GuzzleHttp\Exception\RequestException;

$client = new Client([
    'base_uri' => 'https://api.bkash.com/',
    'connect_timeout' => 3,    // কানেকশন: সর্বোচ্চ ৩ সেকেন্ড
    'timeout' => 10,           // মোট: সর্বোচ্চ ১০ সেকেন্ড
    'read_timeout' => 7,       // রিড: সর্বোচ্চ ৭ সেকেন্ড
]);

try {
    $response = $client->post('checkout/payment/create', [
        'json' => ['amount' => 500, 'currency' => 'BDT'],
        'timeout' => 5   // এই নির্দিষ্ট request-এর জন্য override
    ]);
} catch (ConnectException $e) {
    // কানেকশন ব্যর্থ — সার্ভার unreachable
    Log::error('bKash কানেকশন ব্যর্থ', ['error' => $e->getMessage()]);
    return fallbackPayment();
} catch (RequestException $e) {
    // Read timeout বা অন্য ত্রুটি
    Log::error('bKash রিকোয়েস্ট ব্যর্থ', ['error' => $e->getMessage()]);
    return fallbackPayment();
}
```

### JavaScript Timeout (AbortController ও axios)

```javascript
// ১. AbortController দিয়ে fetch timeout
async function fetchWithTimeout(url, options = {}, timeoutMs = 5000) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

    try {
        const response = await fetch(url, {
            ...options,
            signal: controller.signal
        });
        return response;
    } catch (error) {
        if (error.name === 'AbortError') {
            throw new Error(`⏱️ Request timeout: ${timeoutMs}ms অতিক্রম করেছে`);
        }
        throw error;
    } finally {
        clearTimeout(timeoutId);
    }
}

// Prothom Alo API থেকে ব্রেকিং নিউজ — ৩ সেকেন্ড timeout
const news = await fetchWithTimeout(
    'https://api.prothomalo.com/breaking-news',
    { headers: { 'Accept': 'application/json' } },
    3000
);

// ২. axios timeout
const axios = require('axios');
const gpClient = axios.create({
    baseURL: 'https://api.grameenphone.com/',
    timeout: 5000,                    // ৫ সেকেন্ড
    timeoutErrorMessage: 'GP API timeout হয়ে গেছে'
});
```

---

## 🚢 বাল্কহেড প্যাটার্ন (Bulkhead Pattern)

### মূল ধারণা

জাহাজে (ship) আলাদা আলাদা **ওয়াটারটাইট কম্পার্টমেন্ট** (bulkhead) থাকে। একটি কম্পার্টমেন্টে পানি ঢুকলেও বাকি কম্পার্টমেন্ট নিরাপদ থাকে — পুরো জাহাজ ডোবে না।

সফটওয়্যারেও একই নীতি: একটি সার্ভিস বা ফিচারের সমস্যা যেন পুরো সিস্টেমকে প্রভাবিত না করে।

```
        জাহাজের Bulkhead বনাম সফটওয়্যার Bulkhead
        ==========================================

  জাহাজ:
  ┌──────┬──────┬──────┬──────┬──────┐
  │ 🚢   │ 🚢   │ 💧💧 │ 🚢   │ 🚢   │
  │ নিরাপদ│ নিরাপদ│ ফুটো! │ নিরাপদ│ নিরাপদ│
  └──────┴──────┴──────┴──────┴──────┘
      ↑ দেয়ালগুলো পানি ছড়াতে বাধা দেয়

  সফটওয়্যার (Pathao):
  ┌──────────┬──────────┬──────────┬──────────┐
  │ Ride     │ Food     │ Payment  │ Parcel   │
  │ Pool:    │ Pool:    │ Pool:    │ Pool:    │
  │ 50 conn  │ 30 conn  │ 20 conn  │ 10 conn  │
  │ ✅ সচল   │ ✅ সচল   │ 💀 ধীর    │ ✅ সচল   │
  └──────────┴──────────┴──────────┴──────────┘
      ↑ আলাদা পুল: Payment ধীর হলেও Ride ঠিক আছে
```

### Bulkhead-এর প্রকারভেদ

```
  ১. Thread Pool Isolation (থ্রেড পুল আইসোলেশন):
     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
     প্রতিটি সার্ভিস কলের জন্য আলাদা থ্রেড পুল
     একটি পুল পূর্ণ হলেও অন্যগুলো কাজ করবে

  ২. Connection Pool Isolation (কানেকশন পুল আইসোলেশন):
     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
     প্রতিটি downstream সার্ভিসের জন্য আলাদা DB/HTTP কানেকশন পুল

  ৩. Semaphore Isolation (সেমাফোর আইসোলেশন):
     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
     concurrent কলের সংখ্যা সীমিত করা (লাইটওয়েট)
```

### PHP Bulkhead ইমপ্লিমেন্টেশন (Semaphore)

```php
<?php

class Bulkhead
{
    private int $maxConcurrent;
    private int $currentCount = 0;
    private int $maxWaitTimeMs;

    public function __construct(int $maxConcurrent = 10, int $maxWaitTimeMs = 5000)
    {
        $this->maxConcurrent = $maxConcurrent;
        $this->maxWaitTimeMs = $maxWaitTimeMs;
    }

    public function execute(callable $action): mixed
    {
        if ($this->currentCount >= $this->maxConcurrent) {
            throw new BulkheadRejectedException(
                "Bulkhead পূর্ণ: সর্বোচ্চ {$this->maxConcurrent} concurrent কল অনুমোদিত"
            );
        }

        $this->currentCount++;
        try {
            return $action();
        } finally {
            $this->currentCount--;
        }
    }
}

// Daraz — আলাদা bulkhead: ইনভেন্টরি ও পেমেন্ট
$inventoryBulkhead = new Bulkhead(maxConcurrent: 20);
$paymentBulkhead = new Bulkhead(maxConcurrent: 10);

// পেমেন্ট ধীর হলেও ইনভেন্টরি চেক চলবে
$stock = $inventoryBulkhead->execute(fn() => $inventoryService->check($productId));
$payment = $paymentBulkhead->execute(fn() => $paymentService->charge($amount));
```

### JavaScript Bulkhead

```javascript
class Bulkhead {
    constructor(maxConcurrent = 10) {
        this.maxConcurrent = maxConcurrent;
        this.running = 0;
        this.queue = [];
    }

    async execute(fn) {
        if (this.running >= this.maxConcurrent) {
            // কিউতে রেখে অপেক্ষা
            await new Promise((resolve, reject) => {
                this.queue.push({ resolve, reject });
            });
        }

        this.running++;
        try {
            return await fn();
        } finally {
            this.running--;
            if (this.queue.length > 0) {
                const { resolve } = this.queue.shift();
                resolve();
            }
        }
    }
}

// Pathao Food — সেলার API ও রাইডার API আলাদা bulkhead
const sellerBulkhead = new Bulkhead(15);
const riderBulkhead = new Bulkhead(10);

const menu = await sellerBulkhead.execute(() => fetchSellerMenu(sellerId));
const rider = await riderBulkhead.execute(() => findNearestRider(location));
```

---

## 🔀 ফলব্যাক প্যাটার্ন (Fallback Pattern)

### মূল ধারণা

প্রাথমিক কার্যপদ্ধতি ব্যর্থ হলে একটি বিকল্প (fallback) কার্যপদ্ধতি ব্যবহার করা। ব্যবহারকারী যেন সবসময় কিছু না কিছু পায় — এমনকি সেটি সীমিত হলেও।

### ফলব্যাক কৌশল

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   ফলব্যাক কৌশলসমূহ                            │
  │                                                              │
  │  ১. Default Values (ডিফল্ট মান):                              │
  │     → প্রোডাক্ট রেটিং API ডাউন? ডিফল্ট ৪.০ দেখাও            │
  │                                                              │
  │  ২. Cached Response (ক্যাশ থেকে):                             │
  │     → Prothom Alo API ডাউন? Redis-এ শেষ সংবাদ দেখাও          │
  │                                                              │
  │  ৩. Degraded Functionality (সীমিত সেবা):                      │
  │     → bKash payment ডাউন? শুধু COD অপশন দেখাও                │
  │                                                              │
  │  ৪. Static Fallback Page (স্ট্যাটিক পেজ):                     │
  │     → পুরো API ডাউন? CDN থেকে স্ট্যাটিক HTML দেখাও            │
  │                                                              │
  │  ৫. Queue for Later (পরে প্রক্রিয়া):                          │
  │     → SMS gateway ডাউন? মেসেজ কিউতে রাখো, পরে পাঠাও          │
  └──────────────────────────────────────────────────────────────┘
```

### PHP ফলব্যাক উদাহরণ

```php
<?php

class ProductService
{
    public function __construct(
        private HttpClient $apiClient,
        private CacheInterface $cache,
        private LoggerInterface $logger
    ) {}

    // Daraz প্রোডাক্ট ডিটেইল — মাল্টি-লেভেল ফলব্যাক
    public function getProductDetails(string $productId): array
    {
        // চেষ্টা ১: লাইভ API
        try {
            $product = $this->apiClient->get("/products/{$productId}");
            $this->cache->set("product:{$productId}", $product, ttl: 300);
            return $product;
        } catch (\Throwable $e) {
            $this->logger->warning("API ব্যর্থ, ক্যাশ চেষ্টা করা হচ্ছে", [
                'productId' => $productId
            ]);
        }

        // চেষ্টা ২: ক্যাশ থেকে
        $cached = $this->cache->get("product:{$productId}");
        if ($cached) {
            $cached['_source'] = 'cache';
            $cached['_stale'] = true;
            return $cached;
        }

        // চেষ্টা ৩: ডাটাবেস থেকে মৌলিক তথ্য
        try {
            $basic = DB::table('products')
                ->select(['id', 'name', 'price', 'image_url'])
                ->find($productId);
            if ($basic) {
                return array_merge((array) $basic, ['_source' => 'database', '_limited' => true]);
            }
        } catch (\Throwable $e) {
            $this->logger->error("ডাটাবেসও ব্যর্থ!");
        }

        // চেষ্টা ৪: ডিফল্ট
        return [
            'id' => $productId,
            'name' => 'প্রোডাক্ট তথ্য সাময়িক অনুপলব্ধ',
            'price' => null,
            '_source' => 'fallback'
        ];
    }
}
```

### JavaScript ফলব্যাক উদাহরণ

```javascript
// Prothom Alo — ব্রেকিং নিউজ ফলব্যাক চেইন
async function getBreakingNews() {
    // চেষ্টা ১: লাইভ API
    try {
        const response = await fetchWithTimeout(
            'https://api.prothomalo.com/v1/breaking',
            {}, 3000
        );
        const news = await response.json();
        await redis.set('breaking-news', JSON.stringify(news), 'EX', 60);
        return { data: news, source: 'live' };
    } catch (e) {
        console.warn('Live API ব্যর্থ:', e.message);
    }

    // চেষ্টা ২: Redis ক্যাশ
    try {
        const cached = await redis.get('breaking-news');
        if (cached) {
            return { data: JSON.parse(cached), source: 'cache', stale: true };
        }
    } catch (e) {
        console.warn('Redis ব্যর্থ:', e.message);
    }

    // চেষ্টা ৩: স্ট্যাটিক ফলব্যাক
    return {
        data: [{ title: 'সর্বশেষ সংবাদের জন্য পেজ রিফ্রেশ করুন', url: '/' }],
        source: 'static-fallback'
    };
}
```

---

## 🚦 রেট লিমিটার — ক্লায়েন্ট সাইড (Client-Side Rate Limiting)

### মূল ধারণা

সার্ভার সাইড রেট লিমিটিং সার্ভারকে রক্ষা করে, কিন্তু **ক্লায়েন্ট সাইড রেট লিমিটিং** আপনার অ্যাপকে 429 (Too Many Requests) ত্রুটি থেকে বাঁচায় এবং থার্ড-পার্টি API-র সাথে সুসম্পর্ক বজায় রাখে।

### Token Bucket ক্লায়েন্ট সাইড

```
  Token Bucket (ক্লায়েন্ট সাইড):
  ================================

  ┌─────────────────────┐
  │   Token Bucket      │     প্রতি সেকেন্ডে ১০ টোকেন রিফিল হয়
  │   ○ ○ ○ ○ ○ ●       │     ← ৫টি টোকেন আছে
  │   Max: 10           │     
  └─────────┬───────────┘     
            │                 
   API কল করতে চাও?          
            │                 
     ┌──────┴──────┐          
     │ টোকেন আছে? │          
     └──────┬──────┘          
      হ্যাঁ │       │ না       
           ▼       ▼          
      ✅ কল করো  ⏳ অপেক্ষা   
      টোকেন -১   টোকেন রিফিল  
                  হওয়া পর্যন্ত 
```

### 429 Status হ্যান্ডলিং

```javascript
async function callWithRateLimit(url, options = {}) {
    const response = await fetch(url, options);

    if (response.status === 429) {
        // Retry-After হেডার চেক
        const retryAfter = response.headers.get('Retry-After');
        const waitMs = retryAfter
            ? parseInt(retryAfter) * 1000
            : 60000; // ডিফল্ট ১ মিনিট

        console.log(`⚠️ রেট লিমিট! ${waitMs / 1000}s পর retry করা হবে...`);
        await new Promise(r => setTimeout(r, waitMs));
        return callWithRateLimit(url, options); // পুনরায় চেষ্টা
    }

    return response;
}

// Grameenphone API — SMS পাঠানোর রেট লিমিট
class RateLimitedClient {
    constructor(maxPerSecond = 10) {
        this.tokens = maxPerSecond;
        this.maxTokens = maxPerSecond;
        this.lastRefill = Date.now();
    }

    async call(fn) {
        this._refillTokens();

        if (this.tokens <= 0) {
            const waitTime = 1000 / this.maxTokens;
            await new Promise(r => setTimeout(r, waitTime));
            this._refillTokens();
        }

        this.tokens--;
        return fn();
    }

    _refillTokens() {
        const now = Date.now();
        const elapsed = (now - this.lastRefill) / 1000;
        this.tokens = Math.min(this.maxTokens, this.tokens + elapsed * this.maxTokens);
        this.lastRefill = now;
    }
}
```

---

## 💚 হেলথ চেক ও রেডিনেস (Health Checks & Readiness)

### Liveness বনাম Readiness প্রোব

```
  ┌────────────────────────────────────────────────────────────┐
  │          Liveness বনাম Readiness প্রোব                      │
  │                                                            │
  │  Liveness (জীবিত আছে?):                                    │
  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                                   │
  │  প্রশ্ন: "প্রসেস চলছে? ডেডলক নেই?"                         │
  │  ব্যর্থ হলে: কন্টেইনার রিস্টার্ট করো                        │
  │  এন্ডপয়েন্ট: GET /health/live                              │
  │                                                            │
  │  Readiness (কাজ করতে প্রস্তুত?):                            │
  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄                               │
  │  প্রশ্ন: "DB কানেক্ট আছে? ক্যাশ রেডি? ডিপেন্ডেন্সি ঠিক?"  │
  │  ব্যর্থ হলে: ট্রাফিক পাঠানো বন্ধ করো (কিন্তু kill করো না)  │
  │  এন্ডপয়েন্ট: GET /health/ready                             │
  └────────────────────────────────────────────────────────────┘

  সময়রেখা:
  ──────────────────────────────────────────────────────
  App Start → [DB কানেক্ট হচ্ছে] → [Cache ওয়ার্ম হচ্ছে] → Ready!
              │                                           │
              Liveness: ✅                                 Readiness: ✅
              Readiness: ❌ (এখনো প্রস্তুত না)
```

### PHP Health Check ইমপ্লিমেন্টেশন

```php
<?php

class HealthCheckController
{
    // GET /health/live — Kubernetes liveness probe
    public function liveness(): JsonResponse
    {
        return response()->json([
            'status' => 'UP',
            'timestamp' => now()->toISOString()
        ]);
    }

    // GET /health/ready — Kubernetes readiness probe
    public function readiness(): JsonResponse
    {
        $checks = [];
        $allHealthy = true;

        // ডাটাবেস চেক
        try {
            DB::connection()->getPdo();
            $checks['database'] = ['status' => 'UP'];
        } catch (\Throwable $e) {
            $checks['database'] = ['status' => 'DOWN', 'error' => $e->getMessage()];
            $allHealthy = false;
        }

        // Redis চেক
        try {
            Redis::ping();
            $checks['redis'] = ['status' => 'UP'];
        } catch (\Throwable $e) {
            $checks['redis'] = ['status' => 'DOWN', 'error' => $e->getMessage()];
            $allHealthy = false;
        }

        // bKash API চেক (Daraz-এর জন্য গুরুত্বপূর্ণ)
        try {
            $response = Http::timeout(2)->get('https://api.bkash.com/health');
            $checks['bkash_api'] = ['status' => $response->ok() ? 'UP' : 'DEGRADED'];
        } catch (\Throwable $e) {
            $checks['bkash_api'] = ['status' => 'DOWN'];
            // bKash ডাউন হলেও সিস্টেম ready — COD চালু আছে
        }

        $status = $allHealthy ? 'UP' : 'DOWN';
        $httpCode = $allHealthy ? 200 : 503;

        return response()->json([
            'status' => $status,
            'checks' => $checks,
            'timestamp' => now()->toISOString()
        ], $httpCode);
    }
}
```

### JavaScript Health Check ইমপ্লিমেন্টেশন

```javascript
const express = require('express');
const app = express();

// Liveness
app.get('/health/live', (req, res) => {
    res.json({ status: 'UP', uptime: process.uptime() });
});

// Readiness — সব dependency চেক
app.get('/health/ready', async (req, res) => {
    const checks = {};
    let allHealthy = true;

    // MongoDB চেক
    try {
        await mongoose.connection.db.admin().ping();
        checks.mongodb = { status: 'UP' };
    } catch (e) {
        checks.mongodb = { status: 'DOWN', error: e.message };
        allHealthy = false;
    }

    // Redis চেক
    try {
        await redisClient.ping();
        checks.redis = { status: 'UP' };
    } catch (e) {
        checks.redis = { status: 'DOWN', error: e.message };
        allHealthy = false;
    }

    // Kafka চেক (Pathao event system)
    try {
        const admin = kafka.admin();
        await admin.connect();
        await admin.disconnect();
        checks.kafka = { status: 'UP' };
    } catch (e) {
        checks.kafka = { status: 'DOWN', error: e.message };
        allHealthy = false;
    }

    const statusCode = allHealthy ? 200 : 503;
    res.status(statusCode).json({
        status: allHealthy ? 'UP' : 'DOWN',
        checks,
        timestamp: new Date().toISOString()
    });
});
```

---

## 🐒 ক্যাওস ইঞ্জিনিয়ারিং (Chaos Engineering) — মৌলিক ধারণা

### Netflix Chaos Monkey

Netflix সর্বপ্রথম **Chaos Monkey** তৈরি করে — একটি টুল যা প্রোডাকশনে এলোমেলোভাবে সার্ভার বন্ধ করে দেয়। উদ্দেশ্য: সিস্টেম যদি এলোমেলো ব্যর্থতা সহ্য করতে পারে, তাহলে বাস্তব ব্যর্থতাতেও টিকে থাকবে।

```
  Chaos Engineering পিরামিড:
  ===========================

  ┌───────────────────────────────┐
  │      Chaos Monkey             │ ← সার্ভার বন্ধ করে দেওয়া
  ├───────────────────────────────┤
  │    Latency Injection          │ ← কৃত্রিম ল্যাটেন্সি যোগ
  ├───────────────────────────────┤
  │   Error Injection             │ ← কৃত্রিম ত্রুটি তৈরি
  ├───────────────────────────────┤
  │  Network Partition            │ ← নেটওয়ার্ক বিচ্ছিন্নতা সিমুলেশন
  ├───────────────────────────────┤
  │ Resource Exhaustion           │ ← CPU/মেমোরি/ডিস্ক পূর্ণ করা
  └───────────────────────────────┘
```

### Fault Injection ধারণা

```
  ┌──────────────────────────────────────────────────────────┐
  │              Fault Injection কৌশল                         │
  │                                                          │
  │  ১. Latency Injection:                                   │
  │     API response-এ কৃত্রিম বিলম্ব যোগ                    │
  │     "Payment API-তে ৫ সেকেন্ড delay দিলে কী হয়?"         │
  │                                                          │
  │  ২. Error Injection:                                     │
  │     নির্দিষ্ট % request-এ HTTP 500 রিটার্ন               │
  │     "১০% request ব্যর্থ হলে UI কীভাবে react করে?"        │
  │                                                          │
  │  ৩. Resource Exhaustion:                                 │
  │     মেমোরি বা CPU কৃত্রিমভাবে ব্যস্ত রাখা                │
  │     "Redis OOM হলে কি অ্যাপ ক্র্যাশ করে?"                │
  │                                                          │
  │  ৪. Dependency Kill:                                     │
  │     একটি downstream সার্ভিস সম্পূর্ণ বন্ধ                │
  │     "bKash API সম্পূর্ণ unreachable হলে Daraz কী করে?"   │
  └──────────────────────────────────────────────────────────┘
```

### Game Day (গেম ডে)

**Game Day** হলো একটি পরিকল্পিত অনুশীলন যেখানে দল ইচ্ছাকৃতভাবে সিস্টেমে ব্যর্থতা সৃষ্টি করে এবং দেখে সিস্টেম কীভাবে সামলায়। এটি বাংলাদেশের ফায়ার ড্রিলের মতো — আগুন লাগার আগেই প্রস্তুতি নেওয়া।

### টুলস ওভারভিউ

```
  ┌──────────────────────┬──────────────────────────────────┐
  │ টুল                   │ কাজ                               │
  ├──────────────────────┼──────────────────────────────────┤
  │ Chaos Monkey         │ এলোমেলো instance বন্ধ             │
  │ Gremlin              │ ব্যাপক fault injection platform   │
  │ Litmus               │ Kubernetes chaos testing          │
  │ Toxiproxy            │ নেটওয়ার্ক ত্রুটি সিমুলেশন         │
  │ tc (Linux)           │ নেটওয়ার্ক latency/loss সিমুলেশন   │
  │ stress-ng            │ CPU/মেমোরি stress testing         │
  └──────────────────────┴──────────────────────────────────┘
```

---

## 🎯 সব প্যাটার্ন একসাথে (Combining Patterns)

### একটি সম্পূর্ণ Resilient সার্ভিস কল

বাস্তবে এই প্যাটার্নগুলো একা একা ব্যবহার হয় না — একসাথে মিলিয়ে ব্যবহার করা হয়:

```
  সম্পূর্ণ Resilient কল ফ্লো — Daraz থেকে bKash পেমেন্ট:
  ==========================================================

  ইউজার Request
       │
       ▼
  ┌──────────┐
  │ Bulkhead │ ← concurrent কল সীমিত (সর্বোচ্চ ২০)
  └────┬─────┘
       │ পাস হলে
       ▼
  ┌──────────────┐
  │ Rate Limiter │ ← bKash API রেট লিমিটের মধ্যে থাকো
  └────┬─────────┘
       │
       ▼
  ┌─────────────────┐
  │ Circuit Breaker │ ← ওপেন হলে সরাসরি fallback-এ
  └────┬────────────┘
       │ CLOSED/HALF_OPEN
       ▼
  ┌──────────┐
  │ Timeout  │ ← সর্বোচ্চ ৫ সেকেন্ড
  └────┬─────┘
       │
       ▼
  ┌────────┐
  │ Retry  │ ← সর্বোচ্চ ৩ বার, exponential backoff + jitter
  └────┬───┘
       │
       ▼
  ┌──────────┐    ┌──────────┐
  │ bKash    │──→ │ Fallback │ ← ব্যর্থ হলে: কিউতে রাখো
  │ API Call │    │          │    বা COD অপশন দেখাও
  └──────────┘    └──────────┘
```

### PHP — সম্পূর্ণ Resilient সার্ভিস

```php
<?php

class ResilientPaymentService
{
    private CircuitBreaker $circuitBreaker;
    private Bulkhead $bulkhead;
    private CacheInterface $cache;

    public function __construct()
    {
        $this->circuitBreaker = new CircuitBreaker(
            service: 'bkash-payment',
            failureThreshold: 5,
            recoveryTimeout: 30,
            successThreshold: 3
        );
        $this->bulkhead = new Bulkhead(maxConcurrent: 20);
    }

    public function processPayment(string $orderId, float $amount): array
    {
        // ধাপ ১: Bulkhead — concurrent কল সীমিত
        return $this->bulkhead->execute(function () use ($orderId, $amount) {

            // ধাপ ২: Circuit Breaker + Fallback
            return $this->circuitBreaker->call(
                action: function () use ($orderId, $amount) {
                    // ধাপ ৩: Retry with backoff
                    return RetryHandler::execute(
                        action: function () use ($orderId, $amount) {
                            // ধাপ ৪: Timeout সহ API কল
                            $client = new \GuzzleHttp\Client([
                                'timeout' => 5,
                                'connect_timeout' => 2
                            ]);

                            $response = $client->post(
                                'https://api.bkash.com/checkout/payment/create',
                                [
                                    'json' => [
                                        'amount' => $amount,
                                        'currency' => 'BDT',
                                        'merchantInvoiceNumber' => $orderId,
                                        'idempotencyKey' => "pay-{$orderId}"
                                    ]
                                ]
                            );

                            return json_decode($response->getBody(), true);
                        },
                        maxRetries: 2,
                        baseDelayMs: 500,
                        useJitter: true
                    );
                },
                fallback: function () use ($orderId, $amount) {
                    // Fallback: পেমেন্ট কিউতে রাখো
                    Queue::push(new ProcessPaymentJob($orderId, $amount));
                    return [
                        'status' => 'deferred',
                        'message' => 'পেমেন্ট প্রক্রিয়াধীন, শীঘ্রই নিশ্চিত করা হবে',
                        'orderId' => $orderId
                    ];
                }
            );
        });
    }
}
```

### JavaScript — সম্পূর্ণ Resilient সার্ভিস

```javascript
class ResilientServiceCaller {
    constructor(options = {}) {
        this.circuitBreaker = new CircuitBreaker({
            failureThreshold: options.failureThreshold || 5,
            recoveryTimeout: options.recoveryTimeout || 30000,
            successThreshold: options.successThreshold || 3
        });
        this.bulkhead = new Bulkhead(options.maxConcurrent || 20);
        this.maxRetries = options.maxRetries || 3;
        this.timeoutMs = options.timeoutMs || 5000;
    }

    async call(fn, fallbackFn = null) {
        // ধাপ ১: Bulkhead
        return this.bulkhead.execute(async () => {
            // ধাপ ২: Circuit Breaker
            return this.circuitBreaker.execute(
                async () => {
                    // ধাপ ৩: Retry
                    return retryWithBackoff(
                        async () => {
                            // ধাপ ৪: Timeout
                            return Promise.race([
                                fn(),
                                new Promise((_, reject) =>
                                    setTimeout(
                                        () => reject(new Error('Timeout')),
                                        this.timeoutMs
                                    )
                                )
                            ]);
                        },
                        { maxRetries: this.maxRetries, baseDelay: 500, useJitter: true }
                    );
                },
                fallbackFn
            );
        });
    }
}

// --- ব্যবহার: Nagad পেমেন্ট সার্ভিস ---
const nagadCaller = new ResilientServiceCaller({
    failureThreshold: 5,
    recoveryTimeout: 30000,
    maxConcurrent: 15,
    maxRetries: 2,
    timeoutMs: 4000
});

const paymentResult = await nagadCaller.call(
    // মূল কল
    async () => {
        const res = await fetch('https://api.nagad.com.bd/payment/create', {
            method: 'POST',
            body: JSON.stringify({ amount: 500, orderId: 'ORD-123' })
        });
        if (!res.ok) throw new Error(`Nagad API: ${res.status}`);
        return res.json();
    },
    // ফলব্যাক
    async () => {
        await messageQueue.publish('deferred-payments', { orderId: 'ORD-123', amount: 500 });
        return { status: 'queued', message: 'পেমেন্ট শীঘ্রই প্রসেস হবে' };
    }
);
```

### ডিসিশন ম্যাট্রিক্স — কোন প্যাটার্ন কখন

```
  ┌──────────────────┬────────────┬─────────┬──────────┬──────────┐
  │ পরিস্থিতি          │ Circuit    │ Retry   │ Timeout  │ Bulkhead │
  │                    │ Breaker    │         │          │          │
  ├──────────────────┼────────────┼─────────┼──────────┼──────────┤
  │ API বারবার ব্যর্থ   │ ✅ অবশ্যই   │ ⚠️ সীমিত │ ✅        │ ✅        │
  │ নেটওয়ার্ক গ্লিচ    │ ❌          │ ✅ আদর্শ  │ ✅        │ ❌        │
  │ ধীর dependency   │ ✅          │ ❌       │ ✅ অবশ্যই │ ✅ অবশ্যই │
  │ থার্ড-পার্টি API   │ ✅          │ ✅       │ ✅        │ ✅        │
  │ ইন্টার্নাল সার্ভিস  │ ✅          │ ✅       │ ✅        │ ⚠️       │
  │ ব্যাচ প্রসেসিং     │ ❌          │ ✅       │ ⚠️       │ ❌        │
  └──────────────────┴────────────┴─────────┴──────────┴──────────┘

  ✅ = অবশ্যই ব্যবহার করুন
  ⚠️ = পরিস্থিতি বুঝে ব্যবহার করুন
  ❌ = সাধারণত প্রয়োজন নেই
```

---

## 📌 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন

```
  ১. মাইক্রোসার্ভিস আর্কিটেকচারে — যেখানে সার্ভিস-টু-সার্ভিস কল আছে
     উদাহরণ: Pathao Ride → Payment → Notification

  ২. থার্ড-পার্টি API ইন্টিগ্রেশনে
     উদাহরণ: Daraz → bKash/Nagad API, Grameenphone SMS API

  ৩. ডিস্ট্রিবিউটেড সিস্টেমে — যেখানে নেটওয়ার্ক ব্যর্থতা স্বাভাবিক
     উদাহরণ: মাল্টি-রিজিয়ন ডিপ্লয়মেন্ট

  ৪. উচ্চ ট্রাফিক সিস্টেমে
     উদাহরণ: বিকাশ ঈদের দিন, Daraz 11.11 সেল

  ৫. যেখানে SLA / uptime গ্যারান্টি দরকার
     উদাহরণ: ফিনটেক (bKash, Nagad) — 99.9% uptime
```

### ❌ কখন ব্যবহার করবেন না

```
  ১. সিম্পল মনোলিথিক অ্যাপ — যেখানে সব কোড একই প্রসেসে চলে
     অতিরিক্ত complexity যোগ হবে অকারণে

  ২. ব্যাচ জব / অফলাইন প্রসেসিং — যেখানে ব্যবহারকারী অপেক্ষা করছে না
     সরল retry loop যথেষ্ট

  ৩. ডেভেলপমেন্ট / টেস্ট পরিবেশে
     ডিবাগিং কঠিন হয়ে যায়

  ৪. যেখানে ব্যর্থতা permanent (transient নয়)
     উদাহরণ: ভুল API key → retry করলেও কাজ হবে না

  ৫. সিম্পল CRUD অ্যাপ
     যেমন: একটি ব্লগ সাইট — জটিল resilience pattern দরকার নেই
```

---

## ❌ অ্যান্টি-প্যাটার্ন (Bad Patterns) বনাম ✅ সঠিক প্যাটার্ন

### ❌ Bad: Timeout ছাড়া API কল

```php
// ❌ খারাপ — কোনো timeout নেই, চিরকাল ব্লক হতে পারে
$response = Http::get('https://api.bkash.com/payment/status');
```

### ✅ Good: সবসময় timeout দিন

```php
// ✅ ভালো — timeout সেট করা আছে
$response = Http::timeout(5)
    ->connectTimeout(2)
    ->get('https://api.bkash.com/payment/status');
```

---

### ❌ Bad: Retry ছাড়া jitter/backoff

```javascript
// ❌ খারাপ — সবাই ১ সেকেন্ড পর একসাথে retry করবে (thundering herd)
async function badRetry(fn, retries = 3) {
    for (let i = 0; i < retries; i++) {
        try { return await fn(); }
        catch (e) { await sleep(1000); } // ← ফিক্সড delay!
    }
}
```

### ✅ Good: Exponential backoff + jitter

```javascript
// ✅ ভালো — retry ছড়িয়ে পড়বে
async function goodRetry(fn, retries = 3) {
    for (let i = 0; i < retries; i++) {
        try { return await fn(); }
        catch (e) {
            const delay = Math.pow(2, i) * 1000;
            const jitter = Math.random() * delay * 0.5;
            await sleep(delay + jitter);
        }
    }
}
```

---

### ❌ Bad: Non-idempotent অপারেশনে retry

```php
// ❌ বিপজ্জনক — ডুপ্লিকেট পেমেন্ট হতে পারে!
RetryHandler::execute(function () use ($amount) {
    return $paymentGateway->charge($amount); // কোনো idempotency key নেই
});
```

### ✅ Good: Idempotency key সহ retry

```php
// ✅ নিরাপদ — একই key দিয়ে বারবার কল করলেও একবারই charge হবে
RetryHandler::execute(function () use ($orderId, $amount) {
    return $paymentGateway->charge($amount, idempotencyKey: "charge-{$orderId}");
});
```

---

### ❌ Bad: সব error-এ retry

```javascript
// ❌ খারাপ — 400 Bad Request বা 401 Unauthorized-এও retry করছে!
async function badRetryAll(fn) {
    for (let i = 0; i < 3; i++) {
        try { return await fn(); }
        catch (e) { await sleep(1000); } // সব error-এ retry
    }
}
```

### ✅ Good: শুধু retryable error-এ retry

```javascript
// ✅ ভালো — শুধু transient error-এ retry
async function smartRetry(fn) {
    const retryableStatuses = [408, 429, 500, 502, 503, 504];
    for (let i = 0; i < 3; i++) {
        try { return await fn(); }
        catch (e) {
            if (e.status && !retryableStatuses.includes(e.status)) {
                throw e; // permanent error — retry করো না
            }
            await sleep(Math.pow(2, i) * 1000);
        }
    }
}
```

---

### ❌ Bad: Circuit Breaker ছাড়া সরাসরি কল

```
  ❌ ব্যর্থ সার্ভিসে বারবার কল — রিসোর্স অপচয়:

  Service A ──→ Service B (💀 ডাউন)
  Service A ──→ Service B (💀 ডাউন)    ← প্রতিটি কলে ৫ সেকেন্ড timeout
  Service A ──→ Service B (💀 ডাউন)       Thread/Connection অপচয়
  Service A ──→ Service B (💀 ডাউন)
  ...১০০+ কল... ──→ Service A-ও ধীর হয়ে গেল 💀
```

### ✅ Good: Circuit Breaker সহ

```
  ✅ Circuit Breaker — ৫ বার ব্যর্থ হলে কল বন্ধ:

  Service A ──→ Service B (💀) ← failure count: 1
  Service A ──→ Service B (💀) ← failure count: 5 → OPEN!
  Service A ──→ [CIRCUIT OPEN] ──→ Fallback (instant!) ✅
  Service A ──→ [CIRCUIT OPEN] ──→ Fallback (instant!) ✅
  ... ৩০ সেকেন্ড পর ...
  Service A ──→ Service B (✅) ← HALF-OPEN → test call সফল → CLOSED
```

---

### ❌ Bad: একটি shared connection pool

```
  ❌ সব সার্ভিসের জন্য একটি pool — একটি ধীর হলে সব আটকে যায়:

  ┌──────────────────────────────────────────┐
  │        Shared Connection Pool (50)       │
  │ ████████████████████████████████████████ │ ← ৫০-এর ৪৫টি Payment-এ আটকা!
  │ Payment (ধীর): 45   Ride: 3   Food: 2   │
  └──────────────────────────────────────────┘
  Ride ও Food-ও ভোগে 😢
```

### ✅ Good: Bulkhead — আলাদা pool

```
  ✅ আলাদা pool — Payment ধীর হলেও Ride ও Food ঠিক:

  ┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
  │ Payment Pool (20)│ │ Ride Pool(20)│ │ Food Pool(10)│
  │ ██████████████████│ │ ███░░░░░░░░░░│ │ ████░░░░░░░░░│
  │ 20/20 (পূর্ণ)    │ │ 3/20 (ঠিক)  │ │ 4/10 (ঠিক)  │
  └──────────────────┘ └──────────────┘ └──────────────┘
  Payment পূর্ণ হলেও Ride ও Food স্বাভাবিক ✅
```

---

## 🔗 উপকারী রিসোর্স

```
  📚 বই:
  ┄┄┄┄┄┄
  • "Release It!" — Michael Nygard
  • "Designing Distributed Systems" — Brendan Burns
  • "Site Reliability Engineering" — Google SRE Book

  🔧 লাইব্রেরি:
  ┄┄┄┄┄┄┄┄┄┄┄┄┄
  • PHP: ganesha (circuit breaker), guzzle-retry-middleware
  • JS: opossum (circuit breaker), cockatiel, p-retry, p-timeout
  • Java: resilience4j, Hystrix (legacy)

  📝 নিবন্ধ:
  ┄┄┄┄┄┄┄┄┄┄┄
  • Martin Fowler — "Circuit Breaker" pattern
  • AWS Architecture Blog — "Exponential Backoff and Jitter"
  • Netflix Tech Blog — "Fault Tolerance in a High Volume, Distributed System"
```

---

## 📖 সারসংক্ষেপ

```
  ┌──────────────────────┬───────────────────────────────────────────┐
  │ প্যাটার্ন              │ মূল কাজ                                   │
  ├──────────────────────┼───────────────────────────────────────────┤
  │ Circuit Breaker      │ ক্যাসকেডিং ফেইলিউর রোধ                    │
  │ Retry + Backoff      │ সাময়িক ব্যর্থতা স্বয়ংক্রিয় পুনরায় চেষ্টা   │
  │ Timeout              │ অনির্দিষ্টকাল ব্লক হওয়া রোধ                │
  │ Bulkhead             │ রিসোর্স আইসোলেশন                          │
  │ Fallback             │ বিকল্প সেবা প্রদান                         │
  │ Rate Limiter         │ API রেট লিমিটের মধ্যে থাকা                 │
  │ Health Check         │ সিস্টেম ও dependency পর্যবেক্ষণ             │
  │ Chaos Engineering    │ প্র্যাক্টিসে ব্যর্থতা সহনশীলতা পরীক্ষা      │
  └──────────────────────┴───────────────────────────────────────────┘

  মনে রাখুন: "Hope is not a strategy" — সিস্টেম ব্যর্থ হবেই,
  প্রস্তুতিই আসল পার্থক্য তৈরি করে। 🛡️
```
