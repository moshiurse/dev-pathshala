# ⚡ সার্কিট ব্রেকার প্যাটার্ন (Circuit Breaker Pattern)

> **"যখন একটি সার্ভিস অসুস্থ, তখন তাকে আরও কল দিয়ে মেরে ফেলবেন না — সার্কিট খুলে দিন, একে শ্বাস নিতে দিন।"**

ডিস্ট্রিবিউটেড সিস্টেমে ব্যর্থতা অনিবার্য। কিন্তু একটি ব্যর্থতা যখন *ক্যাসকেড* হয়ে পুরো সিস্টেম ফেলে দেয় — সেটাই
প্রকৃত বিপর্যয়। **Circuit Breaker** হলো সেই রেজিলিয়েন্স প্যাটার্ন যা ক্যাসকেডিং ফেইলিউর প্রতিরোধ করে, downstream
service-কে রিকভার হওয়ার সময় দেয় এবং upstream-কে দ্রুত fail-fast রেসপন্স দেয়।

এই ডকুমেন্টে আমরা ইলেকট্রিক্যাল analogy থেকে শুরু করে state machine, configuration tuning, library তুলনা,
distributed circuit breaker, observability এবং Daraz/bKash/Pathao-র প্রকৃত সিনারিও পর্যন্ত গভীরে যাব।

---

## 📑 সূচিপত্র

1. [সংজ্ঞা ও ইলেকট্রিক্যাল analogy](#-সংজ্ঞা--ইলেকট্রিকাল-analogy)
2. [কোন সমস্যা সমাধান করে: Cascading Failure](#-কোন-সমস্যা-সমাধান-করে-cascading-failure)
3. [State Machine: CLOSED, OPEN, HALF_OPEN](#-state-machine-closed-open-half_open)
4. [Failure Detection — সব strategy](#-failure-detection--সব-strategy)
5. [Configuration Knobs](#-configuration-knobs)
6. [অন্যান্য প্যাটার্নের সাথে combine](#-অন্যান্য-pattern-এর-সাথে-combine)
7. [Library তুলনা](#-library-তুলনা)
8. [Node.js: opossum + bKash payment](#-nodejs-opossum--bkash-payment)
9. [PHP: state machine + Guzzle](#-php-state-machine--guzzle)
10. [Distributed Circuit Breaker](#-distributed-circuit-breaker)
11. [Observability](#-observability)
12. [Anti-patterns](#️-anti-patterns)
13. [BD Scenario: Daraz Checkout](#-bd-scenario-daraz-checkout)
14. [Decision flow](#-decision-flow)
15. [Checklist](#-checklist)

---

## 📌 সংজ্ঞা ও ইলেকট্রিক্যাল analogy

আপনার বাড়ির meter board-এ একটা **Circuit Breaker** আছে। যদি ফ্রিজের কম্প্রেসর shorted হয়ে অতিরিক্ত current
টানে — breaker "trip" করে যায়। এতে:

- ফ্রিজ থামে (ব্যর্থ device)
- কিন্তু পুরো বাড়ির wiring পুড়ে যায় না (cascading failure প্রতিরোধ)
- আপনি technician ডেকে repair করেন (recovery time)
- তারপর breaker "reset" করেন (HALF_OPEN test)

**সফটওয়্যার circuit breaker** ঠিক একই কাজ করে — শুধু "current" এর জায়গায় "request" এবং "short" এর জায়গায়
"failure" বা "slow response":

```
       ইলেকট্রিকাল Analogy → সফটওয়্যার Mapping
       ═══════════════════════════════════════════

   Electrical World          │   Software World
   ──────────────────────────┼──────────────────────────────
   Current (A)               │   Request rate (req/sec)
   Short circuit             │   Service down / 5xx flood
   Overload                  │   Slow response / timeout
   Breaker tripped           │   Circuit OPEN (fail-fast)
   Manual reset              │   HALF_OPEN trial probe
   Fuse blown (one-shot)     │   No-half-open antipattern
   Surge protector           │   Bulkhead + Timeout
```

> **Definition (Martin Fowler):** Circuit Breaker একটি stateful proxy যা downstream call wrap করে; configured
> failure threshold অতিক্রম হলে এটি subsequent call-গুলোকে বাস্তবে invoke না করে সরাসরি error/fallback
> দেয়, এবং পর্যায়ক্রমে limited probe call পাঠিয়ে recovery test করে।

---

## 🔥 কোন সমস্যা সমাধান করে: Cascading Failure

ধরুন **Daraz checkout** flow:

```
   User → Checkout Service → Payment Service (bKash adapter)
                          → Inventory Service
                          → Pricing Service
                          → Notification Service
```

`bKash adapter` slow হয়ে গেছে (P99 = 30s, normally 200ms)। কী ঘটে **circuit breaker ছাড়া**?

```
  T+0s:   Checkout এর HTTP client → bKash call → 30s অপেক্ষা
  T+1s:   ১০০ নতুন user checkout শুরু করল → ১০০ thread/coroutine bKash এ আটকা
  T+5s:   Node.js event loop / PHP-FPM worker pool exhausted
  T+10s:  Checkout service নিজেই 503 দিচ্ছে — যদিও Inventory/Pricing fine ছিল
  T+15s:  Frontend retry storm → load 5x → Inventory ও পড়ে গেল
  T+30s:  💀 সম্পূর্ণ checkout down — Daraz-এর ৫ মিনিটে কোটি টাকার ক্ষতি
```

একটিমাত্র "slow dependency" পুরো sub-system-কে নামিয়ে দিল। এটাই **cascading failure** এবং thread/connection
**resource exhaustion**।

**Circuit breaker দিয়ে:**

```
  T+0s:   bKash slow → first 20 call timeout
  T+2s:   Failure threshold (50%) breach → breaker OPEN
  T+2s+:  পরবর্তী bKash call instantly fail (500µs) → fallback: "COD ব্যবহার করুন"
  T+30s:  bKash recover → HALF_OPEN probe → success → CLOSED
  Result: Checkout-এর বাকি ৭০% (COD/Card) user সফলভাবে order করল।
```

---

## 🔄 State Machine: CLOSED, OPEN, HALF_OPEN

```
                     CIRCUIT BREAKER STATE MACHINE
   ════════════════════════════════════════════════════════════════════

                          ┌──────────────────────┐
                          │       CLOSED         │
       ┌─────────────────▶│  (স্বাভাবিক অবস্থা)   │
       │                  │  সব call pass        │
       │                  │  failure count track │
       │                  └──────────┬───────────┘
       │                             │
       │                  failureRate ≥ threshold
       │                  AND callCount ≥ minCalls
       │                             │
       │                             ▼
       │                  ┌──────────────────────┐
       │                  │        OPEN          │
       │                  │  সব call instant fail │
       │                  │  fallback() execute  │
       │                  │  (timer চলছে)         │
       │                  └──────────┬───────────┘
       │                             │
       │                  waitDuration শেষ
       │                             │
       │                             ▼
       │                  ┌──────────────────────┐
       │                  │     HALF_OPEN        │
       │                  │  N টি probe call     │
       │                  │  permit (concurrent  │
       │                  │  limit সহ)            │
       │                  └──────┬─────────┬─────┘
       │                         │         │
       │            সব probe success    যেকোনো probe fail
       │                         │         │
       └─────────────────────────┘         │
                                           │
                                           ▼
                                   ┌──────────────┐
                                   │     OPEN     │ ◀── (back to OPEN)
                                   └──────────────┘
```

### State বিস্তারিত

#### 🟢 CLOSED — স্বাভাবিক
- প্রতিটি call downstream এ যায়
- success/failure count sliding window-এ track করা হয়
- `failureRate ≥ failureThreshold` AND `totalCalls ≥ minimumNumberOfCalls` হলে → **OPEN**

#### 🔴 OPEN — fail-fast
- কোনো call downstream এ যায় না
- অবিলম্বে `CircuitBreakerOpenException` throw বা fallback execute
- `waitDurationInOpenState` (e.g., 30s) timer চালু
- timer শেষে → **HALF_OPEN**

#### 🟡 HALF_OPEN — probe / trial
- শুধু `permittedNumberOfCallsInHalfOpenState` (e.g., 5) call allow
- বাকি call: instantly fail বা reject
- সব probe success → metrics reset → **CLOSED**
- ১টি probe fail → **OPEN** (timer reset)

> **গুরুত্বপূর্ণ:** Half-open state ছাড়া breaker হলো একটি "fuse" — একবার blown, manual reset দরকার। এটা
> **anti-pattern**: production-এ self-healing দরকার।

---

## 🎯 Failure Detection — সব strategy

কখন একটি call "failed" বলে গণ্য হবে?

### ১. Error-based — exception/HTTP status

```javascript
const isFailure = (err, response) => {
  if (err) return true;                    // network error, timeout
  if (response.status >= 500) return true; // server error
  if (response.status === 429) return true; // rate limited (debatable)
  if (response.status >= 400 && response.status < 500) return false; // client error: NOT breaker's job
  return false;
};
```

> **Pitfall:** 4xx (validation error) breaker trip করানো ভুল — এটা downstream এর সমস্যা না, caller-এর সমস্যা।

### ২. Slow-call rate

```
  call duration > slowCallDurationThreshold (e.g., 2s)
  → counted as "slow" (not failed, but treated as failure for breaker purposes)
```

bKash যদি 30s নেয় (timeout পর্যন্ত পৌঁছায় না), তবু সেটা breaker trip করানো উচিত — কারণ thread/socket
দখল করছে।

### ৩. Consecutive failures (legacy)

```
  consecutiveFailures ≥ N → OPEN
```

Simple কিন্তু noisy। এক ঝাঁক success-এর পরে ১টি transient failure breaker trip করাবে না — অপ্রত্যাশিত।

### ৪. Sliding window — count vs time

| Window Type | কীভাবে কাজ করে | কখন উপযুক্ত |
|---|---|---|
| **Count-based** | শেষ N call (e.g., last 100) এর উপর rate | predictable traffic, batch jobs |
| **Time-based** | শেষ T সেকেন্ড (e.g., last 60s) এর সব call এর উপর rate | bursty traffic, user-facing API |

```
  Count-based (size=10):
  [✓✓✗✓✗✗✓✗✗✗]  → 6/10 fail = 60%

  Time-based (60s window):
  T=0s ────────── T=60s
   ✓ ✓ ✗ ✓ ✗ ✓ ✗ ✗ ✗   (8 calls in last 60s, 5 failed = 62.5%)
```

### ৫. Composite / multi-signal

Resilience4j-এ `recordExceptions`, `ignoreExceptions`, `recordResultPredicate` — একসাথে combine করে সিদ্ধান্ত
নেওয়া হয়।

---

## 🎛️ Configuration Knobs

প্রতিটি library-তে নাম আলাদা হলেও concept গুলো same:

```
┌──────────────────────────────────────┬───────────────────────────────────────────┐
│ Knob                                 │ ব্যাখ্যা ও typical value                    │
├──────────────────────────────────────┼───────────────────────────────────────────┤
│ failureRateThreshold                 │ % failure → OPEN. Typical: 50%             │
│ slowCallRateThreshold                │ % slow → OPEN. Typical: 100% (off) বা 80%  │
│ slowCallDurationThreshold            │ কোন duration-এর পরে call "slow"। 2s       │
│ minimumNumberOfCalls                 │ এর কম হলে rate calculate করা যাবে না।    │
│                                       │ Typical: 10-20 (statistical সিগ্নিফিকান্স)│
│ slidingWindowSize                    │ count: 100, time: 60s                      │
│ slidingWindowType                    │ COUNT_BASED বা TIME_BASED                  │
│ waitDurationInOpenState              │ OPEN → HALF_OPEN delay. Typical: 30s-60s   │
│ permittedNumberOfCallsInHalfOpenState│ probe সংখ্যা। Typical: 5-10               │
│ maxConcurrentCalls (bulkhead-like)   │ একসাথে ক'টি call। Typical: pool size      │
│ automaticTransitionFromOpenToHalfOpen│ true: timer based, false: next call trig   │
└──────────────────────────────────────┴───────────────────────────────────────────┘
```

### Tuning rule of thumb

```
  ┌───────────────────────────────────────────────────────────────┐
  │ 1. minimumNumberOfCalls সবার আগে set করুন (statistical sense)  │
  │ 2. failureRateThreshold = 50% (বেশি conservative হলে noisy)    │
  │ 3. slowCallDurationThreshold = downstream P95 latency × 3      │
  │ 4. waitDurationInOpenState = downstream healing time অনুমান    │
  │    (DB connection pool reset = ~30s; container restart = ~60s) │
  │ 5. permittedHalfOpenCalls কম (5) — accidental flood এড়াতে     │
  └───────────────────────────────────────────────────────────────┘
```

---

## 🧱 অন্যান্য pattern এর সাথে combine

Circuit Breaker একা **যথেষ্ট নয়** — এটি resilience-এর একটি স্তর মাত্র। Layered defense:

```
                Layered Resilience Strategy
                ═══════════════════════════════

         ┌──────────────────────────────────┐
         │  ⏱️  Timeout (per request)        │  ← Layer 1: প্রতিটি call bound
         └──────────────┬───────────────────┘
                        │
         ┌──────────────▼───────────────────┐
         │  🔁 Retry (exponential + jitter) │  ← Layer 2: transient error
         └──────────────┬───────────────────┘
                        │
         ┌──────────────▼───────────────────┐
         │  ⚡ Circuit Breaker              │  ← Layer 3: cascading prevention
         └──────────────┬───────────────────┘
                        │
         ┌──────────────▼───────────────────┐
         │  🚧 Bulkhead (resource isolate)  │  ← Layer 4: blast radius limit
         └──────────────┬───────────────────┘
                        │
         ┌──────────────▼───────────────────┐
         │  🛟 Fallback (cached/default)    │  ← Layer 5: graceful degradation
         └──────────────────────────────────┘
```

### Order matters

```javascript
// সঠিক order: Bulkhead → CircuitBreaker → Retry → Timeout → call
const result = await bulkhead(() =>
  circuitBreaker(() =>
    retryWithBackoff(() =>
      withTimeout(callBkash(), 2000)
    , { attempts: 3, baseDelayMs: 200, jitter: true })
  )
);
```

> **কেন এই order?**
> - **Timeout** সবচেয়ে ভেতরে — প্রতিটি call bound
> - **Retry** তার বাইরে — failed call repeat
> - **Circuit Breaker** retry-এর বাইরে — যদি retry-ও বারবার fail হয়, তবে breaker বাকিদের protect করবে
> - **Bulkhead** সবার বাইরে — overall resource budget enforce

> ⚠️ **Common mistake:** Retry breaker-এর বাইরে রাখলে, breaker OPEN হলেও retry-storm চলতে থাকবে — defeats
> the purpose।

### Retry + jitter formula

```
  delay = min(baseDelay * 2^attempt, maxDelay)
  delay = delay/2 + random(0, delay/2)        ← "decorrelated jitter"
```

---

## 📚 Library তুলনা

```
┌────────────────────────┬──────────┬─────────────┬───────────────────────────────┐
│ Library                │ Stack    │ Maintained  │ মূল বৈশিষ্ট্য                  │
├────────────────────────┼──────────┼─────────────┼───────────────────────────────┤
│ Hystrix (Netflix)      │ JVM      │ ❌ Deprec.  │ Pioneer, thread pool isolation│
│ Resilience4j           │ JVM      │ ✅ Active   │ Functional, Hystrix successor │
│ Polly                  │ .NET     │ ✅ Active   │ Fluent policy composition     │
│ opossum                │ Node.js  │ ✅ Active   │ EventEmitter, simple API      │
│ cockatiel              │ Node.js  │ ✅ Active   │ Polly-inspired, TypeScript    │
│ gobreaker (Sony)       │ Go       │ ✅ Active   │ Minimal, idiomatic            │
│ failsafe-go            │ Go       │ ✅ Active   │ Multi-policy composition      │
│ Sentinel (Alibaba)     │ Java/Go  │ ✅ Active   │ Adaptive, hot-spot detection  │
│ Ackintosh/php-resilience│ PHP     │ 🟡 Limited  │ PSR-friendly                  │
│ spatie/laravel-cb      │ PHP/Lar. │ 🟡          │ Laravel facade integration    │
│ Istio/Envoy            │ Mesh     │ ✅ Active   │ Sidecar-level, language-agno. │
└────────────────────────┴──────────┴─────────────┴───────────────────────────────┘
```

### Application-level vs Service-mesh-level

```
  Application-level (opossum, resilience4j):
  ──────────────────────────────────────────
  ✅ Code-aware: business logic-এর সাথে fallback সহজ
  ✅ Per-method granular tuning
  ❌ প্রতিটি service-এ আলাদা integration
  ❌ Polyglot environment-এ inconsistent

  Service-mesh-level (Istio, Envoy outlier detection):
  ────────────────────────────────────────────────────
  ✅ Language agnostic
  ✅ Centralized control plane, declarative
  ✅ Network-level metrics built-in
  ❌ Business-aware fallback করা কঠিন
  ❌ Per-instance ejection (cluster-wide breaker না)
```

> **Best practice:** দু'টোই combine করুন। Mesh-level outlier detection দিয়ে dead instance eject; app-level
> breaker দিয়ে business fallback।

---

## 💻 Node.js: opossum + bKash payment

পূর্ণাঙ্গ example: bKash payment API call wrap, retry-with-backoff, fallback (cached response + queued retry):

```javascript
// circuit-breaker-bkash.js
const CircuitBreaker = require('opossum');
const axios = require('axios');
const Redis = require('ioredis');
const { v4: uuid } = require('uuid');

const redis = new Redis(process.env.REDIS_URL);

// ─── 1. Actual downstream call ───
async function callBkashCharge({ trxId, amount, msisdn }) {
  // bKash Tokenized Checkout API
  const { data } = await axios.post(
    'https://tokenized.pay.bka.sh/v1.2.0-beta/tokenized/checkout/create',
    { amount, currency: 'BDT', merchantInvoiceNumber: trxId, payerReference: msisdn },
    {
      timeout: 2000,                                  // ← inner timeout
      headers: { Authorization: `Bearer ${process.env.BKASH_TOKEN}` },
    }
  );
  if (data.statusCode !== '0000') {
    const err = new Error(`bKash error ${data.statusCode}: ${data.statusMessage}`);
    err.bkashCode = data.statusCode;
    throw err;
  }
  return data;
}

// ─── 2. Retry wrapper (decorrelated jitter) ───
async function retryWithBackoff(fn, { attempts = 3, baseMs = 150, maxMs = 1500 } = {}) {
  let lastErr;
  let delay = baseMs;
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      lastErr = err;
      // 4xx থেকে retry না; idempotent operation নিশ্চিত করুন
      if (err.bkashCode && err.bkashCode.startsWith('20')) throw err;
      if (i === attempts - 1) break;
      const jitter = delay / 2 + Math.random() * (delay / 2);
      await new Promise((r) => setTimeout(r, jitter));
      delay = Math.min(delay * 2, maxMs);
    }
  }
  throw lastErr;
}

// ─── 3. Circuit breaker configuration ───
const breaker = new CircuitBreaker(
  (payload) => retryWithBackoff(() => callBkashCharge(payload)),
  {
    timeout: 5000,                          // total time budget (retry সহ)
    errorThresholdPercentage: 50,           // 50% fail হলে OPEN
    resetTimeout: 30_000,                   // 30s OPEN, তারপর HALF_OPEN
    rollingCountTimeout: 60_000,            // 60s sliding time window
    rollingCountBuckets: 10,                // 6s প্রতি bucket
    volumeThreshold: 20,                    // < 20 call হলে rate calc করো না
    name: 'bkash-charge',
    group: 'payment',
  }
);

// ─── 4. Fallback: cached + queue for later ───
breaker.fallback(async (payload, err) => {
  console.warn(`[bkash] fallback fired: ${err?.message}`);

  // (a) idempotent লুকআপ — যদি একই trxId-এর জন্য আগেই success cached থাকে
  const cached = await redis.get(`bkash:trx:${payload.trxId}`);
  if (cached) return JSON.parse(cached);

  // (b) async queue → পরে retry worker pick করবে
  await redis.lpush(
    'bkash:retry-queue',
    JSON.stringify({ ...payload, queuedAt: Date.now(), id: uuid() })
  );

  // (c) frontend-কে graceful response — "প্রসেসিং চলছে"
  return {
    status: 'PENDING',
    message: 'পেমেন্ট প্রসেস হচ্ছে। SMS-এ confirmation পাবেন।',
    fallback: true,
  };
});

// ─── 5. Observability ───
breaker.on('open',     () => metric('cb.bkash.open', 1));
breaker.on('halfOpen', () => metric('cb.bkash.halfOpen', 1));
breaker.on('close',    () => metric('cb.bkash.close', 1));
breaker.on('reject',   () => metric('cb.bkash.reject', 1)); // OPEN-এ থাকাকালীন
breaker.on('timeout',  () => metric('cb.bkash.timeout', 1));
breaker.on('failure', (err) => metric('cb.bkash.failure', 1, { code: err.bkashCode }));
breaker.on('success', (_, latencyMs) => histogram('cb.bkash.latency', latencyMs));

// ─── 6. Express handler ───
const express = require('express');
const app = express();
app.use(express.json());

app.post('/payment/bkash', async (req, res) => {
  try {
    const result = await breaker.fire({
      trxId: req.body.trxId,
      amount: req.body.amount,
      msisdn: req.body.msisdn,
    });
    if (result.status === 'PENDING' && result.fallback) {
      return res.status(202).json(result); // 202 Accepted — async
    }
    return res.status(200).json(result);
  } catch (err) {
    return res.status(503).json({ error: 'Service Unavailable', detail: err.message });
  }
});

function metric(name, val, tags = {}) { /* StatsD/Prometheus emit */ }
function histogram(name, val) { /* ... */ }

app.listen(3000);
```

### এখানে কী ঘটছে?

```
   Client → /payment/bkash → breaker.fire()
                              │
              ┌───────────────┴───────────────┐
              │                               │
        breaker CLOSED                  breaker OPEN
              │                               │
              ▼                               ▼
     retryWithBackoff()             fallback() instantly
              │                               │
       callBkashCharge()              ┌───────┴────────┐
              │                       │                │
       success/fail                cached?         queue retry
              │                       │                │
       metric → window           return cached    202 PENDING
```

---

## 🐘 PHP: state machine + Guzzle

PHP-এ "real" multi-instance circuit breaker করতে হলে state শেয়ার করতে হবে (Redis)। নিচে both
in-memory simple এবং Redis-backed example।

### 6.1 Pure PHP state machine (single-process)

```php
<?php
// CircuitBreaker.php
namespace App\Resilience;

use RuntimeException;

class CircuitBreakerOpenException extends RuntimeException {}

class CircuitBreaker
{
    public const CLOSED = 'closed';
    public const OPEN = 'open';
    public const HALF_OPEN = 'half_open';

    private string $state = self::CLOSED;
    private array $window = [];        // [['ts' => float, 'ok' => bool], ...]
    private float $openedAt = 0.0;
    private int $halfOpenSuccesses = 0;

    public function __construct(
        private string $name,
        private int $failureThreshold = 50,        // %
        private int $minimumCalls = 10,
        private int $windowSeconds = 60,
        private int $waitDurationOpen = 30,        // seconds
        private int $halfOpenPermitted = 5,
        private float $slowCallSeconds = 2.0,
    ) {}

    public function execute(callable $fn, ?callable $fallback = null): mixed
    {
        $this->transitionFromOpenIfReady();

        if ($this->state === self::OPEN) {
            return $fallback
                ? $fallback(new CircuitBreakerOpenException("Breaker {$this->name} OPEN"))
                : throw new CircuitBreakerOpenException("Breaker {$this->name} OPEN");
        }

        $start = microtime(true);
        try {
            $result = $fn();
            $duration = microtime(true) - $start;
            $this->onSuccess($duration);
            return $result;
        } catch (\Throwable $e) {
            $this->onFailure();
            if ($fallback) return $fallback($e);
            throw $e;
        }
    }

    private function onSuccess(float $duration): void
    {
        $isSlow = $duration >= $this->slowCallSeconds;
        $this->record($isSlow ? false : true);

        if ($this->state === self::HALF_OPEN) {
            if (++$this->halfOpenSuccesses >= $this->halfOpenPermitted) {
                $this->reset();
            }
        }
    }

    private function onFailure(): void
    {
        $this->record(false);
        if ($this->state === self::HALF_OPEN) {
            $this->trip();   // ১টি probe fail = back to OPEN
            return;
        }
        if ($this->shouldTrip()) $this->trip();
    }

    private function record(bool $ok): void
    {
        $now = microtime(true);
        $this->window[] = ['ts' => $now, 'ok' => $ok];
        $this->window = array_values(array_filter(
            $this->window,
            fn($e) => $now - $e['ts'] <= $this->windowSeconds
        ));
    }

    private function shouldTrip(): bool
    {
        $total = count($this->window);
        if ($total < $this->minimumCalls) return false;
        $failed = count(array_filter($this->window, fn($e) => !$e['ok']));
        return ($failed / $total) * 100 >= $this->failureThreshold;
    }

    private function trip(): void
    {
        $this->state = self::OPEN;
        $this->openedAt = microtime(true);
        $this->halfOpenSuccesses = 0;
        // metric emit করুন
        error_log("[cb] {$this->name} → OPEN");
    }

    private function transitionFromOpenIfReady(): void
    {
        if ($this->state !== self::OPEN) return;
        if (microtime(true) - $this->openedAt >= $this->waitDurationOpen) {
            $this->state = self::HALF_OPEN;
            $this->halfOpenSuccesses = 0;
            error_log("[cb] {$this->name} → HALF_OPEN");
        }
    }

    private function reset(): void
    {
        $this->state = self::CLOSED;
        $this->window = [];
        $this->halfOpenSuccesses = 0;
        error_log("[cb] {$this->name} → CLOSED");
    }
}
```

### 6.2 Redis-backed (multi-instance) + Guzzle for SSLCommerz

```php
<?php
// RedisCircuitBreaker + GuzzleCall
namespace App\Resilience;

use Predis\Client as Redis;
use GuzzleHttp\Client as Guzzle;
use GuzzleHttp\Exception\RequestException;

class RedisCircuitBreaker
{
    public function __construct(
        private Redis $redis,
        private string $name,
        private int $failureThreshold = 50,
        private int $minimumCalls = 10,
        private int $windowSeconds = 60,
        private int $waitDurationOpen = 30,
    ) {}

    public function execute(callable $fn, ?callable $fallback = null): mixed
    {
        $stateKey = "cb:{$this->name}:state";
        $state = $this->redis->get($stateKey) ?? 'closed';

        if ($state === 'open') {
            $openedAt = (float) $this->redis->get("cb:{$this->name}:openedAt");
            if (microtime(true) - $openedAt >= $this->waitDurationOpen) {
                // SET NX দিয়ে race-safe transition (একটিই pod HALF_OPEN-এ যাবে)
                $ok = $this->redis->set("cb:{$this->name}:state", 'half_open', 'NX');
                if (!$ok) {
                    return $fallback ? $fallback(null) : throw new CircuitBreakerOpenException();
                }
                $state = 'half_open';
            } else {
                return $fallback ? $fallback(null) : throw new CircuitBreakerOpenException();
            }
        }

        $start = microtime(true);
        try {
            $result = $fn();
            $this->record(true);
            if ($state === 'half_open') {
                $this->redis->set($stateKey, 'closed');
                $this->redis->del("cb:{$this->name}:counters");
            }
            return $result;
        } catch (\Throwable $e) {
            $this->record(false);
            if ($state === 'half_open' || $this->shouldTrip()) $this->trip();
            if ($fallback) return $fallback($e);
            throw $e;
        }
    }

    private function record(bool $ok): void
    {
        $bucket = (int) (microtime(true) / 6); // 6s buckets
        $key = "cb:{$this->name}:bucket:{$bucket}:" . ($ok ? 'ok' : 'fail');
        $this->redis->incr($key);
        $this->redis->expire($key, $this->windowSeconds + 6);
    }

    private function shouldTrip(): bool
    {
        $now = (int) (microtime(true) / 6);
        $buckets = $this->windowSeconds / 6;
        $ok = $fail = 0;
        for ($i = 0; $i < $buckets; $i++) {
            $b = $now - $i;
            $ok   += (int) $this->redis->get("cb:{$this->name}:bucket:{$b}:ok");
            $fail += (int) $this->redis->get("cb:{$this->name}:bucket:{$b}:fail");
        }
        $total = $ok + $fail;
        if ($total < $this->minimumCalls) return false;
        return ($fail / $total) * 100 >= $this->failureThreshold;
    }

    private function trip(): void
    {
        $this->redis->set("cb:{$this->name}:state", 'open');
        $this->redis->set("cb:{$this->name}:openedAt", (string) microtime(true));
    }
}

// ─── ব্যবহার: SSLCommerz IPN call wrap ───
$breaker = new RedisCircuitBreaker($redis, 'sslcommerz-ipn');
$guzzle  = new Guzzle(['timeout' => 2.5]);

$response = $breaker->execute(
    fn() => $guzzle->post('https://sandbox.sslcommerz.com/validator/api/validationserverAPI.php', [
        'form_params' => ['val_id' => $valId, 'store_id' => $storeId, 'store_passwd' => $pass],
    ])->getBody()->getContents(),
    fallback: function ($e) use ($valId) {
        // queued retry: Laravel job
        \App\Jobs\RetrySslcommerzValidate::dispatch($valId)->delay(now()->addMinutes(2));
        return ['status' => 'PENDING', 'message' => 'Validation queued']; 
    }
);
```

> **Production tip:** PHP-FPM short-lived process — তাই in-memory state কাজ করে না। Redis-backed
> approach **must** যদি আপনার একাধিক worker / pod থাকে।

---

## 🌐 Distributed Circuit Breaker

### Per-instance vs Cluster-wide

```
  Per-Instance (default opossum, resilience4j):
  ─────────────────────────────────────────────
  pod-1 ─→ bKash (3 fail)  → pod-1 OPEN
  pod-2 ─→ bKash (0 fail)  → pod-2 CLOSED
  pod-3 ─→ bKash (5 fail)  → pod-3 OPEN
  
  ✅ Pros: state share-এ network round-trip নেই, fast
  ✅ Pros: instance-specific issue (ফাইলডেস্ক্রিপ্টর leak) detect
  ❌ Cons: কিছু pod traffic পাঠাচ্ছে — total recovery delay
  ❌ Cons: একই metric ৩ বার আলাদা গণনা

  Cluster-wide (Redis-backed):
  ────────────────────────────
  সব pod → Redis-এ shared counter → একসাথে OPEN/CLOSE
  
  ✅ Pros: faster collective trip, unified observability
  ❌ Cons: Redis dependency (latency + SPOF)
  ❌ Cons: Redis fail হলে breaker behavior unknown
  ❌ Cons: instance-specific bug (e.g., ১টি pod-এর DNS issue) hide
```

### Hybrid pattern (recommended)

```
  ┌───────────────────────────────────────────────────┐
  │  Local breaker (fast path, low latency)            │
  │   ↓ trip                                           │
  │  Cluster breaker (Redis aggregation, slower trip)  │
  │   ↓ trip                                           │
  │  Service mesh outlier detection (Envoy)            │
  └───────────────────────────────────────────────────┘
```

---

## 📊 Observability

### Must-have metrics

```
┌────────────────────────────────────────────────────────────────────┐
│ Metric Name                          │ Type      │ Use case          │
├──────────────────────────────────────┼───────────┼───────────────────┤
│ cb_state{name=...}                   │ Gauge     │ 0=closed,1=ho,2=op│
│ cb_calls_total{name,result}          │ Counter   │ rate dashboard    │
│ cb_slow_calls_total{name}            │ Counter   │ slow detection    │
│ cb_state_transitions_total{from,to}  │ Counter   │ flap detection    │
│ cb_open_duration_seconds             │ Histogram │ recovery insight  │
│ cb_call_duration_seconds_bucket      │ Histogram │ P95/P99 latency  │
└──────────────────────────────────────┴───────────┴───────────────────┘
```

### Critical alerts

```yaml
# Prometheus alert rules
- alert: CircuitBreakerStuckOpen
  expr: cb_state == 2  # OPEN
  for: 5m
  annotations:
    summary: "{{ $labels.name }} breaker OPEN > 5min — downstream still down?"

- alert: CircuitBreakerFlapping
  expr: rate(cb_state_transitions_total[5m]) > 0.1
  annotations:
    summary: "{{ $labels.name }} oscillating — threshold tuning দরকার"
```

### Distributed tracing tag

OpenTelemetry-এ span attribute:

```
  span.set_attribute("circuit_breaker.name", "bkash-charge")
  span.set_attribute("circuit_breaker.state", "OPEN")
  span.set_attribute("circuit_breaker.short_circuit", true)
```

---

## ⚠️ Anti-patterns

### ১. Open-loop reset (no half-open testing)
```
  ❌ OPEN → (timer) → CLOSED সরাসরি
  → recovery hয়েছে কিনা না জেনেই full traffic পাঠানো → আবার OPEN
```

### ২. Too aggressive thresholds
```
  ❌ failureThreshold = 10%, minimumCalls = 3
  → ১টি network blip = breaker OPEN → false positive flood
```

### ৩. Missing fallback
```
  ❌ Breaker OPEN → unhandled exception throw → user-facing 500
  ✅ Fallback: cached / default / async queue / friendly message
```

### ৪. Per-call breaker (instead of per-dependency)
```
  ❌ প্রতিটি API endpoint-এ আলাদা breaker
  ✅ প্রতিটি **downstream dependency**-এর জন্য breaker
     (bkash-charge, inventory-check, pricing-lookup — আলাদা)
```

### ৫. Retry inside breaker without idempotency
```
  ❌ POST /payment/charge → retry → idempotency key নেই → double charge
  ✅ Idempotency-Key header সব mutating call-এ
```

### ৬. Treating 4xx as failure
```
  ❌ 400 Bad Request → breaker count → user input error breaker trip করাচ্ছে
  ✅ শুধু 5xx, network error, timeout → failure
```

### ৭. Single global breaker for all dependencies
```
  ❌ একটি breaker = সব downstream → একটার failure = সবাই blocked
```

### ৮. No bulkhead
```
  bKash slow → 1000 concurrent waiter → breaker OPEN হওয়ার আগেই thread pool exhausted
  ✅ maxConcurrentCalls সেট করুন
```

---

## 🛒 BD Scenario: Daraz Checkout

ধরুন **Daraz** এর **Checkout Service** এর dependency map:

```
                    ┌──────────────────────┐
                    │  Checkout Service    │
                    └──────────┬───────────┘
                               │
        ┌──────────────┬───────┼───────────┬─────────────────┐
        ▼              ▼       ▼           ▼                 ▼
  ┌──────────┐  ┌──────────┐ ┌──────┐  ┌──────────┐  ┌───────────┐
  │Inventory │  │ Pricing  │ │ User │  │ Payment  │  │Notification│
  │ Service  │  │ Service  │ │ Svc  │  │  (bKash, │  │  (SMS)    │
  │          │  │          │ │      │  │  SSL,COD)│  │           │
  └──────────┘  └──────────┘ └──────┘  └──────────┘  └───────────┘
       CB           CB         CB          CB              CB
```

### কোন breaker কোথায় protect করে?

| Breaker | Failure-এ User Impact | Fallback Strategy |
|---|---|---|
| **Inventory CB** | Stock check fail | Last-known stock (5min stale) + post-check at order |
| **Pricing CB** | Live discount unavailable | Fall back to **listed price** (no surge) |
| **User CB** | Address fetch fail | User-এর last default address use করো |
| **bKash CB** | Online payment fail | **COD-only mode** show করো; bKash button disable |
| **SSLCommerz CB** | Card payment fail | bKash + COD only |
| **Notification CB** | SMS দিতে পারছে না | Async queue, in-app notification first |

### Flash Sale (১১.১১) সিনারিও

```
  T+0:   Sale start, traffic 50× → Pricing service overwhelmed
  T+10s: Pricing CB OPEN → checkout listed price দেখাচ্ছে (slight margin loss, but selling)
  T+30s: bKash overload → bKash CB OPEN → শুধু COD/Card option visible
  T+90s: Pricing recover → HALF_OPEN probe → CLOSED
  T+120s: bKash recover → HALF_OPEN → CLOSED
  
  Result: Checkout 100% available throughout — শুধু partial features degraded
          (without breaker → full outage হতো)
```

### Pathao + bKash combo

Pathao Foods order place → bKash payment। Pathao-র side-এ bKash adapter-এর CB:

```javascript
// bKash CB OPEN হলে:
fallback: () => ({
  paymentMethods: ['CASH_ON_DELIVERY', 'NAGAD'],  // bKash hide
  message: 'bKash সাময়িক সমস্যায়; অন্য option ব্যবহার করুন'
})
```

---

## 🤔 Decision flow

কখন Circuit Breaker, কখন শুধু Retry, কখন কিছুই না?

```
                  ┌─────────────────────────────────┐
                  │  Downstream call করছেন?         │
                  └────────────┬────────────────────┘
                               │
                          Yes  ▼
                  ┌─────────────────────────────────┐
                  │ Failure কি transient (network,  │
                  │ blip, partial outage) হতে পারে? │
                  └────────────┬────────────────────┘
                          Yes  │   No → just timeout + alert
                               ▼
                  ┌─────────────────────────────────┐
                  │ Failure rate কি 0% / কাছাকাছি   │
                  │ খুব rare?                        │
                  └────────────┬────────────────────┘
                          No   │  Yes → Retry + Timeout enough
                               ▼
                  ┌─────────────────────────────────┐
                  │ Downstream slowdown কি upstream-│
                  │ কে block / starve করতে পারে?    │
                  └────────────┬────────────────────┘
                          Yes  │  No → Retry + Bulkhead
                               ▼
                       ⚡ CIRCUIT BREAKER ⚡
                  + Timeout + Retry + Bulkhead + Fallback
```

### ব্যবহার করুন:
- ✅ External API call (bKash, SSLCommerz, Sundarban courier API)
- ✅ Inter-service RPC (Daraz: checkout → pricing)
- ✅ Database/Cache call যা throughput-critical
- ✅ Slow dependency যা thread/socket starve করতে পারে

### ব্যবহার করবেন না / প্রয়োজন নেই:
- ❌ In-process function call (no I/O)
- ❌ Idempotent local cache lookup
- ❌ Background batch job যেখানে retry-with-backoff যথেষ্ট
- ❌ Single-call CLI script (state preserve করার জায়গা নেই)
- ❌ যেখানে fallback meaningful না (e.g., real-time bidding final price)

---

## ✅ Checklist

```
Configuration:
[ ] failureRateThreshold (50%) সেট করা
[ ] minimumNumberOfCalls (≥10) সেট, false positive এড়াতে
[ ] slowCallDurationThreshold downstream P95 × 3 অনুসারে
[ ] waitDurationInOpenState realistic (30-60s)
[ ] permittedCallsInHalfOpenState ≤ 10
[ ] slidingWindowType (count vs time) চয়ন করা
[ ] 4xx ignore (recordExceptions list)

Composition:
[ ] Timeout (innermost) configured
[ ] Retry with exponential backoff + jitter
[ ] Idempotency-key on mutating retry
[ ] Bulkhead / max concurrency
[ ] Fallback function defined (cached, queued, or default)

Per-dependency:
[ ] প্রতিটি external dependency-এর জন্য আলাদা breaker
[ ] dependency নাম breaker name-এ included
[ ] dependency-specific tuning (bKash vs SSLCommerz vs SMS)

Observability:
[ ] State, calls, slow calls, transitions metric
[ ] Alert: stuck OPEN > 5min
[ ] Alert: flapping (rapid state change)
[ ] Tracing span attribute: breaker name + state
[ ] Dashboard: per-breaker state timeline

Distributed:
[ ] Multi-pod হলে: Redis-backed বা mesh-level outlier detection
[ ] Redis dependency-এর fallback (Redis down → fail-open বা fail-closed?)

Testing:
[ ] Chaos test: downstream forced down → breaker OPEN verify
[ ] Recovery test: downstream up → HALF_OPEN → CLOSED transition
[ ] Load test: breaker state থাকা অবস্থায় latency P99 measure

Documentation:
[ ] Runbook: each breaker OPEN হলে কী করতে হবে
[ ] On-call alert routing: breaker OPEN → who's paged
```

---

## 📚 আরও পড়ুন

- [Fault Tolerance Patterns](../03-system-design/fault-tolerance.md) — overall resilience strategy
- [Backpressure](./backpressure.md) — slow consumer protection
- [Load Shedding](../03-system-design/load-shedding.md) — system-wide overload
- [Rate Limiting](../03-system-design/rate-limiting.md) — per-client throttling
- [Thread Pools](./thread-pools.md) — bulkhead resource isolation
- [Async/Await](./async-await.md) — concurrency primitives

> **চূড়ান্ত কথা:** Circuit Breaker একটি single defense layer — silver bullet নয়। Timeout, retry, bulkhead,
> fallback, observability — সবকিছু একসাথে integrate করতে হবে। Production-এ unverified breaker = breaker
> না থাকার সমান।
