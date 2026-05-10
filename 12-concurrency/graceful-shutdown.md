# 🛑 গ্রেসফুল শাটডাউন (Graceful Shutdown) — নিরাপদে সিস্টেম বন্ধ করার কৌশল

> **"সিস্টেম বন্ধ করা সহজ — KILL করে দিলেই হয়। কিন্তু চলমান ১০০টি bKash পেমেন্ট মাঝপথে হারিয়ে না ফেলে বন্ধ করা — সেটাই আসল চ্যালেঞ্জ।"**

প্রোডাকশন সিস্টেমে deploy, scale-down, বা crash recovery-র সময় সার্ভার বন্ধ করতে হয়। কিন্তু ভুলভাবে বন্ধ করলে
in-flight request হারিয়ে যায়, ডাটাবেস corrupt হয়, message queue-র offset commit হয় না, এবং user-এর টাকা
কেটে যায় কিন্তু order confirm হয় না। **Graceful Shutdown** হলো সেই systematic process যা নিশ্চিত করে — সবকিছু
গুছিয়ে, চলমান কাজ শেষ করে, সব connection বন্ধ করে, তারপর প্রস্থান।

এই ডকুমেন্টে আমরা Unix signal handling থেকে শুরু করে Node.js/Express, PHP/Laravel, Kubernetes preStop hooks,
load balancer deregistration, এবং Daraz/bKash/Pathao-র বাস্তব সিনারিও পর্যন্ত গভীরে যাব।

---

## 📑 সূচিপত্র

1. [গ্রেসফুল শাটডাউন কী?](#-গ্রেসফুল-শাটডাউন-কী)
2. [কেন গুরুত্বপূর্ণ?](#-কেন-গুরুত্বপূর্ণ)
3. [সিগন্যাল হ্যান্ডলিং — SIGTERM, SIGINT, SIGKILL](#-সিগন্যাল-হ্যান্ডলিং--sigterm-sigint-sigkill)
4. [গ্রেসফুল শাটডাউন ফ্লো](#-গ্রেসফুল-শাটডাউন-ফ্লো)
5. [Node.js ইমপ্লিমেন্টেশন](#-nodejs-ইমপ্লিমেন্টেশন)
6. [PHP ইমপ্লিমেন্টেশন](#-php-ইমপ্লিমেন্টেশন)
7. [ডাটাবেস কানেকশন ক্লিনআপ](#-ডাটাবেস-কানেকশন-ক্লিনআপ)
8. [মেসেজ কিউ কনজিউমার](#-মেসেজ-কিউ-কনজিউমার)
9. [কুবারনেটিস ও ডকার](#-কুবারনেটিস-ও-ডকার)
10. [লোড ব্যালেন্সার ইন্টিগ্রেশন](#-লোড-ব্যালেন্সার-ইন্টিগ্রেশন)
11. [সুবিধা ও অসুবিধা](#-সুবিধা-ও-অসুবিধা)
12. [বেস্ট প্র্যাকটিস চেকলিস্ট](#-বেস্ট-প্র্যাকটিস-চেকলিস্ট)
13. [সাধারণ ভুল (Anti-patterns)](#️-সাধারণ-ভুল-anti-patterns)
14. [BD Scenario: Daraz Flash Sale Deploy](#-bd-scenario-daraz-flash-sale-deploy)

---

## 📌 গ্রেসফুল শাটডাউন কী?

ধরুন **Pathao** rider app-এর backend server বন্ধ করতে হবে নতুন version deploy-এর জন্য। এই মুহূর্তে:

```
   চলমান কাজের অবস্থা:
   ══════════════════════════════════════════════════════
   Request #1:  Rider location update → DB write চলছে
   Request #2:  Trip fare calculation → 70% complete
   Request #3:  Payment settlement → bKash API call চলছে
   Request #4:  Push notification → FCM-এ পাঠানো হচ্ছে
   Request #5:  নতুন ride matching → এইমাত্র শুরু হয়েছে
```

**Abrupt Shutdown (kill -9):**

```
   kill -9 pathao-server
   ═══════════════════════════════════════════════
   Request #1:  ❌ DB write incomplete → data corrupt
   Request #2:  ❌ Fare calculation lost → rider ভুল ভাড়া পাবে
   Request #3:  ❌ bKash টাকা কেটেছে কিন্তু settlement record নেই
   Request #4:  ❌ Notification হারিয়ে গেছে
   Request #5:  ❌ Ride match lost → user app-এ "কিছু হচ্ছে না"
   ═══════════════════════════════════════════════
   ফলাফল: ৫টি user-ই সমস্যায় পড়বে, support ticket বাড়বে
```

**Graceful Shutdown:**

```
   SIGTERM received → "আমি বন্ধ হচ্ছি, কিন্তু গুছিয়ে"
   ═══════════════════════════════════════════════
   Step 1: ✋ নতুন request গ্রহণ বন্ধ (503 দাও নতুনদের)
   Step 2: ⏳ চলমান ৫টি request শেষ হতে দাও (timeout: 30s)
   Step 3: 💾 DB connection pool drain করো
   Step 4: 📨 Message queue offset commit করো
   Step 5: 🔌 সব connection বন্ধ করো
   Step 6: ✅ Exit code 0 — পরিষ্কার প্রস্থান
   ═══════════════════════════════════════════════
   ফলাফল: সব user-ই তাদের response পেয়েছে, কিছুই হারায়নি
```

> **Graceful Shutdown** = "নতুন কাজ নেওয়া বন্ধ করো → চলমান কাজ শেষ করো → সব resource ছেড়ে দাও → তারপর exit।"

---

## 🔥 কেন গুরুত্বপূর্ণ?

### ১. Deployment — সবচেয়ে ঘন ঘন ঘটে

```
   বাস্তব সিনারিও: bKash Payment Service Deploy
   ════════════════════════════════════════════════════════════

   দিনে ৩-৪ বার deploy হয় (CI/CD pipeline)
   প্রতি deploy-এ ৮টি pod terminate → ৮টি নতুন pod start

   যদি graceful shutdown না থাকে:
   ─────────────────────────────────────────────────
   প্রতি deploy-এ ~50 in-flight payment request lost
   দিনে ৪ deploy × ৫০ = ২০০ failed payment
   মাসে ৬,০০০ failed payment
   প্রতিটিতে support ticket + manual reconciliation
   → মাসে লক্ষ টাকার operational cost + user trust হারানো
```

### ২. Auto-scaling — Scale-down Event

```
   রাত ২:০০ AM — traffic কমে গেছে
   ══════════════════════════════════════════
   Kubernetes HPA: 20 pods → 8 pods (12 pods terminate হবে)

   12 pods-এ চলমান কাজ:
   ├── 45 HTTP requests processing
   ├── 12 Kafka messages mid-consumption
   └── 8 scheduled jobs running

   Graceful shutdown ছাড়া → 65টি কাজ হারিয়ে যাবে
   Graceful shutdown সহ    → 0 loss, সব শেষ হয়ে তারপর exit
```

### ৩. Crash Recovery — Unexpected Failures

```
   OOM Kill / Hardware Failure → সত্যিকারের abrupt shutdown
   ═══════════════════════════════════════════════════════
   এই ক্ষেত্রে graceful shutdown সরাসরি সাহায্য করে না,
   কিন্তু graceful shutdown-এর জন্য যে design করা হয়
   (idempotency, transaction logs, offset tracking)
   — সেই design-ই crash recovery-তেও কাজে আসে।
```

---

## 📡 সিগন্যাল হ্যান্ডলিং — SIGTERM, SIGINT, SIGKILL

Unix/Linux-এ process-কে বিভিন্ন **signal** পাঠিয়ে নির্দেশ দেওয়া হয়:

```
   Signal         Number   Default Action   Catch করা যায়?   কে পাঠায়?
   ═══════════════════════════════════════════════════════════════════════════
   SIGTERM          15     Terminate          ✅ হ্যাঁ         kill, Kubernetes, Docker
   SIGINT            2     Terminate          ✅ হ্যাঁ         Ctrl+C (terminal)
   SIGKILL           9     Force kill         ❌ না            kill -9, OOM killer
   SIGQUIT           3     Core dump          ✅ হ্যাঁ         Ctrl+\ (terminal)
   SIGHUP            1     Terminate          ✅ হ্যাঁ         Terminal close, config reload
   SIGUSR1          10     User-defined       ✅ হ্যাঁ         Custom (e.g., log rotation)
   SIGUSR2          12     User-defined       ✅ হ্যাঁ         Custom (e.g., debug toggle)
```

### গুরুত্বপূর্ণ পার্থক্য: SIGTERM vs SIGKILL

```
   SIGTERM (সিগনাল ১৫) — "ভদ্রভাবে বলছি, বন্ধ হও"
   ═══════════════════════════════════════════════
   ┌──────────┐  SIGTERM   ┌──────────────────┐
   │  Caller  │ ─────────► │    Process       │
   │(K8s/Docker)          │  "আচ্ছা, আমি     │
   └──────────┘            │   গুছিয়ে বন্ধ    │
                           │   হচ্ছি..."       │
                           │  [cleanup code]  │
                           │  [close conns]   │
                           │  [flush buffers] │
                           │  exit(0) ✅       │
                           └──────────────────┘

   SIGKILL (সিগনাল ৯) — "এখনই মরো, কোনো কথা নেই"
   ═══════════════════════════════════════════════
   ┌──────────┐  SIGKILL   ┌──────────────────┐
   │  Caller  │ ─────────► │    Process       │
   │(kill -9) │            │                  │
   └──────────┘            │  💀 DEAD         │
                           │  (কোনো cleanup  │
                           │   code চলে না)   │
                           └──────────────────┘
   → open files corrupt
   → DB transactions rollback হবে (server side)
   → in-memory data হারানো
```

### Kubernetes-এর Shutdown Sequence

```
   Kubernetes Pod Termination Timeline
   ═══════════════════════════════════════════════════════════════

   T+0s     Pod status: Terminating
            ├── Endpoint removed from Service (LB থেকে সরানো)
            └── preStop hook execute (যদি থাকে)

   T+Xs     preStop শেষ → SIGTERM পাঠানো হয়

   T+Xs     Application graceful shutdown শুরু
     to     ├── নতুন request বন্ধ
   T+30s    ├── চলমান request শেষ
            ├── DB/cache/queue connections বন্ধ
            └── exit(0)

   T+30s    terminationGracePeriodSeconds শেষ
   (default)└── যদি এখনও চলছে → SIGKILL 💀

   ─────────────────────────────────────────────
   Timeline:  preStop ──── SIGTERM ──── ... ──── SIGKILL
              │                                    │
              0s                                  30s (default)
```

---

## 🔄 গ্রেসফুল শাটডাউন ফ্লো

সম্পূর্ণ graceful shutdown process-এর বিস্তারিত ফ্লো:

```
   ┌─────────────────────────────────────────────────────────────────┐
   │                    GRACEFUL SHUTDOWN FLOW                       │
   └─────────────────────────────────────────────────────────────────┘

   SIGTERM received
        │
        ▼
   ┌─────────────────────────┐
   │ 1. isShuttingDown = true │   ← Flag সেট করো
   │    Log: "Shutting down"  │
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 2. Health check → 503   │   ← LB নতুন traffic পাঠানো বন্ধ করবে
   │    /health → unhealthy  │
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 3. Stop accepting new   │   ← server.close() / stop listening
   │    requests/connections │
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 4. Wait for in-flight   │   ← activeRequests counter == 0
   │    requests to complete │      অথবা timeout (30s)
   │    ┌──────────────────┐ │
   │    │ Request #1 ████░ │ │
   │    │ Request #2 ██░░░ │ │
   │    │ Request #3 █████ ✓│ │
   │    └──────────────────┘ │
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 5. Flush buffers        │   ← Log buffer, metric buffer
   │    Commit offsets        │      Kafka offset, Redis pipeline
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 6. Close DB connections │   ← pool.end(), connection drain
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 7. Close MQ connections │   ← consumer.disconnect()
   │    Close cache conns    │      redis.quit()
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 8. Cleanup temp files   │   ← /uploads/temp, PID files
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │ 9. process.exit(0)      │   ← Clean exit
   │    Log: "Shutdown done"  │
   └─────────────────────────┘
```

### Request Lifecycle During Shutdown

```
   Timeline: ──────────────────────────────────────────────►

   SIGTERM
      │
      ▼
   ═══╤══════════════════════════════════════════╤════════
      │                                          │
      │  In-flight requests:                     │ Timeout
      │  ┌──────── Req A ─────────┐ ✅ complete  │ (30s)
      │  ┌────────── Req B ──────────────┐ ✅    │
      │  ┌──── Req C ──┐ ✅                      │
      │                                          │
      │  নতুন requests:                          │
      │     ✕ Req D → 503 Service Unavailable    │
      │     ✕ Req E → 503 Service Unavailable    │
      │     ✕ Req F → Connection Refused         │
      │                                          │
   ═══╧══════════════════════════════════════════╧════════
      │                                          │
   server.close()                           process.exit()
```

---

## 🟢 Node.js ইমপ্লিমেন্টেশন

### Express Server — সম্পূর্ণ Graceful Shutdown

```javascript
// server.js — bKash Payment Gateway Server

const express = require('express');
const { Pool } = require('pg');
const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

const app = express();
const SHUTDOWN_TIMEOUT = 30_000; // 30 seconds

// ─── Resource Setup ─────────────────────────────────
const dbPool = new Pool({
  host: 'pg-primary.bkash.internal',
  database: 'payments',
  max: 20,
  idleTimeoutMillis: 30000,
});

const redis = new Redis({
  host: 'redis-cluster.bkash.internal',
  port: 6379,
});

const kafka = new Kafka({ brokers: ['kafka-1:9092', 'kafka-2:9092'] });
const producer = kafka.producer();

// ─── Shutdown State ─────────────────────────────────
let isShuttingDown = false;
let activeRequests = 0;
let server;

// ─── Middleware: Request Tracking ───────────────────
app.use((req, res, next) => {
  if (isShuttingDown) {
    res.set('Connection', 'close');
    return res.status(503).json({
      error: 'সার্ভার বন্ধ হচ্ছে, অনুগ্রহ করে পুনরায় চেষ্টা করুন',
      retryAfter: 5,
    });
  }

  activeRequests++;
  res.on('finish', () => {
    activeRequests--;
  });
  next();
});

// ─── Health Check ───────────────────────────────────
app.get('/health', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  res.json({ status: 'healthy', activeRequests });
});

// ─── Payment Route ──────────────────────────────────
app.post('/api/payments/process', async (req, res) => {
  const { userId, amount, merchantId } = req.body;

  const client = await dbPool.connect();
  try {
    await client.query('BEGIN');
    await client.query(
      'INSERT INTO transactions (user_id, amount, merchant_id, status) VALUES ($1, $2, $3, $4)',
      [userId, amount, merchantId, 'processing']
    );
    // ... payment processing logic ...
    await client.query('COMMIT');
    res.json({ status: 'success', transactionId: 'txn_123' });
  } catch (err) {
    await client.query('ROLLBACK');
    res.status(500).json({ error: err.message });
  } finally {
    client.release();
  }
});

// ─── Server Start ───────────────────────────────────
server = app.listen(3000, () => {
  console.log('bKash Payment Server চালু হয়েছে port 3000-এ');
});

// ─── Graceful Shutdown Function ─────────────────────
async function gracefulShutdown(signal) {
  console.log(`\n⚠️  ${signal} received — graceful shutdown শুরু...`);
  isShuttingDown = true;

  // Step 1: HTTP server বন্ধ (নতুন connection নেবে না)
  server.close(() => {
    console.log('✅ HTTP server বন্ধ — নতুন connection নেই');
  });

  // Step 2: চলমান requests শেষ হওয়ার অপেক্ষা
  const waitForRequests = () =>
    new Promise((resolve) => {
      const check = setInterval(() => {
        console.log(`⏳ চলমান requests: ${activeRequests}`);
        if (activeRequests === 0) {
          clearInterval(check);
          resolve();
        }
      }, 1000);
    });

  // Step 3: Timeout সহ অপেক্ষা
  const shutdownTimeout = new Promise((resolve) => {
    setTimeout(() => {
      console.log(`⚠️  Timeout! ${activeRequests} requests জোর করে বন্ধ করা হচ্ছে`);
      resolve();
    }, SHUTDOWN_TIMEOUT);
  });

  await Promise.race([waitForRequests(), shutdownTimeout]);

  // Step 4: Kafka producer disconnect
  try {
    await producer.disconnect();
    console.log('✅ Kafka producer disconnected');
  } catch (err) {
    console.error('❌ Kafka disconnect error:', err.message);
  }

  // Step 5: Redis connection বন্ধ
  try {
    await redis.quit();
    console.log('✅ Redis connection বন্ধ');
  } catch (err) {
    console.error('❌ Redis close error:', err.message);
  }

  // Step 6: Database pool drain
  try {
    await dbPool.end();
    console.log('✅ Database pool drained');
  } catch (err) {
    console.error('❌ DB pool close error:', err.message);
  }

  console.log('🏁 Graceful shutdown সম্পন্ন। বিদায়!');
  process.exit(0);
}

// ─── Signal Handlers ────────────────────────────────
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Unhandled rejection-এও graceful shutdown
process.on('unhandledRejection', (reason) => {
  console.error('Unhandled Rejection:', reason);
  gracefulShutdown('UNHANDLED_REJECTION');
});
```

### Connection Keep-Alive সমস্যা ও সমাধান

```javascript
// সমস্যা: Keep-alive connection থাকলে server.close() অপেক্ষা করতে থাকে

// সমাধান: Active connections track করো এবং idle connections বন্ধ করো
const connections = new Set();

server.on('connection', (conn) => {
  connections.add(conn);
  conn.on('close', () => connections.delete(conn));
});

function destroyIdleConnections() {
  for (const conn of connections) {
    // idle connection-গুলো বন্ধ করো
    conn.end();
  }
}

async function gracefulShutdown(signal) {
  isShuttingDown = true;
  server.close();
  
  // ৫ সেকেন্ড পর idle connections জোর করে বন্ধ
  setTimeout(() => destroyIdleConnections(), 5000);
  
  // বাকি cleanup...
}
```

---

## 🐘 PHP ইমপ্লিমেন্টেশন

### Laravel Queue Worker — Graceful Shutdown

```php
<?php
// app/Console/Commands/PaymentQueueWorker.php
// Daraz Payment Processing Worker

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Services\PaymentService;
use App\Services\NotificationService;

class PaymentQueueWorker extends Command
{
    protected $signature = 'queue:payment-worker';
    protected $description = 'Daraz payment queue processor with graceful shutdown';

    private bool $shouldStop = false;
    private bool $isProcessing = false;
    private string $currentJobId = '';

    public function handle(): int
    {
        $this->info('🚀 Payment Worker চালু হয়েছে');

        // SIGTERM ও SIGINT হ্যান্ডলার রেজিস্টার
        if (extension_loaded('pcntl')) {
            pcntl_async_signals(true);

            pcntl_signal(SIGTERM, function () {
                $this->warn('⚠️  SIGTERM received — graceful shutdown শুরু...');
                $this->shouldStop = true;
            });

            pcntl_signal(SIGINT, function () {
                $this->warn('⚠️  SIGINT received — graceful shutdown শুরু...');
                $this->shouldStop = true;
            });
        }

        // Main processing loop
        while (!$this->shouldStop) {
            $job = $this->fetchNextJob();

            if ($job === null) {
                // কোনো job নেই — ১ সেকেন্ড অপেক্ষা
                sleep(1);
                continue;
            }

            $this->isProcessing = true;
            $this->currentJobId = $job->id;

            try {
                $this->processPayment($job);
                $this->markJobComplete($job);
                $this->info("✅ Job {$job->id} সফলভাবে process হয়েছে");
            } catch (\Throwable $e) {
                $this->error("❌ Job {$job->id} ব্যর্থ: {$e->getMessage()}");
                $this->markJobFailed($job, $e);
            } finally {
                $this->isProcessing = false;
                $this->currentJobId = '';
            }
        }

        // Shutdown cleanup
        $this->performCleanup();

        $this->info('🏁 Payment Worker সফলভাবে বন্ধ হয়েছে');
        return Command::SUCCESS;
    }

    private function processPayment(object $job): void
    {
        $this->info("💳 Processing payment: {$job->id} — ৳{$job->amount}");

        // bKash/Nagad API কল
        $paymentGateway = app(PaymentService::class);
        $result = $paymentGateway->charge($job->userId, $job->amount);

        // Transaction record আপডেট
        \DB::table('transactions')
            ->where('id', $job->transactionId)
            ->update([
                'status'       => 'completed',
                'gateway_ref'  => $result->referenceId,
                'completed_at' => now(),
            ]);

        // Notification পাঠাও
        app(NotificationService::class)->sendPaymentConfirmation(
            $job->userId,
            $job->amount,
            $result->referenceId
        );
    }

    private function performCleanup(): void
    {
        $this->info('🧹 Cleanup শুরু...');

        // ১. চলমান job-এর status log করো
        if ($this->isProcessing) {
            $this->warn("⚠️  Job {$this->currentJobId} চলমান ছিল — " .
                        "এটি পরবর্তী worker pickup করবে");
        }

        // ২. Database connection বন্ধ
        \DB::disconnect();
        $this->info('  ✅ Database connection বন্ধ');

        // ৩. Cache connection বন্ধ
        \Cache::getStore()->getRedis()->close();
        $this->info('  ✅ Redis connection বন্ধ');

        // ৪. Metric flush
        if (app()->bound('metrics')) {
            app('metrics')->flush();
            $this->info('  ✅ Metrics flushed');
        }
    }
}
```

### PHP-FPM Graceful Shutdown

```
   PHP-FPM Process Management
   ═══════════════════════════════════════════════════════════

   Master Process (PID 1)
   ├── Worker #1 (PID 101) — idle
   ├── Worker #2 (PID 102) — processing request (Daraz checkout)
   ├── Worker #3 (PID 103) — processing request (bKash callback)
   └── Worker #4 (PID 104) — idle

   Signal: kill -QUIT <master_pid>   ← graceful stop
   ═══════════════════════════════════════════════════════════
   ① Master: নতুন request accept বন্ধ
   ② Worker #1, #4: idle ছিল → তাৎক্ষণিক terminate
   ③ Worker #2: Daraz checkout শেষ হতে দাও → তারপর terminate
   ④ Worker #3: bKash callback শেষ হতে দাও → তারপর terminate
   ⑤ Master: সব worker শেষ → master exit

   php-fpm signals:
   ─────────────────────────────────────────
   SIGQUIT  → Graceful stop (চলমান request শেষ হতে দাও)
   SIGTERM  → Quick stop (চলমান request-ও বন্ধ)
   SIGUSR1  → Log reopen
   SIGUSR2  → Graceful reload (নতুন config সহ restart)
```

```ini
; /etc/php/8.2/fpm/php-fpm.conf

; Graceful shutdown timeout (seconds)
; এই সময়ের মধ্যে চলমান request শেষ না হলে force kill
process_control_timeout = 30

; Daemonize বন্ধ (Docker-এ foreground-এ চলবে)
daemonize = no

; Slow log — ৫ সেকেন্ডের বেশি সময় নিলে log করো
request_slowlog_timeout = 5s
slowlog = /var/log/php-fpm/slow.log

; Maximum execution time per request
request_terminate_timeout = 60s
```

---

## 💾 ডাটাবেস কানেকশন ক্লিনআপ

### Connection Pool Lifecycle During Shutdown

```
   Normal Operation:
   ═══════════════════════════════════════════════
   Pool (max: 20, active: 12, idle: 8)
   ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
   │A │A │A │A │A │A │A │A │A │A │A │A │I │I │I │I │I │I │I │I │
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
   A = Active (query চলছে), I = Idle (ব্যবহার হচ্ছে না)

   Shutdown Phase 1 — Idle বন্ধ:
   ═══════════════════════════════════════════════
   Pool (active: 12, idle: 0, closing: 8)
   ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
   │A │A │A │A │A │A │A │A │A │A │A │A │  ← শুধু active বাকি
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘

   Shutdown Phase 2 — Active শেষ হচ্ছে:
   ═══════════════════════════════════════════════
   Pool (active: 5, idle: 0)
   ┌──┬──┬──┬──┬──┐
   │A │A │A │A │A │  ← query শেষ হচ্ছে একে একে
   └──┴──┴──┴──┴──┘

   Shutdown Phase 3 — সব বন্ধ:
   ═══════════════════════════════════════════════
   Pool (active: 0, idle: 0) → pool.end() ✅
```

### Node.js — PostgreSQL Pool Drain

```javascript
// Pathao Trip Service — DB connection cleanup

const { Pool } = require('pg');

const pool = new Pool({
  host: 'pg-primary.pathao.internal',
  database: 'trips',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

async function drainDatabasePool() {
  console.log(`📊 Pool status: total=${pool.totalCount}, idle=${pool.idleCount}, waiting=${pool.waitingCount}`);
  
  // pool.end() অপেক্ষা করে যতক্ষণ সব active query শেষ না হয়
  // তারপর সব connection বন্ধ করে
  const drainTimeout = setTimeout(() => {
    console.warn('⚠️  DB pool drain timeout — force closing');
    // PostgreSQL client-এ force close নেই
    // Timeout হলে process.exit() করবো main function থেকে
  }, 15000);

  try {
    await pool.end();
    clearTimeout(drainTimeout);
    console.log('✅ Database pool drained successfully');
  } catch (err) {
    clearTimeout(drainTimeout);
    console.error('❌ DB pool drain error:', err.message);
  }
}
```

### PHP — PDO Connection Cleanup

```php
<?php
// Database connection cleanup for Laravel/Pathao Order Service

class DatabaseCleanup
{
    public static function drainConnections(): void
    {
        $connections = config('database.connections');

        foreach (array_keys($connections) as $name) {
            try {
                // চলমান transaction থাকলে rollback করো
                if (\DB::connection($name)->transactionLevel() > 0) {
                    \DB::connection($name)->rollBack();
                    echo "  ⚠️  {$name}: চলমান transaction rollback হয়েছে\n";
                }

                // Connection বন্ধ
                \DB::connection($name)->disconnect();
                echo "  ✅ {$name}: connection বন্ধ\n";
            } catch (\Throwable $e) {
                echo "  ❌ {$name}: disconnect error — {$e->getMessage()}\n";
            }
        }
    }
}
```

---

## 📨 মেসেজ কিউ কনজিউমার

### Kafka Consumer Graceful Shutdown

```
   Kafka Consumer Shutdown Flow
   ═══════════════════════════════════════════════════════════════

   SIGTERM
      │
      ▼
   ┌──────────────────────────┐
   │ 1. consumer.pause()      │  ← নতুন message fetch বন্ধ
   │    (সব partition pause)  │
   └───────────┬──────────────┘
               │
               ▼
   ┌──────────────────────────┐
   │ 2. Process remaining     │  ← ইতিমধ্যে fetch করা
   │    messages in buffer    │     messages process করো
   │    ┌────────────────┐    │
   │    │ msg 1 → ✅      │    │
   │    │ msg 2 → ✅      │    │
   │    │ msg 3 → ✅      │    │
   │    └────────────────┘    │
   └───────────┬──────────────┘
               │
               ▼
   ┌──────────────────────────┐
   │ 3. Commit offsets        │  ← consumer.commitOffsets()
   │    partition 0: offset 42│     সর্বশেষ processed offset
   │    partition 1: offset 87│     Kafka-তে জানাও
   └───────────┬──────────────┘
               │
               ▼
   ┌──────────────────────────┐
   │ 4. Leave consumer group  │  ← consumer.disconnect()
   │    Trigger rebalance     │     অন্য consumers partition
   │                          │     নিয়ে নেবে
   └──────────────────────────┘
```

### Node.js — KafkaJS Consumer

```javascript
// Daraz Order Event Consumer

const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'daraz-order-service',
  brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],
});

const consumer = kafka.consumer({ groupId: 'order-processing-group' });
let isShuttingDown = false;

async function startConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'order-events', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      if (isShuttingDown) return; // shutdown চলছে — নতুন message process করো না

      const event = JSON.parse(message.value.toString());
      console.log(`📦 Order event: ${event.type} — Order #${event.orderId}`);

      switch (event.type) {
        case 'ORDER_PLACED':
          await processNewOrder(event);
          break;
        case 'PAYMENT_CONFIRMED':
          await confirmPayment(event);
          break;
        case 'ORDER_CANCELLED':
          await cancelOrder(event);
          break;
      }
    },
  });
}

async function shutdownConsumer() {
  console.log('📨 Kafka consumer shutdown শুরু...');
  isShuttingDown = true;

  try {
    // consumer.disconnect() অভ্যন্তরীণভাবে:
    // ১. polling বন্ধ করে
    // ২. চলমান message processing শেষ হতে দেয়
    // ৩. offset commit করে
    // ৪. consumer group ছেড়ে দেয়
    await consumer.disconnect();
    console.log('✅ Kafka consumer disconnected — offsets committed');
  } catch (err) {
    console.error('❌ Kafka consumer shutdown error:', err.message);
  }
}

startConsumer().catch(console.error);

process.on('SIGTERM', async () => {
  await shutdownConsumer();
  process.exit(0);
});
```

### RabbitMQ Consumer — PHP

```php
<?php
// bKash Notification Consumer — RabbitMQ

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class NotificationConsumer
{
    private AMQPStreamConnection $connection;
    private $channel;
    private bool $shouldStop = false;

    public function __construct()
    {
        $this->connection = new AMQPStreamConnection(
            'rabbitmq.bkash.internal', 5672, 'guest', 'guest'
        );
        $this->channel = $this->connection->channel();

        // Queue declare
        $this->channel->queue_declare('notifications', false, true, false, false);

        // Prefetch: একবারে ১০টি message নাও
        $this->channel->basic_qos(null, 10, null);
    }

    public function consume(): void
    {
        // Signal handler
        pcntl_async_signals(true);
        pcntl_signal(SIGTERM, function () {
            echo "⚠️  SIGTERM — consuming বন্ধ করছি...\n";
            $this->shouldStop = true;
            // channel-এ basic_cancel পাঠাও
            $this->channel->basic_cancel($this->consumerTag);
        });

        $this->consumerTag = $this->channel->basic_consume(
            'notifications',
            'bkash-notifier',
            false, // no_local
            false, // no_ack (manual ack)
            false, // exclusive
            false, // nowait
            function (AMQPMessage $msg) {
                $this->processMessage($msg);
            }
        );

        echo "🚀 Notification consumer চালু — waiting for messages...\n";

        // consume loop — shouldStop হলে loop বন্ধ হবে
        while ($this->channel->is_consuming() && !$this->shouldStop) {
            $this->channel->wait(null, false, 2); // 2s timeout
        }

        $this->cleanup();
    }

    private function processMessage(AMQPMessage $msg): void
    {
        $data = json_decode($msg->getBody(), true);
        echo "📱 Sending notification to user: {$data['userId']}\n";

        try {
            // SMS/Push notification পাঠাও
            $this->sendNotification($data);
            // সফল — ACK পাঠাও
            $msg->ack();
        } catch (\Throwable $e) {
            // ব্যর্থ — NACK পাঠাও, requeue করো
            $msg->nack(true);
            echo "❌ Notification failed: {$e->getMessage()}\n";
        }
    }

    private function cleanup(): void
    {
        echo "🧹 RabbitMQ cleanup...\n";

        try {
            $this->channel->close();
            echo "  ✅ Channel বন্ধ\n";
        } catch (\Throwable $e) {
            echo "  ❌ Channel close error: {$e->getMessage()}\n";
        }

        try {
            $this->connection->close();
            echo "  ✅ Connection বন্ধ\n";
        } catch (\Throwable $e) {
            echo "  ❌ Connection close error: {$e->getMessage()}\n";
        }

        echo "🏁 Consumer সফলভাবে বন্ধ হয়েছে\n";
    }
}
```

---

## ☸️ কুবারনেটিস ও ডকার

### Kubernetes Pod Shutdown — বিস্তারিত

```yaml
# k8s/deployment.yaml — Daraz Order Service

apiVersion: apps/v1
kind: Deployment
metadata:
  name: daraz-order-service
spec:
  replicas: 8
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # ২টি extra pod চালু করো আগে
      maxUnavailable: 1   # একবারে মাত্র ১টি বন্ধ করো
  template:
    spec:
      # Graceful shutdown timeout — default 30s, বাড়ানো হয়েছে ৬০s
      terminationGracePeriodSeconds: 60

      containers:
        - name: order-service
          image: daraz/order-service:v2.3.1
          ports:
            - containerPort: 3000

          # Liveness — আমি কি জীবিত?
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 15

          # Readiness — আমি কি traffic নিতে পারি?
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5

          lifecycle:
            preStop:
              exec:
                # SIGTERM-এর আগে ৫ সেকেন্ড অপেক্ষা
                # যাতে LB endpoint সরিয়ে নিতে পারে
                command: ["sh", "-c", "sleep 5"]
```

### Kubernetes Shutdown Race Condition

```
   সমস্যা: SIGTERM ও Endpoint Removal একই সাথে ঘটে
   ═══════════════════════════════════════════════════════════════

   সময়     │  kube-proxy / iptables    │  Application Pod
   ─────────┼──────────────────────────┼─────────────────────────
   T+0s     │  Endpoint removal শুরু   │  SIGTERM পেয়েছে
            │  (async — সময় লাগে)      │  server.close() করেছে
   T+0.5s   │  এখনও পুরাতন iptables    │  নতুন request reject
            │  rule আছে — traffic      │  করছে 503 দিয়ে
            │  আসছে এই pod-এ!          │
   T+2s     │  Endpoint সরানো হয়েছে   │  
            │  নতুন traffic আসবে না     │  
   ─────────┴──────────────────────────┴─────────────────────────

   সমাধান: preStop hook-এ sleep দাও
   ═══════════════════════════════════════════════════════════════

   lifecycle:
     preStop:
       exec:
         command: ["sh", "-c", "sleep 5"]

   T+0s     │  Endpoint removal শুরু   │  preStop: sleep 5
   T+2s     │  Endpoint সরানো হয়েছে   │  এখনও sleep চলছে
            │  নতুন traffic আসবে না     │  (কিন্তু পুরাতন traffic
            │                          │   handle করছে)
   T+5s     │                          │  sleep শেষ → SIGTERM
            │                          │  → graceful shutdown
```

### Docker — STOPSIGNAL ও Best Practices

```dockerfile
# Dockerfile — Pathao Rider Service

FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production

COPY . .

# PID 1 সমস্যা সমাধান — tini ব্যবহার করো
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]

# STOPSIGNAL — docker stop পাঠালে কোন signal যাবে
STOPSIGNAL SIGTERM

# Health check
HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Grace period: docker stop --time=30
# Default 10s — আমাদের app-এর জন্য 30s দরকার

CMD ["node", "server.js"]
```

```
   Docker PID 1 সমস্যা
   ═══════════════════════════════════════════════════════

   সমস্যা: Node.js/PHP সরাসরি PID 1 হলে signal handle করে না

   ❌ ভুল:
   CMD node server.js
   → Shell (sh -c "node server.js") হয় PID 1
   → sh signal forward করে না → SIGTERM হারিয়ে যায়
   → 10s পর docker SIGKILL পাঠায় → abrupt shutdown 💀

   ✅ সমাধান ১: tini (init process)
   ENTRYPOINT ["/sbin/tini", "--"]
   CMD ["node", "server.js"]
   → tini হয় PID 1, signal সঠিকভাবে forward করে

   ✅ সমাধান ২: exec form ব্যবহার করো
   CMD ["node", "server.js"]    ← exec form (JSON array)
   CMD node server.js           ← shell form (❌ signal সমস্যা)

   ✅ সমাধান ৩: dumb-init
   RUN apt-get install dumb-init
   ENTRYPOINT ["dumb-init", "--"]
   CMD ["node", "server.js"]
```

---

## ⚖️ লোড ব্যালেন্সার ইন্টিগ্রেশন

### Deregistration Flow

```
   Load Balancer Deregistration During Shutdown
   ═══════════════════════════════════════════════════════════════

   স্বাভাবিক অবস্থা:
   ┌────────┐     ┌─────────────────────────────────────┐
   │  User  │────►│         Load Balancer (Nginx/ALB)    │
   └────────┘     │                                     │
                  │  ┌──────────┐  ┌──────────┐  ┌──────────┐
                  │  │ Server A │  │ Server B │  │ Server C │
                  │  │  (✅)    │  │  (✅)    │  │  (✅)    │
                  │  └──────────┘  └──────────┘  └──────────┘
                  └─────────────────────────────────────┘

   Server B shutdown শুরু:
   ┌────────┐     ┌─────────────────────────────────────┐
   │  User  │────►│         Load Balancer               │
   └────────┘     │                                     │
                  │  ┌──────────┐  ┌──────────┐  ┌──────────┐
                  │  │ Server A │  │ Server B │  │ Server C │
                  │  │  (✅)    │  │  (🔄)    │  │  (✅)    │
                  │  └──────────┘  │ draining │  └──────────┘
                  │                │ in-flight│
                  │     ← নতুন traffic শুধু A ও C-তে →    │
                  └─────────────────────────────────────┘

   Server B shutdown সম্পূর্ণ:
   ┌────────┐     ┌─────────────────────────────────────┐
   │  User  │────►│         Load Balancer               │
   └────────┘     │                                     │
                  │  ┌──────────┐               ┌──────────┐
                  │  │ Server A │               │ Server C │
                  │  │  (✅)    │               │  (✅)    │
                  │  └──────────┘               └──────────┘
                  └─────────────────────────────────────┘
```

### AWS ALB Connection Draining

```
   AWS ALB — Deregistration / Connection Draining
   ═══════════════════════════════════════════════════════════

   Target Group Setting:
   ┌─────────────────────────────────────────────┐
   │ Deregistration delay: 300 seconds (default) │
   │ bKash-র জন্য: 60 seconds (fast deploy)      │
   └─────────────────────────────────────────────┘

   Timeline:
   ─────────────────────────────────────────────────────────
   T+0s    Target deregistered from ALB
           ├── নতুন connection: অন্য target-এ যাবে ✅
           └── existing connection: এই target-এই থাকবে

   T+0-60s Connection draining period
           ├── existing request শেষ হচ্ছে
           ├── keep-alive connection-এ নতুন request আসতে পারে
           └── ALB: "Connection: close" header যোগ করে

   T+60s   Drain period শেষ
           └── বাকি connection force close
   ─────────────────────────────────────────────────────────
```

### Nginx Upstream Health Check

```nginx
# nginx.conf — Daraz API Gateway

upstream daraz_api {
    server api-1.daraz.internal:3000;
    server api-2.daraz.internal:3000;
    server api-3.daraz.internal:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://daraz_api;
        proxy_next_upstream error timeout http_503;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;

        # Server 503 দিলে পরবর্তী server-এ পাঠাও
        # graceful shutdown-এ server 503 দেয় — Nginx
        # স্বয়ংক্রিয়ভাবে অন্য server বেছে নেবে
    }

    # Active health check (Nginx Plus)
    location /health {
        proxy_pass http://daraz_api;
        health_check interval=5s fails=2 passes=1;
    }
}
```

---

## 📊 সুবিধা ও অসুবিধা

```
   ┌───────────────────────────────────────────┬─────────────────────────────────────────┐
   │           ✅ সুবিধা                       │           ❌ অসুবিধা                     │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Zero data loss — চলমান কাজ হারায় না      │ Shutdown সময় বেশি লাগে (30-60s)         │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Zero downtime deployment সম্ভব হয়        │ Implementation complexity বাড়ে           │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ User experience ভালো — error দেখে না     │ Resource ধরে রাখে shutdown-এর সময়       │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Database consistency বজায় থাকে           │ Timeout tuning দরকার (too short/long)   │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Message queue offset সঠিক থাকে —         │ প্রতিটি resource-এর জন্য আলাদা          │
   │ duplicate processing কমে                 │ cleanup logic লিখতে হয়                  │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Operational cost কমে — support ticket     │ Testing কঠিন — production environment   │
   │ কমে, manual reconciliation কমে           │ simulate করা সহজ না                     │
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Kubernetes/Docker-এ graceful rolling      │ Long-running request (file upload,      │
   │ update সম্ভব হয়                          │ report generation) আলাদা handle করতে হয়│
   ├───────────────────────────────────────────┼─────────────────────────────────────────┤
   │ Audit trail সম্পূর্ণ থাকে — কোনো        │ Distributed system-এ সব service-ই       │
   │ "ghost transaction" থাকে না               │ graceful হতে হবে — একটি না হলেও সমস্যা  │
   └───────────────────────────────────────────┴─────────────────────────────────────────┘
```

---

## ✅ বেস্ট প্র্যাকটিস চেকলিস্ট

```
   Graceful Shutdown Implementation Checklist
   ═══════════════════════════════════════════════════════════

   Signal Handling:
   ☐ SIGTERM handler রেজিস্টার করা হয়েছে
   ☐ SIGINT handler রেজিস্টার করা হয়েছে (local dev-এর জন্য)
   ☐ Signal handler idempotent (একাধিকবার call হলেও সমস্যা নেই)
   ☐ Signal handler async-safe (blocking operation নেই)

   HTTP Server:
   ☐ server.close() কল করা হয়েছে (নতুন connection বন্ধ)
   ☐ Health endpoint shutdown-এ 503 দেয়
   ☐ In-flight request tracking আছে (activeRequests counter)
   ☐ Keep-alive connection handle করা হয়েছে
   ☐ "Connection: close" header যোগ করা হয়েছে shutdown-এ

   Database:
   ☐ Connection pool drain করা হয়েছে
   ☐ চলমান transaction commit/rollback হয়েছে
   ☐ Prepared statements cleanup হয়েছে

   Message Queue:
   ☐ Consumer pause/stop করা হয়েছে
   ☐ In-flight messages process শেষ হয়েছে
   ☐ Offset commit হয়েছে (Kafka)
   ☐ Acknowledgement পাঠানো হয়েছে (RabbitMQ)
   ☐ Consumer group ছেড়ে দেওয়া হয়েছে

   Cache/Redis:
   ☐ Pipeline flush হয়েছে
   ☐ Connection quit() করা হয়েছে

   Kubernetes/Docker:
   ☐ terminationGracePeriodSeconds সঠিকভাবে সেট আছে
   ☐ preStop hook আছে (race condition এড়াতে)
   ☐ readinessProbe shutdown-এ fail করে
   ☐ PID 1 সমস্যা সমাধান হয়েছে (tini/dumb-init)
   ☐ STOPSIGNAL সেট আছে Dockerfile-এ

   Monitoring:
   ☐ Shutdown শুরু/শেষ log করা হয়েছে
   ☐ Shutdown duration metric track করা হয়েছে
   ☐ Forced shutdown (timeout) alert আছে
   ☐ Active request count shutdown-এ log হচ্ছে

   Testing:
   ☐ Graceful shutdown integration test আছে
   ☐ Timeout scenario test আছে
   ☐ Multiple signal scenario test আছে
   ☐ Load test-এর সময় rolling restart test করা হয়েছে
```

---

## ⚠️ সাধারণ ভুল (Anti-patterns)

### ১. process.exit() সরাসরি কল করা

```javascript
// ❌ ভুল — কোনো cleanup ছাড়াই exit
process.on('SIGTERM', () => {
  console.log('বন্ধ হচ্ছি');
  process.exit(0); // DB connection open, requests pending!
});

// ✅ সঠিক — cleanup করে তারপর exit
process.on('SIGTERM', async () => {
  console.log('Graceful shutdown শুরু...');
  server.close();
  await waitForActiveRequests();
  await dbPool.end();
  await redis.quit();
  process.exit(0);
});
```

### ২. Shutdown Timeout না রাখা

```javascript
// ❌ ভুল — অনন্তকাল অপেক্ষা করবে
async function shutdown() {
  await waitForActiveRequests(); // যদি কোনো request stuck থাকে?
  // → process কখনো exit হবে না
  // → Kubernetes 30s পর SIGKILL করবে
  // → ততক্ষণে DB cleanup হয়নি
}

// ✅ সঠিক — timeout সহ অপেক্ষা
async function shutdown() {
  const timeout = setTimeout(() => {
    console.error('Shutdown timeout — force exit');
    process.exit(1);
  }, 25000); // K8s-এর 30s-এর আগেই exit করো

  await waitForActiveRequests();
  await dbPool.end();
  clearTimeout(timeout);
  process.exit(0);
}
```

### ৩. Signal Handler একাধিকবার চলা

```javascript
// ❌ ভুল — SIGTERM দুইবার আসলে দুইবার shutdown চালাবে
process.on('SIGTERM', () => gracefulShutdown());
// rapid SIGTERM → race condition → double cleanup → error

// ✅ সঠিক — একবারই চলবে
let shuttingDown = false;
process.on('SIGTERM', () => {
  if (shuttingDown) return;
  shuttingDown = true;
  gracefulShutdown();
});
```

### ৪. preStop Hook না দেওয়া (Kubernetes)

```yaml
# ❌ ভুল — preStop নেই, race condition হবে
spec:
  containers:
    - name: api
      # SIGTERM আসবে কিন্তু LB এখনও traffic পাঠাচ্ছে

# ✅ সঠিক — preStop দিয়ে LB sync হতে সময় দাও
spec:
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

### ৫. Worker Process-এ Signal Forward না করা

```
   ❌ Master-Worker Architecture-এ ভুল:
   ═══════════════════════════════════════════════
   SIGTERM → Master (PID 1)
             ├── Worker 1 (PID 101) — SIGTERM পায়নি, চলছে
             ├── Worker 2 (PID 102) — SIGTERM পায়নি, চলছে
             └── Worker 3 (PID 103) — SIGTERM পায়নি, চলছে
   Master exit → Workers orphaned → SIGKILL 💀

   ✅ সঠিক:
   ═══════════════════════════════════════════════
   SIGTERM → Master (PID 1)
             ├── Master: SIGTERM forward → Worker 1 → graceful stop
             ├── Master: SIGTERM forward → Worker 2 → graceful stop
             └── Master: SIGTERM forward → Worker 3 → graceful stop
             Master: সব worker শেষ → Master exit
```

### ৬. Shutdown-এ নতুন Async কাজ শুরু করা

```javascript
// ❌ ভুল — shutdown-এর সময়ও নতুন setTimeout/setInterval চালু
async function shutdown() {
  setInterval(() => checkStatus(), 1000); // নতুন interval!
  // → process.exit() block হতে পারে
  
  await cleanup();
}

// ✅ সঠিক — সব timer clear করো
async function shutdown() {
  clearInterval(healthCheckTimer);
  clearInterval(metricsFlushTimer);
  clearTimeout(cacheRefreshTimer);
  
  await cleanup();
  process.exit(0);
}
```

---

## 🇧🇩 BD Scenario: Daraz Flash Sale Deploy

```
   সিনারিও: Daraz 11.11 Flash Sale — নতুন version deploy
   ═══════════════════════════════════════════════════════════════

   পরিস্থিতি:
   - রাত ১২:০০ AM — Flash sale শুরু হবে
   - রাত ১১:৫৫ PM — critical bug fix deploy করতে হবে
   - প্রতি সেকেন্ডে ৫,০০০ request আসছে (pre-sale traffic)
   - ১৬টি pod চালু আছে
   - প্রতি pod-এ ~৩০০ active connection

   Rolling Update Strategy:
   ─────────────────────────────────────────────────────────────

   Step 1: Pod #1 shutdown শুরু
   ┌─────────────────────────────────────────────────────────┐
   │ T+0s   preStop hook: sleep 5                            │
   │         → ALB health check fail → deregister from LB   │
   │ T+5s   SIGTERM → app receives signal                   │
   │         → isShuttingDown = true                         │
   │         → /health → 503                                 │
   │         → server.close()                                │
   │ T+6s   ২৮৭টি active request চলছে                       │
   │ T+12s  ১৪২টি active request বাকি                       │
   │ T+18s  ২৩টি active request বাকি                        │
   │ T+22s  ০টি active request → cleanup শুরু               │
   │ T+23s  DB pool drained                                  │
   │ T+24s  Redis disconnected                               │
   │ T+25s  Kafka producer disconnected                      │
   │ T+25s  exit(0) ✅                                       │
   └─────────────────────────────────────────────────────────┘

   Pod #1 নতুন version-এ start হচ্ছে...
   Pod #2 shutdown শুরু...
   (একই process repeat)

   মোট সময়: ১৬ pods × ~30s = ~8 minutes rolling update
   Lost requests: 0 ✅
   User impact: কেউ error দেখেনি ✅

   ─────────────────────────────────────────────────────────────

   যদি Graceful Shutdown না থাকতো:
   ┌─────────────────────────────────────────────────────────┐
   │ প্রতি pod kill-এ ~300 request lost                      │
   │ ১৬ pods × ৩০০ = ৪,৮০০ failed requests                  │
   │ এর মধ্যে ~৫০০ payment processing ছিল                   │
   │ ৫০০ × গড় ৳১,৫০০ = ৳৭,৫০,০০০ এর transaction ঝুঁকিতে   │
   │ + হাজার হাজার angry customer, social media-তে viral     │
   │ + Bangladesh Bank-এর নোটিশ                              │
   └─────────────────────────────────────────────────────────┘
```

---

## 🎯 সারসংক্ষেপ

```
   Graceful Shutdown = নিরাপদে বন্ধ হওয়ার শিষ্টাচার
   ═══════════════════════════════════════════════════════════

   মূল নীতি:
   ┌─────────────────────────────────────────────────────┐
   │  ১. Signal ধরো (SIGTERM/SIGINT)                     │
   │  ২. নতুন কাজ নেওয়া বন্ধ করো                       │
   │  ৩. চলমান কাজ শেষ হতে দাও (timeout সহ)            │
   │  ৪. সব resource গুছিয়ে ছেড়ে দাও                   │
   │  ৫. পরিষ্কারভাবে exit করো                          │
   └─────────────────────────────────────────────────────┘

   মনে রাখো:
   ─────────────────────────────────────────────────────────
   "kill -9 হলো ঘুমন্ত মানুষকে বালতি পানি ঢেলে জাগানো।
    SIGTERM হলো আলতো করে ডেকে বলা — উঠো, সময় হয়েছে।
    Graceful shutdown হলো — উঠে, মুখ ধুয়ে, বিছানা গুছিয়ে,
    লাইট বন্ধ করে, তারপর ঘর থেকে বের হওয়া।"
   ─────────────────────────────────────────────────────────
```
