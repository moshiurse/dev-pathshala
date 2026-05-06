# 🔄 Retry Strategies & Patterns

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Retry Storm সমস্যা](#retry-storm-সমস্যা)
- [Backoff Strategies](#backoff-strategies)
- [Jitter Algorithms](#jitter-algorithms)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Retry Pattern** হলো কোনো অপারেশন ব্যর্থ হলে নির্দিষ্ট কৌশল অনুসরণ করে পুনরায় চেষ্টা করা। Distributed system-এ transient failure (সাময়িক সমস্যা) খুব সাধারণ — network glitch, temporary overload, brief outage — এগুলো retry করলে প্রায়ই সফল হয়।

### কেন Retry দরকার?
- 🌐 Network unreliable: packet loss, timeout, connection reset
- 🔄 Transient failures: temporary database lock, brief service restart
- 📊 Load spikes: momentary overload, rate limiting (429)
- 🔧 Deployment: rolling update-এ brief downtime

### Retry-র ঝুঁকি:
- ⚡ Retry Storm: সবাই একসাথে retry করলে server overwhelm হয়
- 💀 Cascading failure: retry = additional load = more failure
- 🔁 Non-idempotent operations-এ duplicate action

### মূল নীতি:
> "Retry বুদ্ধিমত্তার সাথে করতে হবে — অন্ধভাবে নয়!"

---

## 🌍 বাস্তব জীবনের উদাহরণ

### bKash → Bank API Retry

bKash যখন Dutch-Bangla Bank-এর API call করে মোবাইল recharge-এর জন্য:

```
Scenario: গ্রাহক ১০০ টাকা recharge করছে

Attempt 1: Bank API → 503 Service Unavailable (bank server busy)
   ⏳ Wait 1 second...
   
Attempt 2: Bank API → Timeout (network glitch)
   ⏳ Wait 2 seconds...
   
Attempt 3: Bank API → 200 OK ✓ (সফল!)

Result: গ্রাহক জানেও না যে retry হয়েছে!
```

### Daraz Order Processing:
- Payment gateway call fail → 1s → retry → 2s → retry → success
- Inventory check timeout → retry with backoff
- Shipping API error → retry 3 times → fallback to manual queue

### Grameenphone MyGP:
- Balance check API timeout → immediate retry (transient)
- Recharge API 500 error → exponential backoff retry
- Bundle activation failure → retry with idempotency key

---

## ⚡ Retry Storm সমস্যা

### কী হয় Retry Storm-এ?

```
Normal Load: 1000 req/s → Server handles fine ✓

Server momentarily slow:
- 1000 requests fail
- All 1000 retry immediately = 2000 req/s load!
- More failures → more retries → 3000, 4000 req/s!
- Server completely overwhelmed 💀

"Retry storm" = Thundering herd problem
```

### সমাধান:
1. **Exponential Backoff**: ক্রমবর্ধমান delay
2. **Jitter**: Random variation যাতে সবাই একসাথে retry না করে
3. **Retry Budget**: Organization-wide retry limit
4. **Circuit Breaker**: বেশি failure-এ retry বন্ধ করা
5. **Retry-After Header**: Server বলে দেয় কখন retry করতে হবে

---

## 📈 Backoff Strategies

### 1. No Backoff (Immediate Retry)
```
Attempt: 1 → 2 → 3 → 4 → 5
Delay:   0    0    0    0    (কোনো wait নেই)
```
⚠️ ক্ষতিকর! সবচেয়ে aggressive, server-এ load বাড়ায়।

### 2. Fixed Delay
```
Attempt: 1 → 2 → 3 → 4 → 5
Delay:   1s   1s   1s   1s   (সব সমান)
```
সমস্যা: সবাই একসাথে retry করে (synchronized retries)।

### 3. Linear Backoff
```
Attempt: 1 →  2 →  3 →  4 →  5
Delay:   1s   2s   3s   4s   5s
```
মাঝারি — কিন্তু exponential-এর তুলনায় কম effective।

### 4. Exponential Backoff ⭐ (সবচেয়ে জনপ্রিয়)
```
Attempt: 1 →  2 →  3 →   4 →    5
Delay:   1s   2s   4s    8s    16s
Formula: base × 2^(attempt-1)
```
প্রতিবার delay দ্বিগুণ হয় — server-কে recover করার সময় দেয়।

### 5. Exponential Backoff with Jitter ⭐⭐ (সর্বোত্তম)
```
Attempt: 1 →     2 →      3 →       4
Delay:   0.7s   2.3s    5.1s     9.8s
Formula: random(0, base × 2^(attempt-1))
```
Random variation যোগ করে — retry storm প্রতিরোধ করে!

---

## 🎲 Jitter Algorithms

### 1. Full Jitter
```
delay = random(0, base × 2^attempt)
```
- সর্বোচ্চ spread
- কিছু retry প্রায় immediately, কিছু অনেক পরে
- Best for reducing correlation

### 2. Equal Jitter
```
half = (base × 2^attempt) / 2
delay = half + random(0, half)
```
- Minimum delay guarantee (অর্ধেক)
- অর্ধেক fixed + অর্ধেক random
- Balance between spread and minimum wait

### 3. Decorrelated Jitter
```
delay = random(base, previousDelay × 3)
```
- পূর্ববর্তী delay-এর উপর নির্ভর করে
- AWS recommended approach
- Aggressive spread without correlation

### তুলনা:

| Strategy | Min Delay | Max Delay | Spread | Best For |
|----------|-----------|-----------|--------|----------|
| Full Jitter | 0 | 2^n × base | Maximum | High contention |
| Equal Jitter | 2^(n-1) × base | 2^n × base | Medium | Balanced |
| Decorrelated | base | prev × 3 | Variable | AWS services |

---

## 📊 ASCII ডায়াগ্রাম

### Exponential Backoff Visualization:

```
Time ──────────────────────────────────────────────────────►

Immediate Retry (❌ Bad):
│R│R│R│R│R│  ← সব retry একসাথে, server overwhelm!
0  1  2  3  4

Fixed Delay (⚠️ OK):
│R│  │R│  │R│  │R│  │R│
0  1s  2s  3s  4s  5s

Exponential Backoff (✓ Good):
│R│ │R│    │R│          │R│                    │R│
0 1s  3s      7s              15s                    31s

Exponential + Jitter (⭐ Best):
│R││R│     │R│       │R│                │R│
0 0.7s  3.2s    6.8s          14.1s              28.5s
         (random variation — সবাই আলাদা সময়ে!)
```

### Retry Storm ও সমাধান:

```
    ═══ Without Jitter (Retry Storm) ═══

    Time: 0s     1s      2s      4s
          │      │       │       │
    C1:   ✗──────retry───retry───retry
    C2:   ✗──────retry───retry───retry
    C3:   ✗──────retry───retry───retry
    C4:   ✗──────retry───retry───retry
    C5:   ✗──────retry───retry───retry
                 ▲       ▲       ▲
                 │       │       │
           ALL AT ONCE! Server spikes! 💥


    ═══ With Full Jitter (Spread Out) ═══

    Time: 0s   0.5  1.0  1.5  2.0  2.5  3.0  3.5  4.0
          │     │    │    │    │    │    │    │    │
    C1:   ✗─────────retry────────────────retry───────
    C2:   ✗──────retry───────────retry───────────────
    C3:   ✗────────────retry──────────retry──────────
    C4:   ✗───retry─────────────────retry────────────
    C5:   ✗──────────────retry──────────────retry────
                 ▲
                 │
           SPREAD! Server handles smoothly ✓
```

### Retry with Circuit Breaker:

```
┌──────────────────────────────────────────────────────────┐
│                Retry + Circuit Breaker Flow                │
│                                                           │
│  Request                                                  │
│    │                                                      │
│    ▼                                                      │
│  ┌───────────────────┐                                    │
│  │  Circuit Breaker   │──── OPEN? ──── YES ──► Fail Fast │
│  │  State Check       │                         (no retry)│
│  └────────┬──────────┘                                    │
│           │ CLOSED/HALF-OPEN                              │
│           ▼                                               │
│  ┌───────────────────┐                                    │
│  │  Retry Logic       │                                   │
│  │                    │                                   │
│  │  attempt 1 ──► ✗  │                                   │
│  │  wait (backoff)    │                                   │
│  │  attempt 2 ──► ✗  │─── failure ──► CB failure count++ │
│  │  wait (backoff)    │                                   │
│  │  attempt 3 ──► ✓  │─── success ──► CB reset           │
│  │                    │                                   │
│  └───────────────────┘                                    │
│                                                           │
│  Rules:                                                   │
│  - CB OPEN → কোনো retry নেই (fail fast)                   │
│  - CB CLOSED → normal retry                              │
│  - CB HALF-OPEN → ১টি retry test                         │
│  - Max retries exhausted → CB failure count বাড়ে          │
└──────────────────────────────────────────────────────────┘
```

### Transient vs Permanent Failure Detection:

```
┌─────────────────────────────────────────────────────────┐
│           Error Classification for Retry                 │
│                                                          │
│  HTTP Response                                           │
│       │                                                  │
│       ├── 4xx (Client Error)                             │
│       │    ├── 400 Bad Request    → ❌ NEVER retry       │
│       │    ├── 401 Unauthorized   → ❌ NEVER retry       │
│       │    ├── 403 Forbidden      → ❌ NEVER retry       │
│       │    ├── 404 Not Found      → ❌ NEVER retry       │
│       │    ├── 408 Request Timeout→ ✅ RETRY             │
│       │    └── 429 Too Many Req   → ✅ RETRY (with R-A)  │
│       │                                                  │
│       ├── 5xx (Server Error)                             │
│       │    ├── 500 Internal Error → ⚠️ MAYBE retry       │
│       │    ├── 502 Bad Gateway    → ✅ RETRY             │
│       │    ├── 503 Unavailable    → ✅ RETRY             │
│       │    └── 504 Gateway Timeout→ ✅ RETRY             │
│       │                                                  │
│       └── Network Error                                  │
│            ├── Connection Refused → ✅ RETRY             │
│            ├── Connection Reset   → ✅ RETRY             │
│            ├── DNS Resolution Fail→ ⚠️ MAYBE retry       │
│            └── TLS Handshake Fail → ❌ NEVER retry       │
│                                                          │
│  R-A = Retry-After header                                │
└─────────────────────────────────────────────────────────┘
```

### Retry Budget Concept:

```
┌────────────────────────────────────────────────────┐
│           Retry Budget (Organization Level)         │
│                                                     │
│  Total Request Budget per second: 1000              │
│  Retry Budget: 10% = 100 retries/second max        │
│                                                     │
│  ┌─────────────────────────────────────────┐        │
│  │████████████████████████████████████░░░░░│        │
│  │◄── Original Requests: 900 ──►│◄─ 100 ─►│        │
│  │                               │ retries │        │
│  └─────────────────────────────────────────┘        │
│                                                     │
│  Rule: Total retries ≤ 10% of total traffic         │
│                                                     │
│  যদি retry budget exhausted:                        │
│  → নতুন retry allowed নয়                           │
│  → Fail immediately                                │
│  → Alert: "System under stress"                    │
└────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Retry Strategies Implementation
 * bKash Bank API Integration
 */

/**
 * Retry Configuration
 */
class RetryConfig
{
    public int $maxAttempts;
    public int $baseDelayMs;
    public int $maxDelayMs;
    public string $backoffStrategy; // 'exponential', 'linear', 'fixed'
    public string $jitterStrategy;  // 'full', 'equal', 'decorrelated', 'none'
    public array $retryableErrors;
    public ?float $retryBudgetPercent;

    public function __construct(array $options = [])
    {
        $this->maxAttempts = $options['maxAttempts'] ?? 3;
        $this->baseDelayMs = $options['baseDelayMs'] ?? 1000;
        $this->maxDelayMs = $options['maxDelayMs'] ?? 30000;
        $this->backoffStrategy = $options['backoffStrategy'] ?? 'exponential';
        $this->jitterStrategy = $options['jitterStrategy'] ?? 'full';
        $this->retryableErrors = $options['retryableErrors'] ?? [408, 429, 500, 502, 503, 504];
        $this->retryBudgetPercent = $options['retryBudgetPercent'] ?? 10.0;
    }
}

/**
 * Retry Engine with multiple strategies
 */
class RetryEngine
{
    private RetryConfig $config;
    private int $totalAttempts = 0;
    private int $totalRetries = 0;
    private float $lastDelay = 0;

    public function __construct(RetryConfig $config)
    {
        $this->config = $config;
    }

    /**
     * মূল retry execution
     */
    public function execute(callable $operation, ?string $idempotencyKey = null): mixed
    {
        $attempt = 0;
        $lastException = null;

        while ($attempt < $this->config->maxAttempts) {
            $attempt++;
            $this->totalAttempts++;

            try {
                echo "  🔄 Attempt {$attempt}/{$this->config->maxAttempts}";
                if ($attempt > 1) {
                    echo " (retry after {$this->lastDelay}ms)";
                }
                echo "\n";

                $result = $operation($attempt, $idempotencyKey);
                
                if ($attempt > 1) {
                    echo "  ✅ সফল! (attempt {$attempt}-এ)\n";
                }
                return $result;

            } catch (\Exception $e) {
                $lastException = $e;

                if (!$this->isRetryable($e)) {
                    echo "  ❌ Permanent failure (retry করা যাবে না): {$e->getMessage()}\n";
                    throw $e;
                }

                if ($attempt >= $this->config->maxAttempts) {
                    echo "  💀 Max attempts exhausted!\n";
                    break;
                }

                // Check retry budget
                if (!$this->hasRetryBudget()) {
                    echo "  🚫 Retry budget exhausted!\n";
                    break;
                }

                $delay = $this->calculateDelay($attempt);
                $this->lastDelay = $delay;
                $this->totalRetries++;

                echo "  ⚠️ Failed: {$e->getMessage()}\n";
                echo "  ⏳ Waiting {$delay}ms before retry...\n";

                usleep((int) ($delay * 1000));
            }
        }

        throw $lastException ?? new \RuntimeException("All retry attempts failed");
    }

    /**
     * Delay calculation based on strategy
     */
    private function calculateDelay(int $attempt): float
    {
        $baseDelay = $this->calculateBaseDelay($attempt);
        $delayWithJitter = $this->applyJitter($baseDelay, $attempt);
        
        return min($delayWithJitter, $this->config->maxDelayMs);
    }

    /**
     * Base delay calculation (strategy-based)
     */
    private function calculateBaseDelay(int $attempt): float
    {
        return match ($this->config->backoffStrategy) {
            'exponential' => $this->config->baseDelayMs * pow(2, $attempt - 1),
            'linear' => $this->config->baseDelayMs * $attempt,
            'fixed' => $this->config->baseDelayMs,
            default => $this->config->baseDelayMs * pow(2, $attempt - 1),
        };
    }

    /**
     * Jitter application
     */
    private function applyJitter(float $baseDelay, int $attempt): float
    {
        return match ($this->config->jitterStrategy) {
            'full' => $this->fullJitter($baseDelay),
            'equal' => $this->equalJitter($baseDelay),
            'decorrelated' => $this->decorrelatedJitter($baseDelay),
            'none' => $baseDelay,
            default => $this->fullJitter($baseDelay),
        };
    }

    /** Full Jitter: random(0, baseDelay) */
    private function fullJitter(float $baseDelay): float
    {
        return mt_rand(0, (int) $baseDelay);
    }

    /** Equal Jitter: half + random(0, half) */
    private function equalJitter(float $baseDelay): float
    {
        $half = $baseDelay / 2;
        return $half + mt_rand(0, (int) $half);
    }

    /** Decorrelated Jitter: random(base, lastDelay * 3) */
    private function decorrelatedJitter(float $baseDelay): float
    {
        $min = $this->config->baseDelayMs;
        $max = max($min, $this->lastDelay * 3);
        return mt_rand((int) $min, (int) $max);
    }

    /**
     * Error retryable কিনা check
     */
    private function isRetryable(\Exception $e): bool
    {
        // HTTP status code check
        if (method_exists($e, 'getStatusCode')) {
            return in_array($e->getStatusCode(), $this->config->retryableErrors);
        }

        // Network errors সবসময় retryable
        if ($e instanceof \RuntimeException && 
            str_contains($e->getMessage(), 'timeout')) {
            return true;
        }

        if ($e instanceof \RuntimeException && 
            str_contains($e->getMessage(), 'connection')) {
            return true;
        }

        // 4xx errors → not retryable
        if ($e instanceof HttpClientException) {
            return false;
        }

        return true; // Default: retry
    }

    /**
     * Retry budget check
     */
    private function hasRetryBudget(): bool
    {
        if ($this->totalAttempts === 0) return true;
        
        $retryPercent = ($this->totalRetries / $this->totalAttempts) * 100;
        return $retryPercent < $this->config->retryBudgetPercent;
    }

    public function getStats(): array
    {
        return [
            'totalAttempts' => $this->totalAttempts,
            'totalRetries' => $this->totalRetries,
            'retryRate' => $this->totalAttempts > 0
                ? round(($this->totalRetries / $this->totalAttempts) * 100, 1) . '%'
                : '0%',
        ];
    }
}

/**
 * HTTP Exception with status code
 */
class HttpException extends \RuntimeException
{
    private int $statusCode;

    public function __construct(int $statusCode, string $message = '')
    {
        $this->statusCode = $statusCode;
        parent::__construct($message ?: "HTTP Error {$statusCode}");
    }

    public function getStatusCode(): int
    {
        return $this->statusCode;
    }
}

class HttpClientException extends HttpException {}

/**
 * bKash Bank API Client with Retry
 */
class BkashBankApiClient
{
    private RetryEngine $retryEngine;
    private int $callCount = 0;

    public function __construct()
    {
        $config = new RetryConfig([
            'maxAttempts' => 4,
            'baseDelayMs' => 1000,
            'maxDelayMs' => 15000,
            'backoffStrategy' => 'exponential',
            'jitterStrategy' => 'full',
            'retryableErrors' => [408, 429, 500, 502, 503, 504],
        ]);

        $this->retryEngine = new RetryEngine($config);
    }

    /**
     * Bank fund transfer with idempotency
     */
    public function transferFund(float $amount, string $from, string $to): array
    {
        $idempotencyKey = md5("{$from}_{$to}_{$amount}_" . date('Y-m-d-H'));

        echo "🏦 Fund Transfer: ৳{$amount} ({$from} → {$to})\n";
        echo "   Idempotency Key: {$idempotencyKey}\n\n";

        $result = $this->retryEngine->execute(
            function (int $attempt, ?string $key) use ($amount, $from, $to) {
                return $this->callBankApi($amount, $from, $to, $key);
            },
            $idempotencyKey
        );

        echo "\n📊 Retry Stats: " . json_encode($this->retryEngine->getStats()) . "\n";
        return $result;
    }

    /**
     * Simulated Bank API call (randomly fails)
     */
    private function callBankApi(float $amount, string $from, string $to, ?string $idempotencyKey): array
    {
        $this->callCount++;
        
        // Simulate failures (first 2 calls fail, 3rd succeeds)
        $random = mt_rand(1, 100);
        
        if ($this->callCount <= 2) {
            $errorType = mt_rand(1, 3);
            match ($errorType) {
                1 => throw new HttpException(503, 'Bank service temporarily unavailable'),
                2 => throw new \RuntimeException('Connection timeout after 10000ms'),
                3 => throw new HttpException(502, 'Bad gateway - upstream error'),
            };
        }

        return [
            'status' => 'success',
            'transactionId' => 'BNK_' . uniqid(),
            'amount' => $amount,
            'from' => $from,
            'to' => $to,
            'idempotencyKey' => $idempotencyKey,
            'timestamp' => date('Y-m-d H:i:s'),
        ];
    }
}

// ========== ব্যবহার ==========

echo "🔄 ═══ bKash Retry Strategy Demo ═══\n\n";

$client = new BkashBankApiClient();

try {
    $result = $client->transferFund(5000, '01712345678', '01898765432');
    echo "\n✅ Transfer সফল!\n";
    echo "   Transaction ID: {$result['transactionId']}\n";
    echo "   Amount: ৳{$result['amount']}\n";
} catch (\Exception $e) {
    echo "\n❌ Transfer ব্যর্থ: {$e->getMessage()}\n";
}
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Advanced Retry Strategies
 * bKash Payment System with Multiple Strategies
 */

class RetryConfig {
    constructor(options = {}) {
        this.maxAttempts = options.maxAttempts || 3;
        this.baseDelayMs = options.baseDelayMs || 1000;
        this.maxDelayMs = options.maxDelayMs || 30000;
        this.backoff = options.backoff || 'exponential'; // exponential, linear, fixed
        this.jitter = options.jitter || 'full';         // full, equal, decorrelated, none
        this.retryableStatusCodes = options.retryableStatusCodes || [408, 429, 500, 502, 503, 504];
        this.retryBudgetPercent = options.retryBudgetPercent || 10;
        this.onRetry = options.onRetry || null;         // callback on each retry
    }
}

/**
 * Retry Engine — সকল strategy support করে
 */
class RetryEngine {
    constructor(config) {
        this.config = config instanceof RetryConfig ? config : new RetryConfig(config);
        this.metrics = {
            totalAttempts: 0,
            totalRetries: 0,
            totalSuccess: 0,
            totalFailure: 0,
            delays: [],
        };
        this.lastDelay = this.config.baseDelayMs;
    }

    /**
     * Retry-protected execution
     */
    async execute(operation, context = {}) {
        let lastError = null;

        for (let attempt = 1; attempt <= this.config.maxAttempts; attempt++) {
            this.metrics.totalAttempts++;

            try {
                const result = await operation(attempt, context);
                this.metrics.totalSuccess++;

                if (attempt > 1) {
                    console.log(`   ✅ সফল! (attempt ${attempt})`);
                }
                return result;

            } catch (error) {
                lastError = error;

                // Permanent error → retry করবে না
                if (!this.isRetryable(error)) {
                    console.log(`   ❌ Permanent error: ${error.message}`);
                    this.metrics.totalFailure++;
                    throw error;
                }

                // Last attempt → throw
                if (attempt >= this.config.maxAttempts) {
                    console.log(`   💀 Max attempts (${this.config.maxAttempts}) exhausted`);
                    this.metrics.totalFailure++;
                    break;
                }

                // Retry budget check
                if (!this.hasRetryBudget()) {
                    console.log(`   🚫 Retry budget (${this.config.retryBudgetPercent}%) exhausted`);
                    break;
                }

                // Calculate delay
                const delay = this.calculateDelay(attempt, error);
                this.lastDelay = delay;
                this.metrics.totalRetries++;
                this.metrics.delays.push(delay);

                console.log(`   ⚠️ Attempt ${attempt} failed: ${error.message}`);
                console.log(`   ⏳ Retry in ${Math.round(delay)}ms (${this.config.backoff} + ${this.config.jitter} jitter)`);

                // Retry-After header respect করা
                const retryAfter = this.getRetryAfterMs(error);
                const effectiveDelay = retryAfter ? Math.max(delay, retryAfter) : delay;

                if (retryAfter) {
                    console.log(`   📋 Server Retry-After: ${retryAfter}ms`);
                }

                // Callback
                if (this.config.onRetry) {
                    this.config.onRetry({ attempt, error, delay: effectiveDelay });
                }

                await this.sleep(effectiveDelay);
            }
        }

        throw lastError;
    }

    /**
     * Delay calculation
     */
    calculateDelay(attempt, error) {
        const baseDelay = this.calculateBaseDelay(attempt);
        const jitteredDelay = this.applyJitter(baseDelay, attempt);
        return Math.min(jitteredDelay, this.config.maxDelayMs);
    }

    calculateBaseDelay(attempt) {
        switch (this.config.backoff) {
            case 'exponential':
                return this.config.baseDelayMs * Math.pow(2, attempt - 1);
            case 'linear':
                return this.config.baseDelayMs * attempt;
            case 'fixed':
                return this.config.baseDelayMs;
            default:
                return this.config.baseDelayMs * Math.pow(2, attempt - 1);
        }
    }

    /**
     * Jitter Algorithms
     */
    applyJitter(baseDelay, attempt) {
        switch (this.config.jitter) {
            case 'full':
                // random(0, baseDelay) — Maximum spread
                return Math.random() * baseDelay;

            case 'equal':
                // half + random(0, half) — Guaranteed minimum
                const half = baseDelay / 2;
                return half + Math.random() * half;

            case 'decorrelated':
                // random(base, lastDelay * 3) — AWS recommended
                const min = this.config.baseDelayMs;
                const max = Math.max(min, this.lastDelay * 3);
                return min + Math.random() * (max - min);

            case 'none':
                return baseDelay;

            default:
                return Math.random() * baseDelay;
        }
    }

    /**
     * Retryable error detection
     */
    isRetryable(error) {
        // HTTP status code check
        if (error.statusCode) {
            return this.config.retryableStatusCodes.includes(error.statusCode);
        }

        // Network errors → retryable
        if (error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT' ||
            error.code === 'ECONNREFUSED' || error.code === 'EPIPE') {
            return true;
        }

        // Timeout errors → retryable
        if (error.name === 'TimeoutError' || error.message.includes('timeout')) {
            return true;
        }

        // DNS errors → maybe not retryable
        if (error.code === 'ENOTFOUND') {
            return false;
        }

        return true; // Default: retry
    }

    /**
     * Retry-After header parse
     */
    getRetryAfterMs(error) {
        if (!error.headers || !error.headers['retry-after']) return null;

        const retryAfter = error.headers['retry-after'];
        
        // Seconds format
        if (/^\d+$/.test(retryAfter)) {
            return parseInt(retryAfter) * 1000;
        }

        // Date format
        const date = new Date(retryAfter);
        if (!isNaN(date.getTime())) {
            return Math.max(0, date.getTime() - Date.now());
        }

        return null;
    }

    hasRetryBudget() {
        if (this.metrics.totalAttempts === 0) return true;
        const retryPercent = (this.metrics.totalRetries / this.metrics.totalAttempts) * 100;
        return retryPercent < this.config.retryBudgetPercent;
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    getMetrics() {
        return {
            ...this.metrics,
            retryRate: this.metrics.totalAttempts > 0
                ? `${((this.metrics.totalRetries / this.metrics.totalAttempts) * 100).toFixed(1)}%`
                : '0%',
            avgDelay: this.metrics.delays.length > 0
                ? `${Math.round(this.metrics.delays.reduce((a, b) => a + b, 0) / this.metrics.delays.length)}ms`
                : '0ms',
        };
    }
}

/**
 * HTTP-like Error class
 */
class HttpError extends Error {
    constructor(statusCode, message, headers = {}) {
        super(message || `HTTP ${statusCode}`);
        this.statusCode = statusCode;
        this.headers = headers;
        this.name = 'HttpError';
    }
}

/**
 * bKash Bank Integration with Retry
 */
class BkashBankClient {
    constructor() {
        this.retryEngine = new RetryEngine({
            maxAttempts: 4,
            baseDelayMs: 1000,
            maxDelayMs: 16000,
            backoff: 'exponential',
            jitter: 'full',
            retryableStatusCodes: [408, 429, 500, 502, 503, 504],
            retryBudgetPercent: 20,
            onRetry: ({ attempt, error, delay }) => {
                console.log(`   📊 Retry metric: attempt=${attempt}, delay=${Math.round(delay)}ms`);
            }
        });
        this.callCount = 0;
    }

    /**
     * Fund Transfer with Idempotency Key
     */
    async transfer(amount, fromAccount, toAccount) {
        const idempotencyKey = this.generateIdempotencyKey(amount, fromAccount, toAccount);

        console.log(`\n🏦 ═══ Bank Transfer ═══`);
        console.log(`   Amount: ৳${amount.toLocaleString()}`);
        console.log(`   From: ${fromAccount} → To: ${toAccount}`);
        console.log(`   Idempotency: ${idempotencyKey}`);
        console.log('');

        try {
            const result = await this.retryEngine.execute(
                async (attempt, ctx) => {
                    return await this.callBankApi(amount, fromAccount, toAccount, idempotencyKey, attempt);
                },
                { amount, fromAccount, toAccount }
            );

            console.log(`\n   ✅ Transfer Complete!`);
            console.log(`   Transaction: ${result.transactionId}`);
            console.log(`   📊 Metrics:`, this.retryEngine.getMetrics());
            return result;

        } catch (error) {
            console.log(`\n   ❌ Transfer Failed: ${error.message}`);
            console.log(`   📊 Metrics:`, this.retryEngine.getMetrics());
            throw error;
        }
    }

    /**
     * Simulated Bank API (fails first 2 attempts)
     */
    async callBankApi(amount, from, to, idempotencyKey, attempt) {
        this.callCount++;

        // Simulate network delay
        await new Promise(r => setTimeout(r, Math.random() * 500 + 100));

        // Simulate failures
        if (this.callCount <= 2) {
            const errors = [
                new HttpError(503, 'Bank service temporarily unavailable'),
                new HttpError(502, 'Bad gateway'),
                (() => { const e = new Error('Connection timeout'); e.code = 'ETIMEDOUT'; return e; })(),
                new HttpError(429, 'Rate limited', { 'retry-after': '2' }),
            ];
            throw errors[this.callCount % errors.length];
        }

        // Success
        return {
            transactionId: `TXN_${Date.now()}_${Math.random().toString(36).substr(2, 8)}`,
            amount,
            from,
            to,
            idempotencyKey,
            status: 'completed',
            timestamp: new Date().toISOString(),
        };
    }

    generateIdempotencyKey(amount, from, to) {
        const data = `${from}_${to}_${amount}_${new Date().toISOString().split('T')[0]}`;
        let hash = 0;
        for (let i = 0; i < data.length; i++) {
            const char = data.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash = hash & hash;
        }
        return `idem_${Math.abs(hash).toString(36)}`;
    }
}

/**
 * Retry Strategy Comparison Demo
 */
async function compareStrategies() {
    console.log('\n📊 ═══ Retry Strategy তুলনা ═══\n');

    const strategies = [
        { name: 'Exponential + Full Jitter', backoff: 'exponential', jitter: 'full' },
        { name: 'Exponential + Equal Jitter', backoff: 'exponential', jitter: 'equal' },
        { name: 'Exponential + No Jitter', backoff: 'exponential', jitter: 'none' },
        { name: 'Linear + Full Jitter', backoff: 'linear', jitter: 'full' },
        { name: 'Fixed + No Jitter', backoff: 'fixed', jitter: 'none' },
    ];

    for (const strategy of strategies) {
        const engine = new RetryEngine({
            maxAttempts: 5,
            baseDelayMs: 1000,
            backoff: strategy.backoff,
            jitter: strategy.jitter,
        });

        const delays = [];
        for (let attempt = 1; attempt <= 5; attempt++) {
            const delay = engine.calculateDelay(attempt, {});
            engine.lastDelay = delay;
            delays.push(Math.round(delay));
        }

        console.log(`  ${strategy.name}:`);
        console.log(`    Delays: [${delays.join('ms, ')}ms]`);
        console.log(`    Total: ${delays.reduce((a, b) => a + b, 0)}ms`);
        console.log('');
    }
}

/**
 * Coordinated Retry Avoidance
 * একাধিক client-এর retry synchronize না হওয়া নিশ্চিত করা
 */
class CoordinatedRetryManager {
    constructor() {
        this.activeRetries = new Map(); // endpoint → count
        this.retryBudget = 100; // max retries per second globally
        this.currentSecondRetries = 0;
        this.lastResetTime = Date.now();
    }

    /**
     * Global retry budget check
     */
    canRetry(endpoint) {
        this.resetBudgetIfNeeded();

        if (this.currentSecondRetries >= this.retryBudget) {
            console.log(`   🚫 Global retry budget exhausted (${this.retryBudget}/s)`);
            return false;
        }

        const endpointRetries = this.activeRetries.get(endpoint) || 0;
        if (endpointRetries >= 10) {
            console.log(`   🚫 Endpoint retry limit reached: ${endpoint}`);
            return false;
        }

        return true;
    }

    recordRetry(endpoint) {
        this.currentSecondRetries++;
        const current = this.activeRetries.get(endpoint) || 0;
        this.activeRetries.set(endpoint, current + 1);
    }

    resetBudgetIfNeeded() {
        if (Date.now() - this.lastResetTime >= 1000) {
            this.currentSecondRetries = 0;
            this.activeRetries.clear();
            this.lastResetTime = Date.now();
        }
    }
}

// ========== ব্যবহার ==========

async function main() {
    // Strategy comparison
    await compareStrategies();

    // bKash bank transfer with retry
    const client = new BkashBankClient();

    try {
        const result = await client.transfer(5000, '01712345678', '01898765432');
        console.log('\n🎉 সফল!', result.status);
    } catch (error) {
        console.log('\n💥 সব retry ব্যর্থ:', error.message);
    }
}

main().catch(console.error);
```

---

## 🛡️ Idempotency — নিরাপদ Retry-র ভিত্তি

### কেন Idempotency দরকার:
```
Scenario: bKash ৫,০০০ টাকা transfer

Attempt 1: Request sent → Response timeout (কিন্তু bank-এ টাকা কাটা হয়ে গেছে!)
Attempt 2: Same request again...
  - Without idempotency: ৫,০০০ আবার কেটে নিল! মোট ১০,০০০ কাটা! 😱
  - With idempotency: Bank দেখে key already processed → "already done" response ✓
```

### Idempotency Key Strategy:
```php
// Good: Unique per logical operation
$key = md5("{$sender}_{$receiver}_{$amount}_{$requestId}");

// Bad: Changes on every call
$key = md5(time() . random_bytes(16)); // প্রতিবার আলাদা = no protection!
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন:
- 🌐 External API calls (Bank API, Payment gateway)
- 🔄 Transient network failures (timeout, connection reset)
- 📡 HTTP 5xx errors (server temporary issue)
- 429 Rate limiting (with Retry-After header)
- 🗄️ Database deadlock/lock timeout
- ☁️ Cloud service API calls (AWS, GCP — SDK-তে built-in)

### ❌ ব্যবহার করবেন না:
- 🔐 Authentication failure (401, 403) — credentials ভুল, retry-তে ঠিক হবে না
- 📝 Validation error (400) — input ভুল, retry-তে ঠিক হবে না
- 🚫 Business logic error (insufficient balance) — permanent failure
- 📄 404 Not Found — resource নেই, retry-তে আসবে না
- 🔁 Non-idempotent operations WITHOUT idempotency key
- 💀 System completely down (Circuit Breaker use করুন)

### 🎯 Retry Configuration Guidelines:

```
Service Type          │ Max Attempts │ Base Delay │ Strategy
──────────────────────┼──────────────┼────────────┼──────────────────────
Bank API              │ 3-5          │ 1000ms     │ Exponential + Full Jitter
Internal Microservice │ 2-3          │ 200ms      │ Exponential + Equal Jitter
Database              │ 2-3          │ 100ms      │ Fixed (deadlock)
Message Queue         │ 5-10         │ 500ms      │ Exponential + Decorrelated
CDN/Static            │ 2            │ 500ms      │ Fixed
Third-party API       │ 3            │ 1000ms     │ Exponential + Full Jitter
```

### ⚠️ সাধারণ ভুল:

1. **Retry without backoff**: Server-এ আরও load দেয়
2. **Retry without jitter**: সব client একসাথে retry করে (storm!)
3. **Retry non-idempotent ops**: Duplicate actions হয়
4. **No max attempts**: Infinite retry = resource waste
5. **Retry permanent errors**: ৪০১ retry করে কোনো লাভ নেই
6. **No retry budget**: System-wide retry storm
7. **Ignoring Retry-After**: Server বলছে "পরে এসো" — শুনুন!

---

## 📚 সারসংক্ষেপ

Retry হলো distributed system-এর সবচেয়ে সাধারণ resilience pattern। কিন্তু "smart retry" এবং "dumb retry"-র মধ্যে পার্থক্য হলো:

| Dumb Retry | Smart Retry |
|------------|-------------|
| Immediate retry | Exponential backoff |
| No jitter | Full/decorrelated jitter |
| No limit | Max attempts + budget |
| All errors retry | Only transient errors |
| No idempotency | Idempotency key |
| No circuit breaker | CB integration |

**মনে রাখুন**: "Retry হলো ওষুধ — সঠিক ডোজে ভালো করে, বেশি ডোজে ক্ষতি করে!" 💊
