# 🚀 Progressive Delivery (Feature Flags, Dark Launch, Shadow Traffic)

## 📖 সংজ্ঞা ও ধারণা

Progressive Delivery হলো **Continuous Delivery-এর একটি superset** — এটি শুধু deploy করে না, বরং **কাকে, কখন, কতটুকু** feature দেখানো হবে সেটা নিয়ন্ত্রণ করে। এটি deploy এবং release-কে আলাদা করে (decouple deploy from release)।

### 🎯 মূল ধারণা

```
┌─────────────────────────────────────────────────────────┐
│         CONTINUOUS DELIVERY vs PROGRESSIVE DELIVERY       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Continuous Delivery:                                    │
│    Code → Build → Test → Deploy → EVERYONE gets it      │
│                                                          │
│  Progressive Delivery:                                   │
│    Code → Build → Test → Deploy → CONTROL who gets it   │
│                           │                              │
│                           ├──▶ Feature Flags             │
│                           ├──▶ Dark Launch               │
│                           ├──▶ Shadow Traffic            │
│                           ├──▶ Canary Release            │
│                           └──▶ Ring-based Deployment     │
│                                                          │
│  "Deploy করা মানেই release করা নয়"                      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 🏷️ Feature Flag Types (Toggle Types)

```
┌──────────────┬────────────────────────────────────────────────────┐
│ Toggle Type  │ উদ্দেশ্য ও বৈশিষ্ট্য                                │
├──────────────┼────────────────────────────────────────────────────┤
│ Release      │ নতুন feature ধীরে ধীরে enable করা                  │
│ Toggle       │ Lifecycle: Short (days/weeks)                      │
│              │ Example: bKash-এ নতুন QR payment feature            │
├──────────────┼────────────────────────────────────────────────────┤
│ Experiment   │ A/B testing, multivariate testing                  │
│ Toggle       │ Lifecycle: Medium (weeks/months)                   │
│              │ Example: Pathao-এ surge pricing algorithm test      │
├──────────────┼────────────────────────────────────────────────────┤
│ Ops Toggle   │ System behavior control (kill switch)              │
│              │ Lifecycle: Long/Permanent                          │
│              │ Example: Daraz-এ flash sale mode enable/disable     │
├──────────────┼────────────────────────────────────────────────────┤
│ Permission   │ User-specific feature access                       │
│ Toggle       │ Lifecycle: Long/Permanent                          │
│              │ Example: GP Star users-দের premium features         │
└──────────────┴────────────────────────────────────────────────────┘
```

### 🌑 Dark Launch কী?

Dark Launch = কোড production-এ **deploy করা হয়**, কিন্তু user-দের কাছে **দেখানো হয় না**। পেছনে পেছনে কাজ হয়, metrics collect হয়, কিন্তু UI-তে কিছু পরিবর্তন দেখা যায় না।

### 👻 Shadow/Mirror Traffic কী?

Shadow Traffic = Production traffic-এর একটি **কপি** নতুন service-এ পাঠানো হয়। Response user-কে দেওয়া হয় না, শুধু comparison-এর জন্য ব্যবহৃত হয়।

---

## 🌍 বাস্তব জীবনের উদাহরণ (বাংলাদেশ প্রসঙ্গ)

### 🚗 Pathao — Surge Pricing Algorithm (Shadow Traffic)

Pathao তাদের surge pricing algorithm update করতে চায়। কিন্তু সরাসরি নতুন algorithm চালু করলে:
- ভুল pricing দিলে rider হারাবে
- অতিরিক্ত price দিলে customer হারাবে
- কম price দিলে Pathao-র loss

**Shadow Traffic Solution:**
```
┌────────────────────────────────────────────────────────────┐
│                                                             │
│  Real User Request:                                         │
│  "Dhanmondi → Uttara, এখন কত ভাড়া?"                      │
│                                                             │
│  ┌─────────┐    ┌────────────────────┐    ┌─────────────┐ │
│  │  User   │───▶│  API Gateway       │───▶│ Old Algo    │ │
│  │ Request │    │  (Traffic Mirror)  │    │ (v1) ✅     │ │
│  └─────────┘    └─────────┬──────────┘    │ Response:   │ │
│                           │               │ ৳180        │ │
│                           │ (copy)        └──────┬──────┘ │
│                           ▼                      │        │
│                   ┌────────────────┐             │        │
│                   │  New Algo      │             │        │
│                   │  (v2) 🔍       │             │        │
│                   │  Response:     │             ▼        │
│                   │  ৳195         │     [User পায় ৳180]  │
│                   └───────┬────────┘                      │
│                           │                               │
│                           ▼                               │
│                   ┌────────────────┐                      │
│                   │  Compare &     │                      │
│                   │  Log           │                      │
│                   │  diff: +৳15   │                      │
│                   └────────────────┘                      │
│                                                             │
│  User কিছু জানে না! শুধু team দেখছে v2 কেমন perform করছে │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 📰 Prothom Alo — Dark Launch (New Recommendation Engine)

Prothom Alo তাদের news recommendation engine update করছে:
1. **Dark Launch:** নতুন engine production-এ deploy, কিন্তু শুধু পুরোনো recommendation দেখানো হচ্ছে
2. পেছনে নতুন engine-ও recommendation generate করছে
3. দুটোর comparison log হচ্ছে
4. ৩ সপ্তাহ পর data দেখে decision নেওয়া হবে

### 🛒 Daraz — Ring-based Deployment

```
Ring 0: Internal Employees (Daraz team)     → ১ দিন
Ring 1: Beta Users (opted-in customers)      → ৩ দিন
Ring 2: 10% General Users (random)           → ৫ দিন
Ring 3: 50% General Users                    → ৩ দিন
Ring 4: 100% (General Availability)          → সবাই
```

### 📱 Grameenphone MyGP — Feature Flag

MyGP app-এ নতুন "Internet Balance Prediction" feature:
- প্রথমে শুধু Dhaka zone-এর GP Star customer-দের দেখানো
- তারপর সব GP Star
- তারপর সবাই

---

## 📊 ASCII Diagram — Progressive Delivery Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PROGRESSIVE DELIVERY PIPELINE                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐   ┌────────┐   ┌─────────┐   ┌───────────────────┐    │
│  │  Code  │──▶│ Build  │──▶│  Test   │──▶│     DEPLOY        │    │
│  │ Commit │   │        │   │ (CI)    │   │ (Feature OFF)     │    │
│  └────────┘   └────────┘   └─────────┘   └─────────┬─────────┘    │
│                                                      │              │
│                                                      ▼              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    RELEASE CONTROL                             │  │
│  │                                                               │  │
│  │   ┌─────────────────────────────────────────────────────┐    │  │
│  │   │  Feature Flag Manager (LaunchDarkly/Unleash)        │    │  │
│  │   └────────────────────────┬────────────────────────────┘    │  │
│  │                            │                                  │  │
│  │            ┌───────────────┼───────────────┐                  │  │
│  │            ▼               ▼               ▼                  │  │
│  │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │  │
│  │   │   Internal   │ │    Beta      │ │   General    │        │  │
│  │   │   (Ring 0)   │ │  (Ring 1)    │ │  (Ring 2-4)  │        │  │
│  │   │   Employees  │ │  Opted-in    │ │  Percentage  │        │  │
│  │   └──────────────┘ └──────────────┘ └──────────────┘        │  │
│  │                                                               │  │
│  │   Targeting Rules:                                            │  │
│  │   • User segment (GP Star, Regular)                           │  │
│  │   • Geography (Dhaka, Chittagong)                             │  │
│  │   • Percentage (1%, 5%, 25%, 100%)                            │  │
│  │   • Custom attributes (isPremium, accountAge)                 │  │
│  │                                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Feature Flag Decision Flow

```
┌─────────────────────────────────────────────────────────┐
│             FEATURE FLAG EVALUATION                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Request comes in:                                       │
│  User: Karim, Location: Dhaka, Plan: GP Star            │
│                                                          │
│          ┌─────────────────────┐                        │
│          │  Flag: new-qr-pay  │                        │
│          │  Status: ENABLED    │                        │
│          └──────────┬──────────┘                        │
│                     │                                    │
│          ┌──────────▼──────────┐                        │
│          │  Targeting Rule #1  │                        │
│          │  Segment: "beta"    │──── NO ──┐            │
│          └──────────┬──────────┘          │            │
│                YES  │                     │            │
│                     ▼                     │            │
│          ┌─────────────────────┐          │            │
│          │  Rule #2: Location  │          │            │
│          │  in ["Dhaka"]       │── NO ────┤            │
│          └──────────┬──────────┘          │            │
│                YES  │                     │            │
│                     ▼                     │            │
│          ┌─────────────────────┐          │            │
│          │  Rule #3: Rollout   │          │            │
│          │  25% of matching    │── NO ────┤            │
│          └──────────┬──────────┘          │            │
│                YES  │                     ▼            │
│                     ▼              ┌────────────┐      │
│          ┌────────────────┐       │  FEATURE   │      │
│          │   FEATURE ON   │       │   OFF      │      │
│          │   Show new QR  │       │  Old flow  │      │
│          └────────────────┘       └────────────┘      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Feature Flag System (PHP)

```php
<?php

namespace App\FeatureFlags;

/**
 * Feature Flag Manager
 * Pathao/bKash-এ ব্যবহারযোগ্য Progressive Delivery system
 */
class FeatureFlagManager
{
    private FlagStorage $storage;
    private UserContext $defaultContext;
    private MetricsCollector $metrics;
    private array $cache = [];

    public function __construct(
        FlagStorage $storage,
        MetricsCollector $metrics
    ) {
        $this->storage = $storage;
        $this->metrics = $metrics;
    }

    /**
     * Feature flag evaluate করা
     */
    public function isEnabled(string $flagKey, ?UserContext $context = null): bool
    {
        $context = $context ?? $this->defaultContext;

        // Cache check
        $cacheKey = "{$flagKey}:{$context->getUserId()}";
        if (isset($this->cache[$cacheKey])) {
            return $this->cache[$cacheKey];
        }

        $flag = $this->storage->getFlag($flagKey);

        if (!$flag) {
            $this->metrics->increment("flag.not_found", ['flag' => $flagKey]);
            return false;
        }

        // Flag globally disabled
        if (!$flag->isActive()) {
            return false;
        }

        // Targeting rules evaluate করা
        $result = $this->evaluateRules($flag, $context);

        // Cache & metrics
        $this->cache[$cacheKey] = $result;
        $this->metrics->increment("flag.evaluated", [
            'flag' => $flagKey,
            'result' => $result ? 'on' : 'off',
        ]);

        return $result;
    }

    /**
     * Targeting rules evaluation
     */
    private function evaluateRules(FeatureFlag $flag, UserContext $context): bool
    {
        // Rule 1: Kill switch — specific users always get it
        if (in_array($context->getUserId(), $flag->getAllowList())) {
            return true;
        }

        // Rule 2: Block list
        if (in_array($context->getUserId(), $flag->getBlockList())) {
            return false;
        }

        // Rule 3: User segment targeting
        foreach ($flag->getSegmentRules() as $rule) {
            if (!$this->matchesSegment($rule, $context)) {
                continue; // এই rule apply হয় না, পরবর্তী check
            }

            // Segment match — percentage rollout check
            return $this->isInRolloutPercentage(
                $context->getUserId(),
                $flagKey,
                $rule['percentage']
            );
        }

        // Rule 4: Default rollout percentage
        return $this->isInRolloutPercentage(
            $context->getUserId(),
            $flag->getKey(),
            $flag->getDefaultPercentage()
        );
    }

    /**
     * Consistent hashing — user সবসময় একই bucket-এ থাকবে
     */
    private function isInRolloutPercentage(string $userId, string $flagKey, int $percentage): bool
    {
        if ($percentage === 0) return false;
        if ($percentage === 100) return true;

        // Deterministic hash — user+flag combination সবসময় same result দেবে
        $hash = crc32("{$userId}:{$flagKey}");
        $bucket = abs($hash) % 100;

        return $bucket < $percentage;
    }

    private function matchesSegment(array $rule, UserContext $context): bool
    {
        foreach ($rule['conditions'] as $condition) {
            $attribute = $context->getAttribute($condition['attribute']);

            switch ($condition['operator']) {
                case 'equals':
                    if ($attribute !== $condition['value']) return false;
                    break;
                case 'in':
                    if (!in_array($attribute, $condition['values'])) return false;
                    break;
                case 'greater_than':
                    if ($attribute <= $condition['value']) return false;
                    break;
                case 'contains':
                    if (!str_contains($attribute, $condition['value'])) return false;
                    break;
            }
        }
        return true;
    }
}

/**
 * Feature Flag Definition
 */
class FeatureFlag
{
    private string $key;
    private string $name;
    private string $type; // release, experiment, ops, permission
    private bool $active;
    private int $defaultPercentage;
    private array $allowList;
    private array $blockList;
    private array $segmentRules;
    private ?string $expiresAt;

    public function __construct(array $config)
    {
        $this->key = $config['key'];
        $this->name = $config['name'];
        $this->type = $config['type'];
        $this->active = $config['active'] ?? false;
        $this->defaultPercentage = $config['defaultPercentage'] ?? 0;
        $this->allowList = $config['allowList'] ?? [];
        $this->blockList = $config['blockList'] ?? [];
        $this->segmentRules = $config['segmentRules'] ?? [];
        $this->expiresAt = $config['expiresAt'] ?? null;
    }

    public function isStale(): bool
    {
        if (!$this->expiresAt) return false;
        return new \DateTime() > new \DateTime($this->expiresAt);
    }

    // Getters...
    public function getKey(): string { return $this->key; }
    public function isActive(): bool { return $this->active; }
    public function getDefaultPercentage(): int { return $this->defaultPercentage; }
    public function getAllowList(): array { return $this->allowList; }
    public function getBlockList(): array { return $this->blockList; }
    public function getSegmentRules(): array { return $this->segmentRules; }
}

/**
 * Shadow Traffic / Mirror Traffic Implementation
 * Pathao-র surge pricing algorithm test
 */
class ShadowTrafficRouter
{
    private HttpClient $httpClient;
    private MetricsCollector $metrics;
    private LogStorage $logStorage;

    /**
     * Request mirror করা — shadow service-এ পাঠানো
     */
    public function mirrorRequest(
        string $originalServiceUrl,
        string $shadowServiceUrl,
        array $request
    ): mixed {
        // Primary request — এটাই user-কে response দেবে
        $primaryResponse = $this->httpClient->send($originalServiceUrl, $request);

        // Shadow request — async, user-কে response দেয় না
        $this->sendToShadowAsync($shadowServiceUrl, $request, $primaryResponse);

        return $primaryResponse;
    }

    private function sendToShadowAsync(
        string $shadowUrl,
        array $request,
        mixed $primaryResponse
    ): void {
        // Non-blocking — user experience affect করবে না
        go(function () use ($shadowUrl, $request, $primaryResponse) {
            try {
                $shadowResponse = $this->httpClient->send($shadowUrl, $request);

                // Compare responses
                $comparison = $this->compareResponses($primaryResponse, $shadowResponse);

                // Log for analysis
                $this->logStorage->store([
                    'timestamp' => now(),
                    'request' => $request,
                    'primary_response' => $primaryResponse,
                    'shadow_response' => $shadowResponse,
                    'comparison' => $comparison,
                ]);

                // Metrics
                $this->metrics->histogram('shadow.response_diff', $comparison['priceDiff']);
                $this->metrics->increment('shadow.requests_mirrored');

                if ($comparison['hasMismatch']) {
                    $this->metrics->increment('shadow.mismatches');
                }
            } catch (\Throwable $e) {
                // Shadow failure user-কে affect করবে না
                $this->metrics->increment('shadow.errors');
            }
        });
    }

    private function compareResponses(mixed $primary, mixed $shadow): array
    {
        return [
            'hasMismatch' => $primary['price'] !== $shadow['price'],
            'priceDiff' => abs($primary['price'] - $shadow['price']),
            'primaryPrice' => $primary['price'],
            'shadowPrice' => $shadow['price'],
            'percentDiff' => abs($primary['price'] - $shadow['price']) / $primary['price'] * 100,
        ];
    }
}

/**
 * Dark Launch Controller
 * Feature deploy করা কিন্তু user-কে না দেখানো
 */
class DarkLaunchController
{
    private FeatureFlagManager $flagManager;

    /**
     * Dark launch — পেছনে চালানো, সামনে না দেখানো
     */
    public function executeDarkLaunch(string $feature, callable $newLogic, callable $oldLogic): mixed
    {
        // সবসময় old logic-এর result return করা
        $result = $oldLogic();

        // পেছনে new logic-ও চালানো (async)
        if ($this->flagManager->isEnabled("dark_launch_{$feature}")) {
            $this->executeInBackground(function () use ($feature, $newLogic, $result) {
                $newResult = $newLogic();
                $this->compareAndLog($feature, $result, $newResult);
            });
        }

        return $result; // User সবসময় old result পায়
    }

    private function compareAndLog(string $feature, mixed $old, mixed $new): void
    {
        // Comparison metrics log করা
        // Dashboard-এ দেখা যাবে new logic কেমন perform করছে
    }

    private function executeInBackground(callable $task): void
    {
        // Queue-তে পাঠানো বা async execution
    }
}

/**
 * Flag Cleanup Automation
 * Stale flags detect ও remove করা
 */
class FlagCleanupService
{
    private FlagStorage $storage;
    private NotificationService $notifier;

    /**
     * Stale flags খুঁজে বের করা
     */
    public function findStaleFlags(): array
    {
        $allFlags = $this->storage->getAllFlags();
        $staleFlags = [];

        foreach ($allFlags as $flag) {
            $staleness = $this->calculateStaleness($flag);

            if ($staleness['isStale']) {
                $staleFlags[] = [
                    'flag' => $flag,
                    'reason' => $staleness['reason'],
                    'age' => $staleness['age'],
                    'recommendation' => $staleness['recommendation'],
                ];
            }
        }

        return $staleFlags;
    }

    private function calculateStaleness(FeatureFlag $flag): array
    {
        // Release toggle যা ১০০% rollout — remove করা উচিত
        if ($flag->getType() === 'release' && $flag->getDefaultPercentage() === 100) {
            return [
                'isStale' => true,
                'reason' => 'Release flag at 100% — code এ permanently enable করুন',
                'recommendation' => 'REMOVE',
            ];
        }

        // Expired flags
        if ($flag->isStale()) {
            return [
                'isStale' => true,
                'reason' => 'Expiry date পার হয়ে গেছে',
                'recommendation' => 'REVIEW_AND_REMOVE',
            ];
        }

        return ['isStale' => false];
    }

    /**
     * Weekly stale flag report
     */
    public function sendWeeklyReport(): void
    {
        $staleFlags = $this->findStaleFlags();

        if (empty($staleFlags)) return;

        $this->notifier->send(
            channel: '#tech-debt',
            message: sprintf(
                "🧹 %d টি stale feature flag পাওয়া গেছে! Please review:\n%s",
                count($staleFlags),
                implode("\n", array_map(
                    fn($f) => "  - {$f['flag']->getKey()}: {$f['reason']}",
                    $staleFlags
                ))
            )
        );
    }
}

// === ব্যবহার ===

// Pathao-তে নতুন surge pricing — Shadow Traffic
$router = new ShadowTrafficRouter(new HttpClient(), new MetricsCollector(), new LogStorage());

$request = ['from' => 'Dhanmondi', 'to' => 'Uttara', 'time' => 'peak_hour'];
$response = $router->mirrorRequest(
    'http://pricing-v1.pathao.internal/calculate',
    'http://pricing-v2.pathao.internal/calculate',
    $request
);
// User পাবে v1-এর price, কিন্তু v2-ও চলছে পেছনে

// bKash-এ নতুন QR feature — Feature Flag
$flagManager = new FeatureFlagManager(new RedisStorage(), new PrometheusMetrics());
$context = new UserContext(userId: 'user_123', attributes: [
    'location' => 'Dhaka',
    'plan' => 'premium',
    'accountAge' => 365,
]);

if ($flagManager->isEnabled('new-qr-payment', $context)) {
    // নতুন QR payment UI দেখাও
    showNewQRPayment();
} else {
    // পুরোনো payment flow
    showLegacyPayment();
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Feature Flag SDK (JavaScript)

```javascript
/**
 * Progressive Delivery SDK
 * Feature Flags + Dark Launch + Shadow Traffic
 */

class ProgressiveDeliverySDK {
    constructor(config) {
        this.apiUrl = config.apiUrl;
        this.sdkKey = config.sdkKey;
        this.flags = new Map();
        this.cache = new Map();
        this.metrics = config.metrics || new NoopMetrics();
        this.refreshInterval = config.refreshInterval || 30000;

        this.startPolling();
    }

    /**
     * Feature flag check করা
     */
    isEnabled(flagKey, userContext = {}) {
        const flag = this.flags.get(flagKey);
        if (!flag || !flag.active) {
            this.metrics.track('flag_check', { flag: flagKey, result: 'off', reason: 'not_found' });
            return false;
        }

        const result = this.evaluate(flag, userContext);

        this.metrics.track('flag_check', {
            flag: flagKey,
            result: result ? 'on' : 'off',
            userId: userContext.userId,
        });

        return result;
    }

    /**
     * Feature variation পাওয়া (multivariate flags)
     */
    getVariation(flagKey, userContext = {}, defaultValue = null) {
        const flag = this.flags.get(flagKey);
        if (!flag || !flag.active) return defaultValue;

        const targetVariation = this.evaluateVariation(flag, userContext);
        return targetVariation ?? defaultValue;
    }

    /**
     * Targeting rules evaluation
     */
    evaluate(flag, context) {
        // Allow list check
        if (flag.allowList?.includes(context.userId)) return true;

        // Block list check
        if (flag.blockList?.includes(context.userId)) return false;

        // Segment rules — order matters
        for (const rule of flag.rules || []) {
            if (this.matchesRule(rule, context)) {
                return this.isInPercentage(context.userId, flag.key, rule.percentage);
            }
        }

        // Default percentage
        return this.isInPercentage(context.userId, flag.key, flag.defaultPercentage || 0);
    }

    matchesRule(rule, context) {
        return rule.conditions.every(condition => {
            const value = context[condition.attribute] ?? context.attributes?.[condition.attribute];

            switch (condition.operator) {
                case 'equals': return value === condition.value;
                case 'not_equals': return value !== condition.value;
                case 'in': return condition.values.includes(value);
                case 'not_in': return !condition.values.includes(value);
                case 'greater_than': return value > condition.value;
                case 'less_than': return value < condition.value;
                case 'contains': return String(value).includes(condition.value);
                case 'starts_with': return String(value).startsWith(condition.value);
                case 'regex': return new RegExp(condition.value).test(value);
                default: return false;
            }
        });
    }

    /**
     * Deterministic percentage bucketing
     * একই user সবসময় একই result পাবে (sticky)
     */
    isInPercentage(userId, flagKey, percentage) {
        if (percentage === 0) return false;
        if (percentage >= 100) return true;

        const hash = this.murmurHash(`${userId}:${flagKey}`);
        const bucket = Math.abs(hash) % 100;
        return bucket < percentage;
    }

    murmurHash(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            const char = str.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash |= 0;
        }
        return hash;
    }

    evaluateVariation(flag, context) {
        // For A/B testing — multiple variations
        const bucket = Math.abs(this.murmurHash(`${context.userId}:${flag.key}`)) % 100;
        let cumulative = 0;

        for (const variation of flag.variations || []) {
            cumulative += variation.weight;
            if (bucket < cumulative) {
                return variation.value;
            }
        }
        return flag.defaultVariation;
    }

    async startPolling() {
        await this.fetchFlags();
        setInterval(() => this.fetchFlags(), this.refreshInterval);
    }

    async fetchFlags() {
        try {
            const response = await fetch(`${this.apiUrl}/flags`, {
                headers: { 'Authorization': `Bearer ${this.sdkKey}` },
            });
            const data = await response.json();
            data.flags.forEach(flag => this.flags.set(flag.key, flag));
        } catch (error) {
            console.error('Flag fetch failed, using cached values');
        }
    }
}

/**
 * Shadow Traffic Manager
 * Pathao surge pricing testing
 */
class ShadowTrafficManager {
    constructor(config) {
        this.primaryUrl = config.primaryUrl;
        this.shadowUrl = config.shadowUrl;
        this.comparisonStore = config.comparisonStore;
        this.metrics = config.metrics;
        this.sampleRate = config.sampleRate || 1.0; // ১০০% mirror by default
    }

    /**
     * Request mirror — primary + shadow parallel call
     */
    async handleRequest(request) {
        // Primary request — এটাই user-কে response দেবে
        const primaryPromise = this.callService(this.primaryUrl, request);

        // Shadow request — non-blocking, fire-and-forget
        if (Math.random() < this.sampleRate) {
            this.callShadow(request, primaryPromise);
        }

        // User শুধু primary response পায়
        return await primaryPromise;
    }

    async callShadow(request, primaryPromise) {
        try {
            const [primaryResponse, shadowResponse] = await Promise.all([
                primaryPromise,
                this.callService(this.shadowUrl, request),
            ]);

            // Compare
            const comparison = this.compare(primaryResponse, shadowResponse);
            await this.logComparison(request, comparison);

            // Metrics
            this.metrics.histogram('shadow.latency_diff',
                shadowResponse.latency - primaryResponse.latency
            );
            this.metrics.histogram('shadow.result_diff', comparison.diff);

            if (comparison.hasDifference) {
                this.metrics.increment('shadow.differences');
            }
        } catch (error) {
            // Shadow failure — NEVER affects user
            this.metrics.increment('shadow.errors');
        }
    }

    compare(primary, shadow) {
        const priceDiff = Math.abs(primary.data.price - shadow.data.price);
        const percentDiff = (priceDiff / primary.data.price) * 100;

        return {
            hasDifference: percentDiff > 5, // ৫% এর বেশি হলে significant
            diff: priceDiff,
            percentDiff,
            primary: primary.data,
            shadow: shadow.data,
        };
    }

    async callService(url, request) {
        const start = Date.now();
        const response = await fetch(url, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(request),
        });
        const data = await response.json();
        return { data, latency: Date.now() - start };
    }

    async logComparison(request, comparison) {
        await this.comparisonStore.save({
            timestamp: new Date(),
            request,
            comparison,
        });
    }
}

/**
 * Dark Launch Implementation
 * Deploy without exposing to users
 */
class DarkLaunchManager {
    constructor(flagManager, metrics) {
        this.flagManager = flagManager;
        this.metrics = metrics;
    }

    /**
     * Dark launch wrapper — old logic returns, new logic runs silently
     */
    async execute(featureName, { oldLogic, newLogic, context }) {
        // ALWAYS return old logic result to user
        const oldResult = await oldLogic();

        // Run new logic in background if dark launch is enabled
        if (this.flagManager.isEnabled(`dark_launch_${featureName}`, context)) {
            setImmediate(async () => {
                try {
                    const start = Date.now();
                    const newResult = await newLogic();
                    const duration = Date.now() - start;

                    // Compare and log
                    this.metrics.histogram(`dark_launch.${featureName}.latency`, duration);
                    this.metrics.increment(`dark_launch.${featureName}.executions`);

                    const match = JSON.stringify(oldResult) === JSON.stringify(newResult);
                    this.metrics.increment(`dark_launch.${featureName}.${match ? 'match' : 'mismatch'}`);

                    if (!match) {
                        console.log(`[DarkLaunch] ${featureName} mismatch:`, {
                            old: oldResult,
                            new: newResult,
                        });
                    }
                } catch (error) {
                    this.metrics.increment(`dark_launch.${featureName}.errors`);
                }
            });
        }

        return oldResult;
    }
}

/**
 * Ring-based Deployment Controller
 * Daraz-এ ব্যবহৃত
 */
class RingDeploymentController {
    constructor(flagManager) {
        this.flagManager = flagManager;
        this.rings = [
            { name: 'internal', percentage: 100, segment: 'employees', duration: '1d' },
            { name: 'beta', percentage: 100, segment: 'beta_users', duration: '3d' },
            { name: 'early_adopters', percentage: 10, segment: 'all', duration: '5d' },
            { name: 'general_50', percentage: 50, segment: 'all', duration: '3d' },
            { name: 'general_100', percentage: 100, segment: 'all', duration: '0' },
        ];
    }

    getCurrentRing(featureName) {
        const config = this.flagManager.getVariation(`ring_${featureName}`, {});
        return this.rings.find(r => r.name === config?.currentRing) || this.rings[0];
    }

    async promoteToNextRing(featureName) {
        const currentRing = this.getCurrentRing(featureName);
        const currentIndex = this.rings.indexOf(currentRing);

        if (currentIndex >= this.rings.length - 1) {
            console.log('✅ Already at GA (General Availability)');
            return;
        }

        const nextRing = this.rings[currentIndex + 1];
        console.log(`📍 Promoting ${featureName}: ${currentRing.name} → ${nextRing.name}`);

        // Update flag rules
        await this.flagManager.updateFlag(`ring_${featureName}`, {
            currentRing: nextRing.name,
            segment: nextRing.segment,
            percentage: nextRing.percentage,
        });
    }
}

/**
 * Feature Flag Cleanup Automation
 * Technical debt management
 */
class FlagCleanupBot {
    constructor(flagManager, codeSearch, notifier) {
        this.flagManager = flagManager;
        this.codeSearch = codeSearch;
        this.notifier = notifier;
    }

    /**
     * Stale flags detect করা
     */
    async findStaleFlags() {
        const allFlags = await this.flagManager.getAllFlags();
        const staleFlags = [];

        for (const flag of allFlags) {
            const staleness = await this.assessStaleness(flag);
            if (staleness.isStale) {
                staleFlags.push({ flag, ...staleness });
            }
        }

        return staleFlags;
    }

    async assessStaleness(flag) {
        const now = new Date();
        const createdAt = new Date(flag.createdAt);
        const ageInDays = (now - createdAt) / (1000 * 60 * 60 * 24);

        // Release flag at 100% for > 7 days
        if (flag.type === 'release' && flag.defaultPercentage === 100 && ageInDays > 7) {
            return {
                isStale: true,
                reason: 'Release flag at 100% — code-এ permanently enable করুন',
                ageInDays: Math.round(ageInDays),
                action: 'REMOVE_FLAG_AND_DEAD_CODE',
            };
        }

        // Flag unused in code
        const usageCount = await this.codeSearch.findUsages(flag.key);
        if (usageCount === 0) {
            return {
                isStale: true,
                reason: 'Code-এ ব্যবহৃত হচ্ছে না',
                action: 'DELETE_FLAG',
            };
        }

        // Experiment flag older than 90 days
        if (flag.type === 'experiment' && ageInDays > 90) {
            return {
                isStale: true,
                reason: 'Experiment ৯০ দিনের বেশি পুরোনো — decision নিন',
                action: 'DECIDE_AND_REMOVE',
            };
        }

        return { isStale: false };
    }

    /**
     * Weekly cleanup report
     */
    async sendCleanupReport() {
        const staleFlags = await this.findStaleFlags();

        if (staleFlags.length === 0) {
            console.log('✅ No stale flags found!');
            return;
        }

        const report = staleFlags.map(({ flag, reason, action }) =>
            `• \`${flag.key}\` (${flag.type}) — ${reason}\n  Action: ${action}`
        ).join('\n\n');

        await this.notifier.send({
            channel: '#tech-debt',
            title: `🧹 Feature Flag Cleanup Report — ${staleFlags.length} stale flags`,
            body: report,
        });
    }
}

// === ব্যবহার ===

// Initialize SDK
const sdk = new ProgressiveDeliverySDK({
    apiUrl: 'https://flags.pathao.internal',
    sdkKey: 'sdk-production-key-xxx',
    refreshInterval: 30000,
    metrics: new PrometheusMetrics(),
});

// Feature flag check — Pathao নতুন surge pricing
const userContext = {
    userId: 'rider_123',
    attributes: {
        city: 'Dhaka',
        rideCount: 250,
        isPremium: true,
        zone: 'Dhanmondi',
    },
};

if (sdk.isEnabled('new-surge-pricing-v2', userContext)) {
    // নতুন surge algorithm
    const price = await calculateSurgePriceV2(ride);
} else {
    // পুরোনো algorithm
    const price = await calculateSurgePriceV1(ride);
}

// A/B Testing — কোন UI ভালো perform করছে?
const uiVariation = sdk.getVariation('checkout-ui-experiment', userContext, 'control');
// Returns: 'control', 'variant_a', or 'variant_b'

// Shadow Traffic — নতুন pricing test
const shadowManager = new ShadowTrafficManager({
    primaryUrl: 'http://pricing-v1.pathao.internal/calculate',
    shadowUrl: 'http://pricing-v2.pathao.internal/calculate',
    comparisonStore: new ElasticsearchStore(),
    metrics: new PrometheusMetrics(),
    sampleRate: 0.5, // ৫০% request mirror
});

const price = await shadowManager.handleRequest({
    from: { lat: 23.7461, lng: 90.3742 }, // Dhanmondi
    to: { lat: 23.8759, lng: 90.3795 },   // Uttara
    rideType: 'car',
    time: new Date(),
});
```

---

## 🛠️ Feature Flag Management Tools

```
┌─────────────────────────────────────────────────────────────┐
│              FEATURE FLAG PLATFORMS                           │
├──────────────────┬──────────────────────────────────────────┤
│ LaunchDarkly     │ Enterprise, best UI, expensive            │
│                  │ Real-time flag updates, A/B testing       │
│                  │ Best for: বড় দল, budget আছে             │
├──────────────────┼──────────────────────────────────────────┤
│ Unleash          │ Open-source, self-hosted                  │
│                  │ Good feature set, free                    │
│                  │ Best for: Budget-conscious teams          │
├──────────────────┼──────────────────────────────────────────┤
│ Flagsmith        │ Open-source + Cloud, good middle ground  │
│                  │ Remote config + flags                     │
│                  │ Best for: Startups (Pathao, bKash)        │
├──────────────────┼──────────────────────────────────────────┤
│ Split.io         │ Enterprise, data-driven                   │
│                  │ Strong analytics integration              │
│                  │ Best for: Data-heavy decisions            │
├──────────────────┼──────────────────────────────────────────┤
│ ConfigCat        │ Simple, affordable                        │
│                  │ Quick setup                               │
│                  │ Best for: ছোট দল, সহজ setup চান          │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 🎯 Feature Flag Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│              FEATURE FLAG BEST PRACTICES                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ DO:                                                      │
│  ─────                                                       │
│  • Flag-এ meaningful নাম দিন (new-qr-payment, not flag1)   │
│  • Expiry date set করুন (release flags-এ)                  │
│  • Flag owner assign করুন (কে maintain করবে)              │
│  • Default OFF রাখুন (safe default)                        │
│  • Flag evaluation কে fast রাখুন (< 1ms)                  │
│  • Cleanup schedule maintain করুন                           │
│  • Flag-এ description রাখুন                                │
│  • Metrics track করুন (কত % ON, OFF)                       │
│                                                              │
│  ❌ DON'T:                                                   │
│  ────────                                                    │
│  • Flag-এর ভেতর flag nest করবেন না                         │
│  • Business logic flag-এ রাখবেন না                         │
│  • Flag remove না করে ফেলে রাখবেন না (tech debt!)          │
│  • সব কিছুতে flag ব্যবহার করবেন না                        │
│  • Flag dependency chain তৈরি করবেন না                     │
│  • Production-এ flag toggle test করবেন না (staging-এ করুন)│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔄 Flag Lifecycle ও Technical Debt

```
┌─────────────────────────────────────────────────────────────┐
│                FLAG LIFECYCLE                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Created     Active        Fully Rolled    Cleanup          │
│     │          │           Out (100%)        │              │
│     ▼          ▼               ▼             ▼              │
│  ┌──────┐  ┌──────┐      ┌──────┐      ┌──────┐          │
│  │ NEW  │─▶│ LIVE │─────▶│ STALE│─────▶│REMOVE│          │
│  └──────┘  └──────┘      └──────┘      └──────┘          │
│                                              │              │
│                                              ▼              │
│                                    ┌────────────────┐      │
│                                    │ • Flag config  │      │
│                                    │   delete       │      │
│                                    │ • if/else code │      │
│                                    │   remove       │      │
│                                    │ • Dead code    │      │
│                                    │   remove       │      │
│                                    │ • Tests update │      │
│                                    └────────────────┘      │
│                                                              │
│  ⚠️ Average company: ৫০+ stale flags = significant          │
│     technical debt!                                          │
│                                                              │
│  🤖 Automation:                                              │
│  • Weekly bot scan for stale flags                          │
│  • PR auto-created to remove dead flags                     │
│  • Dashboard showing flag age distribution                  │
│  • Alert when flag > 30 days at 100%                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন

| কৌশল | কখন ব্যবহার করবেন |
|------|-------------------|
| **Feature Flags** | নতুন feature gradually release করতে, A/B testing, kill switch |
| **Dark Launch** | নতুন system/algorithm production-এ validate করতে (user affect ছাড়া) |
| **Shadow Traffic** | নতুন service-এর performance/accuracy compare করতে |
| **Ring Deployment** | বড় org-এ structured rollout (internal → beta → GA) |
| **A/B Testing** | User behavior measure করতে (কোন variant বেশি convert করে) |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| সব feature-এ flag (over-flagging) | Code complexity বাড়ে, flag jungle |
| Flag cleanup plan ছাড়া | Technical debt pile up |
| Simple bug fix | Flag-এর overhead worth না |
| Shadow traffic + write operations | Duplicate writes! Data corruption! |
| Regulatory requirement (সবাইকে দিতেই হবে) | Flag-এ select করার অর্থ নেই |
| খুব short-lived change (< 1 hour) | Setup overhead > benefit |

---

## 📝 সারসংক্ষেপ

> **Progressive Delivery = "Deploy করো সবার জন্য, কিন্তু Release করো নিয়ন্ত্রিতভাবে"**

মনে রাখুন:
1. **Deploy ≠ Release** — code production-এ থাকতে পারে কিন্তু user দেখবে না
2. **Feature Flags** — কাকে দেখাবো control করে (targeting rules)
3. **Dark Launch** — নতুন logic চলছে পেছনে, কিন্তু user পুরোনো result পাচ্ছে
4. **Shadow Traffic** — real traffic copy করে নতুন service test করো
5. **Flag Cleanup** — সবচেয়ে গুরুত্বপূর্ণ! না করলে tech debt monster তৈরি হবে
6. **Ring Deployment** — Internal → Beta → Early Adopters → Everyone
