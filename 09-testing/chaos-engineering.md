# 🔥 Chaos Engineering & Fault Injection

## 📖 সংজ্ঞা ও ধারণা

Chaos Engineering হলো একটি ডিসিপ্লিন যেখানে আমরা **ইচ্ছাকৃতভাবে সিস্টেমে ব্যর্থতা ইনজেক্ট করি** যাতে সিস্টেমের দুর্বলতাগুলো আগে থেকেই চিহ্নিত করা যায়। এটি reactive নয়, বরং **proactive** পদ্ধতি — সমস্যা হওয়ার আগেই সমস্যা খুঁজে বের করা।

### 🎬 Netflix Chaos Monkey এর ইতিহাস

২০১০ সালে Netflix যখন তাদের ডাটা সেন্টার থেকে AWS-এ মাইগ্রেট করছিল, তখন তারা বুঝতে পারলো যে distributed system-এ যেকোনো সময় যেকোনো কিছু fail হতে পারে। তাই তারা **Chaos Monkey** তৈরি করলো — একটি টুল যা randomly production server গুলোকে kill করে। এভাবে তাদের ইঞ্জিনিয়াররা বাধ্য হলো resilient system ডিজাইন করতে।

> "আমরা চাই আমাদের সিস্টেম এতটাই শক্তিশালী হোক যে একটা সার্ভার মারা গেলেও কাস্টমার বুঝতেই না পারে।" — Netflix Engineering

### 🧠 Chaos Engineering এর মূলনীতি

```
১. Steady State Hypothesis (স্থিতিশীল অবস্থার অনুমান)
   → সিস্টেমের স্বাভাবিক আচরণ কী হওয়া উচিত তা define করুন

২. Vary Real-World Events (বাস্তব ঘটনা অনুকরণ)
   → সত্যিকারের ব্যর্থতা অনুকরণ করুন (network lag, server crash)

৩. Run in Production (প্রোডাকশনে চালান)
   → staging-এ পাওয়া সমস্যা production-এর সমস্যার সাথে মেলে না

৪. Automate (স্বয়ংক্রিয় করুন)
   → মানুষের উপর নির্ভর না করে automated experiment চালান

৫. Minimize Blast Radius (ক্ষতির পরিধি সীমিত রাখুন)
   → ছোট পরিসরে শুরু করুন, ধীরে ধীরে বাড়ান
```

---

## 🌍 বাস্তব জীবনের উদাহরণ (বাংলাদেশ প্রসঙ্গ)

### 🏦 bKash — ঈদের সময় ব্যাংক গেটওয়ে অকেজো

কল্পনা করুন, ঈদুল ফিতরের দিন সকাল ১০টায় লক্ষ লক্ষ মানুষ bKash দিয়ে সালামি পাঠাচ্ছে। ঠিক সেই মুহূর্তে **Dutch-Bangla Bank গেটওয়ে unreachable** হয়ে গেলো।

**Chaos Engineering ছাড়া:**
- সিস্টেম hang হয়ে যাবে
- সব transaction timeout হবে
- কাস্টমার হেল্পলাইনে কল বন্যা
- বিজনেস লস কোটি টাকা

**Chaos Engineering দিয়ে আগেই টেস্ট করলে:**
- Circuit breaker সক্রিয় হবে
- Alternative gateway-তে route হবে
- Customer কে graceful error message দেখাবে
- Retry queue-তে transaction জমা হবে

### 🚗 Pathao — GPS Service Down

Pathao-র ride matching নির্ভর করে GPS location-এর উপর। যদি GPS service provider-এ সমস্যা হয়?

**Chaos Experiment:** GPS service-এ ৫ সেকেন্ড latency inject করো এবং দেখো ride matching কতটুকু degrade হয়।

### 📰 Prothom Alo — CDN Failure

Breaking news-এর সময় যদি CDN fail করে? Chaos testing দিয়ে আগে থেকে origin server-এর capacity check করা।

---

## 📊 ASCII Diagram — Chaos Engineering Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                  CHAOS ENGINEERING WORKFLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   STEADY     │    │   INJECT     │    │   OBSERVE    │      │
│  │   STATE      │───▶│   FAILURE    │───▶│   BEHAVIOR   │      │
│  │   DEFINE     │    │              │    │              │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                                        │               │
│         │                                        ▼               │
│         │                                ┌──────────────┐       │
│         │                                │   COMPARE    │       │
│         └───────────────────────────────▶│   WITH       │       │
│                                          │   HYPOTHESIS │       │
│                                          └──────┬───────┘       │
│                                                 │               │
│                              ┌──────────────────┼────────┐      │
│                              ▼                           ▼      │
│                     ┌──────────────┐          ┌──────────────┐  │
│                     │  PASSED ✅   │          │  FAILED ❌   │  │
│                     │  System is   │          │  Found weak  │  │
│                     │  resilient   │          │  point! Fix! │  │
│                     └──────────────┘          └──────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 🎯 Blast Radius Control

```
┌────────────────────────────────────────────────────────┐
│              BLAST RADIUS PROGRESSION                    │
├────────────────────────────────────────────────────────┤
│                                                         │
│  Level 1: Development     ████░░░░░░░░░░  (নিরাপদ)    │
│           └─ Local testing, mock failures               │
│                                                         │
│  Level 2: Staging         ████████░░░░░░  (মাঝারি)    │
│           └─ Full integration chaos                     │
│                                                         │
│  Level 3: Production      ████████████░░  (ঝুঁকিপূর্ণ)│
│           (Non-Critical)                                │
│           └─ Non-customer-facing services               │
│                                                         │
│  Level 4: Production      ██████████████  (সর্বোচ্চ)  │
│           (Critical Path)                               │
│           └─ Customer-facing, revenue path              │
│                                                         │
└────────────────────────────────────────────────────────┘
```

---

## 🧪 ব্যর্থতার প্রকারভেদ (Types of Failures to Inject)

```
┌─────────────────┬──────────────────────────────────────────┐
│ ব্যর্থতার ধরন   │ বিবরণ                                    │
├─────────────────┼──────────────────────────────────────────┤
│ Network Latency │ API call-এ ২-১০ সেকেন্ড delay যোগ করা   │
│ Service Crash   │ একটি মাইক্রোসার্ভিস হঠাৎ বন্ধ হওয়া      │
│ Disk Full       │ ডিস্ক স্পেস ১০০% ভর্তি করা              │
│ CPU Spike       │ CPU ব্যবহার ৯৫%+ করা                    │
│ DNS Failure     │ DNS resolution ব্যর্থ করা                │
│ Clock Skew      │ সিস্টেম ক্লক ৫ মিনিট এগিয়ে/পিছিয়ে দেওয়া│
│ Memory Leak     │ ধীরে ধীরে memory খরচ বাড়ানো             │
│ Packet Loss     │ ১০-৫০% নেটওয়ার্ক প্যাকেট হারানো         │
└─────────────────┴──────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Chaos Experiment Runner (PHP)

```php
<?php

namespace App\Chaos;

/**
 * Chaos Experiment Engine
 * bKash-এর মতো ফিনটেক সিস্টেমে chaos testing
 */
class ChaosExperimentRunner
{
    private array $experiments = [];
    private MetricsCollector $metrics;
    private AlertManager $alertManager;
    private float $blastRadiusLimit;

    public function __construct(
        MetricsCollector $metrics,
        AlertManager $alertManager,
        float $blastRadiusLimit = 0.05 // সর্বোচ্চ ৫% ট্র্যাফিক প্রভাবিত
    ) {
        $this->metrics = $metrics;
        $this->alertManager = $alertManager;
        $this->blastRadiusLimit = $blastRadiusLimit;
    }

    /**
     * Steady State Hypothesis সংজ্ঞায়িত করা
     */
    public function defineSteadyState(string $experimentName, array $hypothesis): self
    {
        $this->experiments[$experimentName] = [
            'hypothesis' => $hypothesis,
            'status' => 'defined',
            'startedAt' => null,
            'results' => [],
        ];
        return $this;
    }

    /**
     * Chaos Experiment চালানো
     */
    public function runExperiment(string $name, callable $faultInjector): ExperimentResult
    {
        $experiment = $this->experiments[$name] ?? null;
        if (!$experiment) {
            throw new \RuntimeException("Experiment '{$name}' পাওয়া যায়নি!");
        }

        // Step 1: Steady state measure করা (আগে)
        $beforeMetrics = $this->measureSteadyState($experiment['hypothesis']);

        // Step 2: Safety check — abort button প্রস্তুত
        $killSwitch = $this->prepareKillSwitch($name);

        // Step 3: Fault inject করা
        $this->experiments[$name]['status'] = 'running';
        $this->experiments[$name]['startedAt'] = microtime(true);

        try {
            $faultInjector();

            // Step 4: প্রভাব observe করা
            sleep(30); // ৩০ সেকেন্ড অপেক্ষা

            $afterMetrics = $this->measureSteadyState($experiment['hypothesis']);

            // Step 5: তুলনা করা
            $result = $this->compareMetrics($beforeMetrics, $afterMetrics, $experiment['hypothesis']);

            // Blast radius check
            if ($result->impactPercentage > $this->blastRadiusLimit * 100) {
                $killSwitch->activate();
                $this->alertManager->critical("Blast radius অতিক্রম! Experiment বন্ধ করা হচ্ছে।");
            }

            return $result;
        } catch (\Throwable $e) {
            $killSwitch->activate();
            $this->alertManager->critical("Chaos experiment ব্যর্থ: " . $e->getMessage());
            throw $e;
        } finally {
            $this->experiments[$name]['status'] = 'completed';
        }
    }

    private function measureSteadyState(array $hypothesis): array
    {
        $measurements = [];
        foreach ($hypothesis as $metric => $threshold) {
            $measurements[$metric] = $this->metrics->getCurrentValue($metric);
        }
        return $measurements;
    }

    private function compareMetrics(array $before, array $after, array $hypothesis): ExperimentResult
    {
        $passed = true;
        $details = [];

        foreach ($hypothesis as $metric => $threshold) {
            $deviation = abs($after[$metric] - $before[$metric]);
            $withinLimit = $deviation <= $threshold['maxDeviation'];

            if (!$withinLimit) {
                $passed = false;
            }

            $details[$metric] = [
                'before' => $before[$metric],
                'after' => $after[$metric],
                'deviation' => $deviation,
                'limit' => $threshold['maxDeviation'],
                'passed' => $withinLimit,
            ];
        }

        return new ExperimentResult($passed, $details);
    }

    private function prepareKillSwitch(string $experimentName): KillSwitch
    {
        return new KillSwitch($experimentName, $this->alertManager);
    }
}

/**
 * Network Latency Fault Injector
 * bKash → Bank Gateway-তে latency inject করা
 */
class NetworkLatencyInjector implements FaultInjector
{
    private string $targetService;
    private int $latencyMs;
    private float $percentage;

    public function __construct(string $targetService, int $latencyMs, float $percentage = 0.5)
    {
        $this->targetService = $targetService;
        $this->latencyMs = $latencyMs;
        $this->percentage = $percentage; // ৫০% request-এ latency
    }

    public function inject(): void
    {
        // iptables বা tc (traffic control) ব্যবহার করে latency inject
        $command = sprintf(
            'tc qdisc add dev eth0 root netem delay %dms %dms distribution normal',
            $this->latencyMs,
            $this->latencyMs / 4
        );

        echo "🔥 Injecting {$this->latencyMs}ms latency to {$this->targetService}\n";
        echo "📊 Affecting {$this->percentage * 100}% of traffic\n";

        // Production-এ সত্যিকারের command চালানো হবে
        // exec($command);
    }

    public function rollback(): void
    {
        $command = 'tc qdisc del dev eth0 root';
        echo "✅ Latency injection বন্ধ করা হলো\n";
        // exec($command);
    }
}

/**
 * Service Kill Experiment
 * Microservice হঠাৎ মারা গেলে কী হয় — টেস্ট
 */
class ServiceKillExperiment
{
    public function killRandomInstance(string $serviceName): void
    {
        $instances = $this->getRunningInstances($serviceName);

        if (count($instances) <= 1) {
            throw new \RuntimeException("শুধুমাত্র ১টি instance আছে! Kill করা নিরাপদ নয়।");
        }

        $victim = $instances[array_rand($instances)];
        echo "💀 Killing instance: {$victim['id']} of {$serviceName}\n";

        // Container kill
        // exec("docker kill {$victim['id']}");
    }

    private function getRunningInstances(string $serviceName): array
    {
        // Kubernetes বা Docker থেকে instance list
        return [
            ['id' => 'bkash-payment-abc123', 'status' => 'running'],
            ['id' => 'bkash-payment-def456', 'status' => 'running'],
            ['id' => 'bkash-payment-ghi789', 'status' => 'running'],
        ];
    }
}

// === ব্যবহার ===
$runner = new ChaosExperimentRunner(
    new MetricsCollector(),
    new AlertManager(),
    blastRadiusLimit: 0.05
);

// bKash Bank Gateway Latency Test
$runner->defineSteadyState('bank-gateway-latency', [
    'transaction_success_rate' => ['maxDeviation' => 2.0],   // ২% এর বেশি drop না
    'p99_latency_ms' => ['maxDeviation' => 500],              // ৫০০ms এর বেশি না বাড়ে
    'error_rate' => ['maxDeviation' => 1.0],                  // ১% এর বেশি error না বাড়ে
]);

$result = $runner->runExperiment('bank-gateway-latency', function () {
    $injector = new NetworkLatencyInjector('bank-gateway', 3000, 0.3);
    $injector->inject();
});

echo $result->passed ? "✅ সিস্টেম resilient!" : "❌ দুর্বলতা পাওয়া গেছে!";
```

---

## 🟨 JavaScript কোড উদাহরণ

### Chaos Toolkit Implementation (Node.js)

```javascript
/**
 * Chaos Engineering Framework
 * Grameenphone-এর মতো টেলকম সিস্টেমে chaos testing
 */

class ChaosEngine {
    constructor(config) {
        this.experiments = new Map();
        this.metrics = config.metricsCollector;
        this.alerting = config.alerting;
        this.maxBlastRadius = config.maxBlastRadius || 0.05;
        this.isRunning = false;
    }

    /**
     * Experiment define করা
     */
    defineExperiment(name, { steadyState, faultInjector, duration, rollback }) {
        this.experiments.set(name, {
            name,
            steadyState,
            faultInjector,
            duration: duration || 60000, // ডিফল্ট ১ মিনিট
            rollback,
            status: 'defined',
            history: [],
        });
        return this;
    }

    /**
     * Experiment চালানো
     */
    async runExperiment(name) {
        const experiment = this.experiments.get(name);
        if (!experiment) throw new Error(`Experiment "${name}" খুঁজে পাওয়া যায়নি`);

        console.log(`\n🔬 Chaos Experiment শুরু: "${name}"`);
        console.log(`⏱️  Duration: ${experiment.duration / 1000}s`);
        console.log(`📏 Max Blast Radius: ${this.maxBlastRadius * 100}%\n`);

        this.isRunning = true;
        const result = {
            name,
            startedAt: new Date(),
            metrics: { before: {}, after: {} },
            passed: false,
            findings: [],
        };

        try {
            // Step 1: Steady State পরিমাপ
            console.log('📊 Steady State পরিমাপ করা হচ্ছে...');
            result.metrics.before = await this.measureSteadyState(experiment.steadyState);
            console.log('   Before:', JSON.stringify(result.metrics.before));

            // Step 2: Fault Inject
            console.log('\n🔥 Fault Injection শুরু...');
            await experiment.faultInjector();

            // Step 3: Duration অপেক্ষা + continuous monitoring
            await this.monitorDuring(experiment, result);

            // Step 4: Steady State আবার পরিমাপ
            console.log('\n📊 Post-injection পরিমাপ...');
            result.metrics.after = await this.measureSteadyState(experiment.steadyState);
            console.log('   After:', JSON.stringify(result.metrics.after));

            // Step 5: তুলনা ও রায়
            result.passed = this.evaluateHypothesis(
                experiment.steadyState,
                result.metrics.before,
                result.metrics.after
            );

        } catch (error) {
            result.passed = false;
            result.findings.push(`Error: ${error.message}`);
            this.alerting.send('critical', `Chaos experiment "${name}" ব্যর্থ: ${error.message}`);
        } finally {
            // সবসময় rollback করা
            console.log('\n🔄 Rollback করা হচ্ছে...');
            await experiment.rollback();
            this.isRunning = false;
        }

        experiment.history.push(result);
        this.printResult(result);
        return result;
    }

    async monitorDuring(experiment, result) {
        const checkInterval = 5000; // প্রতি ৫ সেকেন্ডে check
        const checks = experiment.duration / checkInterval;

        for (let i = 0; i < checks; i++) {
            await this.sleep(checkInterval);

            const currentMetrics = await this.measureSteadyState(experiment.steadyState);
            const blastRadius = this.calculateBlastRadius(currentMetrics);

            if (blastRadius > this.maxBlastRadius) {
                console.log(`\n🚨 ABORT! Blast radius ${(blastRadius * 100).toFixed(1)}% > limit!`);
                await experiment.rollback();
                result.findings.push('Auto-aborted: blast radius exceeded');
                throw new Error('Blast radius limit অতিক্রম করেছে');
            }

            process.stdout.write(`  ⏳ ${i + 1}/${checks} checks OK (radius: ${(blastRadius * 100).toFixed(1)}%)\r`);
        }
        console.log('');
    }

    evaluateHypothesis(steadyState, before, after) {
        for (const [metric, config] of Object.entries(steadyState)) {
            const deviation = Math.abs(after[metric] - before[metric]);
            if (deviation > config.maxDeviation) {
                console.log(`  ❌ ${metric}: deviation ${deviation} > limit ${config.maxDeviation}`);
                return false;
            }
            console.log(`  ✅ ${metric}: deviation ${deviation} ≤ limit ${config.maxDeviation}`);
        }
        return true;
    }

    calculateBlastRadius(metrics) {
        // Error rate-কে blast radius হিসেবে ধরা
        return (metrics.errorRate || 0) / 100;
    }

    async measureSteadyState(hypothesis) {
        const results = {};
        for (const metric of Object.keys(hypothesis)) {
            results[metric] = await this.metrics.query(metric);
        }
        return results;
    }

    printResult(result) {
        console.log('\n' + '═'.repeat(50));
        console.log(result.passed
            ? '✅ EXPERIMENT PASSED — সিস্টেম resilient!'
            : '❌ EXPERIMENT FAILED — দুর্বলতা আবিষ্কৃত!'
        );
        console.log('═'.repeat(50));
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// === Fault Injectors ===

class NetworkFaultInjector {
    /**
     * Latency inject করা (proxy level)
     */
    static async injectLatency(targetUrl, latencyMs, percentage = 50) {
        console.log(`  🌐 ${targetUrl}-এ ${latencyMs}ms latency (${percentage}% traffic)`);
        // Envoy/Istio sidecar proxy-তে fault injection rule যোগ
        await fetch('http://localhost:15000/config', {
            method: 'POST',
            body: JSON.stringify({
                faults: [{
                    type: 'delay',
                    target: targetUrl,
                    delay: latencyMs,
                    percentage: percentage,
                }]
            })
        });
    }

    /**
     * Connection refuse করা
     */
    static async injectConnectionRefuse(targetUrl) {
        console.log(`  🚫 ${targetUrl}-এ connection refuse inject`);
        // iptables rule বা proxy config
    }
}

class ResourceFaultInjector {
    /**
     * CPU stress inject
     */
    static async injectCpuStress(percentage = 90, durationSec = 60) {
        console.log(`  💻 CPU stress: ${percentage}% for ${durationSec}s`);
        // stress-ng tool ব্যবহার করে CPU stress
    }

    /**
     * Memory pressure inject
     */
    static async injectMemoryPressure(megabytes = 512) {
        console.log(`  🧠 Memory allocation: ${megabytes}MB`);
        // Allocate memory to create pressure
        const buffers = [];
        const chunkSize = 1024 * 1024; // 1MB chunks
        for (let i = 0; i < megabytes; i++) {
            buffers.push(Buffer.alloc(chunkSize));
        }
        return () => buffers.length = 0; // cleanup function
    }
}

// === ব্যবহার: bKash ঈদের লোড টেস্ট ===

const chaosEngine = new ChaosEngine({
    metricsCollector: {
        async query(metric) {
            // Prometheus/Grafana থেকে metrics fetch
            const mockMetrics = {
                transactionSuccessRate: 99.5,
                p99Latency: 250,
                errorRate: 0.5,
                throughput: 15000,
            };
            return mockMetrics[metric] || 0;
        }
    },
    alerting: {
        send(level, msg) { console.log(`[${level.toUpperCase()}] ${msg}`); }
    },
    maxBlastRadius: 0.05,
});

// Experiment: Bank Gateway Unreachable during Eid
chaosEngine.defineExperiment('bkash-eid-bank-gateway-down', {
    steadyState: {
        transactionSuccessRate: { maxDeviation: 3 },   // ৩% পর্যন্ত গ্রহণযোগ্য
        p99Latency: { maxDeviation: 1000 },            // ১ সেকেন্ড পর্যন্ত OK
        errorRate: { maxDeviation: 2 },                // ২% পর্যন্ত error OK
    },
    faultInjector: async () => {
        await NetworkFaultInjector.injectConnectionRefuse('bank-gateway.internal:443');
    },
    duration: 120000, // ২ মিনিট
    rollback: async () => {
        console.log('  🔄 Bank gateway connection restore করা হচ্ছে...');
        // iptables rule remove / proxy config reset
    },
});

// Run!
chaosEngine.runExperiment('bkash-eid-bank-gateway-down')
    .then(result => {
        if (!result.passed) {
            console.log('\n📋 Action Items:');
            console.log('  1. Circuit breaker timeout কমানো');
            console.log('  2. Fallback bank gateway যোগ করা');
            console.log('  3. Queue-based retry mechanism implement করা');
        }
    });
```

---

## 📈 Chaos Maturity Model

```
┌─────────────────────────────────────────────────────────────┐
│                    CHAOS MATURITY MODEL                       │
├──────────┬──────────────────────────────────────────────────┤
│ Level 0  │ কোনো chaos practice নেই                          │
│ (শূন্য)  │ "সব ঠিক আছে, কিছু হবে না" মানসিকতা            │
├──────────┼──────────────────────────────────────────────────┤
│ Level 1  │ Manual Game Days                                  │
│ (শুরু)   │ বছরে ১-২ বার manual failure injection            │
│          │ ← বাংলাদেশের বেশিরভাগ কোম্পানি এখানে           │
├──────────┼──────────────────────────────────────────────────┤
│ Level 2  │ Automated Chaos in Staging                        │
│ (মধ্যম)  │ CI/CD pipeline-এ chaos test                      │
│          │ Staging environment-এ নিয়মিত experiment          │
├──────────┼──────────────────────────────────────────────────┤
│ Level 3  │ Automated Chaos in Production                     │
│ (উন্নত)  │ Production-এ controlled chaos                    │
│          │ Real-time monitoring + auto-abort                 │
├──────────┼──────────────────────────────────────────────────┤
│ Level 4  │ Continuous Chaos                                  │
│ (মাস্টার)│ সবসময় কিছু না কিছু inject হচ্ছে               │
│          │ Netflix, Google, Amazon এই level-এ               │
└──────────┴──────────────────────────────────────────────────┘
```

---

## 🛠️ Tools ও Platforms

| টুল | বিবরণ | সেরা ব্যবহার |
|------|--------|--------------|
| **Chaos Monkey** | Netflix-এর মূল টুল, random instance kill | Kubernetes/Cloud |
| **Gremlin** | Enterprise chaos platform, সহজ UI | বড় দল, compliance |
| **Litmus Chaos** | Kubernetes-native, open source | K8s environments |
| **Chaos Toolkit** | Python-based, extensible | Custom experiments |
| **AWS FIS** | AWS Fault Injection Simulator | AWS infrastructure |
| **Toxiproxy** | Network-level fault injection | Network chaos |

---

## 🎮 Game Days (পরিকল্পিত chaos দিবস)

```
┌─────────────────────────────────────────────────────┐
│               GAME DAY CHECKLIST                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  আগে (Before):                                     │
│  □ Experiment scope define করা                      │
│  □ Rollback plan প্রস্তুত                          │
│  □ On-call team alert করা                           │
│  □ Customer support team জানানো                     │
│  □ Monitoring dashboard প্রস্তুত                    │
│  □ Kill switch test করা                             │
│                                                      │
│  চলাকালীন (During):                                │
│  □ একটি experiment এ এক সময়ে focus                 │
│  □ সব metrics monitor করা                           │
│  □ যেকোনো unexpected behavior-এ abort              │
│  □ সব কিছু document করা                            │
│                                                      │
│  পরে (After):                                      │
│  □ Post-mortem meeting                              │
│  □ Findings document করা                           │
│  □ Fix prioritize করা                               │
│  □ পরবর্তী game day plan করা                       │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 🔗 Chaos + Observability Integration

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  Chaos   │────▶│  Prometheus  │────▶│  Grafana    │
│  Engine  │     │  (Metrics)   │     │  Dashboard  │
└──────────┘     └──────────────┘     └─────────────┘
     │                                       │
     │           ┌──────────────┐            │
     └──────────▶│    Jaeger    │◀───────────┘
                 │  (Tracing)   │
                 └──────────────┘
                        │
                 ┌──────────────┐
                 │   PagerDuty  │
                 │   (Alerting) │
                 └──────────────┘
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | করবেন? |
|-----------|--------|
| Mission-critical service (bKash payment) | ✅ অবশ্যই |
| High-availability requirement (99.99%) | ✅ অবশ্যই |
| Microservices architecture | ✅ হ্যাঁ |
| Cloud-native application | ✅ হ্যাঁ |
| Eid/Black Friday-র মতো peak traffic আশা করেন | ✅ হ্যাঁ |
| নতুন infrastructure migration | ✅ হ্যাঁ |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| মনিটরিং সিস্টেম নেই | ব্যর্থতা detect করতে পারবেন না |
| Single point of failure জানা আছে কিন্তু fix হয়নি | আগে fix করুন |
| টিমে incident response process নেই | আগে process তৈরি করুন |
| সদ্য launch হওয়া startup (MVP stage) | এখনও এত complexity নেই |
| Compliance/regulatory restrictions | আগে approval নিন |
| কোনো rollback mechanism নেই | বিপদ! আগে rollback তৈরি করুন |

---

## 🔄 Chaos in CI/CD Pipeline

```
┌────────┐   ┌────────┐   ┌─────────┐   ┌────────┐   ┌────────┐
│  Code  │──▶│  Unit  │──▶│  Chaos  │──▶│  Perf  │──▶│ Deploy │
│ Commit │   │  Test  │   │  Test   │   │  Test  │   │  Prod  │
└────────┘   └────────┘   └─────────┘   └────────┘   └────────┘
                                │
                        ┌───────┴───────┐
                        │  Inject:      │
                        │  - Latency    │
                        │  - Crash      │
                        │  - Error      │
                        │               │
                        │  Verify:      │
                        │  - Fallback   │
                        │  - Retry      │
                        │  - Circuit    │
                        │    Breaker    │
                        └───────────────┘
```

---

## 📝 সারসংক্ষেপ

> **Chaos Engineering = "আমরা ইচ্ছাকৃতভাবে সিস্টেম ভাঙি যাতে আসল সময়ে ভাঙলে আমরা প্রস্তুত থাকি।"**

মনে রাখুন:
1. ছোট থেকে শুরু করুন (blast radius ছোট রাখুন)
2. সবসময় rollback plan রাখুন
3. Observability আগে setup করুন
4. টিমকে জানান (surprise chaos নয়!)
5. প্রতিটি experiment থেকে শিখুন এবং document করুন
