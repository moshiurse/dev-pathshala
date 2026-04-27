# 🏗️ সিস্টেম ডিজাইনের মূল ধারণা

> **Advanced Guide** — সিনিয়র ইঞ্জিনিয়ারদের জন্য system design-এর গভীর আলোচনা।
> PHP (Laravel) ও JavaScript (Node.js/Express) উভয় stack-এ বাস্তব উদাহরণসহ।

---

## 📌 সংজ্ঞা — System Design কী এবং কেন গুরুত্বপূর্ণ?

System Design হলো একটি বৃহৎ-স্কেল সফটওয়্যার সিস্টেমের architecture, components, modules, interfaces এবং data flow নির্ধারণ করার প্রক্রিয়া। এটি শুধু code লেখা নয় — বরং **কোন component কোথায় বসবে, কীভাবে communicate করবে, কতটুকু load সামলাবে, failure-এ কী হবে** — এসব সিদ্ধান্ত নেওয়ার শিল্প।

সিনিয়র ইঞ্জিনিয়ার হিসেবে আপনাকে বুঝতে হবে:

- **Trade-offs**: প্রতিটি design decision-এ কিছু পাওয়া যায়, কিছু হারানো যায়
- **Scale thinking**: ১০০ user আর ১০ মিলিয়ন user-এর architecture আলাদা
- **Failure thinking**: সবকিছু fail করবে — প্রশ্ন হলো কখন এবং কীভাবে recover হবে
- **Cost awareness**: বাংলাদেশের context-এ infrastructure cost critical

```
┌─────────────────────────────────────────────────────┐
│              System Design Pillars                   │
│                                                      │
│   Scalability ─── Availability ─── Consistency       │
│       │               │               │              │
│   Reliability ─── Performance ─── Maintainability    │
│       │               │               │              │
│   Security ────── Cost ────────── Operability        │
└─────────────────────────────────────────────────────┘
```

---

## 📊 Core Concepts Deep Dive

---

### ১. Scalability (স্কেলেবিলিটি)

Scalability হলো একটি সিস্টেমের ক্রমবর্ধমান load বা demand সামলানোর ক্ষমতা। দুইটি মূল পদ্ধতি রয়েছে:

#### Vertical Scaling vs Horizontal Scaling

```
  Vertical Scaling (Scale Up)          Horizontal Scaling (Scale Out)
  ┌─────────────────────┐              ┌────────┐ ┌────────┐ ┌────────┐
  │                     │              │ Server │ │ Server │ │ Server │
  │   BIG SERVER        │              │   1    │ │   2    │ │   3    │
  │   (More CPU,        │              └───┬────┘ └───┬────┘ └───┬────┘
  │    More RAM,        │                  │          │          │
  │    More Disk)       │              ┌───┴──────────┴──────────┴───┐
  │                     │              │       Load Balancer          │
  └─────────────────────┘              └─────────────────────────────┘
  
  সুবিধা:                              সুবিধা:
  - সহজ implementation                 - প্রায় unlimited scaling
  - কোন code change লাগে না            - No single point of failure
                                        - Cost-effective (commodity hardware)
  অসুবিধা:                             
  - Hardware limit আছে                 অসুবিধা:
  - Single point of failure             - Complex architecture
  - Expensive at scale                  - Data consistency চ্যালেঞ্জ
```

#### Scaling Cube (AKF Scale Cube)

তিনটি axis-এ scaling চিন্তা করা যায়:

```
                    Y-axis
                    (Functional Decomposition)
                    │
                    │   ┌──────────┐
                    │   │ Service A│
                    │   ├──────────┤
                    │   │ Service B│
                    │   ├──────────┤
                    │   │ Service C│
                    │   └──────────┘
                    │
────────────────────┼────────────────────── X-axis
                    │                       (Horizontal Cloning)
                    │     ┌───┐ ┌───┐ ┌───┐
                    │     │ S │ │ S │ │ S │
                    │     └───┘ └───┘ └───┘
                   ╱
                 ╱   Z-axis (Data Partitioning)
               ╱     ┌────────┐ ┌────────┐ ┌────────┐
             ╱       │Users   │ │Users   │ │Users   │
                     │A - H   │ │I - P   │ │Q - Z   │
                     └────────┘ └────────┘ └────────┘
```

- **X-axis (Cloning)**: একই application-এর multiple copy চালানো — সবচেয়ে সহজ
- **Y-axis (Functional Decomposition)**: Microservices — ভিন্ন ভিন্ন function আলাদা service-এ
- **Z-axis (Data Partitioning)**: Data-র subset ভিন্ন ভিন্ন server-এ — sharding

#### PHP-FPM Scaling উদাহরণ

```php
// config/server.php — Laravel Octane (Swoole/RoadRunner)
// PHP-FPM-এ প্রতিটি request-এ নতুন process spawn হয়
// Octane-এ application boot একবার হয়, পরে request handle হয়

// php-fpm.conf — Horizontal scaling configuration
// pm = dynamic
// pm.max_children = 50
// pm.start_servers = 10
// pm.min_spare_servers = 5
// pm.max_spare_servers = 20

// Laravel Octane দিয়ে scaling
use Laravel\Octane\Facades\Octane;

// Concurrent task execution — CPU-bound কাজ parallel-এ
$results = Octane::concurrently([
    fn () => $this->fetchUserProfile($userId),
    fn () => $this->fetchUserOrders($userId),
    fn () => $this->fetchRecommendations($userId),
]);

// Laravel Horizon দিয়ে queue worker scaling
// config/horizon.php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
            'connection' => 'redis',
            'queue' => ['high', 'default', 'low'],
            'balance' => 'auto', // auto-scaling workers
        ],
    ],
],
```

#### Node.js Cluster Module উদাহরণ

```javascript
// cluster-server.js — Node.js Horizontal Scaling
const cluster = require('cluster');
const os = require('os');
const express = require('express');

if (cluster.isPrimary) {
    const numCPUs = os.cpus().length;
    console.log(`Primary ${process.pid}: ${numCPUs} টি worker spawn হচ্ছে`);

    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }

    // Worker crash হলে নতুন worker spawn
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died. Respawning...`);
        cluster.fork();
    });
} else {
    const app = express();

    app.get('/api/health', (req, res) => {
        res.json({
            worker: process.pid,
            uptime: process.uptime(),
            memory: process.memoryUsage(),
        });
    });

    app.listen(3000, () => {
        console.log(`Worker ${process.pid} started on port 3000`);
    });
}
```

#### কখন কোন Scaling ব্যবহার করবেন?

| পরিস্থিতি | Vertical | Horizontal |
|---|---|---|
| প্রাথমিক পর্যায়, ছোট টিম | ✅ | ❌ |
| Database bottleneck | ✅ (প্রথমে) | ✅ (পরে sharding) |
| Stateless web server | ❌ | ✅ |
| bKash-এর মতো high-traffic | ❌ | ✅ |
| Budget সীমিত (BD startup) | ✅ | ❌ |

---

### ২. Availability (অ্যাভেইলেবিলিটি)

Availability হলো সিস্টেম কতটুকু সময় সচল ও কার্যক্ষম থাকে তার পরিমাপ। বাংলাদেশের fintech (bKash, Nagad) বা e-commerce (Daraz)-এর জন্য high availability অত্যন্ত গুরুত্বপূর্ণ।

#### Availability Percentage ও Downtime

```
┌─────────────┬──────────────────┬───────────────────┬────────────────────┐
│ Availability│ Downtime/Year    │ Downtime/Month    │ Downtime/Day       │
├─────────────┼──────────────────┼───────────────────┼────────────────────┤
│ 99%         │ 3.65 days        │ 7.31 hours        │ 14.40 minutes      │
│ 99.9%       │ 8.77 hours       │ 43.83 minutes     │ 1.44 minutes       │
│ 99.95%      │ 4.38 hours       │ 21.92 minutes     │ 43.20 seconds      │
│ 99.99%      │ 52.60 minutes    │ 4.38 minutes      │ 8.64 seconds       │
│ 99.999%     │ 5.26 minutes     │ 26.30 seconds     │ 864 milliseconds   │
└─────────────┴──────────────────┴───────────────────┴────────────────────┘

bKash target: 99.99% (52 min downtime/year — financial transaction-এ এটাও অনেক)
Daraz target: 99.9% (8.77 hours/year — 11.11 sale-এ এটা critical)
```

#### Single Point of Failure (SPOF) চিহ্নিতকরণ

```
  ❌ SPOF Architecture                    ✅ No SPOF Architecture
  
  ┌────────┐                              ┌────────┐  ┌────────┐
  │ Client │                              │ Client │  │ Client │
  └───┬────┘                              └───┬────┘  └───┬────┘
      │                                       │          │
  ┌───┴────┐  ← SPOF!                   ┌────┴──────────┴────┐
  │   LB   │                            │  LB1 (Active)      │
  └───┬────┘                            │  LB2 (Standby)     │
      │                                  └────┬──────────┬────┘
  ┌───┴────┐  ← SPOF!                        │          │
  │  App   │                            ┌────┴───┐ ┌────┴───┐
  │ Server │                            │ App 1  │ │ App 2  │
  └───┬────┘                            └────┬───┘ └────┬───┘
      │                                      │          │
  ┌───┴────┐  ← SPOF!                  ┌────┴───┐ ┌────┴───┐
  │   DB   │                           │DB Prim │ │DB Repl │
  └────────┘                           └────────┘ └────────┘
```

#### Redundancy Patterns

**Active-Active**: দুটি server-ই একসাথে traffic handle করে। একটি fail হলে অন্যটি সব load নেয়।

**Active-Passive (Standby)**: একটি server active, অন্যটি standby-তে। Active fail হলে passive takeover করে।

```
  Active-Active                          Active-Passive
  ┌────────┐  ┌────────┐               ┌────────┐  ┌────────┐
  │ Node A │  │ Node B │               │ Active │  │Passive │
  │ (Live) │  │ (Live) │               │ (Live) │  │(Stdby) │
  │ 50%    │  │ 50%    │               │ 100%   │  │  0%    │
  └───┬────┘  └───┬────┘               └───┬────┘  └───┬────┘
      │           │                        │           │
      │    ◄──────┤                        │  Heartbeat│
      │  Sync     │                        │◄─────────►│
      ▼           ▼                        ▼           ▼
  ┌──────────────────┐               ┌──────────────────────┐
  │  Shared State    │               │ Failover: Passive    │
  │  (Redis/DB)      │               │ becomes Active       │
  └──────────────────┘               └──────────────────────┘
```

#### HA Architecture — সম্পূর্ণ চিত্র

```
                         ┌─────────────┐
                         │   DNS       │
                         │(Route 53)   │
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
              ┌─────┴─────┐          ┌─────┴─────┐
              │  LB - 1   │◄────────►│  LB - 2   │
              │ (Active)  │ VRRP/    │ (Standby) │
              └─────┬─────┘ Keepalive└───────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────┴───┐ ┌────┴───┐ ┌────┴───┐
   │ App-1  │ │ App-2  │ │ App-3  │
   └────┬───┘ └────┬───┘ └────┬───┘
        │          │          │
   ┌────┴──────────┴──────────┴────┐
   │         Redis Sentinel         │
   │    ┌─────┐┌─────┐┌─────┐     │
   │    │Mstr ││Repl ││Repl │     │
   │    └─────┘└─────┘└─────┘     │
   └───────────────────────────────┘
        │
   ┌────┴──────────────────────────┐
   │     MySQL/PostgreSQL          │
   │  ┌────────┐  ┌────────┐      │
   │  │Primary │─►│Replica │      │
   │  └────────┘  └────────┘      │
   └───────────────────────────────┘
```

```php
// Laravel — Health Check ও Failover Pattern
// app/Http/Controllers/HealthController.php

class HealthController extends Controller
{
    public function check(): JsonResponse
    {
        $checks = [
            'database' => $this->checkDatabase(),
            'redis'    => $this->checkRedis(),
            'queue'    => $this->checkQueue(),
            'disk'     => $this->checkDisk(),
        ];

        $healthy = collect($checks)->every(fn ($c) => $c['status'] === 'ok');

        return response()->json([
            'status' => $healthy ? 'healthy' : 'degraded',
            'checks' => $checks,
            'timestamp' => now()->toIso8601String(),
        ], $healthy ? 200 : 503);
    }

    private function checkDatabase(): array
    {
        try {
            DB::connection()->getPdo();
            return ['status' => 'ok', 'latency_ms' => $this->measureLatency(fn() => DB::select('SELECT 1'))];
        } catch (\Exception $e) {
            return ['status' => 'fail', 'error' => $e->getMessage()];
        }
    }
}
```

```javascript
// Node.js — Health Check ও Graceful Shutdown
const express = require('express');
const app = express();

let isShuttingDown = false;

app.get('/health', async (req, res) => {
    if (isShuttingDown) {
        return res.status(503).json({ status: 'shutting_down' });
    }

    const checks = {
        database: await checkDB(),
        redis: await checkRedis(),
        uptime: process.uptime(),
        memory: process.memoryUsage().heapUsed / 1024 / 1024,
    };

    const healthy = checks.database.ok && checks.redis.ok;
    res.status(healthy ? 200 : 503).json({ status: healthy ? 'healthy' : 'degraded', checks });
});

// Graceful shutdown — চলমান request শেষ হতে দিন
process.on('SIGTERM', () => {
    isShuttingDown = true;
    console.log('SIGTERM received. Graceful shutdown শুরু...');

    server.close(() => {
        console.log('সব connection বন্ধ। Process exit হচ্ছে।');
        process.exit(0);
    });

    // ৩০ সেকেন্ড পরেও বন্ধ না হলে force exit
    setTimeout(() => process.exit(1), 30000);
});
```

---

### ৩. Consistency Models

Distributed system-এ data consistency সবচেয়ে চ্যালেঞ্জিং সমস্যাগুলোর একটি। CAP theorem অনুসারে, নেটওয়ার্ক partition-এর সময় consistency ও availability-র মধ্যে বেছে নিতে হয়।

```
           CAP Theorem
     ┌─────────────────────┐
     │    Consistency (C)   │
     │     ╱          ╲     │
     │    ╱  CA        ╲    │
     │   ╱  (RDBMS)    CP  │
     │  ╱               ╲  │
     │ ╱                 ╲ │
     │╱        AP         ╲│
     │    (Cassandra,      │
     │     DynamoDB)       │
     │                     │
     │ Availability  Partition│
     │    (A)      Tolerance(P)│
     └─────────────────────┘
```

#### Strong Consistency
প্রতিটি read সবসময় সবচেয়ে সাম্প্রতিক write-এর ফলাফল দেখায়। **bKash-এর balance check**-এর মতো জায়গায় এটি mandatory — ভুল balance দেখানো unacceptable।

#### Eventual Consistency
সময়ের সাথে সব replica একই state-এ আসে, কিন্তু তাৎক্ষণিকভাবে নয়। **Daraz-এর product review count**-এ কিছুক্ষণ পুরানো count দেখালে সমস্যা নেই।

#### Causal Consistency
কারণ-ফলাফল সম্পর্কযুক্ত operation-গুলো সঠিক ক্রমে দেখায়। উদাহরণ: Facebook-এ comment-এর reply অবশ্যই comment-এর পরে দেখাবে।

#### Read-your-writes Consistency
আপনি যা write করেছেন তা আপনি নিজে সবসময় read করতে পারবেন। অন্যরা হয়তো দেরিতে দেখবে।

#### Linearizability
সবচেয়ে শক্তিশালী consistency guarantee। সব operation একটি single, real-time ordered sequence মেনে চলে। ব্যয়বহুল কিন্তু critical system-এ (banking, ticketing) প্রয়োজন।

#### কখন কোন Model ব্যবহার করবেন?

| Use Case | Model | কারণ |
|---|---|---|
| bKash balance | Strong/Linearizable | টাকার ব্যাপার — ভুল হলেই বিপদ |
| Daraz product listing | Eventual | কিছু সেকেন্ড পুরানো data acceptable |
| Chat message ordering | Causal | Reply অবশ্যই message-এর পরে |
| Social media profile update | Read-your-writes | নিজের update নিজে দেখা দরকার |
| Flight seat booking | Linearizable | Double booking prevent করতে হবে |

---

### ৪. Reliability (রিলায়েবিলিটি)

Reliability হলো সিস্টেম নির্দিষ্ট সময় ধরে সঠিকভাবে কাজ করার সম্ভাবনা।

#### MTBF ও MTTR

```
  ──────────────────────────────────────────── সময়রেখা ──►
  
  ┌──────────┐     ┌──┐     ┌──────────────┐     ┌──┐
  │ Working  │     │F │     │   Working    │     │F │
  │          │     │a │     │              │     │a │
  │          │     │i │     │              │     │i │
  │          │     │l │     │              │     │l │
  └──────────┘     └──┘     └──────────────┘     └──┘
  
  ◄── MTBF ──►     ◄►      ◄──── MTBF ───►     ◄►
  (Mean Time     MTTR       (Mean Time         MTTR
   Between      (Mean        Between           (Mean
   Failures)     Time        Failures)          Time
                 To                             To
                 Repair)                        Repair)
  
  Availability = MTBF / (MTBF + MTTR)
  
  উদাহরণ: MTBF = 720 hours, MTTR = 1 hour
  Availability = 720 / (720 + 1) = 99.86%
```

#### Circuit Breaker Pattern

যখন downstream service বারবার fail করে, তখন circuit breaker সেই service-এ call বন্ধ রাখে। এতে cascade failure আটকানো যায়।

```
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │  CLOSED  │──fail──►│   OPEN   │──timer─►│HALF-OPEN │
  │(Normal)  │ threshold│(No call) │ expires │(Test 1   │
  │          │◄─────── │          │         │ request)  │
  │          │ success  │          │◄─fail───│          │
  └──────────┘         └──────────┘         └──────────┘
       ▲                                         │
       └──────────── success ─────────────────────┘
```

```php
// Laravel — Circuit Breaker Implementation
class CircuitBreaker
{
    private const STATE_CLOSED = 'closed';
    private const STATE_OPEN = 'open';
    private const STATE_HALF_OPEN = 'half_open';

    public function __construct(
        private string $service,
        private int $failureThreshold = 5,
        private int $recoveryTimeout = 30,
        private \Illuminate\Contracts\Cache\Repository $cache,
    ) {}

    public function call(callable $action): mixed
    {
        $state = $this->getState();

        if ($state === self::STATE_OPEN) {
            if ($this->shouldAttemptReset()) {
                $this->setState(self::STATE_HALF_OPEN);
            } else {
                throw new CircuitOpenException("{$this->service} circuit is OPEN");
            }
        }

        try {
            $result = $action();
            $this->onSuccess();
            return $result;
        } catch (\Throwable $e) {
            $this->onFailure();
            throw $e;
        }
    }

    private function onFailure(): void
    {
        $failures = $this->cache->increment("cb:{$this->service}:failures");
        if ($failures >= $this->failureThreshold) {
            $this->setState(self::STATE_OPEN);
            $this->cache->put("cb:{$this->service}:opened_at", now()->timestamp, $this->recoveryTimeout * 2);
        }
    }

    private function onSuccess(): void
    {
        $this->cache->forget("cb:{$this->service}:failures");
        $this->setState(self::STATE_CLOSED);
    }

    private function getState(): string
    {
        return $this->cache->get("cb:{$this->service}:state", self::STATE_CLOSED);
    }

    private function setState(string $state): void
    {
        $this->cache->put("cb:{$this->service}:state", $state, $this->recoveryTimeout * 2);
    }

    private function shouldAttemptReset(): bool
    {
        $openedAt = $this->cache->get("cb:{$this->service}:opened_at", 0);
        return (now()->timestamp - $openedAt) >= $this->recoveryTimeout;
    }
}
```

```javascript
// Node.js — Circuit Breaker with opossum library pattern
class CircuitBreaker {
    constructor(service, options = {}) {
        this.service = service;
        this.failureThreshold = options.failureThreshold || 5;
        this.recoveryTimeout = options.recoveryTimeout || 30000;
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.lastFailureTime = null;
    }

    async call(action) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime >= this.recoveryTimeout) {
                this.state = 'HALF_OPEN';
            } else {
                throw new Error(`${this.service}: Circuit OPEN — fallback ব্যবহার করুন`);
            }
        }

        try {
            const result = await action();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    onSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }

    onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}

// ব্যবহার
const paymentCB = new CircuitBreaker('bkash-api', { failureThreshold: 3, recoveryTimeout: 60000 });

app.post('/api/payment', async (req, res) => {
    try {
        const result = await paymentCB.call(() => callBkashAPI(req.body));
        res.json(result);
    } catch (err) {
        // Graceful degradation — queue-তে রাখুন, পরে retry হবে
        await queue.add('payment-retry', req.body);
        res.status(202).json({ message: 'Payment queued — পরে প্রক্রিয়া হবে' });
    }
});
```

#### Graceful Degradation

সিস্টেমের কোনো অংশ fail করলে পুরো সিস্টেম বন্ধ না করে কম feature-এ চলতে থাকা। উদাহরণ: Daraz-এর recommendation engine down হলেও product browse চালু থাকে।

---

### ৫. Latency vs Throughput

**Latency**: একটি request-এর শুরু থেকে response পাওয়া পর্যন্ত সময়।
**Throughput**: নির্দিষ্ট সময়ে সিস্টেম কতগুলো request handle করতে পারে।

```
  Latency (একটি গাড়ির গতি)    Throughput (রাস্তায় মোট গাড়ির সংখ্যা)
  
  ──────►                       ──► ──► ──► ──► ──►
  1 request, 50ms               1000 requests/second
  
  কম latency = দ্রুত response    বেশি throughput = বেশি capacity
```

#### Percentile Latencies

গড় (average) latency বিভ্রান্তিকর হতে পারে। Percentile ব্যবহার করুন:

```
  P50  = 50th percentile (median) — অর্ধেক request এর নিচে
  P90  = 90th percentile          — ৯০% request এর নিচে
  P95  = 95th percentile          — ৯৫% request এর নিচে  
  P99  = 99th percentile          — ৯৯% request এর নিচে (tail latency)
  
  উদাহরণ (একটি বাংলাদেশি e-commerce API):
  ┌────────────────────────────────────────────────┐
  │  P50:  45ms   ████████░░░░░░░░░░░░ (ভালো)     │
  │  P90: 120ms   █████████████░░░░░░░ (acceptable)│
  │  P95: 250ms   █████████████████░░░ (সতর্কতা)   │
  │  P99: 800ms   ██████████████████░░ (সমস্যা!)    │
  │ P999: 2.5s    ████████████████████ (critical!)  │
  └────────────────────────────────────────────────┘
  
  বাস্তবতা: আপনার সবচেয়ে বড় customer-ই P99-তে পড়ে
  (কারণ তাদের data বেশি, query complex)
```

#### Latency Numbers Every Programmer Should Know

```
┌───────────────────────────────────────────────┬─────────────────┐
│ Operation                                     │ Time            │
├───────────────────────────────────────────────┼─────────────────┤
│ L1 cache reference                            │ 0.5 ns          │
│ Branch mispredict                             │ 5 ns            │
│ L2 cache reference                            │ 7 ns            │
│ Mutex lock/unlock                             │ 25 ns           │
│ Main memory reference (RAM)                   │ 100 ns          │
│ SSD random read                               │ 16 μs           │
│ Read 1 MB sequentially from memory            │ 250 μs          │
│ Round trip within same datacenter             │ 500 μs          │
│ Read 1 MB sequentially from SSD              │ 1 ms            │
│ HDD disk seek                                 │ 10 ms           │
│ Read 1 MB sequentially from HDD             │ 20 ms           │
│ Send packet Dhaka → Singapore (round trip)   │ 80 ms           │
│ Send packet Dhaka → US East (round trip)     │ 200 ms          │
│ Send packet Dhaka → EU (round trip)          │ 250 ms          │
└───────────────────────────────────────────────┴─────────────────┘

মূল শিক্ষা:
- Memory, SSD-র চেয়ে ১০০x+ দ্রুত — cache ব্যবহার করুন!
- Network call সবচেয়ে ধীর — minimize করুন
- BD থেকে Singapore datacenter ~80ms — regional CDN ব্যবহার করুন
```

---

### ৬. Consistency Patterns

#### Read-after-write Consistency

ব্যবহারকারী data write করার পরপরই সেটি read করতে পারবে। Implementation:

```php
// Laravel — Read-after-write: write-এর পর primary DB থেকে read
class ProfileService
{
    public function updateProfile(int $userId, array $data): UserProfile
    {
        // Primary DB-তে write
        $profile = UserProfile::on('mysql-write')->find($userId);
        $profile->update($data);

        // Cache invalidate করুন যেন stale data না পড়ে
        Cache::tags("user:{$userId}")->flush();

        // Primary DB থেকে read (replica-তে lag থাকতে পারে)
        return UserProfile::on('mysql-write')->find($userId);
    }
}
```

```javascript
// Node.js — Sticky session দিয়ে read-after-write guarantee
const { Pool } = require('pg');

const primaryPool = new Pool({ host: 'primary-db.internal' });
const replicaPool = new Pool({ host: 'replica-db.internal' });

class ConsistentReader {
    constructor() {
        this.recentWrites = new Map(); // userId → timestamp
    }

    async read(userId) {
        const lastWrite = this.recentWrites.get(userId);
        const replicaLag = 2000; // ২ সেকেন্ড assumed lag

        // সাম্প্রতিক write হলে primary থেকে read
        if (lastWrite && Date.now() - lastWrite < replicaLag) {
            return primaryPool.query('SELECT * FROM users WHERE id = $1', [userId]);
        }
        // পুরানো write হলে replica থেকে read (load কমানোর জন্য)
        return replicaPool.query('SELECT * FROM users WHERE id = $1', [userId]);
    }

    markWrite(userId) {
        this.recentWrites.set(userId, Date.now());
    }
}
```

#### Quorum Consensus (W + R > N)

Distributed system-এ consistency guarantee পেতে quorum ব্যবহার হয়:

```
  N = মোট replica সংখ্যা
  W = write-এ কতগুলো replica acknowledge করবে
  R = read-এ কতগুলো replica থেকে পড়বে
  
  শর্ত: W + R > N হলে strong consistency পাওয়া যায়
  
  উদাহরণ: N=3, W=2, R=2 (W+R=4 > 3 ✅)
  
  Write (W=2):                    Read (R=2):
  ┌─────────┐                    ┌─────────┐
  │Replica 1│ ✅ ACK             │Replica 1│ ✅ v2
  ├─────────┤                    ├─────────┤
  │Replica 2│ ✅ ACK             │Replica 2│ ✅ v2
  ├─────────┤                    ├─────────┤
  │Replica 3│ ⏳ (পরে sync হবে)  │Replica 3│ (পড়ার দরকার নেই)
  └─────────┘                    └─────────┘
  
  কমপক্ষে একটি replica-তে latest data থাকবেই!
```

#### Vector Clocks

Distributed system-এ event ordering track করার জন্য vector clock ব্যবহার হয়। প্রতিটি node নিজের logical clock maintain করে।

```
  Node A: [A:1, B:0, C:0] → [A:2, B:0, C:0] → [A:2, B:1, C:0]
  Node B: [A:0, B:1, C:0] → [A:1, B:2, C:0]
  Node C: [A:0, B:0, C:1] → [A:2, B:1, C:2]
  
  তুলনা: [A:2, B:1, C:0] vs [A:1, B:2, C:0]
  → Concurrent! কোনোটি আগে নয় → Conflict resolution দরকার
```

---

### ৭. Network Fundamentals

#### DNS Resolution Flow

```
  ব্যবহারকারী "daraz.com.bd" টাইপ করে
  
  ┌──────────┐     ┌─────────────┐
  │ Browser  │────►│ Local DNS   │ ← OS cache চেক
  │          │     │ Resolver    │
  └──────────┘     └──────┬──────┘
                          │ cache miss
                   ┌──────▼──────┐
                   │  Root DNS   │ ← "bd" এর NS কোথায়?
                   │  Server     │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │  .bd TLD    │ ← "com.bd" এর NS কোথায়?
                   │  Server     │
                   └──────┬──────┘
                          │
                   ┌──────▼──────────┐
                   │ daraz.com.bd    │ ← IP address: 103.x.x.x
                   │ Authoritative   │
                   │ DNS Server      │
                   └─────────────────┘
  
  সময়: সাধারণত 20-120ms (cached হলে <1ms)
```

#### HTTP/1.1 vs HTTP/2 vs HTTP/3

```
  HTTP/1.1                  HTTP/2                    HTTP/3
  ┌────────────────┐       ┌────────────────┐        ┌────────────────┐
  │ TCP Connection │       │ TCP Connection │        │ QUIC (UDP)     │
  │                │       │                │        │                │
  │ Req 1 ──────►  │       │ Req 1 ══╗      │        │ Req 1 ══╗      │
  │ Res 1 ◄──────  │       │ Req 2 ══╬═══►  │        │ Req 2 ══╬═══►  │
  │ Req 2 ──────►  │       │ Req 3 ══╝      │        │ Req 3 ══╝      │
  │ Res 2 ◄──────  │       │                │        │                │
  │ (Blocking!)    │       │ (Multiplexed)  │        │ (No HOL block) │
  │                │       │ (Header Comp)  │        │ (0-RTT)        │
  └────────────────┘       └────────────────┘        └────────────────┘
  
  - Head-of-line blocking  - Binary protocol         - UDP-based
  - Text-based             - Server push             - Connection migration
  - Multiple connections   - Single connection        - Built-in TLS 1.3
```

#### WebSocket vs SSE vs Long Polling

```
  Long Polling               SSE (Server-Sent Events)    WebSocket
  
  Client ──req──► Server     Client ──req──► Server      Client ◄══════► Server
  Client ◄─res─── Server     Client ◄─event─ Server      (Full Duplex)
  Client ──req──► Server     Client ◄─event─ Server      
  Client ◄─res─── Server     Client ◄─event─ Server      
  
  ব্যবহার:                   ব্যবহার:                    ব্যবহার:
  - Legacy support          - Live notifications        - Chat (WhatsApp-like)
  - Simple implementation   - Real-time dashboard       - Gaming
  - Daraz order tracking    - Stock price update        - Collaborative editing
                            - Unidirectional             - Bidirectional দরকার
```

#### CDN Architecture

```
                    ┌──────────────────┐
                    │   Origin Server  │
                    │   (Singapore)    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────┴───┐  ┌──────┴─────┐  ┌────┴────────┐
     │ CDN Edge   │  │ CDN Edge   │  │ CDN Edge    │
     │ (Dhaka)    │  │ (Mumbai)   │  │ (Singapore) │
     │ ~5ms       │  │ ~40ms      │  │ ~80ms       │
     └────────────┘  └────────────┘  └─────────────┘
              │              │              │
         BD Users       India Users    SG Users
  
  বাংলাদেশের জন্য CDN strategy:
  - Static assets → Cloudflare/BunnyCDN (Dhaka PoP আছে)
  - Dynamic content → Singapore origin (সবচেয়ে কাছের AWS region)
  - Image optimization → CDN-level WebP conversion
```

---

### ৮. Storage Types

#### Block vs File vs Object Storage

```
  Block Storage              File Storage              Object Storage
  ┌───┬───┬───┬───┐         ┌──────────────┐          ┌──────────────────┐
  │ B │ B │ B │ B │         │ /home/       │          │  ┌──────┐        │
  │ 1 │ 2 │ 3 │ 4 │         │ ├── docs/    │          │  │Object│+Metadata│
  ├───┼───┼───┼───┤         │ │   ├── a.pdf│          │  │(Blob)│ key:val │
  │ B │ B │ B │ B │         │ │   └── b.doc│          │  └──────┘        │
  │ 5 │ 6 │ 7 │ 8 │         │ └── pics/    │          │  Flat namespace  │
  └───┴───┴───┴───┘         └──────────────┘          │  HTTP API access │
                                                       └──────────────────┘
  ব্যবহার:                   ব্যবহার:                   ব্যবহার:
  - Database storage        - NAS, shared files       - S3, images, backups
  - High IOPS দরকার         - Hierarchical data       - CDN origin
  - EC2 EBS                 - EFS                     - Unlimited scale
```

#### RAID Levels

```
  RAID 0 (Striping)    RAID 1 (Mirroring)    RAID 5 (Strip+Parity)  RAID 10
  ┌────┐ ┌────┐       ┌────┐ ┌────┐         ┌────┐┌────┐┌────┐    ┌────┐┌────┐
  │ A1 │ │ A2 │       │ A  │ │ A  │         │ A1 ││ A2 ││ Ap │    │A1  ││A1' │
  │ B1 │ │ B2 │       │ B  │ │ B  │         │ B1 ││ Bp ││ B2 │    │B1  ││B1' │
  └────┘ └────┘       └────┘ └────┘         └────┘└────┘└────┘    │A2  ││A2' │
  দ্রুত, unsafe        নিরাপদ, ব্যয়বহুল       ব্যালেন্সড            └────┘└────┘
  (কখনো production-এ নয়)                                          সেরা performance
                                                                    + redundancy
```

#### SSD vs HDD Performance

```
┌─────────────────────┬──────────────────┬───────────────────┐
│ Metric              │ SSD (NVMe)       │ HDD (7200 RPM)    │
├─────────────────────┼──────────────────┼───────────────────┤
│ Random Read IOPS    │ 500,000+         │ ~150              │
│ Random Write IOPS   │ 200,000+         │ ~150              │
│ Sequential Read     │ 3,500 MB/s       │ 150 MB/s          │
│ Sequential Write    │ 3,000 MB/s       │ 150 MB/s          │
│ Latency             │ 0.02-0.1 ms      │ 5-10 ms           │
│ Cost per GB         │ $0.08-0.15       │ $0.02-0.04        │
│ Durability          │ TBW limited      │ Mechanical failure │
└─────────────────────┴──────────────────┴───────────────────┘

বাছাই:
- Database primary → SSD (IOPS critical)
- Archive/Backup → HDD (cost critical)
- Log storage → HDD (sequential write, cost sensitive)
- Cache/Temp → SSD (latency critical)
```

---

### ৯. Estimation & Back-of-envelope Calculations

সিনিয়র ইঞ্জিনিয়ার হিসেবে quick estimation করার দক্ষতা অপরিহার্য। Interview এবং real architecture decision উভয়ের জন্যই এটি কাজে লাগে।

#### Traffic Estimation (DAU → QPS)

```
  সূত্র:
  QPS = DAU × (average queries per user per day) / 86400
  Peak QPS ≈ QPS × 2~5 (depending on traffic pattern)
  
  উদাহরণ: বাংলাদেশি social media app
  - DAU = 10 Million (১ কোটি)
  - প্রতি user গড়ে 20 টি API call/day
  
  QPS = 10,000,000 × 20 / 86,400 ≈ 2,315 QPS
  Peak QPS ≈ 2,315 × 3 ≈ ~7,000 QPS
  
  (রাত ১০-১১টায় বাংলাদেশে peak traffic)
```

#### Storage Estimation

```
  উদাহরণ: BD Social Media App (10M users, 5 years)
  
  Daily new posts = 10M × 10% active × 2 posts = 2M posts/day
  
  Text post:
  - Average size: 500 bytes
  - Daily: 2M × 500 B = 1 GB/day
  - 5 years: 1 GB × 365 × 5 ≈ 1.8 TB
  
  Image (with post):
  - 30% posts have image, avg 200 KB
  - Daily: 600K × 200 KB = 120 GB/day
  - 5 years: 120 GB × 365 × 5 ≈ 219 TB
  
  ┌─────────────────────────────────────┐
  │ Total Storage (5 years):            │
  │ Text:   ~2 TB                       │
  │ Images: ~220 TB                     │
  │ Metadata/Index: ~5 TB              │
  │ Replication (3x): ~680 TB          │
  │ ─────────────────────               │
  │ মোট: ~700 TB (with replication)    │
  └─────────────────────────────────────┘
```

#### Bandwidth Estimation

```
  Incoming (Write):
  - Text: 1 GB/day = ~12 MB/s (peak: ~36 MB/s)
  - Images: 120 GB/day = ~1.4 GB/s (peak: ~4.2 GB/s — CDN offload দরকার)
  
  Outgoing (Read — assuming 10:1 read:write ratio):
  - Text: ~120 MB/s peak
  - Images: ~14 GB/s peak (CDN ছাড়া অসম্ভব)
  
  বাংলাদেশের internet context:
  - Average mobile speed: ~30 Mbps (4G)
  - Image optimization mandatory (WebP, lazy loading)
  - CDN with Dhaka PoP essential
```

#### Memory Estimation for Caching

```
  Rule of thumb: Cache top 20% data (80/20 rule)
  
  Daily unique reads = 20M (10M DAU × 2 avg reads)
  
  Cache 20% = 4M entries
  Average cache entry: 1 KB (post metadata)
  
  Memory needed: 4M × 1 KB = 4 GB
  
  With overhead (hash table, pointers, etc.): ~6 GB
  
  Redis instance: 1x r6g.xlarge (13 GB) — যথেষ্ট
  Cost: ~$200/month (Singapore region)
```

#### সম্পূর্ণ উদাহরণ: বাংলাদেশি Social Media App (10M Users)

```
┌────────────────────────────────────────────────────────────────┐
│                    Estimation Summary                          │
├───────────────────────────┬────────────────────────────────────┤
│ DAU                       │ 10 Million                         │
│ QPS (average)             │ ~2,300                             │
│ QPS (peak)                │ ~7,000                             │
│ Storage (5 years)         │ ~700 TB (with replication)         │
│ Bandwidth (peak outgoing) │ ~15 GB/s (CDN offloads 90%+)      │
│ Cache memory              │ ~6 GB (Redis)                      │
│ App servers needed        │ 15-20 (at 500 QPS each)           │
│ DB servers                │ 1 Primary + 3 Read Replicas       │
│ Monthly infra cost (est.) │ $8,000-15,000 (AWS Singapore)     │
└───────────────────────────┴────────────────────────────────────┘

bKash comparison:
- Daily transactions: ~10M+
- Peak TPS: ~5,000+
- Requires: Strong consistency, <200ms latency, 99.99% availability
- Estimated infra: $50,000-100,000/month
```

---

### ১০. System Design Interview Framework

#### RESHADED Method

```
  R ─ Requirements     (Functional + Non-functional চিহ্নিত করুন)
  E ─ Estimation       (Traffic, Storage, Bandwidth হিসাব করুন)
  S ─ Storage Schema   (Database schema, data model ঠিক করুন)
  H ─ High-level Design(Component diagram আঁকুন)
  A ─ API Design       (Endpoints, request/response define করুন)
  D ─ Detailed Design  (Deep dive — সবচেয়ে complex অংশ)
  E ─ Evaluate         (Bottlenecks চিহ্নিত করুন, trade-offs আলোচনা করুন)
  D ─ Deploy           (Monitoring, scaling, deployment strategy)
```

#### ধাপে ধাপে পদ্ধতি

**ধাপ ১ — Requirements (৩-৫ মিনিট)**

প্রশ্ন করুন! Interviewer চায় আপনি clarify করুন।

```
Functional Requirements:
- কোন কোন feature আছে?
- কে ব্যবহার করবে?
- Core use case কী?

Non-functional Requirements:
- কত user?
- Latency requirement কত?
- Availability কত দরকার?
- Consistency কতটুকু important?
```

**ধাপ ২ — Estimation (২-৩ মিনিট)**

Back-of-envelope calculation করুন (Section ৯ দেখুন)।

**ধাপ ৩ — High-level Design (১০-১৫ মিনিট)**

মূল component গুলো আঁকুন এবং data flow দেখান। এই অংশে সবচেয়ে বেশি সময় দিন।

**ধাপ ৪ — Deep Dive (১০-১৫ মিনিট)**

Interviewer যে অংশে আগ্রহী সেখানে গভীরে যান। সাধারণত: database schema, caching strategy, বা scaling bottleneck।

#### Interview-এ সাধারণ ভুলগুলো

```
❌ ভুল                                    ✅ সঠিক
─────────────────────────────────────────────────────────────────
Requirement না জেনে design শুরু          → প্রথমে ৫+ প্রশ্ন করুন
"Let's use microservices"               → Monolith দিয়ে শুরু, প্রয়োজনে split
সব জায়গায় NoSQL                         → Use case অনুযায়ী DB বাছাই
Single server design                    → প্রথম থেকে distributed চিন্তা
শুধু happy path                          → Failure scenario আলোচনা করুন
Over-engineering                        → Simple থেকে শুরু, প্রয়োজনে complex
Estimation skip করা                     → Numbers দিয়ে decision justify করুন
```

---

## 🔥 Complete Architecture Example — E-commerce System (Daraz-scale)

একটি বাংলাদেশি e-commerce platform-এর সম্পূর্ণ architecture যেখানে PHP (Laravel) backend ও Node.js real-time service একসাথে কাজ করে:

```
                              ┌───────────────┐
                              │   Cloudflare   │
                              │   CDN + WAF    │
                              │  (Dhaka PoP)   │
                              └───────┬───────┘
                                      │
                              ┌───────▼───────┐
                              │   DNS-based    │
                              │Load Balancer   │
                              │  (Route 53)    │
                              └───────┬───────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
             ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
             │  Nginx/ALB  │  │  Nginx/ALB  │  │  Nginx/ALB  │
             │   (L7 LB)   │  │   (L7 LB)   │  │  (WebSocket)│
             └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
                    │                │                 │
         ┌──────────┘    ┌───────────┘                │
         ▼               ▼                            ▼
  ┌─────────────┐ ┌─────────────┐              ┌─────────────┐
  │   Laravel   │ │   Laravel   │              │  Node.js    │
  │   App (x6)  │ │   App (x6)  │              │ Real-time   │
  │             │ │             │              │ (Socket.io) │
  │ - Product   │ │ - Order     │              │ (x3)        │
  │ - User      │ │ - Payment   │              │             │
  │ - Cart      │ │ - Inventory │              │ - Notif     │
  └──────┬──────┘ └──────┬──────┘              │ - Chat      │
         │               │                     │ - Tracking  │
         │               │                     └──────┬──────┘
         ▼               ▼                            │
  ┌─────────────────────────────────┐                │
  │         Redis Cluster           │◄───────────────┘
  │  ┌────────┐ ┌────────┐         │
  │  │Session │ │ Cache  │         │
  │  │Store   │ │(Product│         │
  │  │        │ │ data)  │         │
  │  └────────┘ └────────┘         │
  └───────────────┬─────────────────┘
                  │
  ┌───────────────▼─────────────────┐
  │      Message Queue (RabbitMQ)   │
  │  ┌─────────┐ ┌─────────┐       │
  │  │ Orders  │ │ Emails  │       │
  │  │ Queue   │ │ Queue   │       │
  │  └─────────┘ └─────────┘       │
  └───────────────┬─────────────────┘
                  │
  ┌───────────────▼─────────────────────────────────────────┐
  │                    Database Layer                        │
  │                                                         │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
  │  │   MySQL      │  │ Elasticsearch│  │  MongoDB      │  │
  │  │  (Primary)   │  │  (Search)    │  │ (Product     │  │
  │  │   │          │  │              │  │  Catalog)    │  │
  │  │   ├─Replica1 │  │  3-node      │  │              │  │
  │  │   ├─Replica2 │  │  cluster     │  │  Replica Set │  │
  │  │   └─Replica3 │  │              │  │              │  │
  │  └──────────────┘  └──────────────┘  └──────────────┘  │
  └─────────────────────────────────────────────────────────┘
                  │
  ┌───────────────▼─────────────────┐
  │        S3 (Object Storage)      │
  │   Product images, invoices,     │
  │   user uploads                  │
  └─────────────────────────────────┘
```

#### Tech Stack Mapping

```php
// Laravel (PHP) — Main Application Backend
// routes/api.php

Route::middleware(['auth:sanctum', 'throttle:60,1'])->group(function () {
    // Product Service
    Route::get('/products', [ProductController::class, 'index']);       // Elasticsearch query
    Route::get('/products/{id}', [ProductController::class, 'show']);   // Redis cache → MySQL fallback

    // Order Service
    Route::post('/orders', [OrderController::class, 'store']);          // MySQL write + Queue dispatch
    Route::get('/orders/{id}', [OrderController::class, 'show']);

    // Payment Integration (bKash, Nagad, SSLCommerz)
    Route::post('/payments/initiate', [PaymentController::class, 'initiate']);
    Route::post('/payments/webhook', [PaymentController::class, 'handleWebhook']);
});

// Order Creation — Queue-based processing
class OrderController extends Controller
{
    public function store(OrderRequest $request): JsonResponse
    {
        $order = DB::transaction(function () use ($request) {
            // Inventory lock (SELECT FOR UPDATE)
            $items = Product::whereIn('id', $request->product_ids)
                ->lockForUpdate()
                ->get();

            // Stock check
            foreach ($items as $item) {
                if ($item->stock < $request->quantities[$item->id]) {
                    throw new InsufficientStockException($item->name);
                }
            }

            // Order create
            $order = Order::create([
                'user_id' => auth()->id(),
                'total' => $this->calculateTotal($items, $request->quantities),
                'status' => 'pending',
            ]);

            // Stock decrement
            foreach ($items as $item) {
                $item->decrement('stock', $request->quantities[$item->id]);
            }

            return $order;
        });

        // Async jobs dispatch
        ProcessPayment::dispatch($order);
        SendOrderConfirmation::dispatch($order);
        NotifyWarehouse::dispatch($order);

        return response()->json($order, 201);
    }
}
```

```javascript
// Node.js — Real-time Service (Notifications, Chat, Order Tracking)
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const Redis = require('ioredis');

const app = express();
const httpServer = createServer(app);

// Socket.io with Redis adapter — multiple Node.js instance-এ sync
const io = new Server(httpServer, {
    cors: { origin: process.env.FRONTEND_URL },
    adapter: require('@socket.io/redis-adapter').createAdapter(
        new Redis(process.env.REDIS_URL),
        new Redis(process.env.REDIS_URL)
    ),
});

// Authentication middleware
io.use(async (socket, next) => {
    const token = socket.handshake.auth.token;
    try {
        const user = await verifyJWT(token);
        socket.userId = user.id;
        next();
    } catch (err) {
        next(new Error('Authentication failed'));
    }
});

io.on('connection', (socket) => {
    console.log(`User ${socket.userId} connected`);

    // ব্যবহারকারীকে তার নিজের room-এ join করান
    socket.join(`user:${socket.userId}`);

    // Order tracking — real-time status update
    socket.on('track:order', (orderId) => {
        socket.join(`order:${orderId}`);
    });
});

// Laravel থেকে Redis pub/sub দিয়ে event আসবে
const subscriber = new Redis(process.env.REDIS_URL);
subscriber.subscribe('order-updates', 'notifications');

subscriber.on('message', (channel, message) => {
    const data = JSON.parse(message);

    if (channel === 'order-updates') {
        // Order status update সংশ্লিষ্ট user ও order room-এ broadcast
        io.to(`order:${data.orderId}`).emit('order:status', {
            orderId: data.orderId,
            status: data.status,      // 'confirmed', 'shipped', 'delivered'
            updatedAt: data.timestamp,
        });
    }

    if (channel === 'notifications') {
        io.to(`user:${data.userId}`).emit('notification', {
            title: data.title,
            body: data.body,
            type: data.type,
        });
    }
});

httpServer.listen(3001, () => console.log('Real-time service on :3001'));
```

---

## ✅ System Design Checklist

প্রতিটি system design করার সময় নিচের checklist অনুসরণ করুন:

```
┌─────────────────────────────────────────────────────────────────┐
│                    System Design Checklist                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ □ Requirements                                                  │
│   □ Functional requirements (features) সুস্পষ্ট?                │
│   □ Non-functional requirements (scale, latency) নির্ধারিত?     │
│   □ Out of scope কী কী — সেটা পরিষ্কার?                         │
│                                                                 │
│ □ Estimation                                                    │
│   □ DAU / QPS হিসাব হয়েছে?                                     │
│   □ Storage requirement (1yr, 5yr) বের হয়েছে?                  │
│   □ Bandwidth estimate করা হয়েছে?                               │
│   □ Cache size determine করা হয়েছে?                             │
│                                                                 │
│ □ Data Layer                                                    │
│   □ SQL vs NoSQL — justified decision?                          │
│   □ Schema design complete?                                     │
│   □ Indexing strategy planned?                                  │
│   □ Sharding strategy (যদি দরকার)?                              │
│   □ Replication topology নির্ধারিত?                              │
│                                                                 │
│ □ Caching                                                       │
│   □ Cache strategy (write-through, write-back, write-around)?   │
│   □ Cache invalidation plan?                                    │
│   □ Cache stampede prevention?                                  │
│   □ TTL policy ঠিক করা হয়েছে?                                  │
│                                                                 │
│ □ Scalability                                                   │
│   □ Stateless application servers?                              │
│   □ Horizontal scaling plan?                                    │
│   □ Database scaling strategy?                                  │
│   □ Auto-scaling rules define হয়েছে?                            │
│                                                                 │
│ □ Reliability                                                   │
│   □ SPOF চিহ্নিত ও সমাধান হয়েছে?                               │
│   □ Failover mechanism আছে?                                     │
│   □ Data backup ও recovery plan আছে?                            │
│   □ Circuit breaker implement হয়েছে?                            │
│                                                                 │
│ □ Monitoring & Operations                                       │
│   □ Health check endpoints আছে?                                 │
│   □ Logging strategy (structured logs)?                         │
│   □ Alerting thresholds set করা হয়েছে?                          │
│   □ Runbook তৈরি হয়েছে?                                        │
│                                                                 │
│ □ Security                                                      │
│   □ Authentication ও Authorization?                              │
│   □ Rate limiting আছে?                                          │
│   □ Input validation ও sanitization?                            │
│   □ Encryption (at rest + in transit)?                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 সারসংক্ষেপ

### মূল পয়েন্টগুলো

১. **Scalability** — Vertical দিয়ে শুরু করুন, তারপর horizontal-এ যান। AKF cube-এর তিন axis মনে রাখুন।

২. **Availability** — প্রতিটি layer-এ redundancy রাখুন। বাংলাদেশের fintech-এ 99.99% target করুন।

৩. **Consistency** — সব জায়গায় strong consistency লাগে না। Use case অনুযায়ী বাছাই করুন — bKash-এ strong, Daraz feed-এ eventual।

৪. **Reliability** — Circuit breaker, graceful degradation, এবং proper health checks implement করুন। MTBF বাড়ান, MTTR কমান।

৫. **Latency** — P99 track করুন, average নয়। Bangladesh-এর internet context-এ CDN ও regional caching অপরিহার্য।

৬. **Estimation** — সব decision numbers দিয়ে justify করুন। Quick mental math আয়ত্ত করুন।

৭. **Interview** — RESHADED method follow করুন। প্রথমে requirements, তারপর estimation, তারপর design।

### বাংলাদেশ Context-এ বিশেষ বিবেচনা

```
┌─────────────────────────────────────────────────────────────────┐
│  🇧🇩 Bangladesh-specific System Design Considerations           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Infrastructure:                                                │
│  - Nearest AWS region: Singapore (ap-southeast-1) ~80ms        │
│  - Nearest GCP: Singapore/Mumbai                               │
│  - Local hosting: limited, ISP-dependent                       │
│  - Submarine cable: SEA-ME-WE 4/5 (single point of failure!)  │
│                                                                 │
│  Internet Characteristics:                                      │
│  - Mobile-first (95%+ traffic from mobile)                     │
│  - Average speed: ~30 Mbps (4G), ~5 Mbps (3G rural)           │
│  - High packet loss during peak hours                          │
│  - Power outage consideration (mobile app offline support)     │
│                                                                 │
│  Scale References:                                              │
│  - bKash: ~10M daily transactions, ~60M registered users       │
│  - Daraz: ~5M products, 11.11 sale peak ~100K concurrent       │
│  - Pathao: ~1M daily rides, real-time location tracking        │
│  - BD population: 170M+, internet users: ~130M+               │
│                                                                 │
│  Design Implications:                                           │
│  - Image compression mandatory (WebP, AVIF)                    │
│  - API response size minimize করুন (pagination, field select)  │
│  - Offline-first mobile architecture বিবেচনা করুন              │
│  - CDN with Dhaka PoP (Cloudflare, Akamai আছে)               │
│  - Payment gateway: bKash, Nagad, SSLCommerz integration       │
│  - Bangla text search: Elasticsearch with ICU analyzer         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **মনে রাখুন**: কোনো একটি "সেরা" architecture নেই। প্রতিটি সিদ্ধান্ত trade-off। আপনার কাজ হলো সঠিক trade-off বেছে নেওয়া — এবং সেটা justify করতে পারা। এটাই একজন সিনিয়র ইঞ্জিনিয়ারকে জুনিয়র থেকে আলাদা করে।

---

*এই গাইডটি বাংলাদেশি সিনিয়র সফটওয়্যার ইঞ্জিনিয়ারদের জন্য তৈরি। System design interview preparation ও production architecture planning — উভয় ক্ষেত্রে কাজে আসবে।*
