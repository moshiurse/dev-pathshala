# ⏰ Timeout Pattern & Deadline Propagation

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Timeout-এর প্রকারভেদ](#timeout-এর-প্রকারভেদ)
- [Deadline Propagation](#deadline-propagation)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Timeout Pattern** হলো distributed system-এ একটি অপারেশনের জন্য সর্বোচ্চ অপেক্ষার সময় নির্ধারণ করা। যদি নির্দিষ্ট সময়ের মধ্যে response না আসে, তাহলে অপারেশন ব্যর্থ ধরে নেওয়া।

**Deadline Propagation** হলো একটি request-এর মোট সময়সীমা (deadline) service chain-এর মধ্য দিয়ে propagate করা, যাতে প্রতিটি service জানে তার কতটুকু সময় বাকি আছে।

### কেন Timeout গুরুত্বপূর্ণ?

Timeout ছাড়া distributed system-এ:
- 🔄 Thread/connection চিরকালের জন্য আটকে যেতে পারে
- 💀 Resource exhaustion হয় (সব thread/connection ব্যবহৃত)
- 🌊 Cascading failure: একটি slow service পুরো system ডাউন করে
- 💸 ব্যবহারকারী অভিজ্ঞতা খারাপ হয়

---

## 🌍 বাস্তব জীবনের উদাহরণ

### bKash Payment Flow — ৩০ সেকেন্ড Budget

ধরুন, আপনি bKash app থেকে Daraz-এ ৫,০০০ টাকা পেমেন্ট করছেন। পুরো flow-তে মোট ৩০ সেকেন্ড সময় আছে:

```
ব্যবহারকারী → bKash App → Payment Gateway → Bank API → Fraud Check → Notification
                 │              │                │           │            │
                 │   ৫s budget  │    ১০s budget  │  ৮s budget│  ৫s budget │  ২s budget
                 │              │                │           │            │
                 └──────────────┴────────────────┴───────────┴────────────┘
                              মোট: ৩০ সেকেন্ড deadline
```

**সমস্যা**: যদি Bank API ১৫ সেকেন্ড নেয়, তাহলে Fraud Check এবং Notification-এর জন্য মাত্র ৫ সেকেন্ড বাকি আছে!

### Grameenphone MyGP App:
- Recharge request → ১৫s total deadline
- Balance check → ৫s timeout
- Charging API → ৮s timeout
- SMS confirmation → ২s timeout

---

## 📊 Timeout-এর প্রকারভেদ

### 1. Connection Timeout
- TCP connection establish করতে কত সময় অপেক্ষা করবে
- সাধারণত: ৩-৫ সেকেন্ড
- কম রাখা উচিত: সার্ভার unreachable হলে দ্রুত জানা যায়

### 2. Read Timeout (Socket Timeout)
- Connection হওয়ার পর response-এর জন্য কত সময় অপেক্ষা
- সাধারণত: ১০-৩০ সেকেন্ড
- API complexity অনুযায়ী ভিন্ন হয়

### 3. Write Timeout
- Request পাঠাতে কত সময় লাগতে পারে
- বড় payload-এর জন্য গুরুত্বপূর্ণ
- সাধারণত: ৫-১০ সেকেন্ড

### 4. Idle Timeout
- Connection idle থাকলে কতক্ষণ পর বন্ধ করবে
- Connection pool management-এ ব্যবহৃত
- সাধারণত: ৩০-৬০ সেকেন্ড

### 5. Total/Request Timeout
- পুরো request lifecycle-এর মোট সময়
- Retry সহ সব কিছু এই সময়ের মধ্যে শেষ হতে হবে

---

## 🔗 Deadline Propagation

### ধারণা:
প্রতিটি service call-এ remaining time পাঠানো, যাতে downstream service জানে সে কতটুকু সময়ের মধ্যে কাজ শেষ করতে হবে।

### gRPC Deadline:
```
Client sets: deadline = now() + 30s
  → Service A receives: remaining = 28s (2s network)
    → Service B receives: remaining = 20s (8s processing + network)
      → Service C receives: remaining = 12s
```

### HTTP Header Propagation:
```
X-Request-Deadline: 2024-01-15T10:30:30Z
X-Request-Timeout-Ms: 15000
X-Request-Start: 2024-01-15T10:30:00Z
```

### Timeout Budget Calculation:
```
Total Budget: 30s
├── Service A: max 8s
│   └── Remaining after A: 30 - 8 = 22s
├── Service B: max 10s
│   └── Remaining after A+B: 22 - 10 = 12s
├── Service C: max 7s
│   └── Remaining after A+B+C: 12 - 7 = 5s
└── Service D: max 5s (exactly fits!)
    └── Remaining: 0s
```

---

## 📊 ASCII ডায়াগ্রাম

### Timeout Budget Flow — bKash Payment:

```
    ┌────────────────────────────────────────────────────────────────────┐
    │                    মোট Deadline: 30 সেকেন্ড                         │
    │  ┌───────┬──────────┬───────────┬──────────┬────────┐             │
    │  │ bKash │ Payment  │  Bank API │  Fraud   │ Notify │             │
    │  │  App  │ Gateway  │           │  Check   │        │             │
    │  │  5s   │   10s    │    8s     │   5s     │  2s    │             │
    │  └───────┴──────────┴───────────┴──────────┴────────┘             │
    │  |◄─────────────────── 30s ──────────────────────────►|           │
    └────────────────────────────────────────────────────────────────────┘

    Timeline:
    0s        5s         15s        23s       28s     30s
    |─────────|──────────|──────────|─────────|───────|
    │  bKash  │ Gateway  │ Bank API │  Fraud  │Notify │
    │         │          │          │         │       │
    └─────────┴──────────┴──────────┴─────────┴───────┘
```

### Deadline Propagation Chain:

```
┌──────────┐    deadline=30s    ┌──────────────┐
│  Client  │───────────────────>│  API Gateway │
│  (User)  │                    │   (Entry)    │
└──────────┘                    └───────┬──────┘
                                        │
                           remaining=28s │ (2s network overhead)
                                        ▼
                                ┌──────────────┐
                                │  bKash Auth  │ ← "আমার কাছে 28s আছে"
                                │   Service    │    processing: 3s
                                └───────┬──────┘
                                        │
                           remaining=24s │ (3s processing + 1s network)
                                        ▼
                                ┌──────────────┐
                                │   Payment    │ ← "আমার কাছে 24s আছে"
                                │   Service    │    processing: 8s
                                └───────┬──────┘
                                        │
                           remaining=15s │
                                        ▼
                                ┌──────────────┐
                                │   Bank API   │ ← "আমার কাছে 15s আছে"
                                │  (External)  │    processing: 10s
                                └───────┬──────┘
                                        │
                           remaining=4s  │
                                        ▼
                                ┌──────────────┐
                                │ Notification │ ← "আমার কাছে 4s আছে"
                                │   Service    │    processing: 2s ✓
                                └──────────────┘
```

### Cascading Timeout Failure:

```
    সমস্যা: Bank API slow হলে কী হয়?

    ┌──────┐    ┌─────────┐    ┌──────────┐    ┌──────────┐
    │Client│───>│Service A│───>│Service B │───>│ Bank API │
    │30s   │    │timeout: │    │timeout:  │    │          │
    │wait  │    │  25s    │    │  20s     │    │ 💤 SLOW! │
    └──────┘    └─────────┘    └──────────┘    │ (60s)    │
                                               └──────────┘

    Without Timeout:
    ─────────────────────────────────────────────────────►
    Client waits 60+ seconds 😱 All threads blocked!

    With Timeout & Deadline:
    ────────────────────────────────|
    Service B timeout at 20s → Error returned
    Client gets response in ~22s ✓
    Resources freed immediately! ✓
```

### Timeout vs Circuit Breaker সম্পর্ক:

```
┌─────────────────────────────────────────────────────────┐
│              Timeout + Circuit Breaker                    │
│                                                          │
│  Request ──► [Timeout Check] ──► [Circuit Breaker]      │
│                   │                     │                │
│                   │ timeout?            │ open?          │
│                   ▼                     ▼               │
│              ┌─────────┐          ┌──────────┐          │
│              │ Cancel   │          │ Fail Fast│          │
│              │ Request  │          │ (no call)│          │
│              └─────────┘          └──────────┘          │
│                   │                     │                │
│                   └──────────┬──────────┘                │
│                              ▼                           │
│                     Circuit Breaker                       │
│                     failure count++                       │
│                                                          │
│  Timeout → triggers CB    CB Open → no timeout needed    │
└─────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Timeout Pattern with Deadline Propagation
 * bKash Payment System সিমুলেশন
 */

class RequestDeadline
{
    private float $absoluteDeadline;
    private float $startTime;

    public function __construct(float $totalTimeoutMs)
    {
        $this->startTime = microtime(true) * 1000;
        $this->absoluteDeadline = $this->startTime + $totalTimeoutMs;
    }

    /**
     * বাকি কত সময় আছে (milliseconds)
     */
    public function remainingMs(): float
    {
        $remaining = $this->absoluteDeadline - (microtime(true) * 1000);
        return max(0, $remaining);
    }

    /**
     * Deadline পার হয়ে গেছে কিনা
     */
    public function isExpired(): bool
    {
        return $this->remainingMs() <= 0;
    }

    /**
     * Elapsed time
     */
    public function elapsedMs(): float
    {
        return (microtime(true) * 1000) - $this->startTime;
    }

    /**
     * Downstream service-এর জন্য reduced timeout
     */
    public function childTimeout(float $networkOverheadMs = 100): float
    {
        $remaining = $this->remainingMs() - $networkOverheadMs;
        return max(0, $remaining);
    }
}

/**
 * Timeout-aware HTTP Client
 */
class TimeoutHttpClient
{
    private float $connectTimeout;
    private float $readTimeout;

    public function __construct(float $connectTimeoutMs = 3000, float $readTimeoutMs = 10000)
    {
        $this->connectTimeout = $connectTimeoutMs;
        $this->readTimeout = $readTimeoutMs;
    }

    /**
     * Deadline-aware request
     */
    public function request(string $url, array $options, RequestDeadline $deadline): array
    {
        $remaining = $deadline->remainingMs();

        if ($remaining <= 0) {
            throw new TimeoutException("Deadline ইতিমধ্যে শেষ হয়ে গেছে");
        }

        // Effective timeout = min(configured timeout, remaining deadline)
        $effectiveConnectTimeout = min($this->connectTimeout, $remaining);
        $effectiveReadTimeout = min($this->readTimeout, $remaining - $effectiveConnectTimeout);

        echo "    📡 Request: {$url}\n";
        echo "    ⏱️  Connect timeout: {$effectiveConnectTimeout}ms, Read timeout: {$effectiveReadTimeout}ms\n";
        echo "    ⏳ Remaining budget: {$remaining}ms\n";

        // cURL দিয়ে actual request (simplified simulation)
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CONNECTTIMEOUT_MS => (int) $effectiveConnectTimeout,
            CURLOPT_TIMEOUT_MS => (int) ($effectiveConnectTimeout + $effectiveReadTimeout),
            CURLOPT_HTTPHEADER => [
                "X-Request-Deadline: {$deadline->remainingMs()}",
                "X-Request-Timeout-Ms: {$effectiveReadTimeout}",
            ],
        ]);

        // Simulated response
        usleep((int) (rand(100, 500) * 1000)); // 100-500ms simulated delay

        if ($deadline->isExpired()) {
            throw new TimeoutException("Request timeout: deadline পার হয়ে গেছে");
        }

        return ['status' => 200, 'data' => 'success'];
    }
}

class TimeoutException extends \RuntimeException {}

/**
 * bKash Payment Service with Deadline Propagation
 */
class BkashPaymentService
{
    private TimeoutHttpClient $httpClient;

    public function __construct()
    {
        $this->httpClient = new TimeoutHttpClient(3000, 10000);
    }

    /**
     * Payment প্রসেস — ৩০ সেকেন্ড মোট budget
     */
    public function processPayment(float $amount, string $sender, string $receiver): array
    {
        $deadline = new RequestDeadline(30000); // ৩০ সেকেন্ড মোট

        echo "🏦 bKash Payment শুরু\n";
        echo "   Amount: ৳{$amount} | {$sender} → {$receiver}\n";
        echo "   Total Budget: 30,000ms\n";
        echo str_repeat('─', 50) . "\n";

        try {
            // Step 1: Authentication (max 5s)
            $this->authenticate($sender, $deadline);

            // Step 2: Balance Check (max 5s)
            $this->checkBalance($sender, $amount, $deadline);

            // Step 3: Bank API Call (max 10s)
            $this->callBankApi($amount, $sender, $receiver, $deadline);

            // Step 4: Fraud Check (max 5s)
            $this->fraudCheck($amount, $sender, $receiver, $deadline);

            // Step 5: Notification (max 2s)
            $this->sendNotification($sender, $receiver, $amount, $deadline);

            echo "\n✅ Payment সফল! (Total: {$deadline->elapsedMs()}ms)\n";
            return ['status' => 'success', 'elapsed' => $deadline->elapsedMs()];

        } catch (TimeoutException $e) {
            echo "\n⏰ TIMEOUT: {$e->getMessage()}\n";
            echo "   Elapsed: {$deadline->elapsedMs()}ms\n";
            return ['status' => 'timeout', 'error' => $e->getMessage()];

        } catch (\Exception $e) {
            echo "\n❌ ERROR: {$e->getMessage()}\n";
            return ['status' => 'error', 'error' => $e->getMessage()];
        }
    }

    private function authenticate(string $user, RequestDeadline $deadline): void
    {
        echo "\n📋 Step 1: Authentication\n";
        $stepTimeout = min(5000, $deadline->remainingMs());

        if ($stepTimeout <= 0) {
            throw new TimeoutException("Auth-এর জন্য সময় নেই");
        }

        usleep(rand(200, 800) * 1000); // Simulate 200-800ms

        if ($deadline->isExpired()) {
            throw new TimeoutException("Auth timeout");
        }

        echo "   ✓ Authenticated ({$deadline->elapsedMs()}ms elapsed)\n";
    }

    private function checkBalance(string $user, float $amount, RequestDeadline $deadline): void
    {
        echo "\n💰 Step 2: Balance Check\n";
        $stepTimeout = min(5000, $deadline->remainingMs());

        if ($stepTimeout <= 0) {
            throw new TimeoutException("Balance check-এর জন্য সময় নেই");
        }

        usleep(rand(100, 500) * 1000);

        if ($deadline->isExpired()) {
            throw new TimeoutException("Balance check timeout");
        }

        echo "   ✓ Balance sufficient ({$deadline->elapsedMs()}ms elapsed)\n";
    }

    private function callBankApi(float $amount, string $sender, string $receiver, RequestDeadline $deadline): void
    {
        echo "\n🏦 Step 3: Bank API Call\n";
        $stepTimeout = min(10000, $deadline->remainingMs());

        if ($stepTimeout <= 0) {
            throw new TimeoutException("Bank API-র জন্য সময় নেই");
        }

        echo "   ⏱️ Step budget: {$stepTimeout}ms\n";
        usleep(rand(500, 2000) * 1000); // Bank API slow হতে পারে

        if ($deadline->isExpired()) {
            throw new TimeoutException("Bank API timeout — ৳{$amount} transfer pending");
        }

        echo "   ✓ Bank confirmed ({$deadline->elapsedMs()}ms elapsed)\n";
    }

    private function fraudCheck(float $amount, string $sender, string $receiver, RequestDeadline $deadline): void
    {
        echo "\n🔍 Step 4: Fraud Detection\n";
        $stepTimeout = min(5000, $deadline->remainingMs());

        if ($stepTimeout <= 0) {
            // Fraud check skip করা যেতে পারে timeout হলে (graceful degradation)
            echo "   ⚠️ Skipping fraud check (no time remaining) — async check queued\n";
            return;
        }

        usleep(rand(200, 1000) * 1000);

        if ($deadline->isExpired()) {
            echo "   ⚠️ Fraud check timeout — async check queued\n";
            return; // Graceful degradation
        }

        echo "   ✓ No fraud detected ({$deadline->elapsedMs()}ms elapsed)\n";
    }

    private function sendNotification(string $sender, string $receiver, float $amount, RequestDeadline $deadline): void
    {
        echo "\n📱 Step 5: Notification\n";
        $stepTimeout = min(2000, $deadline->remainingMs());

        if ($stepTimeout <= 0) {
            echo "   ⚠️ Notification skipped (no time) — queued for async delivery\n";
            return;
        }

        usleep(rand(100, 300) * 1000);
        echo "   ✓ SMS sent to {$sender} & {$receiver}\n";
    }
}

// ========== ব্যবহার ==========
$bkash = new BkashPaymentService();
$result = $bkash->processPayment(5000, '01712345678', '01898765432');
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Timeout Pattern with Deadline Propagation
 * bKash → Daraz Payment Flow
 */

class DeadlineContext {
    constructor(totalTimeoutMs) {
        this.startTime = Date.now();
        this.absoluteDeadline = this.startTime + totalTimeoutMs;
        this.totalBudget = totalTimeoutMs;
    }

    /** বাকি কত millisecond আছে */
    get remainingMs() {
        return Math.max(0, this.absoluteDeadline - Date.now());
    }

    /** Deadline পার হয়ে গেছে কিনা */
    get isExpired() {
        return this.remainingMs <= 0;
    }

    /** কত সময় পার হয়েছে */
    get elapsedMs() {
        return Date.now() - this.startTime;
    }

    /** Child service-এর জন্য নতুন context */
    createChildContext(networkOverheadMs = 50) {
        const remaining = this.remainingMs - networkOverheadMs;
        return new DeadlineContext(Math.max(0, remaining));
    }

    /** HTTP headers-এ deadline পাঠানো */
    toHeaders() {
        return {
            'X-Deadline-Ms': String(this.remainingMs),
            'X-Request-Start': new Date(this.startTime).toISOString(),
            'X-Total-Budget-Ms': String(this.totalBudget)
        };
    }

    /** Header থেকে deadline reconstruct */
    static fromHeaders(headers) {
        const remainingMs = parseInt(headers['x-deadline-ms'] || '30000');
        return new DeadlineContext(remainingMs);
    }
}

/**
 * Timeout-aware fetch wrapper
 */
async function fetchWithDeadline(url, options = {}, deadline) {
    if (deadline.isExpired) {
        throw new TimeoutError(`Deadline expired before calling ${url}`);
    }

    const controller = new AbortController();
    const effectiveTimeout = Math.min(
        options.timeout || 30000,
        deadline.remainingMs
    );

    const timeoutId = setTimeout(() => controller.abort(), effectiveTimeout);

    try {
        const response = await fetch(url, {
            ...options,
            signal: controller.signal,
            headers: {
                ...options.headers,
                ...deadline.toHeaders()
            }
        });
        return response;
    } catch (error) {
        if (error.name === 'AbortError') {
            throw new TimeoutError(
                `Timeout after ${effectiveTimeout}ms calling ${url} ` +
                `(budget remaining: ${deadline.remainingMs}ms)`
            );
        }
        throw error;
    } finally {
        clearTimeout(timeoutId);
    }
}

class TimeoutError extends Error {
    constructor(message) {
        super(message);
        this.name = 'TimeoutError';
        this.isTimeout = true;
    }
}

/**
 * bKash Payment Gateway — Deadline Propagation সহ
 */
class BkashPaymentGateway {
    constructor(config = {}) {
        this.config = {
            totalTimeout: config.totalTimeout || 30000,
            authTimeout: config.authTimeout || 5000,
            balanceTimeout: config.balanceTimeout || 5000,
            bankApiTimeout: config.bankApiTimeout || 10000,
            fraudTimeout: config.fraudTimeout || 5000,
            notifyTimeout: config.notifyTimeout || 2000,
        };
    }

    /**
     * মূল Payment Processing
     */
    async processPayment({ amount, sender, receiver, merchant }) {
        const deadline = new DeadlineContext(this.config.totalTimeout);

        console.log('🏦 ═══════════════════════════════════════');
        console.log(`   bKash Payment Processing`);
        console.log(`   Amount: ৳${amount.toLocaleString()}`);
        console.log(`   ${sender} → ${receiver} (${merchant})`);
        console.log(`   Budget: ${this.config.totalTimeout}ms`);
        console.log('═══════════════════════════════════════════\n');

        const result = {
            transactionId: `bkash_${Date.now()}`,
            steps: [],
            startTime: Date.now()
        };

        try {
            // Step 1: Authentication
            await this.executeStep('🔐 Authentication', async () => {
                await this.authenticate(sender, deadline);
            }, result, deadline);

            // Step 2: Balance Verification
            await this.executeStep('💰 Balance Check', async () => {
                await this.verifyBalance(sender, amount, deadline);
            }, result, deadline);

            // Step 3: Bank API (heaviest operation)
            await this.executeStep('🏦 Bank Transfer', async () => {
                await this.bankTransfer(amount, sender, receiver, deadline);
            }, result, deadline);

            // Step 4: Fraud Detection (graceful degradation)
            await this.executeStep('🔍 Fraud Check', async () => {
                await this.fraudDetection(amount, sender, deadline);
            }, result, deadline, { optional: true });

            // Step 5: Notifications (fire-and-forget friendly)
            await this.executeStep('📱 Notification', async () => {
                await this.notify(sender, receiver, amount, deadline);
            }, result, deadline, { optional: true });

            result.status = 'SUCCESS';
            result.totalTime = Date.now() - result.startTime;

            console.log(`\n✅ Payment সম্পন্ন! Total: ${result.totalTime}ms`);
            console.log(`   Remaining budget: ${deadline.remainingMs}ms`);

        } catch (error) {
            result.status = error instanceof TimeoutError ? 'TIMEOUT' : 'ERROR';
            result.error = error.message;
            result.totalTime = Date.now() - result.startTime;

            console.log(`\n❌ Payment ব্যর্থ: ${error.message}`);
            console.log(`   Total time: ${result.totalTime}ms`);
        }

        return result;
    }

    /**
     * Individual step execution with timing
     */
    async executeStep(name, fn, result, deadline, options = {}) {
        const stepStart = Date.now();
        console.log(`\n${name}`);
        console.log(`   Budget remaining: ${deadline.remainingMs}ms`);

        if (deadline.isExpired && !options.optional) {
            throw new TimeoutError(`${name}: deadline পার হয়ে গেছে`);
        }

        if (deadline.isExpired && options.optional) {
            console.log(`   ⚠️ Skipped (no time remaining) — queued async`);
            result.steps.push({ name, status: 'skipped', reason: 'deadline' });
            return;
        }

        try {
            await fn();
            const elapsed = Date.now() - stepStart;
            console.log(`   ✓ Done (${elapsed}ms)`);
            result.steps.push({ name, status: 'success', duration: elapsed });
        } catch (error) {
            const elapsed = Date.now() - stepStart;
            if (options.optional) {
                console.log(`   ⚠️ Failed but optional (${elapsed}ms): ${error.message}`);
                result.steps.push({ name, status: 'degraded', duration: elapsed });
            } else {
                throw error;
            }
        }
    }

    async authenticate(sender, deadline) {
        await this.simulateCall(200, 800, deadline, 'auth');
    }

    async verifyBalance(sender, amount, deadline) {
        await this.simulateCall(100, 500, deadline, 'balance');
    }

    async bankTransfer(amount, sender, receiver, deadline) {
        await this.simulateCall(500, 3000, deadline, 'bank');
    }

    async fraudDetection(amount, sender, deadline) {
        await this.simulateCall(200, 1500, deadline, 'fraud');
    }

    async notify(sender, receiver, amount, deadline) {
        await this.simulateCall(100, 500, deadline, 'notify');
    }

    /**
     * Simulated service call with deadline check
     */
    simulateCall(minMs, maxMs, deadline, label) {
        const duration = Math.floor(Math.random() * (maxMs - minMs)) + minMs;
        const effectiveTimeout = Math.min(duration, deadline.remainingMs);

        return new Promise((resolve, reject) => {
            setTimeout(() => {
                if (deadline.isExpired) {
                    reject(new TimeoutError(`${label}: deadline expired during call`));
                } else {
                    resolve();
                }
            }, Math.min(duration, effectiveTimeout));
        });
    }
}

/**
 * Cascading Timeout Calculator
 * Service chain-এ প্রতিটি hop-এর জন্য timeout calculate করে
 */
class TimeoutBudgetCalculator {
    constructor(totalBudgetMs) {
        this.totalBudget = totalBudgetMs;
        this.steps = [];
    }

    addStep(name, maxTimeMs, priority = 'required') {
        this.steps.push({ name, maxTime: maxTimeMs, priority });
        return this;
    }

    calculate() {
        const requiredTime = this.steps
            .filter(s => s.priority === 'required')
            .reduce((sum, s) => sum + s.maxTime, 0);

        const optionalTime = this.steps
            .filter(s => s.priority === 'optional')
            .reduce((sum, s) => sum + s.maxTime, 0);

        const totalNeeded = requiredTime + optionalTime;
        const ratio = totalNeeded > this.totalBudget
            ? this.totalBudget / totalNeeded
            : 1;

        console.log('\n📊 Timeout Budget Breakdown:');
        console.log(`   Total budget: ${this.totalBudget}ms`);
        console.log(`   Required: ${requiredTime}ms | Optional: ${optionalTime}ms`);
        console.log('   ─'.repeat(25));

        let allocated = 0;
        return this.steps.map(step => {
            const adjustedTime = Math.floor(step.maxTime * ratio);
            allocated += adjustedTime;
            console.log(`   ${step.name}: ${adjustedTime}ms (${step.priority})`);
            return { ...step, allocatedTime: adjustedTime };
        });
    }
}

// ========== ব্যবহার ==========

async function main() {
    // Budget Calculator
    const calculator = new TimeoutBudgetCalculator(30000);
    calculator
        .addStep('Authentication', 5000, 'required')
        .addStep('Balance Check', 5000, 'required')
        .addStep('Bank API', 10000, 'required')
        .addStep('Fraud Detection', 5000, 'optional')
        .addStep('Notification', 2000, 'optional');
    calculator.calculate();

    // Payment Processing
    const gateway = new BkashPaymentGateway({ totalTimeout: 30000 });

    const result = await gateway.processPayment({
        amount: 5000,
        sender: '01712345678',
        receiver: '01898765432',
        merchant: 'Daraz Bangladesh'
    });

    console.log('\n📋 Final Result:', JSON.stringify(result, null, 2));
}

main().catch(console.error);
```

---

## 🔄 Timeout vs Circuit Breaker সম্পর্ক

| বৈশিষ্ট্য | Timeout | Circuit Breaker |
|-----------|---------|-----------------|
| উদ্দেশ্য | একটি request-এর সময়সীমা | বারবার ব্যর্থ service বন্ধ করা |
| Scope | Single request | Multiple requests pattern |
| Action | Cancel current request | Prevent future requests |
| Recovery | Retry পরে | Half-open state-এ test |
| Trigger | সময় শেষ | Failure threshold cross |

**একসাথে ব্যবহার**:
```
Request → [Circuit Breaker Check] → [Timeout Wrapper] → Service Call
                    │                        │
                    │ OPEN? → Fail Fast       │ Timeout? → Cancel + CB failure count++
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন:
- 🌐 প্রতিটি external API call-এ (bKash → Bank API)
- 🔗 Microservice-to-microservice communication
- 📡 Database connection-এ
- 🌍 Third-party service integration (Pathao Maps, payment gateway)
- 📱 Mobile app → Backend call (user patience = ~3s)
- 🔄 gRPC/HTTP service mesh-এ deadline propagation

### ❌ ব্যবহার করবেন না:
- 📁 Local file I/O (সাধারণত timeout দরকার হয় না)
- 🧮 CPU-bound computation (timeout কাজ করে না — task cancel দরকার)
- 📊 Batch processing jobs (অনেক সময় নিতে পারে naturally)
- 🔄 Long-polling connections (intentionally long wait)

### 🎯 Timeout Value নির্ধারণ:

```
Service Type         │ Connection │ Read    │ Total
─────────────────────┼────────────┼─────────┼────────
Internal microservice│ 1-2s       │ 3-5s    │ 5-10s
External API (Bank)  │ 3-5s       │ 10-15s  │ 15-30s
Database query       │ 2-3s       │ 5-10s   │ 10-15s
CDN/Static content   │ 1-2s       │ 2-3s    │ 3-5s
User-facing API      │ -          │ -       │ 3-5s (UX)
Background job       │ 5-10s      │ 30-60s  │ 60-120s
```

### ⚠️ সাধারণ ভুল:
1. **Timeout না দেওয়া**: Thread forever blocked
2. **Timeout খুব বড়**: Resource waste, poor UX
3. **Timeout খুব ছোট**: False failures, unnecessary retries
4. **Deadline propagate না করা**: Downstream services সময় নষ্ট করে
5. **Retry + Timeout combo ভুল**: 3 retries × 30s = 90s total wait!

---

## 📚 সারসংক্ষেপ

Timeout এবং Deadline Propagation হলো distributed system-এর "seat belt" — সমস্যা প্রতিরোধ করে না, কিন্তু সমস্যা হলে ক্ষতি কমায়। প্রতিটি network call-এ timeout থাকা উচিত, এবং multi-service chain-এ deadline propagation নিশ্চিত করে যে কোনো service বৃথা কাজ করে সময় নষ্ট করছে না।
