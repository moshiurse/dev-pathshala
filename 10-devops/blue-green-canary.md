# 🔵🟢 Blue-Green & Canary Deployment Deep Dive

## 📖 সংজ্ঞা ও ধারণা

### Blue-Green Deployment

Blue-Green Deployment হলো একটি deployment strategy যেখানে **দুটি identical production environment** থাকে — Blue (বর্তমান) এবং Green (নতুন)। নতুন version Green-এ deploy করা হয়, test করা হয়, তারপর **traffic instantly switch** করা হয়।

### Canary Deployment

Canary Deployment হলো **ধীরে ধীরে traffic shift** করার কৌশল — প্রথমে ১%, তারপর ৫%, ২৫%, এবং সবশেষে ১০০% traffic নতুন version-এ পাঠানো হয়। প্রতিটি ধাপে metrics monitor করা হয়।

### Rolling Deployment

Rolling Deployment-এ **একটি একটি করে instance** নতুন version-এ upgrade হয়। সময় বেশি লাগে কিন্তু resource কম লাগে।

```
┌─────────────────────────────────────────────────────────────┐
│                  তুলনামূলক সারণী                             │
├──────────────┬────────────┬────────────┬───────────────────┤
│ বৈশিষ্ট্য    │ Blue-Green │ Canary     │ Rolling           │
├──────────────┼────────────┼────────────┼───────────────────┤
│ Speed        │ Instant    │ Gradual    │ Gradual           │
│ Risk         │ Low        │ Very Low   │ Medium            │
│ Cost         │ High (2x)  │ Medium     │ Low               │
│ Rollback     │ Instant    │ Fast       │ Slow              │
│ Complexity   │ Medium     │ High       │ Low               │
│ Zero-downtime│ ✅ Yes     │ ✅ Yes     │ ✅ Yes            │
│ DB Migration │ কঠিন       │ কঠিন      │ পর্যায়ক্রমে       │
└──────────────┴────────────┴────────────┴───────────────────┘
```

---

## 🌍 বাস্তব জীবনের উদাহরণ (বাংলাদেশ প্রসঙ্গ)

### 🛒 Daraz — নতুন Checkout Service Deploy

Daraz তাদের checkout service-এ নতুন payment integration (bKash + Nagad combined) deploy করতে চায়। ভুল হলে কোটি টাকার order হারানোর ঝুঁকি!

**Canary Strategy:**
```
ধাপ ১: 1% traffic (শুধু internal team)     → ১ ঘন্টা monitor
ধাপ ২: 5% traffic (Dhaka region only)      → ২ ঘন্টা monitor
ধাপ ৩: 25% traffic (All regions)           → ৪ ঘন্টা monitor
ধাপ ৪: 50% traffic                         → ৮ ঘন্টা monitor
ধাপ ৫: 100% traffic (Full rollout)         → সম্পূর্ণ!
```

### 📡 Grameenphone — MyGP App Update

MyGP app-এর backend API update-এ Blue-Green deployment:
- **Blue:** বর্তমান v2.3 (কোটি user serve করছে)
- **Green:** নতুন v2.4 (new features, tested)
- Load balancer switch → instant migration
- সমস্যা হলে ১ সেকেন্ডে Blue-তে ফিরে আসা

### 🏦 bKash — Critical Payment Gateway

bKash-এ payment processing update:
- Canary দিয়ে প্রথমে ১% internal transaction
- Error rate monitor করা
- P99 latency check করা
- সব OK হলে gradually বাড়ানো

---

## 📊 ASCII Diagram — Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  BEFORE SWITCH:                                                      │
│  ─────────────                                                       │
│                     ┌──────────────────┐                             │
│                ┌───▶│  BLUE (v2.3)     │  ◄── Active, serving       │
│  ┌────────┐   │    │  Production      │      all traffic           │
│  │ Users  │───┤    └──────────────────┘                             │
│  └────────┘   │                                                      │
│               │    ┌──────────────────┐                             │
│               └──X─│  GREEN (v2.4)    │  ◄── Idle, new version     │
│                    │  Ready to switch  │      deployed & tested     │
│                    └──────────────────┘                             │
│                                                                      │
│                                                                      │
│  AFTER SWITCH:                                                       │
│  ────────────                                                        │
│                     ┌──────────────────┐                             │
│                ┌─X──│  BLUE (v2.3)     │  ◄── Idle (rollback-ready)│
│  ┌────────┐   │    │  Standby         │                            │
│  │ Users  │───┤    └──────────────────┘                             │
│  └────────┘   │                                                      │
│               │    ┌──────────────────┐                             │
│               └───▶│  GREEN (v2.4)    │  ◄── Active, serving       │
│                    │  Production      │      all traffic           │
│                    └──────────────────┘                             │
│                                                                      │
│  ROLLBACK: Load Balancer আবার Blue-তে switch = instant!            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Canary Deployment Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CANARY DEPLOYMENT                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐         ┌─────────────────────────────────┐             │
│  │        │         │      LOAD BALANCER              │             │
│  │ Users  │────────▶│   (Traffic Splitting)           │             │
│  │ 100%   │         └──────────┬──────────────────────┘             │
│  └────────┘                    │                                     │
│                       ┌────────┴────────┐                           │
│                       │                 │                            │
│                       ▼                 ▼                            │
│              ┌─────────────┐   ┌─────────────┐                      │
│              │ Stable v2.3 │   │ Canary v2.4 │                      │
│              │   (99%)     │   │    (1%)     │                      │
│              └──────┬──────┘   └──────┬──────┘                      │
│                     │                 │                              │
│                     ▼                 ▼                              │
│              ┌──────────────────────────────┐                       │
│              │      METRICS COMPARISON       │                       │
│              │  ┌─────────┬───────────────┐ │                       │
│              │  │ Metric  │ Stable│Canary │ │                       │
│              │  ├─────────┼───────┼───────┤ │                       │
│              │  │ Error % │ 0.1%  │ 0.2%  │ │  ← OK ✅             │
│              │  │ Latency │ 120ms │ 180ms │ │  ← OK ✅             │
│              │  │ CPU     │ 45%   │ 55%   │ │  ← OK ✅             │
│              │  └─────────┴───────┴───────┘ │                       │
│              └──────────────────────────────┘                       │
│                              │                                       │
│                    ┌─────────┴─────────┐                            │
│                    ▼                   ▼                             │
│           ┌──────────────┐    ┌──────────────┐                      │
│           │  PROMOTE ✅  │    │ ROLLBACK ❌  │                      │
│           │  1%→5%→25%   │    │  Back to 0%  │                      │
│           │  →50%→100%   │    │              │                      │
│           └──────────────┘    └──────────────┘                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Migration Challenges

### Blue-Green + Database

```
┌─────────────────────────────────────────────────────────────┐
│          DATABASE BACKWARD COMPATIBILITY                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  সমস্যা: Blue এবং Green একই database ব্যবহার করে!         │
│                                                              │
│  ┌──────────┐     ┌──────────┐                              │
│  │  BLUE    │────▶│          │◀────┌──────────┐            │
│  │  (v2.3)  │     │ DATABASE │     │  GREEN   │            │
│  └──────────┘     │ (shared) │     │  (v2.4)  │            │
│                    └──────────┘     └──────────┘            │
│                                                              │
│  সমাধান: Expand-Contract Pattern                            │
│  ─────────────────────────────────                          │
│                                                              │
│  Phase 1 (EXPAND): নতুন column যোগ, পুরাতন রাখা            │
│  ┌─────────────────────────────────────────┐                │
│  │ users table:                            │                │
│  │   - phone (existing)                    │                │
│  │   - phone_number (new, nullable)  ← ADD │                │
│  └─────────────────────────────────────────┘                │
│  → Blue v2.3 phone ব্যবহার করে (OK)                        │
│  → Green v2.4 phone_number ব্যবহার করে (OK)                │
│                                                              │
│  Phase 2 (MIGRATE): Data copy করা                           │
│  → phone → phone_number (backfill)                          │
│                                                              │
│  Phase 3 (CONTRACT): পুরাতন column remove                   │
│  → Green v2.5 deploy হলে phone remove                       │
│  → Blue তখন আর দরকার নেই                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Blue-Green Deployment Manager (PHP)

```php
<?php

namespace App\Deployment;

/**
 * Blue-Green Deployment Manager
 * Daraz-এর মতো e-commerce platform-এ ব্যবহার
 */
class BlueGreenDeploymentManager
{
    private LoadBalancer $loadBalancer;
    private HealthChecker $healthChecker;
    private MetricsService $metrics;
    private NotificationService $notifications;

    private string $activeEnvironment; // 'blue' or 'green'
    private array $environments;

    public function __construct(
        LoadBalancer $loadBalancer,
        HealthChecker $healthChecker,
        MetricsService $metrics,
        NotificationService $notifications
    ) {
        $this->loadBalancer = $loadBalancer;
        $this->healthChecker = $healthChecker;
        $this->metrics = $metrics;
        $this->notifications = $notifications;

        $this->environments = [
            'blue' => ['url' => 'https://blue.daraz.internal', 'instances' => 10],
            'green' => ['url' => 'https://green.daraz.internal', 'instances' => 10],
        ];
    }

    /**
     * নতুন version deploy করা (inactive environment-এ)
     */
    public function deployNewVersion(string $version, string $dockerImage): DeploymentResult
    {
        $targetEnv = $this->getInactiveEnvironment();

        echo "🚀 Deploying v{$version} to {$targetEnv} environment\n";
        echo "📦 Image: {$dockerImage}\n";

        // Step 1: Inactive environment-এ deploy
        $this->deployToEnvironment($targetEnv, $dockerImage);

        // Step 2: Health check
        echo "🏥 Health check চলছে...\n";
        $healthy = $this->healthChecker->checkAll($this->environments[$targetEnv]['url']);

        if (!$healthy) {
            return new DeploymentResult(
                success: false,
                message: "❌ Health check ব্যর্থ! Deploy abort।"
            );
        }

        // Step 3: Smoke tests চালানো
        echo "🧪 Smoke tests চলছে...\n";
        $smokeTestsPassed = $this->runSmokeTests($targetEnv);

        if (!$smokeTestsPassed) {
            return new DeploymentResult(
                success: false,
                message: "❌ Smoke tests ব্যর্থ! Deploy abort।"
            );
        }

        echo "✅ {$targetEnv} environment প্রস্তুত। Switch করতে switchTraffic() call করুন।\n";

        return new DeploymentResult(
            success: true,
            message: "Deploy সফল। Traffic switch pending।",
            targetEnvironment: $targetEnv,
            version: $version,
        );
    }

    /**
     * Traffic switch করা — instant cutover
     */
    public function switchTraffic(): SwitchResult
    {
        $currentActive = $this->activeEnvironment;
        $newActive = $this->getInactiveEnvironment();

        echo "🔄 Switching traffic: {$currentActive} → {$newActive}\n";

        // Session draining — existing sessions শেষ হতে দেওয়া
        echo "⏳ Session draining (30s)...\n";
        $this->loadBalancer->drainConnections($currentActive, timeoutSeconds: 30);

        // Load balancer switch
        $this->loadBalancer->setActiveBackend($newActive);
        $this->activeEnvironment = $newActive;

        // Verify switch
        $isServing = $this->verifyTrafficServing($newActive);

        if (!$isServing) {
            // Immediate rollback!
            echo "🚨 Verification failed! Auto-rollback...\n";
            $this->loadBalancer->setActiveBackend($currentActive);
            $this->activeEnvironment = $currentActive;

            return new SwitchResult(success: false, rolledBack: true);
        }

        $this->notifications->send(
            "✅ Traffic switch সম্পন্ন: {$currentActive} → {$newActive}"
        );

        return new SwitchResult(success: true, activeEnv: $newActive);
    }

    /**
     * Instant rollback — আগের environment-এ ফেরত
     */
    public function rollback(): void
    {
        $currentActive = $this->activeEnvironment;
        $previousActive = $this->getInactiveEnvironment();

        echo "⚡ INSTANT ROLLBACK: {$currentActive} → {$previousActive}\n";

        $this->loadBalancer->setActiveBackend($previousActive);
        $this->activeEnvironment = $previousActive;

        $this->notifications->send(
            "🔴 ROLLBACK executed! Active: {$previousActive}"
        );
    }

    private function getInactiveEnvironment(): string
    {
        return $this->activeEnvironment === 'blue' ? 'green' : 'blue';
    }

    private function deployToEnvironment(string $env, string $image): void
    {
        // Kubernetes deployment update বা Docker Swarm service update
    }

    private function runSmokeTests(string $env): bool
    {
        $tests = [
            'homepage_loads' => '/health',
            'api_responds' => '/api/v1/status',
            'db_connected' => '/api/v1/db-check',
            'cache_working' => '/api/v1/cache-check',
        ];

        foreach ($tests as $name => $endpoint) {
            $url = $this->environments[$env]['url'] . $endpoint;
            $response = $this->httpGet($url);

            if ($response->statusCode !== 200) {
                echo "  ❌ {$name} failed ({$url})\n";
                return false;
            }
            echo "  ✅ {$name} passed\n";
        }
        return true;
    }

    private function verifyTrafficServing(string $env): bool
    {
        sleep(5); // ৫ সেকেন্ড অপেক্ষা
        $metrics = $this->metrics->getRequestCount($env, lastSeconds: 5);
        return $metrics > 0;
    }

    private function httpGet(string $url): object
    {
        return (object) ['statusCode' => 200]; // Simplified
    }
}

/**
 * Canary Deployment Controller
 * Automated Canary Analysis সহ
 */
class CanaryDeploymentController
{
    private LoadBalancer $loadBalancer;
    private MetricsService $metrics;
    private AlertService $alerts;

    private array $stages = [
        ['percentage' => 1, 'duration' => 3600, 'label' => 'Internal Team'],
        ['percentage' => 5, 'duration' => 7200, 'label' => 'Dhaka Region'],
        ['percentage' => 25, 'duration' => 14400, 'label' => 'All Regions'],
        ['percentage' => 50, 'duration' => 28800, 'label' => 'Half Traffic'],
        ['percentage' => 100, 'duration' => 0, 'label' => 'Full Rollout'],
    ];

    private array $canaryCriteria = [
        'error_rate' => ['max' => 1.0, 'comparison' => 'absolute'],     // ১% এর বেশি error না
        'p99_latency' => ['max' => 1.5, 'comparison' => 'relative'],    // ১.৫x এর বেশি slow না
        'cpu_usage' => ['max' => 80, 'comparison' => 'absolute'],       // ৮০% এর নিচে
        'memory_usage' => ['max' => 85, 'comparison' => 'absolute'],    // ৮৫% এর নিচে
    ];

    /**
     * Canary deployment শুরু করা
     */
    public function startCanary(string $version, string $image): CanaryResult
    {
        echo "🐤 Canary deployment শুরু: v{$version}\n";
        echo "📊 Stages: " . count($this->stages) . "\n\n";

        foreach ($this->stages as $index => $stage) {
            $stageNum = $index + 1;
            echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n";
            echo "📍 Stage {$stageNum}: {$stage['label']} ({$stage['percentage']}%)\n";
            echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n";

            // Traffic shift
            $this->loadBalancer->setCanaryWeight($stage['percentage']);

            if ($stage['duration'] === 0) {
                echo "🎉 Full rollout complete!\n";
                break;
            }

            // Monitoring period
            $passed = $this->monitorCanary($stage['duration']);

            if (!$passed) {
                echo "\n🚨 Canary FAILED at stage {$stageNum}! Rolling back...\n";
                $this->rollbackCanary();
                return new CanaryResult(
                    success: false,
                    failedAtStage: $stageNum,
                    reason: 'Canary criteria ভায়োলেশন',
                );
            }

            echo "  ✅ Stage {$stageNum} passed! Advancing...\n\n";
        }

        return new CanaryResult(success: true, version: $version);
    }

    /**
     * Automated Canary Analysis
     */
    private function monitorCanary(int $durationSeconds): bool
    {
        $checkInterval = 60; // প্রতি মিনিটে check
        $checks = $durationSeconds / $checkInterval;

        for ($i = 0; $i < $checks; $i++) {
            sleep($checkInterval);

            $stableMetrics = $this->metrics->getMetrics('stable');
            $canaryMetrics = $this->metrics->getMetrics('canary');

            foreach ($this->canaryCriteria as $metric => $criteria) {
                $passed = $this->evaluateMetric(
                    $metric,
                    $stableMetrics[$metric],
                    $canaryMetrics[$metric],
                    $criteria
                );

                if (!$passed) {
                    echo "  ❌ {$metric} criteria ভায়োলেশন!\n";
                    echo "     Stable: {$stableMetrics[$metric]}, Canary: {$canaryMetrics[$metric]}\n";
                    return false;
                }
            }

            $progress = round(($i + 1) / $checks * 100);
            echo "  ⏳ Monitoring: {$progress}% ({$metric} OK)\r";
        }

        return true;
    }

    private function evaluateMetric(string $metric, float $stable, float $canary, array $criteria): bool
    {
        if ($criteria['comparison'] === 'absolute') {
            return $canary <= $criteria['max'];
        }

        // Relative comparison — canary value stable-এর x গুণের বেশি না
        return $canary <= ($stable * $criteria['max']);
    }

    private function rollbackCanary(): void
    {
        $this->loadBalancer->setCanaryWeight(0);
        echo "✅ Canary traffic 0%. Rollback সম্পন্ন।\n";
        $this->alerts->send('Canary deployment rollback হয়েছে');
    }
}

/**
 * Database Migration with Blue-Green
 * Expand-Contract Pattern implementation
 */
class BlueGreenDatabaseMigration
{
    /**
     * Phase 1: EXPAND — নতুন column/table যোগ (backward compatible)
     */
    public function expand(): void
    {
        // পুরাতন code (Blue) এবং নতুন code (Green) দুটোই কাজ করবে
        $migrations = [
            "ALTER TABLE orders ADD COLUMN payment_method_v2 VARCHAR(50) NULL",
            "ALTER TABLE orders ADD COLUMN metadata_json JSON NULL",
            // পুরাতন column 'payment_method' রেখে দেওয়া হচ্ছে!
        ];

        foreach ($migrations as $sql) {
            echo "  📝 Running: {$sql}\n";
            // DB::statement($sql);
        }
    }

    /**
     * Phase 2: MIGRATE — Data backfill
     */
    public function migrateData(): void
    {
        echo "  📊 Backfilling payment_method → payment_method_v2...\n";
        // UPDATE orders SET payment_method_v2 = payment_method WHERE payment_method_v2 IS NULL
        // Batch processing to avoid lock
    }

    /**
     * Phase 3: CONTRACT — পুরাতন column remove (পরবর্তী deploy-এ)
     */
    public function contract(): void
    {
        // শুধুমাত্র যখন Blue environment আর ব্যবহার হচ্ছে না
        $migrations = [
            "ALTER TABLE orders DROP COLUMN payment_method",
            "ALTER TABLE orders RENAME COLUMN payment_method_v2 TO payment_method",
        ];
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Canary Deployment with Argo Rollouts (JavaScript)

```javascript
/**
 * Canary Deployment Controller
 * Daraz checkout service-এ ব্যবহার
 */

class CanaryDeploymentManager {
    constructor(config) {
        this.kubeClient = config.kubeClient;
        this.metricsProvider = config.metricsProvider; // Prometheus
        this.notifier = config.notifier;              // Slack notification
        this.canaryConfig = config.canaryConfig;
    }

    /**
     * Canary rollout শুরু করা
     */
    async startRollout(deployment) {
        const { name, namespace, image, stages } = deployment;

        console.log(`\n🐤 Starting Canary Rollout`);
        console.log(`   Service: ${name}`);
        console.log(`   Image: ${image}`);
        console.log(`   Stages: ${stages.length}`);
        console.log('');

        const rolloutState = {
            name,
            startedAt: new Date(),
            currentStage: 0,
            status: 'in_progress',
            metrics: [],
        };

        for (let i = 0; i < stages.length; i++) {
            const stage = stages[i];
            rolloutState.currentStage = i;

            console.log(`\n${'═'.repeat(50)}`);
            console.log(`📍 Stage ${i + 1}/${stages.length}: ${stage.percentage}% traffic`);
            console.log(`   Duration: ${stage.duration / 60000} minutes`);
            console.log(`${'═'.repeat(50)}`);

            // Traffic weight set করা
            await this.setTrafficWeight(name, namespace, stage.percentage);
            console.log(`   ✅ Traffic weight set: ${stage.percentage}%`);

            // Bake time — metrics collect করার জন্য অপেক্ষা
            if (stage.duration > 0) {
                const analysis = await this.runCanaryAnalysis(name, namespace, stage);

                rolloutState.metrics.push({
                    stage: i + 1,
                    percentage: stage.percentage,
                    analysis,
                });

                if (!analysis.passed) {
                    console.log(`\n🚨 CANARY FAILED at Stage ${i + 1}!`);
                    console.log(`   Reason: ${analysis.failureReason}`);
                    await this.rollback(name, namespace);
                    rolloutState.status = 'failed';
                    await this.notifier.send({
                        level: 'critical',
                        message: `❌ Canary rollout "${name}" ব্যর্থ! Stage ${i + 1} এ rollback হয়েছে।`,
                        details: analysis,
                    });
                    return rolloutState;
                }

                console.log(`   ✅ Stage ${i + 1} analysis passed!`);
            }
        }

        rolloutState.status = 'completed';
        console.log(`\n🎉 Canary rollout সম্পূর্ণ! v${deployment.version} is now serving 100% traffic`);

        await this.notifier.send({
            level: 'success',
            message: `✅ Canary rollout "${name}" সফল!`,
        });

        return rolloutState;
    }

    /**
     * Automated Canary Analysis — metrics-based decision
     */
    async runCanaryAnalysis(serviceName, namespace, stage) {
        const checkInterval = 30000; // ৩০ সেকেন্ড interval
        const totalChecks = Math.ceil(stage.duration / checkInterval);
        const criteria = this.canaryConfig.criteria;

        let consecutiveFailures = 0;
        const maxConsecutiveFailures = 3;

        for (let check = 0; check < totalChecks; check++) {
            await this.sleep(checkInterval);

            // Metrics collect
            const stableMetrics = await this.metricsProvider.query({
                service: serviceName,
                subset: 'stable',
                window: '5m',
            });

            const canaryMetrics = await this.metricsProvider.query({
                service: serviceName,
                subset: 'canary',
                window: '5m',
            });

            // Each criterion evaluate করা
            const results = this.evaluateCriteria(criteria, stableMetrics, canaryMetrics);

            const allPassed = results.every(r => r.passed);

            if (!allPassed) {
                consecutiveFailures++;
                const failedCriteria = results.filter(r => !r.passed);
                console.log(`   ⚠️  Check ${check + 1}/${totalChecks} FAILED (${consecutiveFailures}/${maxConsecutiveFailures})`);
                failedCriteria.forEach(f => {
                    console.log(`      - ${f.metric}: ${f.canaryValue} (limit: ${f.limit})`);
                });

                if (consecutiveFailures >= maxConsecutiveFailures) {
                    return {
                        passed: false,
                        failureReason: `${maxConsecutiveFailures} consecutive failures`,
                        failedCriteria: results.filter(r => !r.passed),
                    };
                }
            } else {
                consecutiveFailures = 0; // Reset on success
                const progress = Math.round(((check + 1) / totalChecks) * 100);
                process.stdout.write(`   ⏳ Analysis: ${progress}% complete (all metrics OK)\r`);
            }
        }

        console.log('');
        return { passed: true, totalChecks, results: 'all_passed' };
    }

    /**
     * Canary criteria evaluation
     */
    evaluateCriteria(criteria, stableMetrics, canaryMetrics) {
        return Object.entries(criteria).map(([metric, config]) => {
            const stableValue = stableMetrics[metric] || 0;
            const canaryValue = canaryMetrics[metric] || 0;

            let passed;
            let limit;

            switch (config.type) {
                case 'threshold':
                    // Absolute threshold — canary value must be below max
                    limit = config.max;
                    passed = canaryValue <= config.max;
                    break;

                case 'deviation':
                    // Relative deviation — canary shouldn't deviate much from stable
                    limit = stableValue * (1 + config.maxDeviation);
                    passed = canaryValue <= limit;
                    break;

                case 'ratio':
                    // Ratio comparison
                    limit = stableValue * config.maxRatio;
                    passed = canaryValue <= limit;
                    break;

                default:
                    passed = true;
                    limit = 'N/A';
            }

            return { metric, stableValue, canaryValue, limit, passed };
        });
    }

    /**
     * Traffic weight set করা (Kubernetes Istio/Nginx)
     */
    async setTrafficWeight(name, namespace, percentage) {
        // Istio VirtualService update
        const virtualService = {
            apiVersion: 'networking.istio.io/v1beta1',
            kind: 'VirtualService',
            metadata: { name, namespace },
            spec: {
                http: [{
                    route: [
                        { destination: { host: name, subset: 'stable' }, weight: 100 - percentage },
                        { destination: { host: name, subset: 'canary' }, weight: percentage },
                    ]
                }]
            }
        };

        await this.kubeClient.apply(virtualService);
    }

    /**
     * Rollback — canary traffic বন্ধ করা
     */
    async rollback(name, namespace) {
        console.log('   🔄 Rolling back canary...');
        await this.setTrafficWeight(name, namespace, 0);
        // Canary pods delete করা
        await this.kubeClient.deleteDeployment(`${name}-canary`, namespace);
        console.log('   ✅ Rollback complete. 100% traffic on stable.');
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

/**
 * Blue-Green Deployment Controller
 */
class BlueGreenController {
    constructor(loadBalancer, healthChecker) {
        this.loadBalancer = loadBalancer;
        this.healthChecker = healthChecker;
        this.activeSlot = 'blue'; // current active
    }

    async deploy(version, image) {
        const targetSlot = this.activeSlot === 'blue' ? 'green' : 'blue';
        console.log(`\n🔵🟢 Blue-Green Deploy`);
        console.log(`   Active: ${this.activeSlot} | Target: ${targetSlot}`);
        console.log(`   Version: ${version}`);

        // Deploy to inactive slot
        await this.deployToSlot(targetSlot, image);

        // Health check
        const healthy = await this.healthChecker.check(targetSlot);
        if (!healthy) {
            throw new Error(`${targetSlot} health check ব্যর্থ!`);
        }

        // Ready to switch
        return {
            ready: true,
            switchCommand: () => this.switchTraffic(targetSlot),
            rollbackCommand: () => this.switchTraffic(this.activeSlot),
        };
    }

    async switchTraffic(targetSlot) {
        // Session draining
        console.log(`   ⏳ Draining sessions from ${this.activeSlot}...`);
        await this.loadBalancer.drain(this.activeSlot, { timeout: 30000 });

        // Switch
        await this.loadBalancer.route('*', targetSlot);
        this.activeSlot = targetSlot;
        console.log(`   ✅ Traffic now serving from: ${targetSlot}`);
    }

    async deployToSlot(slot, image) {
        console.log(`   📦 Deploying ${image} to ${slot}...`);
    }
}

// === ব্যবহার: Daraz Checkout Service ===

const canaryManager = new CanaryDeploymentManager({
    kubeClient: new KubernetesClient(),
    metricsProvider: new PrometheusClient('http://prometheus:9090'),
    notifier: new SlackNotifier('#deployments'),
    canaryConfig: {
        criteria: {
            errorRate: { type: 'threshold', max: 1.0 },        // ১% max error
            p99Latency: { type: 'deviation', maxDeviation: 0.5 }, // ৫০% বেশি slow হলে fail
            successRate: { type: 'threshold', max: -99 },        // Inverted: must be > 99%
            cpuUsage: { type: 'threshold', max: 80 },           // ৮০% max CPU
        },
    },
});

// Run canary deployment
canaryManager.startRollout({
    name: 'checkout-service',
    namespace: 'production',
    version: '2.4.0',
    image: 'daraz/checkout:2.4.0',
    stages: [
        { percentage: 1, duration: 300000 },    // ১% — ৫ মিনিট
        { percentage: 5, duration: 600000 },    // ৫% — ১০ মিনিট
        { percentage: 25, duration: 1800000 },  // ২৫% — ৩০ মিনিট
        { percentage: 50, duration: 3600000 },  // ৫০% — ১ ঘন্টা
        { percentage: 100, duration: 0 },       // ১০০% — complete
    ],
}).then(result => {
    console.log('\nFinal Status:', result.status);
});
```

---

## 🛠️ Tools ও Platforms

```
┌─────────────────────────────────────────────────────────────┐
│                  DEPLOYMENT TOOLS                             │
├──────────────────┬──────────────────────────────────────────┤
│ Argo Rollouts    │ Kubernetes-native, Canary + Blue-Green   │
│                  │ Automated analysis, Istio/Nginx support  │
├──────────────────┼──────────────────────────────────────────┤
│ Flagger          │ Progressive delivery for Kubernetes      │
│                  │ Automated canary with metrics analysis   │
├──────────────────┼──────────────────────────────────────────┤
│ Spinnaker        │ Netflix-built, multi-cloud               │
│                  │ Complex pipeline support                 │
├──────────────────┼──────────────────────────────────────────┤
│ AWS CodeDeploy   │ AWS native, easy setup                   │
│                  │ Blue-Green + Canary on ECS/EC2           │
├──────────────────┼──────────────────────────────────────────┤
│ Istio            │ Service mesh traffic splitting           │
│                  │ Fine-grained routing rules               │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 🔀 Traffic Splitting Mechanisms

```
┌─────────────────────────────────────────────────────────┐
│           TRAFFIC SPLITTING METHODS                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. WEIGHTED ROUTING (শতাংশ ভিত্তিক)                   │
│     ┌─────────────┐  95%  ┌────────┐                   │
│     │             │──────▶│ Stable │                   │
│     │ Load        │       └────────┘                   │
│     │ Balancer    │                                     │
│     │             │   5%  ┌────────┐                   │
│     │             │──────▶│ Canary │                   │
│     └─────────────┘       └────────┘                   │
│                                                          │
│  2. HEADER-BASED ROUTING (Header দিয়ে control)          │
│     X-Canary: true  → Canary version                   │
│     X-Canary: false → Stable version                   │
│     Internal testing-এ খুব উপকারী                      │
│                                                          │
│  3. COOKIE-BASED (User sticky)                          │
│     Set-Cookie: canary=true                             │
│     একবার canary-তে গেলে সেখানেই থাকবে                │
│                                                          │
│  4. GEOGRAPHIC (অঞ্চল ভিত্তিক)                         │
│     Dhaka → Canary                                      │
│     Chittagong → Stable                                 │
│     ← Daraz-এ regional rollout-এ ব্যবহৃত              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন

### Blue-Green:
| পরিস্থিতি | করবেন? |
|-----------|--------|
| Instant rollback দরকার | ✅ Blue-Green |
| Budget আছে 2x infrastructure-এর জন্য | ✅ Blue-Green |
| DB schema change নেই | ✅ Blue-Green |
| Compliance — full environment testing | ✅ Blue-Green |
| Eid/Sale-এর আগে deploy | ✅ Blue-Green |

### Canary:
| পরিস্থিতি | করবেন? |
|-----------|--------|
| High-traffic service (millions of users) | ✅ Canary |
| Risk minimize করতে চান | ✅ Canary |
| Automated metrics analysis সম্ভব | ✅ Canary |
| Gradual confidence build করতে চান | ✅ Canary |
| A/B testing-ও করতে চান | ✅ Canary |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| খুব ছোট application (single server) | Overhead বেশি |
| Database breaking change আছে | Blue-Green কঠিন |
| Real-time monitoring নেই | Canary-তে decision নিতে পারবেন না |
| Budget সীমিত (Blue-Green) | 2x resource দরকার |
| Very low traffic (canary-র metrics insufficient) | Statistical significance পাবেন না |

---

## 📝 সারসংক্ষেপ

> **Blue-Green = দুটি ঘর, একটি switch — সমস্যা হলে আগের ঘরে ফিরে যাও।**
> **Canary = আস্তে আস্তে দরজা খোলো — বিপদ দেখলে বন্ধ করো।**

মনে রাখুন:
1. Blue-Green: Instant switch, high cost, simple
2. Canary: Gradual, metric-driven, complex but safer
3. Database migration সবচেয়ে কঠিন challenge — Expand-Contract pattern ব্যবহার করুন
4. Monitoring ছাড়া কোনো advanced deployment strategy কাজ করবে না
5. সবসময় rollback plan রাখুন — যত automated তত ভালো
