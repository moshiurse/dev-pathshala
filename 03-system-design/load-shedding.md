# 🚦 লোড শেডিং (Load Shedding) — Overload-এ Graceful Degradation

> **"যখন তুফান আসে, পুরো জাহাজ ডুবে যাওয়ার চেয়ে কিছু পণ্য সমুদ্রে ফেলে দেওয়াই বুদ্ধিমানের কাজ।"**

প্রতিটি সিস্টেমের একটি সীমা আছে। ১১.১১ Daraz Big Sale-এ ১০ লাখ user একসাথে clicking করছে, অথবা বৃষ্টিতে
Pathao surge — কোনো একটা সময়ে চাহিদা capacity-কে ছাড়িয়ে যায়। তখন দু'টোই বিকল্প:

- 🟥 **Crash and burn** — সব request slow হোক, eventually সব fail
- 🟩 **Load shed** — কিছু request quickly reject করে, বাকিদের ভালো সেবা দাও

দ্বিতীয়টাই **load shedding** — সচেতন, পরিমাপযোগ্য degradation।

এই ডকুমেন্টে আমরা signals, strategies, implementation patterns (Express middleware, Laravel, Envoy),
adaptive concurrency (Netflix Vegas/Gradient), priority shedding এবং Pathao/Daraz/bKash-এর প্রকৃত
সিনারিও দেখব।

---

## 📑 সূচিপত্র

1. [Definition: Graceful Degradation](#-definition-graceful-degradation)
2. [Rate Limiting / Backpressure এর সাথে পার্থক্য](#-rate-limiting--backpressure-এর-সাথে-পার্থক্য)
3. [Signals — কীভাবে বুঝবেন আপনি overloaded](#-signals--কীভাবে-বুঝবেন-আপনি-overloaded)
4. [Strategies](#-strategies)
5. [Adaptive Concurrency (Netflix Vegas/Gradient, AIMD)](#-adaptive-concurrency-netflix-vegasgradient-aimd)
6. [কোথায় Implement করবেন](#-কোথায়-implement-করবেন)
7. [Implementation Patterns](#-implementation-patterns)
8. [Node.js: Express + event-loop lag](#-nodejs-express--event-loop-lag)
9. [PHP/Laravel: Redis-tracked latency](#-phplaravel-redis-tracked-latency)
10. [Envoy / Istio config](#-envoy--istio-config)
11. [BD Real-world](#-bd-real-world)
12. [Observability](#-observability)
13. [Anti-patterns](#️-anti-patterns)
14. [Checklist](#-checklist)

---

## 📌 Definition: Graceful Degradation

**Load Shedding** = system overload-এ থাকাকালীন ইচ্ছাকৃতভাবে কিছু কাজ refuse করা যাতে অবশিষ্ট কাজ
সফলভাবে complete হয়।

মূল principle:
> "Better to do **less** work **well**, than to do **all** work **poorly**."

### Crash vs Shed

```
  Capacity = 1000 RPS, Demand = 1500 RPS

  ❌ No shedding:
  ─────────────────────────────────────────────────────
  সব ১৫০০ request accept → প্রত্যেকের latency 5× → 
  timeout cascade → eventual 0 RPS success → 100% fail

  ✅ With shedding:
  ─────────────────────────────────────────────────────
  ১০০০ accept (P99 < SLA), ৫০০ instant 503 + Retry-After
  → 67% user happy, 33% retry करে 5s পরে success
  → overall success rate 95%+
```

### Goodput vs Throughput

```
  Throughput = সব request count (success + fail)
  Goodput   = শুধু success request count
  
   ──────────────────────────────────────────────
   Demand →  ১X    ২X    ৪X    ৮X    ১৬X
   ──────────────────────────────────────────────
   No shed →  100  150   180   100   20    (collapse)
   Shed   →   100  150   180   200   200   (steady)
   ──────────────────────────────────────────────
```

Load shedding = goodput maximize, even at the cost of throughput।

---

## 🆚 Rate Limiting / Backpressure এর সাথে পার্থক্য

```
┌──────────────────┬─────────────────────────────────────────────────────┐
│ Pattern          │ Trigger                                              │
├──────────────────┼─────────────────────────────────────────────────────┤
│ Rate Limiting    │ Per-client quota (100 req/min/user)                 │
│                  │ Goal: fairness, abuse prevention                    │
│                  │ Decision: client identity-based                     │
│                  │ Static config                                       │
│                  │                                                     │
│ Backpressure     │ Consumer slower than producer (queue grows)         │
│                  │ Goal: prevent OOM, latency spike                    │
│                  │ Direction: downstream → upstream signal             │
│                  │ Adaptive (instant capacity)                         │
│                  │                                                     │
│ Load Shedding    │ System health degraded (CPU/latency/queue)          │
│                  │ Goal: protect overall service                       │
│                  │ Decision: system-wide, priority-aware               │
│                  │ Adaptive + threshold based                          │
│                  │                                                     │
│ Circuit Breaker  │ Downstream dependency unhealthy                     │
│                  │ Goal: prevent cascading failure                     │
│                  │ Decision: per-dependency state                      │
│                  │ Reactive                                            │
└──────────────────┴─────────────────────────────────────────────────────┘
```

### Visual

```
   ┌──────────┐ rate ┌──────────┐ shed ┌──────────┐ break ┌──────────┐
   │ Client   │─────▶│ Gateway  │─────▶│ Service  │──────▶│Dependency│
   │          │      │ (per-key)│      │(self CPU)│       │ (bKash)  │
   └──────────┘      └──────────┘      └──────────┘       └──────────┘
        ▲                                    ▲
        │                                    │
        │backpressure (queue grow → reject)  │
        └────────────────────────────────────┘
```

আরও বিস্তারিত:
- [Rate Limiting](./rate-limiting.md) — per-client quota
- [Backpressure](../12-concurrency/backpressure.md) — flow control
- [Circuit Breaker](../12-concurrency/circuit-breaker.md) — dependency-level

---

## 📡 Signals — কীভাবে বুঝবেন আপনি overloaded

Load shedding-এর সবচেয়ে বড় চ্যালেঞ্জ — **সঠিক signal চয়ন**। ভুল signal = unnecessary shed (user
impact) বা delayed shed (collapse)।

### Signal Hierarchy

```
   Layer            Signal                       Latency to detect
   ──────────────────────────────────────────────────────────────
   Application →    Queue depth                  সবচেয়ে দ্রুত
                    Inflight request count       
                    Event-loop lag (Node)        
                    Worker pool saturation       
   ──────────────────────────────────────────────────────────────
   SLA →            Latency P95/P99              মাঝারি
                    Error rate                   
   ──────────────────────────────────────────────────────────────
   System →         CPU utilization              ধীর
                    Memory pressure              
                    GC pause %                   
                    File descriptors             
   ──────────────────────────────────────────────────────────────
   Network →        Connection count             দেরিতে
                    TCP retransmits              
```

> **Best practice:** Application-layer signal-এ shed করুন — system signal-এ পৌঁছানোর আগে। CPU 100%
> মানে আপনি ইতিমধ্যে SLA breach করছেন।

### Specific signals

#### Queue depth
```javascript
if (jobQueue.size > MAX_QUEUE) shed();
```
সবচেয়ে clean signal — backpressure-এর direct expression।

#### Latency P99
```javascript
const p99 = histogram.percentile(0.99);
if (p99 > 2 * SLA) shed();
```

#### Event-loop lag (Node.js specific)
```javascript
let lag = 0;
let last = process.hrtime.bigint();
setInterval(() => {
  const now = process.hrtime.bigint();
  lag = Number(now - last) / 1e6 - 100;  // 100ms expected; lag = excess
  last = now;
}, 100);
// lag > 50ms = event loop saturated
```

#### Concurrent inflight
```javascript
let inflight = 0;
app.use((req, res, next) => {
  if (inflight >= MAX_INFLIGHT) return res.status(503).send('Try later');
  inflight++;
  res.on('finish', () => inflight--);
  next();
});
```

---

## 🎯 Strategies

### ১. Reject Early (Fail Fast)
```
   ✅ Request arrive → 1ms-এ 503 → consume কিছু না
   ❌ Request arrive → DB call → 5s wait → 503
   
   Fail fast = কম resource consume = বেশি capacity বাকিদের জন্য
```

### ২. Priority-based shedding

প্রতিটা request-এর সমান গুরুত্ব নেই। **Critical path-কে protect** করুন।

```
   bKash example priority tiers:
   ─────────────────────────────────────────────────
   Tier 0 (must): money transfer, payment confirm
   Tier 1 (should): balance check, login
   Tier 2 (nice): transaction history, analytics
   Tier 3 (optional): promo offer fetch

   80% capacity: shed Tier 3
   90% capacity: shed Tier 3 + 2
   95% capacity: shed Tier 3 + 2 + 1, keep only Tier 0
```

```javascript
function shed(req) {
  const tier = req.tier ?? getTierFromRoute(req.path);
  const load = currentLoadPercent();
  if (load > 95 && tier > 0) return true;
  if (load > 90 && tier > 1) return true;
  if (load > 80 && tier > 2) return true;
  return false;
}
```

### ৩. Cooperative shedding (Retry-After)

Client-কে polite signal দিন:

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{
  "error": "ServiceOverloaded",
  "retryAfter": 30,
  "message": "আমরা একটু ব্যস্ত। ৩০ সেকেন্ড পরে চেষ্টা করুন।"
}
```

Mobile app SDK এই header দেখে exponential backoff প্রয়োগ করতে পারে।

### ৪. Degraded mode (cached / stale)

Daraz Flash Sale-এ:

```
   Normal mode:        Live price + stock + recommendations + reviews
   80% capacity:       cached recommendations (5-min stale)
   90% capacity:       no recommendations + cached reviews
   95% capacity:       only product + price + stock
   99% capacity:       only product + listed price (no live discount)
```

### ৫. LIFO / Drop oldest under overload

ক্লাসিক **FIFO queue + slow consumer** = oldest request প্রায় expired। User waited 30s → eventually
timeout। Better: drop the old, serve the new (which client is still waiting for)।

```
   FIFO under overload:
   t=0s submitted ───────────► t=30s served (user already gone)
   
   LIFO under overload:
   t=29s submitted ──► t=30s served (user still waiting)
```

> **Caution:** LIFO শুধু overload-এ। Normal mode-এ FIFO fairness রাখুন।

### ৬. Tail-drop vs Random-drop vs ECN-marking

Network congestion control থেকে borrowed:

| Strategy | কীভাবে | Use case |
|---|---|---|
| **Tail-drop** | Queue full → newest reject | Simple, default |
| **RED** (Random Early Detection) | Probabilistic drop before full | Smoother, no synchronization |
| **ECN** | Mark packet, let sender slow | Best, but requires cooperation |

App layer parallel:
- Tail-drop = 503 returned
- RED = probabilistic 503 (e.g., 10% of requests at 80% capacity, 50% at 90%)
- ECN-like = `X-Slow-Down: true` header → SDK reduces rate

---

## 🧠 Adaptive Concurrency (Netflix Vegas/Gradient, AIMD)

Static threshold (e.g., "if inflight > 1000, shed") fragile — capacity changes (GC, neighbor pod, DB).
**Adaptive concurrency** real-time capacity discover করে।

### AIMD (Additive Increase, Multiplicative Decrease)

TCP congestion control থেকে borrowed:

```
   Concurrent limit (L)
        ▲
        │      ╱╲          ╱╲
        │     ╱  ╲        ╱  ╲
        │    ╱    ╲      ╱    ╲
        │   ╱      ╲    ╱      ╲
        │  ╱        ╲╱╲╱        ╲╱
        └────────────────────────────► time
        
   Success → L = L + 1 (additive increase)
   Latency degrade → L = L × 0.5 (multiplicative decrease)
```

### Netflix Concurrency Limits — Vegas/Gradient

**Gradient algorithm:**
```
   RTT_min = historical minimum
   RTT_actual = current RTT
   gradient = RTT_min / RTT_actual    (0 < g ≤ 1)

   newLimit = currentLimit × gradient + queueSize
   
   gradient = 1.0  → no congestion → grow
   gradient = 0.5  → 2× latency → shrink
```

**Java example (resilience4j-bulkhead-like):**
```java
AdaptiveLimit limit = AimdLimit.newBuilder()
    .initialLimit(20)
    .minLimit(5)
    .maxLimit(200)
    .increaseBy(1)
    .decreaseRatio(0.9)
    .build();

if (limit.tryAcquire()) {
    try {
        long start = System.nanoTime();
        // process
        limit.recordSuccess(System.nanoTime() - start);
    } catch (Exception e) {
        limit.recordFailure();
    } finally {
        limit.release();
    }
} else {
    return shed();
}
```

---

## 📍 কোথায় Implement করবেন

```
                Multi-layer Shedding
   ╔══════════════════════════════════════════════════════╗
   ║                                                      ║
   ║   1️⃣  CDN / Edge        — DDoS, geo-rate limit       ║
   ║              │                                        ║
   ║   2️⃣  API Gateway       — auth, per-key rate         ║
   ║              │             priority routing           ║
   ║   3️⃣  Service Mesh      — outlier detection,         ║
   ║              │             connection pool limit      ║
   ║   4️⃣  Application       — adaptive concurrency,      ║
   ║              │             event-loop lag, priority   ║
   ║   5️⃣  Queue Consumer    — visibility timeout,        ║
   ║                            DLQ, lag autoscale         ║
   ║                                                      ║
   ╚══════════════════════════════════════════════════════╝
```

কোন layer-এ কী shed?

```
   Edge:       bot, scraper, geo-block
   Gateway:    abusive client, unauthenticated burst
   Mesh:       failed instance eject, connection cap
   App:        priority-based, latency-driven
   Consumer:   poison message, slow-DB-bound jobs
```

আরও: [API Gateway](../06-api-design/api-gateway.md), [Service Mesh](../10-devops/service-mesh.md)

---

## 🧰 Implementation Patterns

### Pattern ১: Token bucket on shared resource budget

```javascript
// 1000 RPS overall budget
class GlobalBudget {
  constructor(rps) { this.rps = rps; this.tokens = rps; setInterval(() => this.refill(), 1000); }
  refill() { this.tokens = this.rps; }
  tryAcquire(weight = 1) {
    if (this.tokens >= weight) { this.tokens -= weight; return true; }
    return false;
  }
}
const budget = new GlobalBudget(1000);

app.use((req, res, next) => {
  const weight = req.path.startsWith('/search') ? 5 : 1;  // search expensive
  if (!budget.tryAcquire(weight)) {
    res.set('Retry-After', '1');
    return res.status(503).json({ error: 'Overloaded' });
  }
  next();
});
```

### Pattern ২: Concurrency semaphore + reject overflow

```javascript
class Semaphore {
  constructor(n) { this.permits = n; this.queue = []; }
  async acquire(timeoutMs = 0) {
    if (this.permits > 0) { this.permits--; return; }
    if (timeoutMs === 0) throw new Error('SHED');
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        this.queue = this.queue.filter(x => x !== entry);
        reject(new Error('SHED_TIMEOUT'));
      }, timeoutMs);
      const entry = { resolve, timer };
      this.queue.push(entry);
    });
  }
  release() {
    const next = this.queue.shift();
    if (next) { clearTimeout(next.timer); next.resolve(); }
    else this.permits++;
  }
}
```

### Pattern ৩: LIFO under overload

```javascript
class LIFOQueue {
  constructor(max, overloadThreshold) {
    this.queue = []; this.max = max; this.threshold = overloadThreshold;
  }
  push(item) {
    if (this.queue.length >= this.max) return false;
    this.queue.push(item);
    return true;
  }
  pop() {
    // overload হলে LIFO, normal হলে FIFO
    return this.queue.length > this.threshold
      ? this.queue.pop()
      : this.queue.shift();
  }
}
```

---

## 🟢 Node.js: Express + event-loop lag

```javascript
// shedding-middleware.js
const toobusy = require('toobusy-js');
toobusy.maxLag(70);            // 70ms event-loop lag threshold
toobusy.interval(500);

const Histogram = require('hdr-histogram-js');
const latencyHist = Histogram.build();

const PRIORITY = {
  '/api/payment': 0,    // critical
  '/api/checkout': 0,
  '/api/profile': 1,
  '/api/cart': 1,
  '/api/recommendations': 2,
  '/api/banner': 3,
  '/api/analytics': 3,
};

function getPriority(path) {
  for (const prefix of Object.keys(PRIORITY)) {
    if (path.startsWith(prefix)) return PRIORITY[prefix];
  }
  return 2;
}

function loadFactor() {
  // 0.0 (idle) — 1.0 (overloaded)
  const lag = toobusy.lag();
  const lagFactor = Math.min(lag / 200, 1);
  const p99 = latencyHist.getValueAtPercentile(99);
  const latencyFactor = Math.min(p99 / 2000, 1);
  return Math.max(lagFactor, latencyFactor);
}

let inflight = 0;
const MAX_INFLIGHT = 500;

module.exports = function sheddingMiddleware(req, res, next) {
  const tier = getPriority(req.path);
  const load = loadFactor();

  // Hard cap
  if (inflight >= MAX_INFLIGHT) {
    return shed(res, 'inflight_cap', 2);
  }

  // Priority-aware shed
  if (load > 0.95 && tier > 0) return shed(res, 'critical_only', 60);
  if (load > 0.85 && tier > 1) return shed(res, 'high_load', 30);
  if (load > 0.70 && tier > 2) return shed(res, 'moderate_load', 15);

  // Probabilistic shed (RED-like)
  if (load > 0.50 && tier > 2) {
    const dropProb = (load - 0.5) * 2;     // 0 → 1 over 50%-100%
    if (Math.random() < dropProb) return shed(res, 'red_drop', 5);
  }

  inflight++;
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    inflight--;
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    latencyHist.recordValue(Math.min(ms, 60_000));
  });
  next();
};

function shed(res, reason, retryAfterSec) {
  metric('shed.total', 1, { reason });
  res.set('Retry-After', String(retryAfterSec));
  res.status(503).json({
    error: 'ServiceOverloaded',
    reason,
    retryAfter: retryAfterSec,
  });
}

function metric(name, val, tags) { /* StatsD/Prometheus */ }
```

### ব্যবহার

```javascript
const express = require('express');
const sheddingMiddleware = require('./shedding-middleware');

const app = express();
app.use(sheddingMiddleware);

app.get('/api/payment/status', /* ... */);  // tier 0 — কখনই shed না (যদি না 95%+)
app.get('/api/recommendations', /* ... */); // tier 2 — early shed
```

---

## 🐘 PHP/Laravel: Redis-tracked latency

PHP-FPM-এ in-process state share হয় না, তাই Redis-এ centralized signal রাখি।

```php
<?php
// app/Http/Middleware/LoadShedding.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class LoadShedding
{
    private const PRIORITY = [
        'api/payment'         => 0,
        'api/checkout'        => 0,
        'api/profile'         => 1,
        'api/cart'            => 1,
        'api/recommendations' => 2,
        'api/banner'          => 3,
    ];

    public function handle(Request $request, Closure $next)
    {
        $tier = $this->priorityFor($request->path());
        $load = $this->loadFactor();

        if ($load >= 0.95 && $tier > 0) return $this->shed(60, 'critical_only');
        if ($load >= 0.85 && $tier > 1) return $this->shed(30, 'high_load');
        if ($load >= 0.70 && $tier > 2) return $this->shed(15, 'moderate_load');

        $start = microtime(true);
        $response = $next($request);
        $this->recordLatency((microtime(true) - $start) * 1000);
        return $response;
    }

    private function priorityFor(string $path): int
    {
        foreach (self::PRIORITY as $prefix => $tier) {
            if (str_starts_with($path, $prefix)) return $tier;
        }
        return 2;
    }

    private function loadFactor(): float
    {
        // Redis sorted set-এ last 60s latency samples; p99 calculate
        $samples = Redis::zrange('app:latency:samples', 0, -1);
        if (count($samples) < 50) return 0.0;
        sort($samples);
        $p99 = (float) $samples[(int) floor(count($samples) * 0.99)];
        $cpu = (float) (Redis::get('app:cpu:percent') ?? 0); // node-exporter scrape
        $latencyFactor = min($p99 / 2000.0, 1.0);
        $cpuFactor = min($cpu / 90.0, 1.0);
        return max($latencyFactor, $cpuFactor);
    }

    private function recordLatency(float $ms): void
    {
        $now = microtime(true);
        Redis::pipeline(function ($pipe) use ($ms, $now) {
            $pipe->zadd('app:latency:samples', $now, $ms . ':' . bin2hex(random_bytes(4)));
            $pipe->zremrangebyscore('app:latency:samples', '-inf', $now - 60);
        });
    }

    private function shed(int $retryAfter, string $reason)
    {
        \Log::info('load_shed', ['reason' => $reason]);
        Redis::incr("metrics:shed:{$reason}");
        return response()->json([
            'error' => 'ServiceOverloaded',
            'reason' => $reason,
            'retryAfter' => $retryAfter,
        ], 503)->header('Retry-After', (string) $retryAfter);
    }
}
```

### Register

```php
// app/Http/Kernel.php
protected $middleware = [
    \App\Http\Middleware\LoadShedding::class,
    // ...
];
```

---

## 🌐 Envoy / Istio config

Service mesh-এ load shedding-এর দু'টো mechanism:

### ১. Connection pool limits (circuit-breaker style)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: bkash-payment
spec:
  host: bkash-payment.payments.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100        # max TCP connection
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

connection pool limit hit হলে Envoy **immediately 503** — sidecar-level shedding।

### ২. Local rate limit (Envoy)

```yaml
http_filters:
- name: envoy.filters.http.local_ratelimit
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
    stat_prefix: http_local_rate_limiter
    token_bucket:
      max_tokens: 1000
      tokens_per_fill: 1000
      fill_interval: 1s
    filter_enabled:
      runtime_key: local_rate_limit_enabled
      default_value: { numerator: 100, denominator: HUNDRED }
    response_headers_to_add:
      - append: false
        header: { key: x-rate-limit, value: "true" }
```

---

## 🇧🇩 BD Real-world

### Pathao surge during rains

বৃষ্টি = ride demand 5×। Driver supply সীমিত। Pathao app-এ:

```
   Tier 0 (must keep):
     - Active ride status update
     - Driver-rider chat
     - Payment confirm
   Tier 1 (degraded):
     - New ride matching (slow but accept)
   Tier 2 (deferred):
     - Rate driver UI
     - Past-trip details
   Tier 3 (shed):
     - Promo banner load
     - Profile picture upload
     - "Refer a friend" page
```

```javascript
// Pathao API gateway
if (load > 0.85 && req.path === '/api/promotions') {
  return res.status(503).set('Retry-After', '60').json({
    error: 'PeakHour',
    message: 'এখন ব্যস্ত সময়; ১ মিনিট পরে চেষ্টা করুন'
  });
}
```

### Daraz Big Sale — image upload shed

Big Sale চলাকালীন seller image upload endpoint resource-heavy। Capacity reach 80% → image-upload
endpoint **fully shed** (admin notice: "১০ মিনিট পরে আপলোড করুন")। Customer-facing checkout
সম্পূর্ণ চালু থাকে।

```nginx
# nginx limit_req for /api/seller/upload
limit_req_zone $binary_remote_addr zone=upload:10m rate=2r/m;

location /api/seller/upload {
  limit_req zone=upload burst=5 nodelay;
  # additional Lua: check Redis flag flash_sale_active
  access_by_lua_block {
    local redis = require "resty.redis"
    local r = redis:new()
    r:connect("redis", 6379)
    if r:get("flash_sale_mode") == "1" then
        ngx.status = 503
        ngx.header["Retry-After"] = "600"
        ngx.say('{"error":"FlashSaleMode","message":"Flash Sale চলাকালীন upload বন্ধ"}')
        ngx.exit(503)
    end
  }
}
```

### bKash partial outage — payment-only mode

DB primary lag → some service degraded। bKash backend:

```
   Health signal: DB replication lag > 5s
   ─────────────────────────────────────
   Mode: PAYMENT_ONLY
   ✅ Authenticated send-money, payment confirm
   ❌ Transaction history (read-replica overloaded)
   ❌ Statement download
   ❌ Add-money via card (needs DB write)
   
   App-এ user-facing message: 
   "আমরা সীমিত মোডে চলছি; শুধু পেমেন্ট ও সেন্ড মানি কাজ করছে"
```

---

## 📊 Observability

### Metrics

```
   shed_total{reason, tier}             counter
   shed_rate                            ratio (shed / total)
   load_factor                          gauge (0-1)
   adaptive_concurrency_limit           gauge
   inflight_requests                    gauge
   event_loop_lag_ms                    histogram
   latency_p99_ms                       gauge
```

### Critical alerts

```yaml
- alert: ShedRateHigh
  expr: rate(shed_total[5m]) / rate(http_requests_total[5m]) > 0.05
  for: 5m
  annotations:
    summary: "{{ $labels.service }} shedding > 5% for 5m"

- alert: ShedRateCritical
  expr: rate(shed_total[5m]) / rate(http_requests_total[5m]) > 0.20
  for: 2m
  annotations:
    severity: page

- alert: ShedTier0    # critical work being shed!
  expr: rate(shed_total{tier="0"}[1m]) > 0
  annotations:
    severity: page
    summary: "Tier-0 shedding — capacity emergency"
```

### Dashboard panels

1. **Shed rate over time** (stacked by reason)
2. **Load factor + threshold lines**
3. **Latency P50/P95/P99 vs shed rate**
4. **Goodput vs incoming RPS** (the famous overload curve)
5. **Per-endpoint shed breakdown**

---

## ⚠️ Anti-patterns

### ১. Silent dropping
```
   ❌ Just `return;` (no response, no log)
   ✅ 503 + Retry-After + log + metric
```

### ২. Shedding critical paths first
```
   ❌ Shed by random — bKash payment shed হয়ে গেল
   ✅ Priority tier explicit
```

### ৩. No client-side cooperation
```
   ❌ Server 503, client immediately retry → infinite loop
   ✅ Retry-After honored, exponential backoff
```

### ৪. Static threshold
```
   ❌ "if CPU > 80% shed" — কিন্তু DB issue-এ CPU 30%, latency 10×
   ✅ Multi-signal (latency, queue, CPU, GC)
```

### ৫. Shed at the wrong layer
```
   ❌ Application shed কিন্তু LB যথারীতি route করছে → unbalanced
   ✅ LB/mesh-level outlier detection + app-level priority
```

### ৬. Single-tenant view
```
   ❌ Premium customer-ও free user-এর সাথে একসাথে shed হলো
   ✅ Tenant-tier-aware (premium = lower shed prob)
```

### ৭. No goodput measurement
```
   ❌ "throughput up" বলে happy → আসলে error rate ৫০%
   ✅ Goodput = success/total monitor
```

### ৮. Test না করা
```
   "Shed code আছে" — কিন্তু production-এ trigger কখনো হয়নি
   ✅ Chaos test: artificial load → verify shed activated
```

---

## ✅ Checklist

```
Strategy:
[ ] Priority tier defined (T0 critical → T3 optional)
[ ] Per-tier shed threshold documented
[ ] Retry-After header included
[ ] Cooperative client SDK (exponential backoff)
[ ] Degraded mode plan (cached / stale / disabled features)

Signals:
[ ] Multi-signal load factor (latency + CPU + queue + lag)
[ ] Application-level signal preferred over system-level
[ ] Adaptive concurrency considered (Vegas/AIMD)

Implementation:
[ ] Fail-fast (shed-decision <5ms)
[ ] Hard inflight cap as safety net
[ ] LIFO during overload (consider)
[ ] RED probabilistic drop in mid-load

Layering:
[ ] Edge: bot/geo
[ ] Gateway: per-key rate
[ ] Mesh: outlier detection + connection pool
[ ] App: priority + adaptive
[ ] Consumer: poison + DLQ

Observability:
[ ] shed_total counter (by reason, tier)
[ ] load_factor gauge
[ ] goodput vs throughput dashboard
[ ] Alert: shed > 5% (warn), > 20% (page), Tier-0 shed (immediate)

Testing:
[ ] Load test: artificial overload → verify shed triggers
[ ] Recovery test: post-overload latency back to baseline
[ ] Game day: simulate downstream slowness
[ ] Client SDK: respects Retry-After

Documentation:
[ ] Runbook: shed-rate-high → কী check করতে হবে
[ ] User-facing message: clear, polite, action-oriented (Bangla!)
[ ] Capacity plan: known shed-trigger thresholds documented
```

---

## 📚 আরও পড়ুন

- [Rate Limiting](./rate-limiting.md)
- [Backpressure](../12-concurrency/backpressure.md)
- [Circuit Breaker](../12-concurrency/circuit-breaker.md)
- [Fault Tolerance](./fault-tolerance.md)
- [API Gateway](../06-api-design/api-gateway.md)
- [Service Mesh](../10-devops/service-mesh.md)
- [Observability](../16-observability/README.md)

> **চূড়ান্ত কথা:** Load shedding = humility। আপনার system-এর সীমা মেনে নেওয়া এবং সেই সীমার মধ্যে
> সেরাটা দেওয়া। প্রতিদিন নয়, কিন্তু flash sale, viral moment, কিংবা downstream outage-এ — load
> shedding-ই আপনার সিস্টেমকে সম্পূর্ণ ভেঙে পড়া থেকে বাঁচাবে।
