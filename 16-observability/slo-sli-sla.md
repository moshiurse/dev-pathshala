# 🎯 Error Budget & SLO/SLI/SLA Definitions

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🧠 সংজ্ঞা ও ধারণা

### SLI (Service Level Indicator) — সার্ভিস লেভেল ইন্ডিকেটর

SLI হলো আপনার সিস্টেমের **পরিমাপযোগ্য মেট্রিক** যা সার্ভিসের গুণগত মান নির্দেশ করে।

**সাধারণ SLI গুলো:**
- **Availability (প্রাপ্যতা):** সফল রিকোয়েস্ট / মোট রিকোয়েস্ট
- **Latency (বিলম্ব):** p50, p95, p99 রেসপন্স টাইম
- **Throughput (থ্রুপুট):** প্রতি সেকেন্ডে প্রসেস করা রিকোয়েস্ট সংখ্যা
- **Error Rate (ত্রুটির হার):** ব্যর্থ রিকোয়েস্ট / মোট রিকোয়েস্ট

### SLO (Service Level Objective) — সার্ভিস লেভেল অবজেক্টিভ

SLO হলো SLI-এর উপর **টার্গেট মান** যা আপনার টিম অভ্যন্তরীণভাবে সেট করে।

> উদাহরণ: "আমাদের payment API-র availability হবে 99.95%"

### SLA (Service Level Agreement) — সার্ভিস লেভেল এগ্রিমেন্ট

SLA হলো **গ্রাহকের সাথে চুক্তি** — SLO ভঙ্গ হলে ক্ষতিপূরণ দিতে হবে।

> উদাহরণ: "99.9% uptime গ্যারান্টি, না হলে ক্রেডিট রিফান্ড"

### 🔑 তিনটির সম্পর্ক

```
SLI ⊂ SLO ⊂ SLA

SLI = কী পরিমাপ করছেন (measurement)
SLO = কত ভালো হওয়া উচিত (target)
SLA = ব্যর্থ হলে কী হবে (consequence)
```

### 💡 Error Budget (ত্রুটি বাজেট)

Error Budget = **100% - SLO**

যদি SLO = 99.95%, তাহলে Error Budget = 0.05%

**৩০ দিনে এর মানে:**
- মোট মিনিট: 30 × 24 × 60 = 43,200 মিনিট
- Error Budget: 43,200 × 0.0005 = **21.6 মিনিট** downtime অনুমোদিত

### 🔥 Burn Rate (বার্ন রেট)

Burn Rate = আপনি কত দ্রুত Error Budget খরচ করছেন।

- Burn Rate 1 = স্বাভাবিক গতিতে budget শেষ হচ্ছে (30 দিনে শেষ)
- Burn Rate 2 = দ্বিগুণ গতিতে (15 দিনে শেষ হবে)
- Burn Rate 10 = 3 দিনে budget শেষ!

### 📊 Multi-Window Burn Rate

Google SRE দুটি উইন্ডো ব্যবহার করে:
- **Long window:** গত 1 ঘণ্টা (sustained problem detect করতে)
- **Short window:** গত 5 মিনিট (problem এখনও চলছে কিনা confirm করতে)

### 🎲 Google SRE Error Budget Policy

Error Budget শেষ হলে:
1. ✋ নতুন feature release বন্ধ
2. 🔧 শুধু reliability improvement কাজ
3. 📊 Post-mortem এবং root cause analysis
4. ✅ Budget recover হলে আবার feature deployment

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏦 bKash Payment API — ঈদে Error Budget Tracking

**পরিস্থিতি:** ঈদুল ফিতরের আগের সপ্তাহ। bKash-এর Send Money API-তে ভয়ংকর লোড।

**SLI সেটআপ:**
- Availability SLI: সফল transaction / মোট transaction attempt
- Latency SLI: p99 response time < 500ms
- Error Rate SLI: HTTP 5xx responses < 0.05%

**SLO:**
- Availability: 99.95% (মাসে 21.6 মিনিট downtime budget)
- Latency p99: 500ms এর নিচে
- Error Rate: 0.05% এর নিচে

**SLA (Merchant Agreement):**
- 99.9% availability guarantee
- SLA breach হলে merchant-দের transaction fee waiver

**ঈদের সপ্তাহে কী হলো:**

```
দিন ১ (সোমবার): Normal load, burn rate 0.5 ✅
দিন ২ (মঙ্গলবার): Load বাড়ছে, burn rate 1.2 ⚠️
দিন ৩ (বুধবার): Database timeout, burn rate 4.0 🔴
দিন ৪ (বৃহস্পতিবার): Hotfix deployed, burn rate 0.8 ✅
দিন ৫ (শুক্রবার - ঈদ): Peak load, burn rate 2.5 🟡
```

**সিদ্ধান্ত:** বুধবার burn rate 4 হওয়ায় নতুন payment method feature freeze করা হলো। শুধু database optimization করা হলো।

### 📱 Grameenphone MyGP App

**SLO:** Balance check API 99.9% available, p95 < 200ms
**Error Budget:** মাসে 43.2 মিনিট
**বাস্তবতা:** প্রতি মাসের ১ তারিখে (recharge peak) burn rate বেড়ে যায়

---

## 📊 ASCII ডায়াগ্রাম

### SLI → SLO → SLA সম্পর্ক

```
┌─────────────────────────────────────────────────────────┐
│                    SLA (চুক্তি)                          │
│  "99.9% uptime, না হলে ক্রেডিট রিফান্ড"                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              SLO (টার্গেট)                       │    │
│  │  "আমরা 99.95% availability maintain করবো"       │    │
│  │                                                 │    │
│  │  ┌─────────────────────────────────────────┐    │    │
│  │  │           SLI (পরিমাপ)                  │    │    │
│  │  │  successful_requests / total_requests   │    │    │
│  │  │  = 99.97% (current)                     │    │    │
│  │  └─────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Error Budget Burn Over Time

```
Error Budget (মিনিট)
  21.6 ┤━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Budget Start
       │╲
  18.0 ┤  ╲─── Normal burn (burn rate 1)
       │    ╲
  14.4 ┤     ╲
       │      ╲
  10.8 ┤       ╲
       │        ╲╲╲ ←── Incident! (burn rate 4)
   7.2 ┤            ╲
       │             ╲
   3.6 ┤              ─── Budget almost gone! 🚨
       │               ╲
   0.0 ┤────────────────╳── Budget EXHAUSTED ❌
       └──┬───┬───┬───┬───┬───┬───┬──→ দিন
          5   10  15  20  25  30
```

### Multi-Window Burn Rate Alert

```
┌────────────────────────────────────────────────────┐
│            Multi-Window Burn Rate Alerting          │
├────────────────────────────────────────────────────┤
│                                                    │
│  Severity │ Long Window │ Short Window │ Burn Rate │
│  ─────────┼─────────────┼──────────────┼───────────│
│  PAGE 🔴  │  1 hour     │  5 min       │  14.4     │
│  PAGE 🔴  │  6 hours    │  30 min      │  6.0      │
│  TICKET🟡 │  3 days     │  6 hours     │  1.0      │
│  LOG  🟢  │  30 days    │  3 days      │  N/A      │
│                                                    │
│  Alert fires ONLY when:                            │
│  Long Window burn rate > threshold                 │
│       AND                                          │
│  Short Window burn rate > threshold                │
│                                                    │
│  (এতে false positive কমে কারণ recovered           │
│   incidents alert trigger করবে না)                 │
└────────────────────────────────────────────────────┘
```

### Error Budget Policy Decision Flow

```
                    ┌──────────────────┐
                    │  Error Budget    │
                    │  Remaining?      │
                    └────────┬─────────┘
                             │
                   ┌─────────┴─────────┐
                   │                   │
              ≥ 50% ✅            < 50% ⚠️
                   │                   │
          ┌────────┴────────┐   ┌──────┴──────┐
          │ Feature release │   │  Budget     │
          │ স্বাভাবিক      │   │  < 0% ?     │
          └─────────────────┘   └──────┬──────┘
                                       │
                              ┌────────┴────────┐
                              │                 │
                         হ্যাঁ ❌           না 🟡
                              │                 │
                    ┌─────────┴──────┐  ┌───────┴───────┐
                    │ Feature Freeze │  │ Careful deploy│
                    │ Only SRE work  │  │ Extra review  │
                    │ Post-mortem    │  │ Canary longer │
                    └────────────────┘  └───────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Error Budget Calculator & SLO Monitor

```php
<?php

/**
 * SLO/SLI/SLA Error Budget Tracking System
 * bKash Payment API-র জন্য Error Budget Manager
 */

class SLIMetric
{
    private string $name;
    private float $totalRequests = 0;
    private float $successfulRequests = 0;
    private array $latencies = [];

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function recordRequest(bool $success, float $latencyMs): void
    {
        $this->totalRequests++;
        if ($success) {
            $this->successfulRequests++;
        }
        $this->latencies[] = $latencyMs;
    }

    public function getAvailability(): float
    {
        if ($this->totalRequests === 0) return 1.0;
        return $this->successfulRequests / $this->totalRequests;
    }

    public function getPercentileLatency(float $percentile): float
    {
        if (empty($this->latencies)) return 0;

        sort($this->latencies);
        $index = (int) ceil(($percentile / 100) * count($this->latencies)) - 1;
        return $this->latencies[max(0, $index)];
    }

    public function getErrorRate(): float
    {
        if ($this->totalRequests === 0) return 0;
        return 1.0 - $this->getAvailability();
    }
}

class ErrorBudget
{
    private float $sloTarget;          // e.g., 0.9995 (99.95%)
    private int $windowDays;           // e.g., 30
    private float $consumedMinutes = 0;

    public function __construct(float $sloTarget, int $windowDays = 30)
    {
        $this->sloTarget = $sloTarget;
        $this->windowDays = $windowDays;
    }

    /**
     * মোট Error Budget (মিনিটে)
     */
    public function getTotalBudgetMinutes(): float
    {
        $totalMinutes = $this->windowDays * 24 * 60;
        return $totalMinutes * (1 - $this->sloTarget);
    }

    /**
     * বাকি Error Budget
     */
    public function getRemainingBudget(): float
    {
        return max(0, $this->getTotalBudgetMinutes() - $this->consumedMinutes);
    }

    /**
     * Error Budget শতাংশে কত বাকি
     */
    public function getRemainingPercentage(): float
    {
        $total = $this->getTotalBudgetMinutes();
        if ($total === 0.0) return 0;
        return ($this->getRemainingBudget() / $total) * 100;
    }

    /**
     * Downtime রেকর্ড করো
     */
    public function consumeBudget(float $downtimeMinutes): void
    {
        $this->consumedMinutes += $downtimeMinutes;
    }

    /**
     * Budget exhausted কিনা চেক
     */
    public function isExhausted(): bool
    {
        return $this->consumedMinutes >= $this->getTotalBudgetMinutes();
    }
}

class BurnRateCalculator
{
    /**
     * Burn Rate = (actual error rate) / (allowed error rate)
     * Burn Rate 1 = budget ঠিক 30 দিনে শেষ হবে
     */
    public function calculate(float $currentErrorRate, float $sloTarget): float
    {
        $allowedErrorRate = 1 - $sloTarget;
        if ($allowedErrorRate === 0.0) return PHP_FLOAT_MAX;
        return $currentErrorRate / $allowedErrorRate;
    }

    /**
     * Multi-Window Burn Rate Alert Check
     * দুটো উইন্ডোতেই threshold cross করলে alert
     */
    public function shouldAlert(
        float $longWindowBurnRate,
        float $shortWindowBurnRate,
        float $threshold
    ): bool {
        return $longWindowBurnRate > $threshold
            && $shortWindowBurnRate > $threshold;
    }

    /**
     * Budget কত দিনে শেষ হবে (বর্তমান burn rate-এ)
     */
    public function daysUntilExhaustion(float $burnRate, int $windowDays = 30): float
    {
        if ($burnRate <= 0) return PHP_FLOAT_MAX;
        return $windowDays / $burnRate;
    }
}

class SLOAlertManager
{
    private array $alertRules = [];

    public function __construct()
    {
        // Google SRE recommended multi-window burn rate rules
        $this->alertRules = [
            ['severity' => 'PAGE',   'longWindow' => 1,    'shortWindow' => 5/60,   'burnRate' => 14.4],
            ['severity' => 'PAGE',   'longWindow' => 6,    'shortWindow' => 0.5,    'burnRate' => 6.0],
            ['severity' => 'TICKET', 'longWindow' => 72,   'shortWindow' => 6,      'burnRate' => 1.0],
        ];
    }

    public function evaluate(float $currentBurnRateLong, float $currentBurnRateShort): ?array
    {
        foreach ($this->alertRules as $rule) {
            if ($currentBurnRateLong >= $rule['burnRate']
                && $currentBurnRateShort >= $rule['burnRate']) {
                return [
                    'severity' => $rule['severity'],
                    'burnRate' => $rule['burnRate'],
                    'message' => $this->generateAlertMessage($rule),
                ];
            }
        }
        return null;
    }

    private function generateAlertMessage(array $rule): string
    {
        return sprintf(
            "[%s] Burn rate %.1f exceeded! Budget %.0f ঘণ্টায় শেষ হবে।",
            $rule['severity'],
            $rule['burnRate'],
            $rule['longWindow']
        );
    }
}

class ErrorBudgetPolicy
{
    /**
     * Error Budget Policy অনুযায়ী সিদ্ধান্ত
     */
    public function getDecision(ErrorBudget $budget): array
    {
        $remaining = $budget->getRemainingPercentage();

        if ($remaining >= 50) {
            return [
                'action' => 'FULL_SPEED',
                'message' => '✅ Budget ভালো আছে। স্বাভাবিক feature release চালু।',
                'canRelease' => true,
                'extraReview' => false,
            ];
        }

        if ($remaining >= 25) {
            return [
                'action' => 'CAUTION',
                'message' => '⚠️ Budget কমছে। Extra review এবং longer canary period।',
                'canRelease' => true,
                'extraReview' => true,
            ];
        }

        if ($remaining > 0) {
            return [
                'action' => 'RESTRICTED',
                'message' => '🟡 Budget প্রায় শেষ। শুধু critical fixes deploy করুন।',
                'canRelease' => false,
                'extraReview' => true,
            ];
        }

        return [
            'action' => 'FREEZE',
            'message' => '❌ Budget EXHAUSTED! Feature freeze। শুধু reliability work।',
            'canRelease' => false,
            'extraReview' => true,
        ];
    }
}

// === ব্যবহার উদাহরণ: bKash Eid Traffic ===

echo "=== 🏦 bKash Payment API — Error Budget Tracking ===\n\n";

// SLO সেটআপ: 99.95% availability
$budget = new ErrorBudget(0.9995, 30);
echo "📊 SLO: 99.95% availability\n";
echo "📊 মোট Error Budget: " . round($budget->getTotalBudgetMinutes(), 1) . " মিনিট/মাস\n\n";

// ঈদের সপ্তাহে incidents
$incidents = [
    ['day' => 'সোমবার', 'downtime' => 2.0, 'desc' => 'Minor timeout spike'],
    ['day' => 'মঙ্গলবার', 'downtime' => 3.5, 'desc' => 'Load balancer issue'],
    ['day' => 'বুধবার', 'downtime' => 8.0, 'desc' => 'Database connection pool exhausted'],
    ['day' => 'বৃহস্পতিবার', 'downtime' => 1.0, 'desc' => 'Hotfix deployed, minor blip'],
    ['day' => 'শুক্রবার (ঈদ)', 'downtime' => 5.0, 'desc' => 'Payment gateway overload'],
];

$policy = new ErrorBudgetPolicy();

foreach ($incidents as $incident) {
    $budget->consumeBudget($incident['downtime']);
    $decision = $policy->getDecision($budget);

    echo "📅 {$incident['day']}: {$incident['desc']}\n";
    echo "   Downtime: {$incident['downtime']} মিনিট\n";
    echo "   Budget বাকি: " . round($budget->getRemainingBudget(), 1) . " মিনিট ";
    echo "(" . round($budget->getRemainingPercentage(), 1) . "%)\n";
    echo "   সিদ্ধান্ত: {$decision['message']}\n\n";
}

// Burn Rate Check
$burnCalc = new BurnRateCalculator();
$currentErrorRate = 0.001; // 0.1% error rate
$burnRate = $burnCalc->calculate($currentErrorRate, 0.9995);
echo "🔥 Current Burn Rate: " . round($burnRate, 1) . "\n";
echo "⏰ Budget শেষ হবে: " . round($burnCalc->daysUntilExhaustion($burnRate), 1) . " দিনে\n";

// Alert Check
$alertManager = new SLOAlertManager();
$alert = $alertManager->evaluate($burnRate, $burnRate);
if ($alert) {
    echo "🚨 ALERT: {$alert['message']}\n";
} else {
    echo "✅ কোনো alert নেই।\n";
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Real-time Error Budget Dashboard & Burn Rate Monitor

```javascript
/**
 * SLO/SLI Error Budget Monitoring System
 * bKash-এর মতো payment service-এর জন্য real-time monitor
 */

class SLICollector {
  constructor(name) {
    this.name = name;
    this.windows = new Map(); // time-window → metrics
    this.latencyHistogram = [];
  }

  record(success, latencyMs, timestamp = Date.now()) {
    const minute = Math.floor(timestamp / 60000);
    
    if (!this.windows.has(minute)) {
      this.windows.set(minute, { total: 0, success: 0, errors: 0 });
    }
    
    const window = this.windows.get(minute);
    window.total++;
    if (success) {
      window.success++;
    } else {
      window.errors++;
    }
    
    this.latencyHistogram.push({ latencyMs, timestamp });
    
    // পুরনো ডেটা cleanup (30 দিনের বেশি পুরনো)
    this.cleanup(timestamp);
  }

  getAvailability(fromTimestamp, toTimestamp) {
    let total = 0, success = 0;
    
    const fromMinute = Math.floor(fromTimestamp / 60000);
    const toMinute = Math.floor(toTimestamp / 60000);
    
    for (const [minute, metrics] of this.windows) {
      if (minute >= fromMinute && minute <= toMinute) {
        total += metrics.total;
        success += metrics.success;
      }
    }
    
    return total === 0 ? 1.0 : success / total;
  }

  getPercentileLatency(percentile, fromTimestamp, toTimestamp) {
    const relevant = this.latencyHistogram
      .filter(l => l.timestamp >= fromTimestamp && l.timestamp <= toTimestamp)
      .map(l => l.latencyMs)
      .sort((a, b) => a - b);
    
    if (relevant.length === 0) return 0;
    
    const index = Math.ceil((percentile / 100) * relevant.length) - 1;
    return relevant[Math.max(0, index)];
  }

  cleanup(currentTimestamp) {
    const cutoff = Math.floor((currentTimestamp - 30 * 24 * 60 * 60 * 1000) / 60000);
    for (const [minute] of this.windows) {
      if (minute < cutoff) this.windows.delete(minute);
    }
  }
}

class ErrorBudget {
  constructor(sloTarget, windowDays = 30) {
    this.sloTarget = sloTarget;
    this.windowDays = windowDays;
  }

  get totalBudgetMinutes() {
    return this.windowDays * 24 * 60 * (1 - this.sloTarget);
  }

  /**
   * বর্তমান SLI থেকে consumed budget হিসাব
   */
  getConsumedMinutes(currentAvailability, elapsedMinutes) {
    const expectedFailureRate = 1 - this.sloTarget;
    const actualFailureRate = 1 - currentAvailability;
    
    // শুধু allowed-এর বেশি failure count করি
    if (actualFailureRate <= expectedFailureRate) return 0;
    
    return (actualFailureRate - expectedFailureRate) * elapsedMinutes;
  }

  getRemainingBudget(consumedMinutes) {
    return Math.max(0, this.totalBudgetMinutes - consumedMinutes);
  }

  getRemainingPercentage(consumedMinutes) {
    if (this.totalBudgetMinutes === 0) return 0;
    return (this.getRemainingBudget(consumedMinutes) / this.totalBudgetMinutes) * 100;
  }

  isExhausted(consumedMinutes) {
    return consumedMinutes >= this.totalBudgetMinutes;
  }
}

class BurnRateAlert {
  constructor(sloTarget) {
    this.sloTarget = sloTarget;
    
    // Multi-window burn rate thresholds (Google SRE)
    this.rules = [
      { severity: 'CRITICAL', longWindowHours: 1,  shortWindowMin: 5,   burnRate: 14.4 },
      { severity: 'CRITICAL', longWindowHours: 6,  shortWindowMin: 30,  burnRate: 6.0  },
      { severity: 'WARNING',  longWindowHours: 72, shortWindowMin: 360, burnRate: 1.0  },
    ];
  }

  /**
   * Burn Rate হিসাব: actual_error_rate / allowed_error_rate
   */
  calculateBurnRate(errorRate) {
    const allowedErrorRate = 1 - this.sloTarget;
    if (allowedErrorRate === 0) return Infinity;
    return errorRate / allowedErrorRate;
  }

  /**
   * Multi-window alert evaluation
   */
  evaluate(sliCollector) {
    const now = Date.now();
    const alerts = [];

    for (const rule of this.rules) {
      const longWindowStart = now - (rule.longWindowHours * 60 * 60 * 1000);
      const shortWindowStart = now - (rule.shortWindowMin * 60 * 1000);

      const longAvailability = sliCollector.getAvailability(longWindowStart, now);
      const shortAvailability = sliCollector.getAvailability(shortWindowStart, now);

      const longBurnRate = this.calculateBurnRate(1 - longAvailability);
      const shortBurnRate = this.calculateBurnRate(1 - shortAvailability);

      // দুটো window-ই threshold cross করলেই alert
      if (longBurnRate >= rule.burnRate && shortBurnRate >= rule.burnRate) {
        alerts.push({
          severity: rule.severity,
          burnRate: longBurnRate,
          threshold: rule.burnRate,
          longWindow: `${rule.longWindowHours}h`,
          shortWindow: `${rule.shortWindowMin}m`,
          message: `${rule.severity}: Burn rate ${longBurnRate.toFixed(1)}x ` +
                   `(threshold: ${rule.burnRate}x) — ` +
                   `Budget ${(30 / longBurnRate).toFixed(1)} দিনে শেষ হবে!`
        });
        break; // শুধু সবচেয়ে severe alert
      }
    }

    return alerts;
  }
}

class ErrorBudgetPolicy {
  decide(remainingPercentage) {
    if (remainingPercentage >= 50) {
      return {
        action: 'FULL_SPEED',
        canDeploy: true,
        canaryDuration: '30 মিনিট',
        message: '✅ Error budget ভালো আছে। Feature release অবাধ।',
      };
    }
    if (remainingPercentage >= 25) {
      return {
        action: 'CAUTION',
        canDeploy: true,
        canaryDuration: '2 ঘণ্টা',
        message: '⚠️ Budget কমছে। Canary duration বাড়ানো হয়েছে।',
      };
    }
    if (remainingPercentage > 0) {
      return {
        action: 'RESTRICTED',
        canDeploy: false,
        canaryDuration: 'N/A',
        message: '🟡 শুধু critical bugfix deploy অনুমোদিত।',
      };
    }
    return {
      action: 'FREEZE',
      canDeploy: false,
      canaryDuration: 'N/A',
      message: '❌ FEATURE FREEZE! শুধু reliability improvement।',
    };
  }
}

// === ব্যবহার: bKash Payment Monitor ===

console.log('=== 🏦 bKash Error Budget Real-time Monitor ===\n');

const sli = new SLICollector('bkash-send-money');
const budget = new ErrorBudget(0.9995, 30); // 99.95% SLO
const alerter = new BurnRateAlert(0.9995);
const policy = new ErrorBudgetPolicy();

console.log(`📊 SLO Target: ${(budget.sloTarget * 100).toFixed(2)}%`);
console.log(`📊 Total Error Budget: ${budget.totalBudgetMinutes.toFixed(1)} মিনিট/মাস`);
console.log(`📊 Per Day Budget: ${(budget.totalBudgetMinutes / 30).toFixed(2)} মিনিট/দিন\n`);

// Simulated traffic — ঈদের দিনে
const scenarios = [
  { name: 'সকাল (স্বাভাবিক)', successRate: 0.9999, requests: 10000 },
  { name: 'দুপুর (লোড বাড়ছে)', successRate: 0.9990, requests: 50000 },
  { name: 'বিকেল (ঈদ সেলামি পিক)', successRate: 0.9970, requests: 100000 },
  { name: 'সন্ধ্যা (স্থিতিশীল)', successRate: 0.9995, requests: 30000 },
];

let consumedTotal = 0;
const now = Date.now();

scenarios.forEach((scenario, index) => {
  const timeOffset = index * 60 * 60 * 1000; // প্রতিটি 1 ঘণ্টা পর
  const timestamp = now - (4 - index) * 60 * 60 * 1000;

  // Traffic simulate
  for (let i = 0; i < scenario.requests; i++) {
    const success = Math.random() < scenario.successRate;
    const latency = success ? 100 + Math.random() * 300 : 500 + Math.random() * 2000;
    sli.record(success, latency, timestamp + i);
  }

  const errorRate = 1 - scenario.successRate;
  const burnRate = alerter.calculateBurnRate(errorRate);
  consumedTotal += errorRate * 60; // 1 ঘণ্টা = 60 মিনিট

  const remaining = budget.getRemainingPercentage(consumedTotal);
  const decision = policy.decide(remaining);

  console.log(`⏰ ${scenario.name}:`);
  console.log(`   Requests: ${scenario.requests.toLocaleString()}`);
  console.log(`   Success Rate: ${(scenario.successRate * 100).toFixed(2)}%`);
  console.log(`   Burn Rate: ${burnRate.toFixed(1)}x`);
  console.log(`   Budget বাকি: ${remaining.toFixed(1)}%`);
  console.log(`   Policy: ${decision.message}\n`);
});

// Latency SLI check
console.log('--- Latency SLI ---');
console.log(`   p50: ${sli.getPercentileLatency(50, now - 5*3600000, now).toFixed(0)}ms`);
console.log(`   p95: ${sli.getPercentileLatency(95, now - 5*3600000, now).toFixed(0)}ms`);
console.log(`   p99: ${sli.getPercentileLatency(99, now - 5*3600000, now).toFixed(0)}ms`);
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন SLO/Error Budget ব্যবহার করবেন

| পরিস্থিতি | কেন |
|-----------|-----|
| Production service চলছে | Reliability objectively measure করতে |
| Team feature vs reliability নিয়ে তর্ক করছে | Error budget data-driven সিদ্ধান্ত দেয় |
| On-call burnout হচ্ছে | Budget থাকলে alert কমানো যায় |
| Stakeholder-দের uptime promise দিতে হবে | SLA define করার ভিত্তি |
| Microservices architecture | প্রতিটি service-এর নিজস্ব SLO |
| ঈদ/বিশেষ দিনে capacity planning | Historical burn rate দেখে plan করা |

### ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|-----------|-----|
| নতুন product, user নেই | পর্যাপ্ত data নেই SLI measure করতে |
| Internal tool শুধু ৫ জন use করে | Overhead worth it না |
| 100% SLO সেট করা | অবাস্তব — কোনো budget থাকবে না |
| শুধু vanity metric হিসেবে | SLO action-এ convert না হলে অর্থহীন |
| SRE culture তৈরি হয়নি team-এ | আগে culture build করুন |

### 🎯 বাস্তব SLO সেট করার টিপস

```
❌ ভুল: "আমাদের SLO হবে 99.99%"
   (এর মানে মাসে শুধু 4.3 মিনিট downtime budget!)

✅ সঠিক: Historical data দেখুন → গত 3 মাসে actual availability কত?
   যদি actual 99.93% হয়, তাহলে SLO 99.9% সেট করুন।
   এতে improvement-এর জন্য pressure থাকবে কিন্তু অবাস্তব নয়।
```

### 📐 SLO বেছে নেওয়ার সূত্র

```
1. Critical path (payment) → 99.95% availability
2. Important (search, booking) → 99.9% availability
3. Non-critical (analytics, reports) → 99.5% availability
4. Internal tools → 99.0% availability
```
