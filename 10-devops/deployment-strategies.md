# 📘 Deployment Strategies

> **Deployment Strategies — প্রোডাকশনে নতুন সফটওয়্যার ভার্সন নিরাপদে ও কম ঝুঁকিতে মোতায়েন করার বিভিন্ন কৌশল।**

---

## 📖 সংজ্ঞা ও মূল ধারণা

Deployment Strategy হলো **নতুন কোড বা সফটওয়্যার ভার্সন প্রোডাকশন পরিবেশে মোতায়েন করার পদ্ধতি**। সঠিক strategy বেছে নেওয়া নির্ভর করে — **ঝুঁকি সহনশীলতা, ডাউনটাইম গ্রহণযোগ্যতা, রিসোর্স বাজেট** এবং **রোলব্যাক প্রয়োজনীয়তার** উপর।

### কৌশলগুলোর সারসংক্ষেপ

| কৌশল | ডাউনটাইম | ঝুঁকি | রিসোর্স খরচ | রোলব্যাক |
|------|----------|-------|-------------|----------|
| **Recreate** | ✅ আছে | 🔴 বেশি | কম | ধীর |
| **Rolling Update** | ❌ নেই | 🟡 মাঝারি | মাঝারি | মাঝারি |
| **Blue/Green** | ❌ নেই | 🟢 কম | 🔴 দ্বিগুণ | তাৎক্ষণিক |
| **Canary** | ❌ নেই | 🟢 কম | মাঝারি | তাৎক্ষণিক |
| **A/B Testing** | ❌ নেই | 🟢 কম | মাঝারি | তাৎক্ষণিক |

---

## 🏠 বাস্তব জীবনের উদাহরণ

### 🏗️ রেস্তোরাঁ সংস্কার

একটি জনপ্রিয় রেস্তোরাঁ চেইন তাদের মেনু ও ডেকোরেশন বদলাতে চায়:

- **Recreate** = পুরো রেস্তোরাঁ বন্ধ করে সংস্কার করা → গ্রাহক হারানোর ঝুঁকি
- **Rolling Update** = একটি একটি করে টেবিল বদলানো → গ্রাহক সেবা চালু থাকে, কিন্তু পুরানো ও নতুন মিশ্রিত থাকে
- **Blue/Green** = পাশের বিল্ডিংয়ে নতুন রেস্তোরাঁ তৈরি করে একদিন সবাইকে সেখানে নিয়ে যাওয়া → কোনো সমস্যা হলে আগের জায়গায় ফেরা সহজ
- **Canary** = শুধু ১টি টেবিলে নতুন মেনু দিয়ে দেখা — গ্রাহকরা পছন্দ করলে সব টেবিলে চালু করা
- **A/B Testing** = অর্ধেক গ্রাহকদের নতুন মেনু, অর্ধেককে পুরানো — কোনটা বেশি বিক্রি হয় তুলনা করা

---

## 1️⃣ Recreate Deployment

পুরানো ভার্সনের সকল instance বন্ধ করে নতুন ভার্সন চালু করা।

```
Recreate Deployment:

সময় ──────────────────────────────────────────►

v1 ████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░
                   ↑ ডাউনটাইম ↓
v2 ░░░░░░░░░░░░░░░░░░░░░░████████████████████

                   │←── Downtime ──►│
```

---

## 2️⃣ Rolling Update

ধাপে ধাপে পুরানো instance বন্ধ করে নতুন instance চালু করা। Kubernetes-এর **default strategy**।

```
Rolling Update:

সময় ──────────────────────────────────────────►

Pod 1 (v1) ██████████████░░░░░░░░░░░░░░░░░░░░░
Pod 1 (v2) ░░░░░░░░░░░░░░████████████████████

Pod 2 (v1) ████████████████████░░░░░░░░░░░░░░░
Pod 2 (v2) ░░░░░░░░░░░░░░░░░░░░██████████████

Pod 3 (v1) ██████████████████████████░░░░░░░░░
Pod 3 (v2) ░░░░░░░░░░░░░░░░░░░░░░░░░░████████

✅ কোনো ডাউনটাইম নেই — সবসময় কিছু pod চালু থাকে
⚠️ সাময়িকভাবে v1 ও v2 একসাথে চলে (compatibility দরকার)
```

### Kubernetes Rolling Update কনফিগারেশন

```yaml
# rolling-update-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # সর্বোচ্চ ১টি অতিরিক্ত pod চালু হতে পারে
      maxUnavailable: 1    # সর্বোচ্চ ১টি pod unavailable থাকতে পারে
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:v2.1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
```

---

## 3️⃣ Blue/Green Deployment

দুটি **সম্পূর্ণ আলাদা পরিবেশ** (Blue ও Green) থাকে। একটিতে বর্তমান ভার্সন চলে, অন্যটিতে নতুন ভার্সন deploy হয়। প্রস্তুত হলে **traffic switch** করা হয়।

```
Blue/Green Deployment:

          Load Balancer
               │
     ┌─────────┼──────────┐
     │         │          │
     ▼         │          ▼
┌──────────┐   │   ┌──────────┐
│  Blue    │   │   │  Green   │
│  (v1)    │◄──┘   │  (v2)    │     ← ধাপ ১: Blue-এ traffic, Green-এ deploy
│ Active   │       │ Standby  │
└──────────┘       └──────────┘

          Load Balancer
               │
     ┌─────────┼──────────┐
     │         │          │
     ▼         │          ▼
┌──────────┐   │   ┌──────────┐
│  Blue    │   │   │  Green   │
│  (v1)    │   └──►│  (v2)    │     ← ধাপ ২: Traffic switch! Green active
│ Standby  │       │ Active   │
└──────────┘       └──────────┘

রোলব্যাক: Load Balancer আবার Blue-তে point করলেই হলো (তাৎক্ষণিক)
```

### Blue/Green Kubernetes Service কনফিগারেশন

```yaml
# blue-green-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: green        # এই selector বদলে blue/green switch হয়
  ports:
    - port: 80
      targetPort: 8080
---
# Blue Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:v1.0.0
---
# Green Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: green
  template:
    metadata:
      labels:
        app: order-service
        version: green
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:v2.0.0
```

---

## 4️⃣ Canary Deployment

নতুন ভার্সন অল্প সংখ্যক ব্যবহারকারীর কাছে পাঠানো হয়। সমস্যা না থাকলে ধীরে ধীরে সকল ব্যবহারকারীর কাছে পৌঁছানো হয়।

```
Canary Deployment (ধাপে ধাপে):

ধাপ ১: ৫% traffic নতুন ভার্সনে
┌──────────────────────────────────────────────────┐
│ v1 ██████████████████████████████████████████████ │ 95%
│ v2 ███                                           │  5%
└──────────────────────────────────────────────────┘

ধাপ ২: সব ঠিক আছে → ২৫% traffic
┌──────────────────────────────────────────────────┐
│ v1 ████████████████████████████████████           │ 75%
│ v2 █████████████                                  │ 25%
└──────────────────────────────────────────────────┘

ধাপ ৩: metrics ভালো → ৫০% traffic
┌──────────────────────────────────────────────────┐
│ v1 ████████████████████████                       │ 50%
│ v2 ████████████████████████                       │ 50%
└──────────────────────────────────────────────────┘

ধাপ ৪: সম্পূর্ণ rollout → ১০০%
┌──────────────────────────────────────────────────┐
│ v2 ██████████████████████████████████████████████ │ 100%
└──────────────────────────────────────────────────┘

⚠️ যেকোনো ধাপে সমস্যা হলে → v1-এ ফিরে যাওয়া (instant rollback)
```

### Istio Canary VirtualService

```yaml
# canary-virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

---

## 5️⃣ A/B Testing Deployment

ব্যবহারকারীর **বৈশিষ্ট্য অনুযায়ী** (location, device, user group) ভিন্ন ভার্সন দেখানো। এটি মূলত **business metric** (conversion, engagement) পরিমাপের জন্য।

```
A/B Testing:

                    Load Balancer
                         │
              ┌──────────┤──────────┐
              │  Routing Rules      │
              │  Header: region=BD  │
              │  Cookie: beta=true  │
              └──────────┤──────────┘
                    ┌────┴────┐
                    │         │
                    ▼         ▼
              ┌──────────┐ ┌──────────┐
              │ Version A│ │ Version B│
              │ (Control)│ │ (Test)   │
              │ ৮০% user │ │ ২০% user │
              └──────────┘ └──────────┘
                    │              │
                    ▼              ▼
              ┌────────────────────────┐
              │   Analytics Platform   │
              │   Conversion তুলনা    │
              └────────────────────────┘
```

---

## 🐘 PHP উদাহরণ — Health Check ও Deployment Readiness

```php
<?php
/**
 * Deployment strategy সমর্থনের জন্য PHP health check system।
 * Kubernetes probes ও load balancer health checks-এর জন্য ব্যবহৃত হয়।
 */

class HealthCheckController
{
    private array $checks = [];

    public function __construct()
    {
        $this->checks = [
            'database' => fn() => $this->checkDatabase(),
            'cache' => fn() => $this->checkRedis(),
            'storage' => fn() => $this->checkDiskSpace(),
            'memory' => fn() => $this->checkMemory(),
        ];
    }

    /**
     * Liveness Probe — অ্যাপ্লিকেশন জীবিত আছে কিনা।
     * ব্যর্থ হলে Kubernetes pod restart করবে।
     */
    public function livenessCheck(): array
    {
        return [
            'status' => 'alive',
            'timestamp' => date('c'),
            'version' => getenv('APP_VERSION') ?: 'unknown',
            'uptime' => $this->getUptime(),
        ];
    }

    /**
     * Readiness Probe — অ্যাপ্লিকেশন traffic গ্রহণের জন্য প্রস্তুত কিনা।
     * ব্যর্থ হলে Service Endpoint থেকে সরিয়ে দেবে (traffic আসবে না)।
     * Rolling/Canary deploy-এ এটি অত্যন্ত গুরুত্বপূর্ণ।
     */
    public function readinessCheck(): array
    {
        $results = [];
        $allHealthy = true;

        foreach ($this->checks as $name => $checkFn) {
            try {
                $checkFn();
                $results[$name] = ['status' => 'healthy'];
            } catch (\Throwable $e) {
                $results[$name] = [
                    'status' => 'unhealthy',
                    'error' => $e->getMessage(),
                ];
                $allHealthy = false;
            }
        }

        if (!$allHealthy) {
            http_response_code(503);
        }

        return [
            'status' => $allHealthy ? 'ready' : 'not_ready',
            'checks' => $results,
            'version' => getenv('APP_VERSION') ?: 'unknown',
        ];
    }

    /**
     * Startup Probe — অ্যাপ্লিকেশন সম্পূর্ণ boot হয়েছে কিনা।
     * ধীর startup-এর ক্ষেত্রে liveness probe কে বিলম্বিত করে।
     */
    public function startupCheck(): array
    {
        $warmupComplete = file_exists('/app/storage/.warmup-complete');
        if (!$warmupComplete) {
            http_response_code(503);
        }

        return [
            'status' => $warmupComplete ? 'started' : 'starting',
            'warmup' => $warmupComplete,
        ];
    }

    /**
     * Blue/Green deployment-এ version-specific health check।
     * Load balancer এই endpoint চেক করে traffic switch করে।
     */
    public function deploymentCheck(): array
    {
        $readiness = $this->readinessCheck();

        return array_merge($readiness, [
            'deployment_slot' => getenv('DEPLOYMENT_SLOT') ?: 'unknown',
            'git_commit' => getenv('GIT_COMMIT') ?: 'unknown',
            'build_time' => getenv('BUILD_TIME') ?: 'unknown',
            'canary' => getenv('CANARY_ENABLED') === 'true',
        ]);
    }

    private function checkDatabase(): void
    {
        $pdo = new PDO(
            getenv('DB_DSN') ?: 'mysql:host=localhost;dbname=app',
            getenv('DB_USER') ?: 'root',
            getenv('DB_PASS') ?: ''
        );
        $pdo->query('SELECT 1');
    }

    private function checkRedis(): void
    {
        $redis = new Redis();
        $redis->connect(getenv('REDIS_HOST') ?: 'localhost', 6379);
        $redis->ping();
    }

    private function checkDiskSpace(): void
    {
        $freeBytes = disk_free_space('/');
        $threshold = 100 * 1024 * 1024; // 100MB minimum
        if ($freeBytes < $threshold) {
            throw new RuntimeException("ডিস্ক স্পেস কম: " . round($freeBytes / 1024 / 1024) . "MB");
        }
    }

    private function checkMemory(): void
    {
        $memUsage = memory_get_usage(true);
        $limit = 256 * 1024 * 1024; // 256MB limit
        if ($memUsage > $limit) {
            throw new RuntimeException("মেমোরি ব্যবহার বেশি: " . round($memUsage / 1024 / 1024) . "MB");
        }
    }

    private function getUptime(): string
    {
        $startFile = '/app/storage/.start-time';
        if (file_exists($startFile)) {
            $seconds = time() - (int)file_get_contents($startFile);
            return gmdate('H:i:s', $seconds);
        }
        return 'unknown';
    }
}

// Router — Kubernetes probes এই endpoints কল করবে
$controller = new HealthCheckController();
$uri = $_SERVER['REQUEST_URI'] ?? '';
header('Content-Type: application/json');

$response = match (true) {
    str_starts_with($uri, '/health/live') => $controller->livenessCheck(),
    str_starts_with($uri, '/health/ready') => $controller->readinessCheck(),
    str_starts_with($uri, '/health/startup') => $controller->startupCheck(),
    str_starts_with($uri, '/health/deployment') => $controller->deploymentCheck(),
    default => ['error' => 'Unknown health endpoint'],
};

echo json_encode($response, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
```

---

## 🟨 JavaScript উদাহরণ — Deployment Readiness ও Graceful Shutdown

```javascript
/**
 * Node.js deployment-ready সার্ভার।
 * Health probes, graceful shutdown, ও deployment slot awareness সহ।
 */

const express = require('express');
const app = express();

let isReady = false;
let isShuttingDown = false;

const APP_VERSION = process.env.APP_VERSION || 'unknown';
const DEPLOYMENT_SLOT = process.env.DEPLOYMENT_SLOT || 'unknown';
const startTime = Date.now();

// ==================== Health Endpoints ====================

/**
 * Liveness Probe — প্রসেস জীবিত আছে কিনা।
 * শুধু process crash detect করার জন্য — dependency চেক নয়।
 */
app.get('/health/live', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  res.json({
    status: 'alive',
    version: APP_VERSION,
    uptime: Math.floor((Date.now() - startTime) / 1000),
    pid: process.pid,
  });
});

/**
 * Readiness Probe — traffic গ্রহণের জন্য প্রস্তুত কিনা।
 * Rolling update-এ নতুন pod ready না হওয়া পর্যন্ত traffic আসবে না।
 * Canary deploy-এ এটি নতুন ভার্সনের সক্ষমতা যাচাই করে।
 */
app.get('/health/ready', async (req, res) => {
  if (isShuttingDown || !isReady) {
    return res.status(503).json({
      status: 'not_ready',
      reason: isShuttingDown ? 'shutting_down' : 'warming_up',
    });
  }

  const checks = await runHealthChecks();
  const allHealthy = Object.values(checks).every(c => c.status === 'healthy');

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ready' : 'not_ready',
    checks,
    version: APP_VERSION,
    slot: DEPLOYMENT_SLOT,
  });
});

/**
 * Deployment-specific endpoint — Blue/Green switch-এর আগে যাচাই করা।
 * Load balancer বা deployment script এই endpoint কল করে।
 */
app.get('/health/deployment', async (req, res) => {
  const checks = await runHealthChecks();
  const allHealthy = Object.values(checks).every(c => c.status === 'healthy');

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'deploy_ready' : 'not_ready',
    checks,
    metadata: {
      version: APP_VERSION,
      slot: DEPLOYMENT_SLOT,
      gitCommit: process.env.GIT_COMMIT || 'unknown',
      buildTime: process.env.BUILD_TIME || 'unknown',
      nodeVersion: process.version,
      memoryUsage: process.memoryUsage(),
    },
  });
});

// ==================== Health Check Functions ====================

async function runHealthChecks() {
  const checks = {};

  // Database check
  try {
    // Simulated DB check
    checks.database = { status: 'healthy', latency: '2ms' };
  } catch (err) {
    checks.database = { status: 'unhealthy', error: err.message };
  }

  // Redis check
  try {
    checks.cache = { status: 'healthy', latency: '1ms' };
  } catch (err) {
    checks.cache = { status: 'unhealthy', error: err.message };
  }

  // Memory check
  const memUsage = process.memoryUsage();
  const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
  checks.memory = {
    status: heapUsedMB < 450 ? 'healthy' : 'unhealthy',
    heapUsedMB,
  };

  return checks;
}

// ==================== Graceful Shutdown ====================

/**
 * Graceful shutdown — Rolling/Canary deploy-এ অত্যন্ত গুরুত্বপূর্ণ।
 * SIGTERM পেলে:
 * ১. নতুন request গ্রহণ বন্ধ (readiness fail)
 * ২. চলমান request শেষ হওয়ার অপেক্ষা
 * ৩. Connection বন্ধ
 * ৪. Process exit
 */
function gracefulShutdown(signal) {
  console.log(`🛑 ${signal} পাওয়া গেছে — graceful shutdown শুরু...`);
  isShuttingDown = true;

  // Kubernetes-কে readiness fail জানানোর জন্য কিছুটা সময় দেওয়া
  setTimeout(() => {
    server.close(() => {
      console.log('✅ সকল connection বন্ধ হয়েছে — process exit');
      process.exit(0);
    });

    // ৩০ সেকেন্ডের মধ্যে বন্ধ না হলে force exit
    setTimeout(() => {
      console.error('⚠️ Force shutdown — ৩০ সেকেন্ড পার হয়ে গেছে');
      process.exit(1);
    }, 30000);
  }, 5000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// ==================== Application Warmup ====================

/**
 * Startup warmup — cache preload, connection pool তৈরি।
 * Warmup শেষ হলে readiness probe pass করবে।
 */
async function warmup() {
  console.log('🔄 Warmup শুরু...');

  // Database connection pool তৈরি
  console.log('  📦 DB connection pool তৈরি হচ্ছে...');
  await new Promise(resolve => setTimeout(resolve, 1000));

  // Cache preload
  console.log('  📦 Cache preload হচ্ছে...');
  await new Promise(resolve => setTimeout(resolve, 500));

  isReady = true;
  console.log('✅ Warmup সম্পন্ন — সার্ভার ready');
}

// ==================== Server Start ====================

const PORT = process.env.PORT || 8080;
const server = app.listen(PORT, '0.0.0.0', async () => {
  console.log(`🚀 Server started on port ${PORT}`);
  console.log(`📌 Version: ${APP_VERSION}, Slot: ${DEPLOYMENT_SLOT}`);
  await warmup();
});
```

---

## ✅ কখন কোন Strategy ব্যবহার করবেন

| Strategy | কখন ব্যবহার করবেন | কখন করবেন না |
|----------|-------------------|--------------|
| **Recreate** | Development/staging পরিবেশ, ডাউনটাইম গ্রহণযোগ্য | Production-এ কখনো না |
| **Rolling Update** | সাধারণ production deploy, backward-compatible পরিবর্তন | Database schema breaking change থাকলে |
| **Blue/Green** | Mission-critical সিস্টেম, তাৎক্ষণিক রোলব্যাক দরকার, compliance | রিসোর্স বাজেট কম থাকলে (দ্বিগুণ সার্ভার লাগে) |
| **Canary** | বড় ব্যবহারকারী base, ঝুঁকিপূর্ণ পরিবর্তন, পারফরম্যান্স যাচাই | ছোট internal অ্যাপ, দ্রুত deploy দরকার |
| **A/B Testing** | Feature comparison, UI/UX পরীক্ষা, business metric পরিমাপ | Infrastructure পরিবর্তন, non-user-facing সার্ভিস |

---

## 📊 কৌশল নির্বাচন Decision Tree

```
নতুন ভার্সন deploy করতে চান?
│
├─ ডাউনটাইম গ্রহণযোগ্য?
│  └─ হ্যাঁ → Recreate
│
├─ তাৎক্ষণিক rollback দরকার?
│  ├─ হ্যাঁ, এবং বাজেট আছে → Blue/Green
│  └─ হ্যাঁ, কিন্তু বাজেট কম → Canary
│
├─ Business metric তুলনা করতে চান?
│  └─ হ্যাঁ → A/B Testing
│
└─ সাধারণ deploy, backward compatible?
   └─ হ্যাঁ → Rolling Update
```

---

## 🔑 মূল শিক্ষা

1. **কোনো একটি strategy সব পরিস্থিতিতে সেরা নয়** — প্রসঙ্গ অনুযায়ী বেছে নিন
2. **Health checks সঠিকভাবে implement করুন** — readiness probe ছাড়া rolling update বিপজ্জনক
3. **Graceful shutdown বাধ্যতামূলক** — চলমান request হারানো user experience নষ্ট করে
4. **Rollback plan সবসময় প্রস্তুত রাখুন** — deploy-এর আগে rollback পরীক্ষা করুন
5. **Canary + automated analysis** সবচেয়ে নিরাপদ — Flagger বা Argo Rollouts ব্যবহার করুন
6. **Monitoring ও alerting** deploy-এর সাথে একীভূত করুন — সমস্যা দ্রুত ধরার জন্য
