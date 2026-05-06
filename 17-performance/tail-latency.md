# ⏱️ Tail Latency & P99 Optimization

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🧠 সংজ্ঞা ও ধারণা

### কেন Average মিথ্যা বলে?

**Average (Mean):** সব মান যোগ করে ভাগ। কিন্তু এটি distribution-এর shape বোঝায় না।

```
Service A: [100, 100, 100, 100, 100]ms → Average = 100ms
Service B: [10, 10, 10, 10, 460]ms     → Average = 100ms

দুটোর average একই! কিন্তু Service B-র একজন user 460ms অপেক্ষা করছে!
```

### Percentile কী?

**Percentile (p):** ডেটার কত শতাংশ একটি নির্দিষ্ট মানের নিচে পড়ে।

| Percentile | মানে | বাংলায় |
|-----------|------|---------|
| **p50 (Median)** | 50% request এর নিচে | "সাধারণ" user experience |
| **p95** | 95% request এর নিচে | প্রতি ২০ জনে ১ জন এর চেয়ে খারাপ |
| **p99** | 99% request এর নিচে | প্রতি ১০০ জনে ১ জন এর চেয়ে খারাপ |
| **p99.9** | 99.9% request এর নিচে | প্রতি ১০০০ জনে ১ জন |

### 🐍 Tail Latency কী?

Tail Latency = distribution-এর **ডানদিকের লেজ** — সবচেয়ে slow requests।

> p99 = 2 সেকেন্ড মানে: প্রতি ১০০টি request-এ ১টি ২+ সেকেন্ড সময় নেয়

**কেন এটা গুরুত্বপূর্ণ?**

1. **High-traffic services:** প্রতিদিন ১ কোটি request হলে p99 = 1% = **১ লাখ** slow request!
2. **Power users affected:** বড় customer-রা বেশি request করে → তাদের tail hit-এর সম্ভাবনা বেশি
3. **Fan-out amplification:** Microservices-এ একটি request ১০টি service call করলে, যেকোনো একটি slow হলেই সব slow

### 🌊 Tail Latency-র কারণ

#### 1. Garbage Collection (GC)
```
Java/Go/C# → GC pause হলে সব request waiting
Stop-the-world GC: 50-200ms pause → p99 spike
```

#### 2. Context Switching
```
OS thread schedule করতে না পারলে → CPU wait
Thread pool exhausted → queuing delay
```

#### 3. Resource Contention
```
Database lock → একটি slow query সবাইকে block করে
Shared cache eviction → cold cache request slow
Network congestion → packet retransmission
```

#### 4. Queuing Delay (Little's Law)
```
Queue length = arrival_rate × wait_time
Server 80% loaded → queue start building
Server 95% loaded → queue exponentially grows!
```

#### 5. Background Tasks
```
Log rotation, Cron jobs, Backup, Index rebuild
এগুলো IO/CPU compete করে → spike তৈরি করে
```

### 🎯 Fan-Out Effect (Latency Amplification)

যখন একটি request অনেক parallel service call করে:

```
User request → Service A → [B, C, D, E, F] parallel calls
                           (5টি downstream call)

প্রতিটি service-এর p99 = 50ms

কিন্তু 5টি parallel call-এর ক্ষেত্রে:
P(অন্তত একটি slow) = 1 - (0.99)^5 = 4.9%

মানে: ~5% request-এ কোনো না কোনো service slow হবে!
আপনার overall p95 ≈ individual service-র p99!
```

**10টি parallel call হলে:** 1 - (0.99)^10 = 9.6% slow!

### 🛡️ সমাধান কৌশল

#### 1. Hedged Requests (বিকল্প অনুরোধ)
একই request দুটি server-এ পাঠানো — যে আগে answer দেয় সেটা নেওয়া।

```
Client → Server A (primary)
      → Server B (hedge, 5ms delay-এর পর)
      ← যে আগে reply করে = winner
```

#### 2. Tied Requests
দুই server-কে বলা: "আমি তোমাকে আর অন্যকে request দিয়েছি। তুমি শুরু করলে অন্যকে cancel করো।"

#### 3. Adaptive Timeouts
Fixed timeout-এর বদলে recent latency distribution দেখে dynamic timeout সেট।

```
Static: timeout = 500ms (সবসময়)
Adaptive: timeout = p99 * 1.5 = dynamic (300ms → 750ms)
```

#### 4. Work Stealing
Idle worker thread অন্য busy worker-এর queue থেকে কাজ নিয়ে নেয়।

#### 5. Request Prioritization
Critical requests (payment) আগে process, non-critical (analytics) পরে।

### 📊 Latency Budget

Microservices-এ total latency budget ভাগ করা:

```
Total SLO: 500ms p99

API Gateway:    50ms  (10%)
Auth Service:   30ms  (6%)
Business Logic: 200ms (40%)
Database:       150ms (30%)
Serialization:  50ms  (10%)
Network:        20ms  (4%)
─────────────────────────────
Total:          500ms (100%)
```

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏍️ Pathao Search — Average vs P99 Problem

**সমস্যা:** Pathao-র ride search feature-এ:
- Dashboard বলছে: "Average response time 200ms" ✅
- User complain করছে: "Search এ অনেক সময় লাগে!" ❌

**Investigation:**

```
Latency Distribution:
  p50:  120ms  (অর্ধেক user-এর experience)
  p90:  350ms  (গ্রহণযোগ্য)
  p95:  800ms  (একটু slow)
  p99:  2100ms (VERY slow!)
  p99.9: 5800ms (অগ্রহণযোগ্য!)

প্রতিদিন 5 লাখ search request:
  p99 affected users: 5,000 users/day = 2.1+ সেকেন্ড wait!
  p99.9 affected: 500 users/day = 5.8+ সেকেন্ড wait!
```

**Root Cause Analysis:**

```
Pathao Search Architecture:
User → API GW → Search Service → [
  Driver Location (Redis Geo),
  ETA Calculation (Map API),
  Surge Pricing (ML Model),
  Promo Check (Promotion DB)
]

Fan-out = 4 parallel calls
Individual p99:
  Redis Geo: 30ms
  Map API: 500ms (external, sometimes 2s!)  ← 🔴
  ML Model: 200ms (GC pause → 1s)           ← 🔴
  Promo DB: 50ms

Worst case: Map API timeout + ML GC = 2s+ at p99
```

**সমাধান:**

1. **Map API Hedged Request:**
   - Primary: Google Maps
   - Hedge (after 300ms): Cached approximation
   - Result: p99 500ms → 350ms

2. **ML Model GC Tuning:**
   - G1GC → ZGC (low-pause)
   - Result: GC pause 200ms → 5ms

3. **Adaptive Timeout:**
   - Fixed 2s timeout → Adaptive p99×1.5 timeout
   - Fail fast, return partial results

4. **Result:**
   ```
   Before: p99 = 2100ms, p99.9 = 5800ms
   After:  p99 = 450ms,  p99.9 = 800ms
   ```

### 🏦 bKash — Peak Hour Latency Spike

**পরিস্থিতি:** প্রতিদিন সন্ধ্যা ৬-৮টায় (salary transfer time):
- p50 স্বাভাবিক (80ms)
- p99 আকাশচুম্বী (3s+)

**কারণ:** Database connection pool exhausted → queuing
**সমাধান:** Connection pool size increase + read replica + query optimization

---

## 📊 ASCII ডায়াগ্রাম

### Latency Distribution — Average Hides The Truth

```
        Response Time Distribution

Count
  │
  │
80│   ██
  │   ██
60│   ████
  │   ████
40│   ██████
  │   ████████
20│   ██████████
  │   ████████████░░░░░░░░░░░░░░░▒▒▒▒▒▓▓▓
  │   ████████████░░░░░░░░░░░░░░░▒▒▒▒▒▓▓▓█  ← Tail!
  └───┴──────────┴───────────────┴─────┴──┴──→ ms
      50   100   200    500     1000  2000  5000
      
      ├── p50 ──┤
      ├────────── p95 ─────────────┤
      ├─────────────── p99 ────────────────┤
      ├──────────────────── p99.9 ─────────────────┤

  Average = 180ms (এটা দেখে মনে হয় সব ঠিক!)
  But p99 = 2000ms (১% user 2 সেকেন্ড+ অপেক্ষা!)
```

### Fan-Out Latency Amplification

```
┌─────────────────────────────────────────────────────────────┐
│            Fan-Out Latency Amplification Effect              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Single Service:                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │ p99 = 50ms    (1% chance of being slow)         │       │
│  └─────────────────────────────────────────────────┘       │
│                                                             │
│  5 Parallel Calls (Fan-out = 5):                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │ P(at least one slow) = 1 - (0.99)^5 = 4.9%     │       │
│  │ Your p95 ≈ downstream p99!                       │       │
│  └─────────────────────────────────────────────────┘       │
│                                                             │
│  10 Parallel Calls:                                         │
│  ┌─────────────────────────────────────────────────┐       │
│  │ P(at least one slow) = 1 - (0.99)^10 = 9.6%    │       │
│  │ Almost 1 in 10 requests affected!                │       │
│  └─────────────────────────────────────────────────┘       │
│                                                             │
│  50 Parallel Calls (Google-scale):                          │
│  ┌─────────────────────────────────────────────────┐       │
│  │ P(at least one slow) = 1 - (0.99)^50 = 39.5%   │       │
│  │ 4 out of 10 requests hit tail latency!           │       │
│  └─────────────────────────────────────────────────┘       │
│                                                             │
│  Visual:                                                    │
│                                                             │
│  User ──→ Gateway ──┬──→ Service A ──→ 30ms                │
│                     ├──→ Service B ──→ 25ms                │
│                     ├──→ Service C ──→ 500ms 🐌 SLOW!      │
│                     ├──→ Service D ──→ 20ms                │
│                     └──→ Service E ──→ 35ms                │
│                                                             │
│  Total latency = max(A, B, C, D, E) = 500ms               │
│  (সবচেয়ে slow service dictates total!)                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Hedged Requests Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    Hedged Requests                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without Hedging:                                            │
│  ┌────────┐        ┌────────────┐                           │
│  │ Client │──req──→│ Server A   │──── 800ms (slow!) ──→ ✅ │
│  └────────┘        └────────────┘                           │
│  Total: 800ms 😢                                            │
│                                                              │
│  With Hedging (5ms delay):                                   │
│  ┌────────┐        ┌────────────┐                           │
│  │ Client │──req──→│ Server A   │────── 800ms... (cancel)   │
│  │        │        └────────────┘                           │
│  │        │  +5ms  ┌────────────┐                           │
│  │        │──req──→│ Server B   │── 45ms ──→ ✅ Winner!    │
│  └────────┘        └────────────┘                           │
│  Total: 50ms (5ms delay + 45ms response) 🎉                │
│                                                              │
│  Cost: ~5% extra load (most hedges cancelled quickly)        │
│  Benefit: p99 dramatically reduced!                          │
│                                                              │
│  Timeline:                                                   │
│  0ms   5ms                    50ms                 800ms    │
│  |─────|──────────────────────|─────────────────────|       │
│  │     │                      │                     │       │
│  Send  Send hedge             B responds ✅         A would │
│  to A  to B                   (cancel A)            respond │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Latency Budget Allocation

```
┌─────────────────────────────────────────────────────────────┐
│           Latency Budget: 500ms Total SLO (p99)              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┬──────┬──────────────────┬─────────┬─────┐     │
│  │  API GW │ Auth │  Business Logic  │   DB    │ N/W │     │
│  │  50ms   │ 30ms │     200ms        │  150ms  │20ms │     │
│  │  (10%)  │ (6%) │     (40%)        │  (30%)  │(4%) │     │
│  └─────────┴──────┴──────────────────┴─────────┴─────┘     │
│  |←──────────────── 500ms total ──────────────────→|        │
│                                                              │
│  ⚠️ Budget Violation Example:                                │
│  ┌─────────┬──────┬──────────────────┬──────────────┬──┐   │
│  │  API GW │ Auth │  Business Logic  │     DB       │NW│   │
│  │  50ms   │ 30ms │     200ms        │    350ms! 🔴 │20│   │
│  └─────────┴──────┴──────────────────┴──────────────┴──┘   │
│  |←────────────────── 650ms total (OVER BUDGET!) ────→|     │
│                                                              │
│  Action: DB team-কে budget breach notify করুন               │
│  Fix: Query optimization বা read replica add                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Queuing Theory — Why p99 Spikes at High Load

```
        Response Time vs Server Utilization

p99 (ms)
   │
5s │                                          ╱ ← p99 explodes!
   │                                        ╱
4s │                                      ╱
   │                                    ╱
3s │                                  ╱
   │                                ╱
2s │                              ╱
   │                           ╱
1s │                        ╱─
   │                    ╱──
500│              ╱────
   │        ╱────
200│  ──────
   │────── ← p99 stable at low load
   └───┬────┬────┬────┬────┬────┬────┬────→ Load %
       10   20   40   60   70   80   90  95

   Rule of Thumb:
   ├── < 60% load: p99 stable ✅
   ├── 60-80% load: p99 growing ⚠️  
   ├── 80-90% load: p99 rapidly increasing 🔴
   └── > 90% load: p99 EXPLODING! 💥

   (এজন্য server 70% এর বেশি load করা উচিত না!)
```

---

## 💻 PHP কোড উদাহরণ

### Tail Latency Measurement & Hedged Request Implementation

```php
<?php

/**
 * Tail Latency Measurement & Optimization
 * Pathao Search Service — P99 Optimization
 */

class LatencyHistogram
{
    private array $values = [];
    private array $buckets;
    private int $count = 0;
    private float $sum = 0;

    public function __construct(array $buckets = [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000])
    {
        $this->buckets = $buckets;
    }

    public function observe(float $valueMs): void
    {
        $this->values[] = $valueMs;
        $this->sum += $valueMs;
        $this->count++;
    }

    public function getPercentile(float $p): float
    {
        if (empty($this->values)) return 0;

        $sorted = $this->values;
        sort($sorted);
        $index = (int) ceil(($p / 100) * count($sorted)) - 1;
        return $sorted[max(0, $index)];
    }

    public function getMean(): float
    {
        return $this->count > 0 ? $this->sum / $this->count : 0;
    }

    public function getStats(): array
    {
        return [
            'count' => $this->count,
            'mean' => round($this->getMean(), 2),
            'p50' => round($this->getPercentile(50), 2),
            'p90' => round($this->getPercentile(90), 2),
            'p95' => round($this->getPercentile(95), 2),
            'p99' => round($this->getPercentile(99), 2),
            'p999' => round($this->getPercentile(99.9), 2),
            'max' => round(max($this->values ?: [0]), 2),
        ];
    }

    /**
     * Fan-out probability calculation
     */
    public function fanOutProbability(int $parallelCalls, float $percentile): float
    {
        $individualProbability = $percentile / 100;
        // P(at least one call exceeds percentile) = 1 - P(all within)^n
        return 1 - pow($individualProbability, $parallelCalls);
    }
}

class AdaptiveTimeout
{
    private array $recentLatencies = [];
    private int $windowSize;
    private float $multiplier;
    private float $minTimeout;
    private float $maxTimeout;

    public function __construct(
        int $windowSize = 100,
        float $multiplier = 1.5,
        float $minTimeout = 100,
        float $maxTimeout = 5000
    ) {
        $this->windowSize = $windowSize;
        $this->multiplier = $multiplier;
        $this->minTimeout = $minTimeout;
        $this->maxTimeout = $maxTimeout;
    }

    public function recordLatency(float $latencyMs): void
    {
        $this->recentLatencies[] = $latencyMs;
        if (count($this->recentLatencies) > $this->windowSize) {
            array_shift($this->recentLatencies);
        }
    }

    /**
     * p99 × multiplier = adaptive timeout
     */
    public function getTimeout(): float
    {
        if (count($this->recentLatencies) < 10) {
            return $this->maxTimeout; // Not enough data, use max
        }

        $sorted = $this->recentLatencies;
        sort($sorted);
        $p99Index = (int) ceil(0.99 * count($sorted)) - 1;
        $p99 = $sorted[$p99Index];

        $timeout = $p99 * $this->multiplier;
        return max($this->minTimeout, min($this->maxTimeout, $timeout));
    }
}

class HedgedRequest
{
    private int $hedgeDelayMs;
    private int $maxAttempts;

    public function __construct(int $hedgeDelayMs = 5, int $maxAttempts = 2)
    {
        $this->hedgeDelayMs = $hedgeDelayMs;
        $this->maxAttempts = $maxAttempts;
    }

    /**
     * Primary request পাঠাও, hedge delay-এর পর backup-ও পাঠাও
     * যে আগে respond করে সেটা নাও
     */
    public function execute(callable $requestFn, array $servers): array
    {
        $results = [];

        // Primary request
        $primaryStart = microtime(true);
        $primaryServer = $servers[0];

        // Simulate: production-এ async/parallel execution হবে
        $primaryResult = $requestFn($primaryServer);
        $primaryLatency = (microtime(true) - $primaryStart) * 1000;

        if ($primaryLatency <= $this->hedgeDelayMs || count($servers) < 2) {
            return [
                'result' => $primaryResult,
                'latency' => $primaryLatency,
                'server' => $primaryServer,
                'hedged' => false,
            ];
        }

        // Hedge request (primary slow ছিল)
        $hedgeServer = $servers[1];
        $hedgeStart = microtime(true);
        $hedgeResult = $requestFn($hedgeServer);
        $hedgeLatency = (microtime(true) - $hedgeStart) * 1000;

        // Winner selection
        if ($hedgeLatency < $primaryLatency) {
            return [
                'result' => $hedgeResult,
                'latency' => $hedgeLatency + $this->hedgeDelayMs,
                'server' => $hedgeServer,
                'hedged' => true,
            ];
        }

        return [
            'result' => $primaryResult,
            'latency' => $primaryLatency,
            'server' => $primaryServer,
            'hedged' => false,
        ];
    }
}

class LatencyBudget
{
    private float $totalBudgetMs;
    private array $allocations = [];
    private array $actuals = [];

    public function __construct(float $totalBudgetMs)
    {
        $this->totalBudgetMs = $totalBudgetMs;
    }

    public function allocate(string $component, float $budgetMs): void
    {
        $this->allocations[$component] = $budgetMs;
    }

    public function record(string $component, float $actualMs): void
    {
        $this->actuals[$component] = $actualMs;
    }

    public function getViolations(): array
    {
        $violations = [];
        foreach ($this->allocations as $component => $budget) {
            $actual = $this->actuals[$component] ?? 0;
            if ($actual > $budget) {
                $violations[] = [
                    'component' => $component,
                    'budget' => $budget,
                    'actual' => $actual,
                    'overage_ms' => $actual - $budget,
                    'overage_pct' => round((($actual - $budget) / $budget) * 100, 1),
                ];
            }
        }
        return $violations;
    }

    public function getTotalActual(): float
    {
        return array_sum($this->actuals);
    }

    public function isWithinBudget(): bool
    {
        return $this->getTotalActual() <= $this->totalBudgetMs;
    }

    public function getReport(): array
    {
        $report = [];
        foreach ($this->allocations as $component => $budget) {
            $actual = $this->actuals[$component] ?? 0;
            $report[] = [
                'component' => $component,
                'budget_ms' => $budget,
                'actual_ms' => round($actual, 2),
                'status' => $actual <= $budget ? '✅' : '🔴',
                'utilization' => round(($actual / $budget) * 100, 1) . '%',
            ];
        }
        return $report;
    }
}

// === Simulation: Pathao Search P99 Analysis ===

echo "=== 🏍️ Pathao Search — Tail Latency Analysis ===\n\n";

$histogram = new LatencyHistogram();

// Simulate realistic latency distribution
// Most requests fast, some slow (bimodal)
for ($i = 0; $i < 10000; $i++) {
    $base = 80 + (mt_rand(0, 100) / 10); // 80-90ms base

    // 5% requests hit cache miss → +200ms
    if (mt_rand(1, 100) <= 5) {
        $base += 200;
    }

    // 1% requests hit GC pause → +500-1500ms
    if (mt_rand(1, 100) <= 1) {
        $base += mt_rand(500, 1500);
    }

    // 0.1% requests hit external API timeout → +3000ms
    if (mt_rand(1, 1000) <= 1) {
        $base += 3000;
    }

    // Random jitter
    $base += mt_rand(0, 50);

    $histogram->observe($base);
}

$stats = $histogram->getStats();
echo "📊 Latency Distribution (10,000 requests):\n";
echo "   Mean:  {$stats['mean']}ms\n";
echo "   p50:   {$stats['p50']}ms\n";
echo "   p90:   {$stats['p90']}ms\n";
echo "   p95:   {$stats['p95']}ms\n";
echo "   p99:   {$stats['p99']}ms   ← ⚠️ Average-এর চেয়ে অনেক বেশি!\n";
echo "   p99.9: {$stats['p999']}ms  ← 🔴 Tail latency!\n";
echo "   Max:   {$stats['max']}ms\n\n";

// Fan-out impact
echo "📡 Fan-Out Amplification Effect:\n";
foreach ([1, 3, 5, 10, 20] as $fanOut) {
    $prob = $histogram->fanOutProbability($fanOut, 99);
    echo "   Fan-out={$fanOut}: " . round($prob * 100, 1) . "% chance of hitting tail\n";
}
echo "\n";

// Adaptive Timeout Demo
echo "⏰ Adaptive Timeout:\n";
$adaptiveTimeout = new AdaptiveTimeout(windowSize: 100, multiplier: 1.5);
for ($i = 0; $i < 100; $i++) {
    $adaptiveTimeout->recordLatency(80 + mt_rand(0, 50));
}
echo "   Normal traffic timeout: " . round($adaptiveTimeout->getTimeout(), 0) . "ms\n";

// Spike traffic
for ($i = 0; $i < 20; $i++) {
    $adaptiveTimeout->recordLatency(500 + mt_rand(0, 200));
}
echo "   After spike timeout: " . round($adaptiveTimeout->getTimeout(), 0) . "ms\n\n";

// Latency Budget
echo "📋 Latency Budget Report (SLO: 500ms p99):\n";
$budget = new LatencyBudget(500);
$budget->allocate('API Gateway', 50);
$budget->allocate('Auth Service', 30);
$budget->allocate('Search Logic', 200);
$budget->allocate('Database', 150);
$budget->allocate('Network', 20);

// Simulate actuals
$budget->record('API Gateway', 45);
$budget->record('Auth Service', 25);
$budget->record('Search Logic', 180);
$budget->record('Database', 280); // Over budget!
$budget->record('Network', 15);

$report = $budget->getReport();
foreach ($report as $item) {
    echo "   {$item['status']} {$item['component']}: {$item['actual_ms']}ms / {$item['budget_ms']}ms ({$item['utilization']})\n";
}
echo "\n   Total: {$budget->getTotalActual()}ms / 500ms\n";
echo "   Status: " . ($budget->isWithinBudget() ? '✅ Within budget' : '🔴 OVER BUDGET!') . "\n";

$violations = $budget->getViolations();
if (!empty($violations)) {
    echo "\n   ⚠️ Budget Violations:\n";
    foreach ($violations as $v) {
        echo "   🔴 {$v['component']}: {$v['overage_ms']}ms over ({$v['overage_pct']}% overage)\n";
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Real-time P99 Tracking, Hedged Requests & Fan-out Simulator

```javascript
/**
 * Tail Latency Optimization — JavaScript Implementation
 * Pathao Search Service with Hedged Requests & Adaptive Timeout
 */

// === Histogram-based Latency Tracker ===

class LatencyTracker {
  constructor(name, windowSize = 1000) {
    this.name = name;
    this.windowSize = windowSize;
    this.values = [];
    this.totalCount = 0;
    this.totalSum = 0;
  }

  record(latencyMs) {
    this.values.push(latencyMs);
    this.totalCount++;
    this.totalSum += latencyMs;

    // Sliding window
    if (this.values.length > this.windowSize) {
      this.values.shift();
    }
  }

  percentile(p) {
    if (this.values.length === 0) return 0;
    const sorted = [...this.values].sort((a, b) => a - b);
    const idx = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[Math.max(0, idx)];
  }

  mean() {
    return this.values.length > 0
      ? this.values.reduce((a, b) => a + b, 0) / this.values.length
      : 0;
  }

  getSnapshot() {
    return {
      name: this.name,
      count: this.values.length,
      mean: this.mean().toFixed(2),
      p50: this.percentile(50).toFixed(2),
      p90: this.percentile(90).toFixed(2),
      p95: this.percentile(95).toFixed(2),
      p99: this.percentile(99).toFixed(2),
      p999: this.percentile(99.9).toFixed(2),
      max: Math.max(...this.values).toFixed(2),
    };
  }
}

// === Hedged Request Implementation ===

class HedgedRequestExecutor {
  constructor(hedgeDelayMs = 10, maxHedges = 1) {
    this.hedgeDelayMs = hedgeDelayMs;
    this.maxHedges = maxHedges;
    this.stats = {
      totalRequests: 0,
      hedgedRequests: 0,
      hedgeWins: 0,
    };
  }

  /**
   * Primary + hedge request execution
   * যে server আগে respond করে সেটা win
   */
  async execute(requestFn, endpoints) {
    this.stats.totalRequests++;

    return new Promise((resolve) => {
      let resolved = false;
      const results = [];

      const finish = (result, index, hedged) => {
        if (resolved) return;
        resolved = true;
        
        if (hedged) this.stats.hedgeWins++;
        
        resolve({
          ...result,
          endpoint: endpoints[index],
          wasHedged: hedged,
          hedgeImprovement: hedged
            ? results[0]?.latencyMs - result.latencyMs
            : 0,
        });
      };

      // Primary request immediately
      const primaryStart = performance.now();
      requestFn(endpoints[0]).then(result => {
        const latencyMs = performance.now() - primaryStart;
        results.push({ latencyMs });
        finish({ ...result, latencyMs }, 0, false);
      });

      // Hedge request after delay
      if (endpoints.length > 1) {
        setTimeout(() => {
          if (resolved) return; // Primary already responded
          
          this.stats.hedgedRequests++;
          const hedgeStart = performance.now();
          
          requestFn(endpoints[1]).then(result => {
            const latencyMs = performance.now() - hedgeStart;
            finish({ ...result, latencyMs }, 1, true);
          });
        }, this.hedgeDelayMs);
      }
    });
  }

  getStats() {
    return {
      ...this.stats,
      hedgeRate: ((this.stats.hedgedRequests / this.stats.totalRequests) * 100).toFixed(1) + '%',
      hedgeWinRate: this.stats.hedgedRequests > 0
        ? ((this.stats.hedgeWins / this.stats.hedgedRequests) * 100).toFixed(1) + '%'
        : '0%',
    };
  }
}

// === Adaptive Timeout ===

class AdaptiveTimeout {
  constructor(config = {}) {
    this.windowSize = config.windowSize || 100;
    this.multiplier = config.multiplier || 2.0;
    this.minTimeout = config.minTimeout || 50;
    this.maxTimeout = config.maxTimeout || 5000;
    this.percentile = config.percentile || 99;
    this.latencies = [];
  }

  record(latencyMs) {
    this.latencies.push(latencyMs);
    if (this.latencies.length > this.windowSize) {
      this.latencies.shift();
    }
  }

  getTimeout() {
    if (this.latencies.length < 10) return this.maxTimeout;

    const sorted = [...this.latencies].sort((a, b) => a - b);
    const idx = Math.ceil((this.percentile / 100) * sorted.length) - 1;
    const pValue = sorted[Math.max(0, idx)];

    const timeout = pValue * this.multiplier;
    return Math.max(this.minTimeout, Math.min(this.maxTimeout, timeout));
  }
}

// === Fan-Out Simulator ===

class FanOutSimulator {
  constructor(serviceCount, individualP99Ms) {
    this.serviceCount = serviceCount;
    this.individualP99Ms = individualP99Ms;
  }

  /**
   * Parallel fan-out simulate — worst case = max of all
   */
  simulate(iterations = 10000) {
    const results = [];

    for (let i = 0; i < iterations; i++) {
      let maxLatency = 0;

      for (let s = 0; s < this.serviceCount; s++) {
        const latency = this._generateLatency();
        maxLatency = Math.max(maxLatency, latency);
      }

      results.push(maxLatency);
    }

    results.sort((a, b) => a - b);

    return {
      serviceCount: this.serviceCount,
      iterations,
      p50: results[Math.floor(iterations * 0.50)].toFixed(1),
      p95: results[Math.floor(iterations * 0.95)].toFixed(1),
      p99: results[Math.floor(iterations * 0.99)].toFixed(1),
      p999: results[Math.floor(iterations * 0.999)].toFixed(1),
      theoreticalP99Probability: (1 - Math.pow(0.99, this.serviceCount) * 100).toFixed(1),
    };
  }

  _generateLatency() {
    // Realistic bimodal distribution
    const rand = Math.random();
    if (rand < 0.95) {
      // 95% fast (normal distribution around 50ms)
      return 30 + Math.random() * 40;
    } else if (rand < 0.99) {
      // 4% medium (cache miss or retry)
      return 100 + Math.random() * 200;
    } else {
      // 1% slow (GC, timeout, contention)
      return this.individualP99Ms + Math.random() * this.individualP99Ms;
    }
  }
}

// === Latency Budget Tracker ===

class LatencyBudgetTracker {
  constructor(totalBudgetMs) {
    this.totalBudgetMs = totalBudgetMs;
    this.components = new Map();
  }

  setBudget(component, budgetMs) {
    this.components.set(component, { budget: budgetMs, actuals: [] });
  }

  recordActual(component, latencyMs) {
    const entry = this.components.get(component);
    if (entry) {
      entry.actuals.push(latencyMs);
    }
  }

  getReport() {
    const report = [];
    let totalActual = 0;

    for (const [name, data] of this.components) {
      const avg = data.actuals.length > 0
        ? data.actuals.reduce((a, b) => a + b, 0) / data.actuals.length
        : 0;
      
      const p99 = this._percentile(data.actuals, 99);
      const withinBudget = p99 <= data.budget;
      totalActual += p99;

      report.push({
        component: name,
        budget: data.budget,
        avgMs: avg.toFixed(1),
        p99Ms: p99.toFixed(1),
        status: withinBudget ? '✅' : '🔴',
        utilization: ((p99 / data.budget) * 100).toFixed(0) + '%',
      });
    }

    return {
      components: report,
      totalBudget: this.totalBudgetMs,
      totalActualP99: totalActual.toFixed(1),
      withinTotalBudget: totalActual <= this.totalBudgetMs,
    };
  }

  _percentile(values, p) {
    if (values.length === 0) return 0;
    const sorted = [...values].sort((a, b) => a - b);
    const idx = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[Math.max(0, idx)];
  }
}

// === Main Simulation ===

async function main() {
  console.log('=== ⏱️ Pathao Search — Tail Latency Deep Dive ===\n');

  // 1. Latency Distribution Analysis
  console.log('📊 1. Latency Distribution (10,000 simulated requests)\n');

  const tracker = new LatencyTracker('pathao-search');

  for (let i = 0; i < 10000; i++) {
    let latency = 80 + Math.random() * 40; // Base 80-120ms

    if (Math.random() < 0.05) latency += 200;  // 5% cache miss
    if (Math.random() < 0.01) latency += 800;  // 1% GC pause
    if (Math.random() < 0.001) latency += 3000; // 0.1% timeout

    tracker.record(latency);
  }

  const snapshot = tracker.getSnapshot();
  console.log(`   Mean:  ${snapshot.mean}ms`);
  console.log(`   p50:   ${snapshot.p50}ms`);
  console.log(`   p90:   ${snapshot.p90}ms`);
  console.log(`   p95:   ${snapshot.p95}ms`);
  console.log(`   p99:   ${snapshot.p99}ms  ← Average-এর ${(snapshot.p99 / snapshot.mean).toFixed(1)}x!`);
  console.log(`   p99.9: ${snapshot.p999}ms`);
  console.log(`   Max:   ${snapshot.max}ms\n`);

  // 2. Fan-Out Amplification
  console.log('📡 2. Fan-Out Latency Amplification\n');

  for (const serviceCount of [1, 3, 5, 10, 20]) {
    const sim = new FanOutSimulator(serviceCount, 500);
    const result = sim.simulate(5000);
    console.log(`   ${serviceCount} services: p50=${result.p50}ms, p99=${result.p99}ms, p99.9=${result.p999}ms`);
  }
  console.log('');

  // 3. Hedged Requests
  console.log('🛡️ 3. Hedged Requests Simulation\n');

  const hedger = new HedgedRequestExecutor(10); // 10ms hedge delay

  // Simulate service with variable latency
  const mockService = async (endpoint) => {
    const baseLatency = endpoint === 'server-a' ? 50 : 45;
    // 10% chance of being slow
    const latency = Math.random() < 0.1
      ? baseLatency + 500 + Math.random() * 1000
      : baseLatency + Math.random() * 30;
    
    await new Promise(r => setTimeout(r, latency));
    return { data: 'result', latencyMs: latency };
  };

  const withHedge = [];
  const withoutHedge = [];

  for (let i = 0; i < 100; i++) {
    // With hedging
    const hedgedResult = await hedger.execute(mockService, ['server-a', 'server-b']);
    withHedge.push(hedgedResult.latencyMs);

    // Without hedging (just primary)
    const start = performance.now();
    await mockService('server-a');
    withoutHedge.push(performance.now() - start);
  }

  withHedge.sort((a, b) => a - b);
  withoutHedge.sort((a, b) => a - b);

  const p99WithHedge = withHedge[Math.floor(withHedge.length * 0.99)];
  const p99WithoutHedge = withoutHedge[Math.floor(withoutHedge.length * 0.99)];

  console.log(`   Without hedging — p99: ${p99WithoutHedge.toFixed(0)}ms`);
  console.log(`   With hedging    — p99: ${p99WithHedge.toFixed(0)}ms`);
  console.log(`   Improvement: ${((1 - p99WithHedge / p99WithoutHedge) * 100).toFixed(0)}% reduction! 🎉`);
  console.log(`   Hedge stats: ${JSON.stringify(hedger.getStats())}\n`);

  // 4. Adaptive Timeout
  console.log('⏰ 4. Adaptive Timeout\n');

  const timeout = new AdaptiveTimeout({ multiplier: 1.5, percentile: 99 });

  // Normal traffic
  for (let i = 0; i < 100; i++) {
    timeout.record(50 + Math.random() * 30);
  }
  console.log(`   Normal traffic → Timeout: ${timeout.getTimeout().toFixed(0)}ms`);

  // Traffic spike
  for (let i = 0; i < 30; i++) {
    timeout.record(300 + Math.random() * 200);
  }
  console.log(`   After spike   → Timeout: ${timeout.getTimeout().toFixed(0)}ms (auto-adjusted!)`);

  // Recovery
  for (let i = 0; i < 80; i++) {
    timeout.record(50 + Math.random() * 30);
  }
  console.log(`   After recovery→ Timeout: ${timeout.getTimeout().toFixed(0)}ms (back to normal)\n`);

  // 5. Latency Budget
  console.log('📋 5. Latency Budget Tracking\n');

  const budgetTracker = new LatencyBudgetTracker(500);
  budgetTracker.setBudget('API Gateway', 50);
  budgetTracker.setBudget('Auth', 30);
  budgetTracker.setBudget('Search Logic', 200);
  budgetTracker.setBudget('Database', 150);
  budgetTracker.setBudget('Network', 20);

  // Simulate 100 requests
  for (let i = 0; i < 100; i++) {
    budgetTracker.recordActual('API Gateway', 20 + Math.random() * 30);
    budgetTracker.recordActual('Auth', 15 + Math.random() * 20);
    budgetTracker.recordActual('Search Logic', 100 + Math.random() * 150);
    budgetTracker.recordActual('Database', 80 + Math.random() * 200); // Variable!
    budgetTracker.recordActual('Network', 5 + Math.random() * 15);
  }

  const report = budgetTracker.getReport();
  console.log(`   Total Budget: ${report.totalBudget}ms (p99 SLO)`);
  console.log(`   Total Actual p99: ${report.totalActualP99}ms`);
  console.log(`   Status: ${report.withinTotalBudget ? '✅ WITHIN BUDGET' : '🔴 OVER BUDGET!'}\n`);

  for (const comp of report.components) {
    console.log(`   ${comp.status} ${comp.component}: p99=${comp.p99Ms}ms / budget=${comp.budget}ms (${comp.utilization})`);
  }
}

main().catch(console.error);
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ Tail Latency Optimization কখন করবেন

| পরিস্থিতি | কেন |
|-----------|-----|
| High fan-out microservices | Amplification effect devastation করবে |
| User-facing API (search, feed) | UX directly affected |
| SLO p99 target আছে | p99 miss করলে error budget eat করে |
| Payment/Financial transactions | Timeout = failed transaction |
| Real-time system (Pathao matching) | Late response = useless response |
| High RPS service (1000+) | 1% of 1M = 10,000 affected users |

### ❌ কখন Overkill

| পরিস্থিতি | কেন |
|-----------|-----|
| Internal batch job | Nobody waiting for it |
| Low-traffic service (< 100 RPM) | p99 = 1 request, statistically meaningless |
| Already within SLO | Don't optimize what isn't broken |
| Read-heavy caching service | Cache hit ratio fix করুন আগে |
| Development/staging | Production-এই focus করুন |

### 🎯 Quick Wins for P99 Reduction

```
1. Connection Pooling:
   ❌ প্রতি request-এ নতুন DB connection
   ✅ Connection pool (p99 impact: -200ms)

2. Async where possible:
   ❌ Synchronous notification send
   ✅ Queue-তে push, async process (p99 impact: -500ms)

3. Timeout tuning:
   ❌ Default 30s timeout (slow requests block pool)
   ✅ Adaptive timeout based on p99 (p99 impact: -1000ms)

4. Caching:
   ❌ প্রতি request-এ compute/DB call
   ✅ Cache frequent data (p99 impact: -300ms)

5. GC Tuning:
   ❌ Default GC settings (stop-the-world pauses)
   ✅ Low-latency GC (ZGC/Shenandoah) (p99 impact: -200ms)

6. Pre-warming:
   ❌ Cold start after deploy
   ✅ Warm up caches/connections before traffic (p99 impact: -500ms)
```

### 📐 P99 Optimization Priority Matrix

```
              Impact on P99
              High            Low
         ┌──────────────┬──────────────┐
    Easy │  🥇 DO FIRST │  🥉 Nice to  │  Effort
         │  • Timeouts  │     have     │
         │  • Caching   │  • Log tuning│
         │  • Pool size │  • Metrics   │
         ├──────────────┼──────────────┤
    Hard │  🥈 Plan for │  ❌ AVOID    │
         │  • Hedging   │  • Rewrite   │
         │  • Sharding  │  • New infra │
         │  • Async     │  • Over-eng  │
         └──────────────┴──────────────┘
```

### 🔑 মনে রাখবেন

```
1. "Average is a lie" — সবসময় percentile দেখুন
2. p99 > 10× mean → আপনার tail latency problem আছে
3. Fan-out = n হলে, overall p99 ≈ individual p(100 - 100/n)
4. 70% server utilization-এর উপরে p99 exponentially বাড়ে
5. Hedged requests: 5% extra load-এ 50%+ p99 improvement
6. SLO p99 target ছাড়া optimization blind optimization
```
