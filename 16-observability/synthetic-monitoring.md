# 🤖 Synthetic Monitoring & Proactive Testing

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🧠 সংজ্ঞা ও ধারণা

### Synthetic Monitoring কী?

Synthetic Monitoring হলো **সিমুলেটেড user behavior** দিয়ে আপনার সিস্টেম continuously test করা — real user আসার আগেই সমস্যা ধরা।

> কল্পনা করুন: আপনার bKash app-এ প্রতি ৫ মিনিটে একটি "ভুয়া" Send Money request পাঠানো হচ্ছে শুধু check করতে — system কাজ করছে কিনা।

### Synthetic vs Real User Monitoring (RUM)

| বিষয় | Synthetic Monitoring | Real User Monitoring (RUM) |
|--------|---------------------|---------------------------|
| **কে test করে** | Bot/Script | সত্যিকার user |
| **কখন চলে** | ২৪/৭, scheduled | User visit করলে |
| **Traffic লাগে?** | না | হ্যাঁ |
| **Baseline detect** | সহজ (controlled) | কঠিন (variable) |
| **Geography test** | Multiple locations থেকে | User যেখানে আছে |
| **Off-peak detect** | ✅ রাত ৩টায়ও ধরবে | ❌ User নেই তো data নেই |
| **Real experience** | Approximation | সত্যিকার experience |

### Synthetic Check-এর প্রকারভেদ

#### 1. HTTP/API Checks
সবচেয়ে সহজ — একটি endpoint-এ request পাঠিয়ে response check।

```
GET https://api.bkash.com/health → 200 OK? < 500ms?
```

#### 2. Browser Checks
Headless browser (Puppeteer/Playwright) দিয়ে full page load simulate।

```
Chrome launch → bkash.com load → Login → Check balance → Assert amount visible
```

#### 3. Multi-Step API Transactions
একটি সম্পূর্ণ business flow API level-এ test।

```
Step 1: Login → Get token
Step 2: Check balance → Assert > 100 TK
Step 3: Send Money → Assert success
Step 4: Verify transaction history
```

#### 4. DNS/TCP/SSL Checks
Infrastructure level health — DNS resolve হচ্ছে? SSL expire হবে?

### 🌍 Global Monitoring — Multiple Locations

**কেন দরকার?** 
- Dhaka-তে কাজ করতে পারে কিন্তু Chittagong-এ CDN issue থাকতে পারে
- ISP-specific routing problem detect
- Regional outage তাড়াতাড়ি ধরা

**সাধারণ locations:**
- Dhaka (primary)
- Chittagong
- Singapore (nearest cloud region)
- Mumbai (AWS/GCP region)

### 🐤 Canary Deployments + Synthetic Tests

Canary deploy করার পর synthetic test দিয়ে verify:

```
1. Deploy new version to 5% traffic (canary)
2. Run synthetic tests against canary endpoint
3. Compare canary metrics vs stable metrics
4. ✅ Metrics ভালো → Gradually increase traffic
5. ❌ Synthetic fails → Auto-rollback
```

### 🔇 Alert Fatigue Management

Synthetic monitoring-এ false positives common। Alert fatigue কমাতে:

1. **Retry before alerting:** 1 failure = retry, 3 consecutive failures = alert
2. **Multi-location confirmation:** Dhaka + Chittagong দুটোই fail → alert
3. **Maintenance windows:** Scheduled deploy-এর সময় alert mute
4. **Severity levels:** Health check fail ≠ critical, transaction fail = critical
5. **Escalation policy:** 5 min → Slack, 15 min → SMS, 30 min → Phone call

### 🛠️ Tools Overview

| Tool | বৈশিষ্ট্য | খরচ |
|------|-----------|------|
| Grafana Synthetic | OSS-friendly, k6 integration | Free (self-host) |
| AWS CloudWatch Synthetics | Canary scripts, Lambda-based | Pay per run |
| Datadog Synthetics | Browser + API, CI integration | $$$ |
| Uptime Robot | Simple HTTP checks | Free tier |
| Checkly | Playwright-based, "Monitoring as Code" | Mid-range |

### 📝 Synthetic Test as Code

Modern approach — tests version control-এ রাখা:

```
repo/
├── synthetic-tests/
│   ├── bkash-send-money.test.js
│   ├── bkash-cashout.test.js
│   ├── bkash-payment.test.js
│   └── config.yaml
├── .github/
│   └── workflows/
│       └── deploy-synthetics.yml
```

Deploy pipeline-এ synthetic tests-ও deploy হয়। Code review হয়। Version history থাকে।

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏦 bKash Send Money — Continuous Synthetic Testing

**সমস্যা:** ঈদের রাতে Send Money service down হয়ে গিয়েছিল। User complain আসার ১৫ মিনিট পর team জানতে পারলো। ততক্ষণে হাজার হাজার failed transaction।

**সমাধান:** Synthetic monitoring সেটআপ

**Test Scenario: "Send Money Flow"**

```
প্রতি ৫ মিনিটে execute হবে, ঢাকা ও চট্টগ্রাম থেকে:

Step 1: POST /api/auth/login
        Body: { phone: "01700000001", pin: "12345" }
        Assert: status 200, token received

Step 2: GET /api/wallet/balance
        Header: Authorization: Bearer {token}
        Assert: status 200, balance >= 10

Step 3: POST /api/transaction/send
        Body: { to: "01700000002", amount: 10, pin: "12345" }
        Assert: status 200, trx_id exists

Step 4: GET /api/transaction/{trx_id}
        Assert: status 200, status == "completed"

Timing Assertions:
  - Total flow: < 3000ms
  - Individual step: < 1000ms

Alert Policy:
  - 2 consecutive failures from same location → WARNING (Slack)
  - 2 consecutive failures from ALL locations → CRITICAL (PagerDuty)
  - Single failure → Retry after 30s, no alert
```

**ফলাফল:** 
- পরের বার যখন payment gateway slow হলো, ৩০ সেকেন্ডের মধ্যে alert পাওয়া গেলো
- User complain আসার আগেই team action নিলো
- MTTR (Mean Time To Recovery) 15 মিনিট থেকে 3 মিনিটে নামলো

### 📱 Daraz Bangladesh — E-commerce Checkout Monitoring

**Browser-based Synthetic Test:**
```
প্রতি ১০ মিনিটে Chrome Headless দিয়ে:

1. daraz.com.bd open → page load < 3s
2. Search "mobile phone" → results appear < 2s
3. Click first product → product page loads < 2s
4. Add to cart → cart updated
5. Proceed to checkout → checkout page loads
6. Verify payment options visible (bKash, Nagad, COD)

Locations: Dhaka, Chittagong, Sylhet
```

### 📰 Prothom Alo — Content Availability

```
প্রতি ২ মিনিটে:
1. Homepage load → Assert breaking news section exists
2. Article page → Assert content renders, ads load
3. Mobile API → Assert JSON response valid, articles > 0
4. CDN check → Asset from CDN loads < 200ms
```

---

## 📊 ASCII ডায়াগ্রাম

### Synthetic Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  Synthetic Monitoring System                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │  📍 Dhaka  │  │📍Chittagong│  │📍 Singapore│  ← Test Probes │
│  │  Probe     │  │  Probe     │  │  Probe     │                │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                │
│        │               │               │                        │
│        └───────────────┼───────────────┘                        │
│                        │                                         │
│                        ▼                                         │
│         ┌──────────────────────────────┐                        │
│         │     Test Orchestrator         │                        │
│         │  (Schedule, Execute, Collect) │                        │
│         └──────────────┬───────────────┘                        │
│                        │                                         │
│              ┌─────────┼─────────┐                              │
│              │         │         │                               │
│              ▼         ▼         ▼                               │
│  ┌──────────────┐ ┌────────┐ ┌──────────┐                      │
│  │ HTTP Checks  │ │Browser │ │Multi-Step│                       │
│  │ (API health) │ │Checks  │ │Flows     │                       │
│  └──────┬───────┘ └───┬────┘ └────┬─────┘                      │
│         │             │           │                              │
│         └─────────────┼───────────┘                              │
│                       │                                          │
│                       ▼                                          │
│         ┌──────────────────────────────┐                        │
│         │      Results Collector        │                        │
│         │  ┌────────┐ ┌─────────────┐  │                        │
│         │  │Pass/Fail│ │ Timing Data │  │                        │
│         │  └────────┘ └─────────────┘  │                        │
│         └──────────────┬───────────────┘                        │
│                        │                                         │
│              ┌─────────┼─────────┐                              │
│              ▼         ▼         ▼                               │
│     ┌─────────┐  ┌─────────┐  ┌───────────┐                    │
│     │📊Metrics│  │🚨Alerts │  │📈Dashboard│                    │
│     │  Store  │  │ Engine  │  │ (Grafana) │                    │
│     └─────────┘  └─────────┘  └───────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Synthetic vs RUM Coverage

```
           24-Hour Timeline
           
Hour:  0  2  4  6  8  10 12 14 16 18 20 22 24
       |  |  |  |  |  |  |  |  |  |  |  |  |

RUM:   ░░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░
       ↑ কম user        ↑ peak hours         ↑ কম
       (blind spot!)     (ভালো coverage)      (blind!)

Synth: ████████████████████████████████████████
       ↑ প্রতি ৫ মিনিটে test — ২৪/৭ সমান coverage

       Legend: █ = monitored, ▓ = heavy data, ░ = sparse data

       ⚠️ রাত ২টায় server crash? 
          RUM: কেউ জানবে না (user নেই)
          Synthetic: ৫ মিনিটের মধ্যে alert!
```

### Alert Fatigue Prevention Flow

```
┌───────────────────────────────────────────────────────────┐
│           Alert Fatigue Prevention Pipeline                 │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  Synthetic Test Failure                                    │
│         │                                                  │
│         ▼                                                  │
│  ┌──────────────┐                                         │
│  │ Retry (30s)  │──→ Pass? ──→ No alert, log only ✅     │
│  └──────┬───────┘                                         │
│         │ Fail again                                       │
│         ▼                                                  │
│  ┌──────────────────────────────┐                         │
│  │ Check other locations        │                         │
│  │ Dhaka ❌ Chittagong ✅       │──→ Local ISP issue      │
│  │                              │    → Low severity ⚠️    │
│  │ Dhaka ❌ Chittagong ❌       │──→ Real outage!        │
│  │                              │    → High severity 🔴   │
│  └──────────────┬───────────────┘                         │
│                 │                                          │
│         ┌───────┴───────┐                                 │
│         ▼               ▼                                  │
│  ┌────────────┐  ┌────────────┐                           │
│  │Maintenance │  │  NOT in    │                           │
│  │  Window?   │  │maintenance │                           │
│  │ → Suppress │  │ → ALERT!   │                           │
│  └────────────┘  └─────┬──────┘                           │
│                         │                                  │
│                         ▼                                  │
│  ┌────────────────────────────────────────┐               │
│  │ Escalation:                             │               │
│  │  0-5 min  → Slack #alerts              │               │
│  │  5-15 min → SMS to on-call             │               │
│  │  15+ min  → Phone call + manager       │               │
│  └────────────────────────────────────────┘               │
│                                                            │
└───────────────────────────────────────────────────────────┘
```

### Canary Deployment + Synthetic Validation

```
                Canary Deploy with Synthetic Gates

     ┌─────────────────────────────────────────────────┐
     │               Production Traffic                 │
     └────────────────────────┬────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │   Load Balancer    │
                    └────┬─────────┬────┘
                         │         │
              ┌──────────┴──┐  ┌───┴──────────┐
              │  Stable v1  │  │  Canary v2   │
              │   (95%)     │  │   (5%)       │
              └─────────────┘  └──────┬───────┘
                                      │
                               ┌──────┴──────┐
                               │  Synthetic  │
                               │  Tests Run  │
                               └──────┬──────┘
                                      │
                          ┌───────────┴───────────┐
                          │                       │
                     ✅ All Pass              ❌ Failure
                          │                       │
                    ┌─────┴─────┐          ┌──────┴──────┐
                    │ Increase  │          │ Auto        │
                    │ to 25%    │          │ Rollback    │
                    │ → 50%     │          │ Alert team  │
                    │ → 100%    │          └─────────────┘
                    └───────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Synthetic Monitor Framework

```php
<?php

/**
 * Synthetic Monitoring Framework
 * bKash Send Money Flow — Multi-Location Testing
 */

interface SyntheticCheck
{
    public function getName(): string;
    public function execute(): CheckResult;
    public function getTimeout(): int;
}

class CheckResult
{
    public bool $success;
    public float $durationMs;
    public ?string $errorMessage;
    public array $metadata;
    public string $location;

    public function __construct(
        bool $success,
        float $durationMs,
        string $location,
        ?string $errorMessage = null,
        array $metadata = []
    ) {
        $this->success = $success;
        $this->durationMs = $durationMs;
        $this->location = $location;
        $this->errorMessage = $errorMessage;
        $this->metadata = $metadata;
    }
}

class HttpCheck implements SyntheticCheck
{
    private string $name;
    private string $url;
    private string $method;
    private array $headers;
    private ?array $body;
    private int $expectedStatus;
    private int $timeoutMs;
    private ?string $assertBodyContains;

    public function __construct(array $config)
    {
        $this->name = $config['name'];
        $this->url = $config['url'];
        $this->method = $config['method'] ?? 'GET';
        $this->headers = $config['headers'] ?? [];
        $this->body = $config['body'] ?? null;
        $this->expectedStatus = $config['expected_status'] ?? 200;
        $this->timeoutMs = $config['timeout_ms'] ?? 5000;
        $this->assertBodyContains = $config['assert_body_contains'] ?? null;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getTimeout(): int
    {
        return $this->timeoutMs;
    }

    public function execute(): CheckResult
    {
        $startTime = microtime(true);
        $location = gethostname();

        try {
            $ch = curl_init($this->url);
            curl_setopt_array($ch, [
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT_MS => $this->timeoutMs,
                CURLOPT_CUSTOMREQUEST => $this->method,
                CURLOPT_HTTPHEADER => $this->formatHeaders(),
            ]);

            if ($this->body) {
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($this->body));
            }

            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            $durationMs = (microtime(true) - $startTime) * 1000;

            if (curl_errno($ch)) {
                return new CheckResult(false, $durationMs, $location, curl_error($ch));
            }

            curl_close($ch);

            // Status code assertion
            if ($httpCode !== $this->expectedStatus) {
                return new CheckResult(
                    false, $durationMs, $location,
                    "Expected status {$this->expectedStatus}, got {$httpCode}"
                );
            }

            // Body contains assertion
            if ($this->assertBodyContains && !str_contains($response, $this->assertBodyContains)) {
                return new CheckResult(
                    false, $durationMs, $location,
                    "Response body doesn't contain: {$this->assertBodyContains}"
                );
            }

            // Timeout assertion
            if ($durationMs > $this->timeoutMs) {
                return new CheckResult(
                    false, $durationMs, $location,
                    "Response too slow: {$durationMs}ms > {$this->timeoutMs}ms"
                );
            }

            return new CheckResult(true, $durationMs, $location, null, [
                'http_code' => $httpCode,
                'body_length' => strlen($response),
            ]);

        } catch (\Throwable $e) {
            $durationMs = (microtime(true) - $startTime) * 1000;
            return new CheckResult(false, $durationMs, $location, $e->getMessage());
        }
    }

    private function formatHeaders(): array
    {
        return array_map(
            fn($key, $value) => "{$key}: {$value}",
            array_keys($this->headers),
            $this->headers
        );
    }
}

class MultiStepCheck implements SyntheticCheck
{
    private string $name;
    private array $steps;
    private array $context = [];

    public function __construct(string $name, array $steps)
    {
        $this->name = $name;
        $this->steps = $steps;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getTimeout(): int
    {
        return 30000; // 30s total for multi-step
    }

    public function execute(): CheckResult
    {
        $startTime = microtime(true);
        $location = gethostname();
        $stepResults = [];

        foreach ($this->steps as $index => $step) {
            $stepStart = microtime(true);

            // Step URL-এ context variable replace
            $url = $this->replaceVariables($step['url']);
            $body = $step['body'] ? $this->replaceVariablesInArray($step['body']) : null;
            $headers = $this->replaceVariablesInArray($step['headers'] ?? []);

            $check = new HttpCheck([
                'name' => $step['name'],
                'url' => $url,
                'method' => $step['method'] ?? 'GET',
                'headers' => $headers,
                'body' => $body,
                'expected_status' => $step['expected_status'] ?? 200,
                'timeout_ms' => $step['timeout_ms'] ?? 5000,
            ]);

            $result = $check->execute();
            $stepDuration = (microtime(true) - $stepStart) * 1000;

            $stepResults[] = [
                'step' => $index + 1,
                'name' => $step['name'],
                'success' => $result->success,
                'duration_ms' => $stepDuration,
                'error' => $result->errorMessage,
            ];

            if (!$result->success) {
                $totalDuration = (microtime(true) - $startTime) * 1000;
                return new CheckResult(
                    false, $totalDuration, $location,
                    "Step {$index + 1} ({$step['name']}) failed: {$result->errorMessage}",
                    ['steps' => $stepResults]
                );
            }

            // Extract variables from response for next steps
            if (isset($step['extract'])) {
                // Simplified extraction — production-এ JSON path use করুন
                $this->context = array_merge($this->context, $step['extract']);
            }
        }

        $totalDuration = (microtime(true) - $startTime) * 1000;
        return new CheckResult(true, $totalDuration, $location, null, [
            'steps' => $stepResults,
            'total_steps' => count($this->steps),
        ]);
    }

    private function replaceVariables(string $text): string
    {
        foreach ($this->context as $key => $value) {
            $text = str_replace("{{$key}}", $value, $text);
        }
        return $text;
    }

    private function replaceVariablesInArray(array $data): array
    {
        return json_decode(
            $this->replaceVariables(json_encode($data)),
            true
        );
    }
}

class SyntheticScheduler
{
    private array $checks = [];
    private array $results = [];
    private AlertEngine $alertEngine;
    private MaintenanceWindow $maintenance;

    public function __construct(AlertEngine $alertEngine, MaintenanceWindow $maintenance)
    {
        $this->alertEngine = $alertEngine;
        $this->maintenance = $maintenance;
    }

    public function addCheck(SyntheticCheck $check, int $intervalSeconds): void
    {
        $this->checks[] = [
            'check' => $check,
            'interval' => $intervalSeconds,
            'lastRun' => 0,
            'consecutiveFailures' => 0,
        ];
    }

    /**
     * একটি cycle execute (production-এ cron/loop চলবে)
     */
    public function runCycle(): array
    {
        $cycleResults = [];
        $now = time();

        foreach ($this->checks as &$entry) {
            if (($now - $entry['lastRun']) < $entry['interval']) {
                continue;
            }

            $check = $entry['check'];
            $result = $check->execute();
            $entry['lastRun'] = $now;

            if ($result->success) {
                $entry['consecutiveFailures'] = 0;
            } else {
                $entry['consecutiveFailures']++;
            }

            $cycleResults[] = [
                'check_name' => $check->getName(),
                'result' => $result,
                'consecutive_failures' => $entry['consecutiveFailures'],
            ];

            // Alert logic
            if (!$this->maintenance->isActive()) {
                $this->alertEngine->evaluate(
                    $check->getName(),
                    $result,
                    $entry['consecutiveFailures']
                );
            }
        }

        return $cycleResults;
    }
}

class AlertEngine
{
    private int $alertThreshold;
    private array $alertHistory = [];

    public function __construct(int $consecutiveFailuresBeforeAlert = 3)
    {
        $this->alertThreshold = $consecutiveFailuresBeforeAlert;
    }

    public function evaluate(string $checkName, CheckResult $result, int $consecutiveFailures): void
    {
        if ($result->success) {
            // Recovery alert
            if (isset($this->alertHistory[$checkName]) && $this->alertHistory[$checkName]['alerting']) {
                $this->sendRecoveryAlert($checkName, $result);
                $this->alertHistory[$checkName]['alerting'] = false;
            }
            return;
        }

        if ($consecutiveFailures >= $this->alertThreshold) {
            $severity = $consecutiveFailures >= 6 ? 'CRITICAL' : 'WARNING';
            $this->sendAlert($checkName, $result, $severity);
            $this->alertHistory[$checkName] = ['alerting' => true, 'since' => time()];
        }
    }

    private function sendAlert(string $checkName, CheckResult $result, string $severity): void
    {
        echo "🚨 [{$severity}] {$checkName} FAILED from {$result->location}\n";
        echo "   Error: {$result->errorMessage}\n";
        echo "   Duration: {$result->durationMs}ms\n";
        // Production: Slack, PagerDuty, SMS integration
    }

    private function sendRecoveryAlert(string $checkName, CheckResult $result): void
    {
        echo "✅ [RECOVERY] {$checkName} recovered ({$result->durationMs}ms)\n";
    }
}

class MaintenanceWindow
{
    private array $windows = [];

    public function schedule(string $name, int $startTime, int $endTime): void
    {
        $this->windows[] = ['name' => $name, 'start' => $startTime, 'end' => $endTime];
    }

    public function isActive(): bool
    {
        $now = time();
        foreach ($this->windows as $window) {
            if ($now >= $window['start'] && $now <= $window['end']) {
                return true;
            }
        }
        return false;
    }
}

// === ব্যবহার: bKash Send Money Synthetic Test ===

echo "=== 🏦 bKash Synthetic Monitor ===\n\n";

$alertEngine = new AlertEngine(consecutiveFailuresBeforeAlert: 2);
$maintenance = new MaintenanceWindow();
$scheduler = new SyntheticScheduler($alertEngine, $maintenance);

// HTTP Health Check — প্রতি ১ মিনিটে
$healthCheck = new HttpCheck([
    'name' => 'bKash API Health',
    'url' => 'https://api.bkash.com/health',
    'expected_status' => 200,
    'timeout_ms' => 2000,
]);
$scheduler->addCheck($healthCheck, 60);

// Multi-step Send Money Flow — প্রতি ৫ মিনিটে
$sendMoneyFlow = new MultiStepCheck('bKash Send Money Flow', [
    [
        'name' => 'Login',
        'url' => 'https://api.bkash.com/auth/login',
        'method' => 'POST',
        'body' => ['phone' => '01700000001', 'pin' => '12345'],
        'expected_status' => 200,
        'timeout_ms' => 3000,
        'extract' => ['token' => 'test-token-123'],
    ],
    [
        'name' => 'Check Balance',
        'url' => 'https://api.bkash.com/wallet/balance',
        'method' => 'GET',
        'headers' => ['Authorization' => 'Bearer {token}'],
        'expected_status' => 200,
        'timeout_ms' => 2000,
    ],
    [
        'name' => 'Send Money',
        'url' => 'https://api.bkash.com/transaction/send',
        'method' => 'POST',
        'headers' => ['Authorization' => 'Bearer {token}'],
        'body' => ['to' => '01700000002', 'amount' => 10],
        'expected_status' => 200,
        'timeout_ms' => 5000,
    ],
]);
$scheduler->addCheck($sendMoneyFlow, 300);

// Maintenance window — deploy-এর সময়
// $maintenance->schedule('Weekly Deploy', strtotime('Tuesday 2:00 AM'), strtotime('Tuesday 2:30 AM'));

echo "📋 Registered checks:\n";
echo "  • bKash API Health (every 60s)\n";
echo "  • bKash Send Money Flow (every 300s)\n\n";

echo "🔄 Running synthetic check cycle...\n\n";
$results = $scheduler->runCycle();

foreach ($results as $r) {
    $status = $r['result']->success ? '✅' : '❌';
    echo "{$status} {$r['check_name']}\n";
    echo "   Duration: " . round($r['result']->durationMs, 1) . "ms\n";
    echo "   Location: {$r['result']->location}\n";
    if ($r['result']->errorMessage) {
        echo "   Error: {$r['result']->errorMessage}\n";
    }
    echo "\n";
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Playwright-based Browser Synthetic & API Monitoring

```javascript
/**
 * Synthetic Monitoring — JavaScript Implementation
 * Browser + API Multi-Step Monitoring with Alert Management
 */

// === Core Synthetic Check Framework ===

class SyntheticResult {
  constructor(checkName, success, durationMs, location, error = null, metadata = {}) {
    this.checkName = checkName;
    this.success = success;
    this.durationMs = durationMs;
    this.location = location;
    this.error = error;
    this.metadata = metadata;
    this.timestamp = new Date().toISOString();
  }
}

class ApiSyntheticCheck {
  constructor(config) {
    this.name = config.name;
    this.steps = config.steps;
    this.location = config.location || 'dhaka';
    this.timeout = config.timeout || 30000;
  }

  async execute() {
    const startTime = performance.now();
    const context = {};
    const stepResults = [];

    for (let i = 0; i < this.steps.length; i++) {
      const step = this.steps[i];
      const stepStart = performance.now();

      try {
        const result = await this._executeStep(step, context);
        const stepDuration = performance.now() - stepStart;

        stepResults.push({
          step: i + 1,
          name: step.name,
          success: true,
          durationMs: stepDuration,
        });

        // Extract variables for next steps
        if (step.extract && result.body) {
          for (const [key, jsonPath] of Object.entries(step.extract)) {
            context[key] = this._extractValue(result.body, jsonPath);
          }
        }
      } catch (error) {
        const stepDuration = performance.now() - stepStart;
        stepResults.push({
          step: i + 1,
          name: step.name,
          success: false,
          durationMs: stepDuration,
          error: error.message,
        });

        const totalDuration = performance.now() - startTime;
        return new SyntheticResult(
          this.name, false, totalDuration, this.location,
          `Step ${i + 1} (${step.name}) failed: ${error.message}`,
          { steps: stepResults }
        );
      }
    }

    const totalDuration = performance.now() - startTime;
    return new SyntheticResult(
      this.name, true, totalDuration, this.location,
      null, { steps: stepResults }
    );
  }

  async _executeStep(step, context) {
    const url = this._interpolate(step.url, context);
    const headers = step.headers
      ? Object.fromEntries(
          Object.entries(step.headers).map(([k, v]) => [k, this._interpolate(v, context)])
        )
      : {};
    
    const options = {
      method: step.method || 'GET',
      headers: { 'Content-Type': 'application/json', ...headers },
      signal: AbortSignal.timeout(step.timeout || 5000),
    };

    if (step.body) {
      options.body = JSON.stringify(
        JSON.parse(this._interpolate(JSON.stringify(step.body), context))
      );
    }

    const response = await fetch(url, options);

    // Status assertion
    const expectedStatus = step.expectedStatus || 200;
    if (response.status !== expectedStatus) {
      throw new Error(`Expected ${expectedStatus}, got ${response.status}`);
    }

    // Parse body
    const body = await response.json().catch(() => null);

    // Body assertions
    if (step.assertions) {
      for (const assertion of step.assertions) {
        this._assert(body, assertion);
      }
    }

    return { status: response.status, body };
  }

  _interpolate(text, context) {
    return text.replace(/\{\{(\w+)\}\}/g, (_, key) => context[key] || '');
  }

  _extractValue(obj, path) {
    return path.split('.').reduce((o, k) => o?.[k], obj);
  }

  _assert(body, assertion) {
    const value = this._extractValue(body, assertion.path);
    switch (assertion.operator) {
      case 'exists':
        if (value === undefined || value === null) {
          throw new Error(`Assertion failed: ${assertion.path} does not exist`);
        }
        break;
      case 'equals':
        if (value !== assertion.value) {
          throw new Error(`Assertion: ${assertion.path} = ${value}, expected ${assertion.value}`);
        }
        break;
      case 'greaterThan':
        if (value <= assertion.value) {
          throw new Error(`Assertion: ${assertion.path} = ${value}, expected > ${assertion.value}`);
        }
        break;
    }
  }
}

// === Browser Synthetic Check (Playwright-style) ===

class BrowserSyntheticCheck {
  constructor(config) {
    this.name = config.name;
    this.location = config.location || 'dhaka';
    this.steps = config.steps; // Playwright-like steps
    this.timeout = config.timeout || 30000;
  }

  async execute() {
    const startTime = performance.now();
    const stepResults = [];

    // Production-এ এখানে Playwright/Puppeteer launch হবে
    // এটি একটি simulation
    console.log(`🌐 Browser check: ${this.name} from ${this.location}`);

    for (let i = 0; i < this.steps.length; i++) {
      const step = this.steps[i];
      const stepStart = performance.now();

      try {
        // Simulate browser action
        await this._simulateBrowserAction(step);
        const stepDuration = performance.now() - stepStart;

        stepResults.push({
          step: i + 1,
          action: step.action,
          target: step.target,
          success: true,
          durationMs: stepDuration,
        });

        console.log(`  ✅ Step ${i + 1}: ${step.action} ${step.target} (${stepDuration.toFixed(0)}ms)`);
      } catch (error) {
        const stepDuration = performance.now() - stepStart;
        stepResults.push({
          step: i + 1,
          action: step.action,
          target: step.target,
          success: false,
          durationMs: stepDuration,
          error: error.message,
        });

        const totalDuration = performance.now() - startTime;
        return new SyntheticResult(
          this.name, false, totalDuration, this.location,
          `Browser step ${i + 1} failed: ${error.message}`,
          { steps: stepResults, screenshot: 'base64_screenshot_data' }
        );
      }
    }

    const totalDuration = performance.now() - startTime;
    return new SyntheticResult(
      this.name, true, totalDuration, this.location,
      null, { steps: stepResults }
    );
  }

  async _simulateBrowserAction(step) {
    // Simulate variable timing
    const delay = 100 + Math.random() * 500;
    await new Promise(resolve => setTimeout(resolve, delay));

    // Simulate occasional failures
    if (step.failureRate && Math.random() < step.failureRate) {
      throw new Error(`Element not found: ${step.target}`);
    }
  }
}

// === Scheduler with Multi-Location Support ===

class SyntheticScheduler {
  constructor() {
    this.checks = [];
    this.results = new Map();
    this.alertManager = new AlertManager();
    this.maintenanceWindows = [];
  }

  register(check, intervalMs) {
    this.checks.push({
      check,
      intervalMs,
      lastRun: 0,
      consecutiveFailures: 0,
      history: [],
    });
  }

  addMaintenanceWindow(name, startTime, endTime) {
    this.maintenanceWindows.push({ name, startTime, endTime });
  }

  isInMaintenance() {
    const now = Date.now();
    return this.maintenanceWindows.some(w => now >= w.startTime && now <= w.endTime);
  }

  async runOnce() {
    const now = Date.now();
    const results = [];

    for (const entry of this.checks) {
      if ((now - entry.lastRun) < entry.intervalMs) continue;

      const result = await entry.check.execute();
      entry.lastRun = now;

      if (result.success) {
        entry.consecutiveFailures = 0;
      } else {
        entry.consecutiveFailures++;
      }

      // History maintain (last 100 results)
      entry.history.push(result);
      if (entry.history.length > 100) entry.history.shift();

      results.push({ entry, result });

      // Alert evaluation
      if (!this.isInMaintenance()) {
        this.alertManager.evaluate(
          entry.check.name,
          result,
          entry.consecutiveFailures
        );
      }
    }

    return results;
  }

  getUptime(checkName, periodMs) {
    const entry = this.checks.find(c => c.check.name === checkName);
    if (!entry) return null;

    const cutoff = Date.now() - periodMs;
    const relevant = entry.history.filter(r => new Date(r.timestamp) >= cutoff);
    if (relevant.length === 0) return 100;

    const successful = relevant.filter(r => r.success).length;
    return (successful / relevant.length) * 100;
  }
}

// === Alert Manager with Escalation ===

class AlertManager {
  constructor() {
    this.threshold = 2; // consecutive failures before alert
    this.activeAlerts = new Map();
    this.channels = [
      { name: 'Slack', delayMs: 0 },
      { name: 'SMS', delayMs: 5 * 60 * 1000 },
      { name: 'Phone', delayMs: 15 * 60 * 1000 },
    ];
  }

  evaluate(checkName, result, consecutiveFailures) {
    if (result.success) {
      if (this.activeAlerts.has(checkName)) {
        this._sendRecovery(checkName, result);
        this.activeAlerts.delete(checkName);
      }
      return;
    }

    if (consecutiveFailures >= this.threshold) {
      if (!this.activeAlerts.has(checkName)) {
        this.activeAlerts.set(checkName, { since: Date.now(), escalationLevel: 0 });
      }

      const alert = this.activeAlerts.get(checkName);
      const elapsed = Date.now() - alert.since;

      // Escalation check
      for (let i = this.channels.length - 1; i >= 0; i--) {
        if (elapsed >= this.channels[i].delayMs && alert.escalationLevel <= i) {
          this._sendAlert(checkName, result, this.channels[i]);
          alert.escalationLevel = i + 1;
          break;
        }
      }
    }
  }

  _sendAlert(checkName, result, channel) {
    console.log(
      `🚨 [${channel.name}] ALERT: ${checkName} FAILED\n` +
      `   Location: ${result.location}\n` +
      `   Error: ${result.error}\n` +
      `   Duration: ${result.durationMs.toFixed(0)}ms`
    );
  }

  _sendRecovery(checkName, result) {
    console.log(
      `✅ [RECOVERY] ${checkName} is back up\n` +
      `   Duration: ${result.durationMs.toFixed(0)}ms`
    );
  }
}

// === Usage: bKash Monitoring Configuration ===

async function main() {
  console.log('=== 🤖 bKash Synthetic Monitoring ===\n');

  const scheduler = new SyntheticScheduler();

  // 1. API Health Check — প্রতি ১ মিনিটে
  const healthCheck = new ApiSyntheticCheck({
    name: 'bKash API Health (Dhaka)',
    location: 'dhaka',
    steps: [
      {
        name: 'Health Endpoint',
        url: 'https://api.bkash.com/health',
        method: 'GET',
        expectedStatus: 200,
        timeout: 2000,
        assertions: [
          { path: 'status', operator: 'equals', value: 'healthy' },
        ],
      },
    ],
  });
  scheduler.register(healthCheck, 60000);

  // 2. Send Money Flow — প্রতি ৫ মিনিটে (Dhaka)
  const sendMoneyDhaka = new ApiSyntheticCheck({
    name: 'bKash Send Money (Dhaka)',
    location: 'dhaka',
    steps: [
      {
        name: 'Login',
        url: 'https://api.bkash.com/auth/login',
        method: 'POST',
        body: { phone: '01700000001', pin: '12345' },
        expectedStatus: 200,
        extract: { token: 'data.token' },
        assertions: [
          { path: 'data.token', operator: 'exists' },
        ],
      },
      {
        name: 'Check Balance',
        url: 'https://api.bkash.com/wallet/balance',
        method: 'GET',
        headers: { Authorization: 'Bearer {{token}}' },
        expectedStatus: 200,
        assertions: [
          { path: 'data.balance', operator: 'greaterThan', value: 100 },
        ],
      },
      {
        name: 'Send Money',
        url: 'https://api.bkash.com/transaction/send',
        method: 'POST',
        headers: { Authorization: 'Bearer {{token}}' },
        body: { to: '01700000002', amount: 10 },
        expectedStatus: 200,
        extract: { trxId: 'data.transaction_id' },
      },
      {
        name: 'Verify Transaction',
        url: 'https://api.bkash.com/transaction/{{trxId}}',
        method: 'GET',
        headers: { Authorization: 'Bearer {{token}}' },
        expectedStatus: 200,
        assertions: [
          { path: 'data.status', operator: 'equals', value: 'completed' },
        ],
      },
    ],
  });
  scheduler.register(sendMoneyDhaka, 300000);

  // 3. Browser Check — Daraz Homepage (প্রতি ১০ মিনিটে)
  const darazBrowser = new BrowserSyntheticCheck({
    name: 'Daraz Homepage (Browser)',
    location: 'dhaka',
    steps: [
      { action: 'navigate', target: 'https://www.daraz.com.bd' },
      { action: 'waitForSelector', target: '.search-bar' },
      { action: 'type', target: '.search-bar', value: 'mobile phone' },
      { action: 'click', target: '.search-button' },
      { action: 'waitForSelector', target: '.product-card' },
      { action: 'assertVisible', target: '.product-card:first-child .price' },
    ],
  });
  scheduler.register(darazBrowser, 600000);

  // Run checks
  console.log('🔄 Running synthetic checks...\n');
  const results = await scheduler.runOnce();

  // Summary
  console.log('\n=== 📊 Results Summary ===\n');
  for (const { entry, result } of results) {
    const icon = result.success ? '✅' : '❌';
    console.log(`${icon} ${result.checkName}`);
    console.log(`   Duration: ${result.durationMs.toFixed(0)}ms`);
    console.log(`   Location: ${result.location}`);
    if (result.error) console.log(`   Error: ${result.error}`);
    if (result.metadata.steps) {
      console.log(`   Steps: ${result.metadata.steps.length} total`);
    }
    console.log('');
  }

  // SLA Validation
  console.log('=== 📋 SLA Validation ===');
  console.log(`   24h Uptime: ${scheduler.getUptime('bKash Send Money (Dhaka)', 86400000)?.toFixed(2) || 'N/A'}%`);
  console.log(`   7d Uptime: ${scheduler.getUptime('bKash Send Money (Dhaka)', 604800000)?.toFixed(2) || 'N/A'}%`);
}

main().catch(console.error);
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Synthetic Monitoring ব্যবহার করবেন

| পরিস্থিতি | কেন |
|-----------|-----|
| Payment gateway (bKash, Nagad) | Transaction failure তাৎক্ষণিক detect |
| E-commerce checkout (Daraz) | Revenue-critical flow monitor |
| SLA commitment আছে | Proactively SLA validate করা |
| Off-peak hour coverage দরকার | রাতে user নেই, synthetic detect করবে |
| Multi-region service | প্রতিটি region থেকে test |
| Canary deployment validation | New version-এ synthetic test |
| Third-party dependency | Payment gateway, SMS API health |
| SSL/Domain expiry tracking | Certificate expire-এর আগে alert |

### ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|-----------|-----|
| শুধু internal tool | RUM যথেষ্ট, synthetic overkill |
| Real user experience measure | RUM ভালো — synthetic approximation |
| Load testing | Synthetic ≠ load test! k6/JMeter ব্যবহার করুন |
| Security testing | Synthetic ≠ penetration test |
| Cost-sensitive & low traffic | Simple uptime check (free) যথেষ্ট |
| Dynamic content testing | Content constantly change হলে flaky হয় |

### 🎯 Best Practices

```
1. Start Simple:
   ❌ Day 1-এ 50টা browser test লিখবেন না
   ✅ Top 3 critical flows দিয়ে শুরু করুন

2. Test What Matters:
   ❌ প্রতিটি page test করা
   ✅ Revenue-generating flows (payment, checkout, signup)

3. Realistic Intervals:
   ❌ প্রতি ১০ সেকেন্ডে heavy test
   ✅ Health check: 1 min, Flows: 5 min, Browser: 10 min

4. Maintenance Windows:
   ❌ Deploy-এর সময় false alert পাওয়া
   ✅ Deploy schedule-এ auto-mute

5. Multi-location:
   ❌ শুধু একটি location থেকে test
   ✅ কমপক্ষে ২টি location (ISP issue vs real outage)
   
6. Alert Fatigue Prevention:
   ❌ প্রথম failure-এই page করা
   ✅ 2-3 consecutive failures + multi-location confirm
```

### 💰 Cost Comparison

```
┌─────────────────────────────────────────────────────────┐
│ Tool               │ API Check/mo │ Browser/mo │ Free?  │
├────────────────────┼──────────────┼────────────┼────────┤
│ UptimeRobot        │ 50 (free)    │ ❌         │ ✅     │
│ Checkly            │ 10K          │ 1.2K       │ Free*  │
│ Datadog Synthetic  │ 10K          │ 1K         │ $$$    │
│ AWS CloudWatch Syn │ Pay per run  │ Pay/run    │ $$     │
│ Grafana Cloud      │ Included     │ Included   │ Free*  │
│ Self-hosted (k6)   │ Unlimited    │ Unlimited  │ Free   │
└─────────────────────────────────────────────────────────┘
* Free tier limits apply

bKash-এর মতো service-এর জন্য recommendation:
├── Critical flows (Send Money): Checkly/Datadog (reliable)
├── Health checks: Self-hosted or free tier
└── Browser: Playwright + self-hosted runner
```
