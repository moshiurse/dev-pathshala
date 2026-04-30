# 📌 থ্রেড পুল (Thread Pool Pattern)

## থ্রেড পুল কী এবং কেন দরকার?

থ্রেড পুল হলো আগে থেকে তৈরি করা কিছু থ্রেড (বা ওয়ার্কার) এর একটি গ্রুপ, যারা একটি টাস্ক
কিউ থেকে কাজ নিয়ে সম্পন্ন করে। প্রতিটি নতুন কাজের জন্য নতুন থ্রেড তৈরি না করে, আগে থেকে
তৈরি থ্রেড পুনরায় ব্যবহার (reuse) করা হয়।

### 🏠 বাস্তব উদাহরণ — bKash কাস্টমার সার্ভিস

```
কল্পনা করুন bKash এর কাস্টমার সার্ভিস সেন্টার:

🏢 bKash কল সেন্টার
├── 50 জন এজেন্ট (= 50 টি থ্রেড)
├── প্রতিদিন 10,000+ কল আসে (= 10,000+ টাস্ক)
└── প্রতিটি কলের জন্য নতুন এজেন্ট হায়ার করা অসম্ভব!

❌ থ্রেড পুল ছাড়া (Bad Approach):
   কল আসলো → নতুন এজেন্ট হায়ার → ট্রেনিং → কল হ্যান্ডেল → ফায়ার
   কল আসলো → নতুন এজেন্ট হায়ার → ট্রেনিং → কল হ্যান্ডেল → ফায়ার
   (প্রতিবার হায়ার/ফায়ার = ব্যয়বহুল!)

✅ থ্রেড পুল দিয়ে (Good Approach):
   50 এজেন্ট আগে থেকেই রেডি → কল আসলে ফ্রি এজেন্ট ধরে → কাজ শেষে পরের কল নেয়
   (এজেন্ট reuse হচ্ছে = সাশ্রয়ী!)
```

### থ্রেড তৈরির খরচ কত?

```
+--------------------------------------------------+
|        থ্রেড তৈরির খরচ (Thread Creation Cost)     |
+--------------------------------------------------+
|                                                    |
|  প্রতিটি নতুন থ্রেড তৈরি করতে লাগে:              |
|  ┌─────────────────────────────────────────┐      |
|  │ • মেমোরি অ্যালোকেশন (Stack ~1MB)       │      |
|  │ • OS কার্নেল রিসোর্স                    │      |
|  │ • কনটেক্সট সুইচিং ওভারহেড              │      |
|  │ • রেজিস্টার সেভ/রিস্টোর                │      |
|  └─────────────────────────────────────────┘      |
|                                                    |
|  1 টাস্ক    → 1 থ্রেড = ঠিক আছে                  |
|  100 টাস্ক  → 100 থ্রেড = ধীর হয়ে যাচ্ছে         |
|  10K টাস্ক  → 10K থ্রেড = 💥 সার্ভার ক্র্যাশ!     |
+--------------------------------------------------+
```

---

## 📊 থ্রেড পুল আর্কিটেকচার

থ্রেড পুলের তিনটি প্রধান উপাদান:
1. **টাস্ক কিউ (Task Queue)** — কাজ জমা রাখার সারি
2. **ওয়ার্কার থ্রেড (Worker Threads)** — কাজ সম্পন্নকারী
3. **পুল ম্যানেজার (Pool Manager)** — কাজ বণ্টনকারী

```
    Task Queue              Thread Pool                  Results
+---------------+     +--------------------------+     +----------+
| ┌───────────┐ |     | ┌────────┐               |     |          |
| │  Task 1   │ | ──> | │Worker 1│ ⚙️  busy      | ──> | Result 1 |
| └───────────┘ |     | └────────┘               |     |          |
| ┌───────────┐ |     | ┌────────┐               |     |          |
| │  Task 2   │ | ──> | │Worker 2│ ⚙️  busy      | ──> | Result 2 |
| └───────────┘ |     | └────────┘               |     |          |
| ┌───────────┐ |     | ┌────────┐               |     |          |
| │  Task 3   │ |     | │Worker 3│ 💤 idle       |     |          |
| └───────────┘ |     | └────────┘               |     |          |
| ┌───────────┐ |     | ┌────────┐               |     |          |
| │  Task 4   │ |     | │Worker 4│ 💤 idle       |     |          |
| └───────────┘ |     | └────────┘               |     |          |
| ┌───────────┐ |     +--------------------------+     +----------+
| │  Task 5   │ |           Pool Manager
| └───────────┘ |      (ফ্রি ওয়ার্কারে টাস্ক দেয়)
+---------------+
```

### লাইফসাইকেল ডায়াগ্রাম

```
                    থ্রেড পুল লাইফসাইকেল
                    ════════════════════

  [পুল তৈরি]
       │
       ▼
  ┌─────────────┐     ┌─────────────────────────────────┐
  │  Initialize  │────>│  N সংখ্যক ওয়ার্কার থ্রেড তৈরি  │
  └─────────────┘     └─────────────────────────────────┘
       │
       ▼
  ┌─────────────┐     ┌─────────────────────────────────┐
  │  Ready       │────>│  ওয়ার্কাররা কিউ থেকে কাজ নেয়    │
  └─────────────┘     └─────────────────────────────────┘
       │                          │
       │                    ┌─────┴──────┐
       │                    ▼            ▼
       │              [কাজ আছে]    [কাজ নেই]
       │                    │            │
       │                    ▼            ▼
       │              ⚙️ Execute    💤 Wait
       │                    │            │
       │                    └─────┬──────┘
       │                          │
       ▼                          ▼
  ┌─────────────┐     ┌─────────────────────────────────┐
  │  Shutdown    │────>│  সব ওয়ার্কার থামানো, রিসোর্স ফ্রি │
  └─────────────┘     └─────────────────────────────────┘
```

---

## 📏 পুল সাইজিং স্ট্র্যাটেজি

সঠিক পুল সাইজ নির্ধারণ করা অত্যন্ত গুরুত্বপূর্ণ। খুব কম ওয়ার্কার থাকলে টাস্ক পেন্ডিং পড়ে থাকবে,
আর খুব বেশি থাকলে কনটেক্সট সুইচিং ওভারহেড বেড়ে যাবে।

### ⚡ CPU-Bound কাজের জন্য

```
╔══════════════════════════════════════════════════╗
║  Formula: pool_size = number_of_cores            ║
║                                                  ║
║  কেন? CPU-bound কাজে থ্রেড সবসময় CPU ব্যবহার   ║
║  করে। কোর সংখ্যার বেশি থ্রেড দিলে শুধু           ║
║  context switching বাড়বে, কাজ দ্রুত হবে না।      ║
╚══════════════════════════════════════════════════╝

উদাহরণ: Pathao এর রাইড ম্যাচিং অ্যালগরিদম
─────────────────────────────────────────────
সার্ভার: 8-core CPU

pool_size = 8  (8 কোর = 8 ওয়ার্কার)

প্রতিটি ওয়ার্কার একটি করে রাইড ম্যাচিং ক্যালকুলেশন করে।
ভারী গণনা — দূরত্ব, ট্রাফিক, দাম সব হিসাব করতে হয়।
```

### 🌐 I/O-Bound কাজের জন্য

```
╔══════════════════════════════════════════════════════════════╗
║  Formula: pool_size = cores × (1 + wait_time / compute_time)║
║                                                              ║
║  কেন? I/O-bound কাজে থ্রেড বেশিরভাগ সময় অপেক্ষা করে       ║
║  (DB query, API call, ফাইল রিড)। তাই বেশি থ্রেড রাখলে      ║
║  অপেক্ষার সময়ে অন্য থ্রেড CPU ব্যবহার করতে পারে।          ║
╚══════════════════════════════════════════════════════════════╝

উদাহরণ: Daraz এর প্রোডাক্ট ক্যাটালগ API
─────────────────────────────────────────
সার্ভার: 4-core CPU
গড় DB query wait: 200ms
গড় compute time: 50ms

pool_size = 4 × (1 + 200/50)
         = 4 × (1 + 4)
         = 4 × 5
         = 20 ওয়ার্কার

অর্থাৎ, 4 কোরের সার্ভারে 20 টি ওয়ার্কার দিলে সর্বোচ্চ
throughput পাওয়া যাবে!
```

### 📊 সাইজিং চার্ট

```
কাজের ধরন        | কোর | Wait | Compute | পুল সাইজ
═════════════════╪══════╪══════╪═════════╪══════════
CPU-bound        |  8   |  0   |   100%  |    8
মিক্সড (50/50)   |  8   |  50  |   50    |   16
I/O-heavy        |  8   | 200  |   50    |   40
খুব I/O-heavy    |  8   | 500  |   25    |  168
```

---

## 🐘 PHP তে থ্রেড পুল বিকল্প

PHP তে সরাসরি থ্রেড সাপোর্ট নেই (PHP মূলত সিঙ্গেল-থ্রেডেড)। তবে বেশ কিছু বিকল্প আছে:

### বিকল্প ১: pcntl_fork দিয়ে প্রসেস পুল

```php
<?php
/**
 * pcntl_fork ভিত্তিক প্রসেস পুল
 * 
 * Nagad এর বাল্ক ট্রানজ্যাকশন প্রসেসিং এর উদাহরণ:
 * হাজার হাজার ট্রানজ্যাকশন একসাথে ভেরিফাই করতে হবে
 */
function processPool(array $tasks, int $poolSize = 4): void
{
    $chunks = array_chunk($tasks, (int) ceil(count($tasks) / $poolSize));
    $pids = [];

    foreach ($chunks as $index => $chunk) {
        $pid = pcntl_fork();

        if ($pid === -1) {
            die("Fork ব্যর্থ হয়েছে!");
        }

        if ($pid === 0) {
            // চাইল্ড প্রসেস — নিজের অংশের কাজ করবে
            foreach ($chunk as $task) {
                processTransaction($task);
            }
            exit(0);
        }

        // প্যারেন্ট প্রসেস — চাইল্ড PID রাখে
        $pids[] = $pid;
        echo "ওয়ার্কার #{$index} শুরু হলো (PID: {$pid})\n";
    }

    // সব চাইল্ড প্রসেস শেষ হওয়ার অপেক্ষা
    foreach ($pids as $pid) {
        pcntl_waitpid($pid, $status);
        if (pcntl_wifexited($status)) {
            $exitCode = pcntl_wexitstatus($status);
            echo "PID {$pid} শেষ হলো, exit code: {$exitCode}\n";
        }
    }
}

function processTransaction(array $txn): void
{
    // ট্রানজ্যাকশন ভেরিফিকেশন লজিক
    usleep(100_000); // 100ms সিমুলেশন
    echo "ট্রানজ্যাকশন #{$txn['id']} ভেরিফাইড (PID: " . getmypid() . ")\n";
}

// ব্যবহার
$transactions = [];
for ($i = 1; $i <= 100; $i++) {
    $transactions[] = ['id' => $i, 'amount' => rand(100, 50000)];
}

processPool($transactions, poolSize: 4);
```

### বিকল্প ২: parallel এক্সটেনশন (PHP 8+)

```php
<?php
/**
 * PHP parallel এক্সটেনশন — সত্যিকারের থ্রেড পুল
 * 
 * Grameenphone এর SMS ব্রডকাস্ট সিস্টেম:
 * লক্ষ লক্ষ গ্রাহককে একসাথে SMS পাঠাতে হবে
 */
use parallel\Runtime;
use parallel\Future;
use parallel\Channel;

function smsThreadPool(array $phoneNumbers, string $message, int $poolSize = 8): array
{
    $results = [];
    $runtimes = [];
    $futures = [];

    // পুল তৈরি — $poolSize সংখ্যক Runtime (থ্রেড)
    for ($i = 0; $i < $poolSize; $i++) {
        $runtimes[] = new Runtime();
    }

    // টাস্ক বণ্টন — রাউন্ড-রবিন পদ্ধতিতে
    foreach ($phoneNumbers as $index => $phone) {
        $workerIndex = $index % $poolSize;

        $futures[] = $runtimes[$workerIndex]->run(
            function (string $phone, string $msg): array {
                // SMS পাঠানোর লজিক
                $success = sendSms($phone, $msg);
                return ['phone' => $phone, 'status' => $success ? 'sent' : 'failed'];
            },
            [$phone, $message]
        );
    }

    // সব ফলাফল সংগ্রহ
    foreach ($futures as $future) {
        $results[] = $future->value();
    }

    return $results;
}
```

### বিকল্প ৩: Symfony Process দিয়ে প্রসেস পুল

```php
<?php
/**
 * Symfony Process কম্পোনেন্ট দিয়ে প্রসেস পুল
 * 
 * Daraz এর প্রোডাক্ট ইমেজ রিসাইজ:
 * হাজার হাজার ছবি একসাথে রিসাইজ করতে হবে
 */
use Symfony\Component\Process\Process;

class ImageProcessPool
{
    private int $maxWorkers;
    /** @var Process[] */
    private array $running = [];

    public function __construct(int $maxWorkers = 4)
    {
        $this->maxWorkers = $maxWorkers;
    }

    public function processImages(array $imagePaths): void
    {
        $queue = $imagePaths;

        while (!empty($queue) || !empty($this->running)) {
            // ফ্রি স্লট থাকলে নতুন প্রসেস চালু
            while (count($this->running) < $this->maxWorkers && !empty($queue)) {
                $image = array_shift($queue);
                $process = new Process([
                    'php', 'resize-worker.php', $image
                ]);
                $process->start();
                $this->running[$image] = $process;
                echo "🖼️ রিসাইজ শুরু: {$image}\n";
            }

            // চলমান প্রসেস চেক
            foreach ($this->running as $image => $process) {
                if (!$process->isRunning()) {
                    if ($process->isSuccessful()) {
                        echo "✅ সম্পন্ন: {$image}\n";
                    } else {
                        echo "❌ ব্যর্থ: {$image}\n";
                    }
                    unset($this->running[$image]);
                }
            }

            usleep(50_000); // 50ms পর আবার চেক
        }
    }
}

// ব্যবহার
$pool = new ImageProcessPool(maxWorkers: 6);
$pool->processImages(glob('/uploads/products/*.jpg'));
```

---

## ⚡ Node.js worker_threads পুল

Node.js এ `worker_threads` মডিউল দিয়ে সত্যিকারের মাল্টি-থ্রেডেড কাজ করা যায়।
`piscina` এবং `workerpool` লাইব্রেরি দিয়ে সহজেই থ্রেড পুল বানানো যায়।

### piscina দিয়ে থ্রেড পুল

```javascript
/**
 * piscina — হাই-পারফরম্যান্স থ্রেড পুল
 * 
 * Pathao Food এর অর্ডার প্রসেসিং:
 * প্রতিটি অর্ডারের দূরত্ব, দাম, ডেলিভারি সময় হিসাব করা
 */
const Piscina = require('piscina');

// পুল কনফিগারেশন
const pool = new Piscina({
    filename: './order-worker.js',
    maxThreads: 4,         // সর্বোচ্চ 4 থ্রেড
    minThreads: 2,         // সর্বনিম্ন 2 থ্রেড সবসময় রেডি
    idleTimeout: 30_000,   // 30s পর আইডল থ্রেড বন্ধ
    maxQueue: 1000,        // সর্বোচ্চ 1000 টাস্ক কিউতে থাকবে
});

async function processOrders(orders) {
    console.log(`📦 ${orders.length} টি অর্ডার প্রসেস হচ্ছে...`);
    console.log(`🔧 ওয়ার্কার থ্রেড: ${pool.threads.length}`);

    // সব অর্ডার একসাথে সাবমিট
    const results = await Promise.all(
        orders.map(order => pool.run(order))
    );

    console.log(`✅ সব অর্ডার প্রসেস সম্পন্ন!`);
    return results;
}

// ব্যবহার
const orders = [
    { id: 1, restaurant: 'Star Kabab', customer: 'Dhanmondi' },
    { id: 2, restaurant: 'Sulatan\'s Dine', customer: 'Gulshan' },
    { id: 3, restaurant: 'Chillox', customer: 'Mirpur' },
    { id: 4, restaurant: 'Pizza Hut', customer: 'Uttara' },
    { id: 5, restaurant: 'Madchef', customer: 'Banani' },
];

processOrders(orders).then(results => {
    results.forEach(r => {
        console.log(`অর্ডার #${r.id}: ${r.distance}km, ${r.price}৳, ${r.eta} মিনিট`);
    });
});
```

### order-worker.js (ওয়ার্কার ফাইল)

```javascript
/**
 * প্রতিটি ওয়ার্কার থ্রেডে এই কোড চলে
 * মেইন থ্রেড থেকে আলাদা — CPU-intensive কাজ করতে পারে
 */
module.exports = async (order) => {
    // ভারী হিসাব — মেইন থ্রেডকে ব্লক করবে না
    const distance = calculateDistance(
        order.restaurant,
        order.customer
    );
    const price = calculatePrice(distance, order);
    const eta = estimateDeliveryTime(distance);

    return {
        id: order.id,
        distance: distance.toFixed(1),
        price: Math.round(price),
        eta: Math.round(eta),
    };
};

function calculateDistance(from, to) {
    // জটিল দূরত্ব হিসাব (Haversine formula ইত্যাদি)
    let sum = 0;
    for (let i = 0; i < 1_000_000; i++) {
        sum += Math.sqrt(i) * Math.sin(i);
    }
    return 2 + Math.random() * 15;
}

function calculatePrice(distance, order) {
    return 50 + distance * 12; // বেস ৫০৳ + প্রতি km ১২৳
}

function estimateDeliveryTime(distance) {
    return 15 + distance * 3; // বেস ১৫ মিনিট + প্রতি km ৩ মিনিট
}
```

### workerpool দিয়ে থ্রেড পুল

```javascript
/**
 * workerpool — সহজ API এর থ্রেড পুল
 * 
 * bKash ট্রানজ্যাকশন রিপোর্ট জেনারেশন
 */
const workerpool = require('workerpool');

// পুল তৈরি
const pool = workerpool.pool('./report-worker.js', {
    minWorkers: 2,
    maxWorkers: 4,
    workerType: 'thread',
});

async function generateMonthlyReports(merchants) {
    const promises = merchants.map(merchant =>
        pool.exec('generateReport', [merchant])
            .then(report => {
                console.log(`✅ ${merchant.name} রিপোর্ট তৈরি`);
                return report;
            })
            .catch(err => {
                console.error(`❌ ${merchant.name} ব্যর্থ: ${err.message}`);
                return null;
            })
    );

    const reports = await Promise.all(promises);

    // পুল বন্ধ
    await pool.terminate();

    return reports.filter(Boolean);
}
```

### ❌ Bad: থ্রেড পুল ছাড়া

```javascript
// ❌ প্রতিটি রিকোয়েস্টে নতুন Worker তৈরি — ব্যয়বহুল!
const { Worker } = require('worker_threads');

app.post('/process', async (req, res) => {
    // প্রতিবার নতুন Worker = নতুন থ্রেড তৈরি ও ধ্বংস
    const worker = new Worker('./heavy-task.js', {
        workerData: req.body
    });

    worker.on('message', result => res.json(result));
    worker.on('error', err => res.status(500).json({ error: err.message }));
});

// সমস্যা:
// 1. প্রতিটি রিকোয়েস্টে থ্রেড তৈরি/ধ্বংসের ওভারহেড
// 2. কোনো সীমা নেই — 10K রিকোয়েস্ট = 10K থ্রেড!
// 3. মেমোরি দ্রুত শেষ হয়ে যাবে
```

### ✅ Good: থ্রেড পুল সহ

```javascript
// ✅ আগে থেকে তৈরি পুল — দ্রুত ও নিরাপদ
const Piscina = require('piscina');
const pool = new Piscina({
    filename: './heavy-task.js',
    maxThreads: 4,
    maxQueue: 500, // সর্বোচ্চ 500 টাস্ক কিউতে
});

app.post('/process', async (req, res) => {
    try {
        // পুলে সাবমিট — ফ্রি থ্রেড পেলেই রান হবে
        const result = await pool.run(req.body);
        res.json(result);
    } catch (err) {
        if (err.message === 'Task queue is full') {
            res.status(503).json({ error: 'সার্ভার ব্যস্ত, পরে চেষ্টা করুন' });
        } else {
            res.status(500).json({ error: err.message });
        }
    }
});

// সুবিধা:
// 1. থ্রেড reuse — তৈরি/ধ্বংসের ওভারহেড নেই
// 2. সর্বোচ্চ 4 থ্রেড — রিসোর্স নিয়ন্ত্রিত
// 3. কিউ ফুল হলে 503 রিটার্ন — গ্রেসফুল ডিগ্রেডেশন
```

---

## 🔗 কানেকশন পুল vs থ্রেড পুল

দুটি আলাদা কিন্তু সম্পর্কিত ধারণা। একটি ওয়েব অ্যাপ্লিকেশনে দুটোই একসাথে কাজ করে।

### পার্থক্য

```
╔══════════════════╦══════════════════════╦══════════════════════╗
║                  ║    থ্রেড পুল          ║    কানেকশন পুল       ║
╠══════════════════╬══════════════════════╬══════════════════════╣
║ কী পুল করা হয়?  ║ ওয়ার্কার থ্রেড       ║ DB/Redis কানেকশন    ║
║ উদ্দেশ্য         ║ CPU কাজ সমান্তরালে   ║ কানেকশন reuse       ║
║ ব্যবহার          ║ ভারী গণনা, I/O       ║ ডেটাবেস কুয়েরি      ║
║ সীমা কে ঠিক করে ║ CPU কোর সংখ্যা      ║ DB max connections   ║
║ ওভারহেড         ║ মেমোরি, context switch║ TCP handshake, auth  ║
╚══════════════════╩══════════════════════╩══════════════════════╝
```

### Daraz ব্যাকএন্ডে দুটি একসাথে কাজ করে

```
                    Daraz ব্যাকএন্ড আর্কিটেকচার
                    ═══════════════════════════

  ক্লায়েন্ট (অ্যাপ/ওয়েব)
       │
       ▼
  ┌──────────────────────────────────────────────────────┐
  │                    Load Balancer                      │
  └──────────────────┬───────────────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────────────────┐
  │           অ্যাপ্লিকেশন সার্ভার (Node.js)             │
  │                                                      │
  │  ┌──────────────────────────────────────────────┐    │
  │  │          থ্রেড পুল (4 Workers)                │    │
  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐│    │
  │  │  │Worker 1│ │Worker 2│ │Worker 3│ │Worker 4││    │
  │  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘│    │
  │  │      │          │          │          │      │    │
  │  └──────┼──────────┼──────────┼──────────┼──────┘    │
  │         │          │          │          │           │
  │  ┌──────┼──────────┼──────────┼──────────┼──────┐    │
  │  │      ▼          ▼          ▼          ▼      │    │
  │  │     DB কানেকশন পুল (10 connections)          │    │
  │  │  ┌────┐┌────┐┌────┐┌────┐┌────┐ ... ┌────┐ │    │
  │  │  │ C1 ││ C2 ││ C3 ││ C4 ││ C5 │     │C10 │ │    │
  │  │  └──┬─┘└──┬─┘└──┬─┘└──┬─┘└──┬─┘     └──┬─┘ │    │
  │  └─────┼─────┼─────┼─────┼─────┼──────────┼───┘    │
  └────────┼─────┼─────┼─────┼─────┼──────────┼────────┘
           │     │     │     │     │          │
           ▼     ▼     ▼     ▼     ▼          ▼
  ┌──────────────────────────────────────────────────────┐
  │               MySQL / PostgreSQL সার্ভার              │
  └──────────────────────────────────────────────────────┘

  ফ্লো:
  1. ক্লায়েন্ট রিকোয়েস্ট আসে
  2. থ্রেড পুল থেকে ফ্রি Worker কাজ নেয়
  3. Worker এর DB দরকার হলে কানেকশন পুল থেকে কানেকশন নেয়
  4. কুয়েরি শেষে কানেকশন পুলে ফেরত দেয়
  5. কাজ শেষে Worker পুলে ফিরে আসে
```

---

## ⚠️ ব্যাকপ্রেশার হ্যান্ডলিং

যখন টাস্ক আসার গতি ওয়ার্কারদের কাজ শেষ করার গতির চেয়ে বেশি হয়, তখন ব্যাকপ্রেশার
তৈরি হয়। এটি হ্যান্ডেল না করলে মেমোরি ফুরিয়ে যাবে বা সিস্টেম ক্র্যাশ করবে।

### ব্যাকপ্রেশার কখন হয়?

```
সাধারণ অবস্থা:
  টাস্ক আসে    ████████░░░░░░░░  (8/sec)
  প্রসেস হয়     ██████████░░░░░░  (10/sec)
  ✅ কিউ খালি থাকে — সব ঠিক আছে

ব্যাকপ্রেশার:
  টাস্ক আসে    ████████████████  (16/sec)
  প্রসেস হয়     ██████████░░░░░░  (10/sec)
  ❌ কিউ ভরে যাচ্ছে — বিপদ!

  সময়   কিউ সাইজ
  0s     ░░░░░░░░░░  0
  1s     ██░░░░░░░░  6      (+16 আসলো, -10 প্রসেস হলো)
  2s     ████░░░░░░  12
  3s     ██████░░░░  18
  4s     ████████░░  24
  5s     ██████████  30     কিউ ফুল! 💥
```

### স্ট্র্যাটেজি ১: বাউন্ডেড কিউ (Bounded Queue)

```javascript
/**
 * বাউন্ডেড কিউ — কিউ ফুল হলে নতুন টাস্ক প্রত্যাখ্যান
 * Nagad এর পেমেন্ট প্রসেসিং
 */
class BoundedThreadPool {
    constructor(workerFile, { maxThreads = 4, maxQueueSize = 100 } = {}) {
        this.pool = new Piscina({
            filename: workerFile,
            maxThreads,
            maxQueue: maxQueueSize,
        });
    }

    async submit(task) {
        // piscina স্বয়ংক্রিয়ভাবে কিউ ফুল হলে Error দেয়
        try {
            return await this.pool.run(task);
        } catch (err) {
            if (err.code === 'ERR_PISCINA_QUEUE_FULL') {
                console.warn('⚠️ কিউ ভর্তি! টাস্ক প্রত্যাখ্যান হলো।');
                throw new Error('সার্ভার ব্যস্ত, পরে চেষ্টা করুন');
            }
            throw err;
        }
    }
}
```

### স্ট্র্যাটেজি ২: রিজেক্ট পলিসি (Reject Policy)

```php
<?php
/**
 * বিভিন্ন রিজেক্ট পলিসি — Grameenphone এর API গেটওয়ে
 */
class ThreadPoolWithPolicy
{
    private int $maxQueue;
    private array $queue = [];
    private string $rejectPolicy;

    public function __construct(
        int $maxQueue = 100,
        string $rejectPolicy = 'abort'  // abort | caller_runs | discard_oldest
    ) {
        $this->maxQueue = $maxQueue;
        $this->rejectPolicy = $rejectPolicy;
    }

    public function submit(callable $task): void
    {
        if (count($this->queue) >= $this->maxQueue) {
            $this->handleRejection($task);
            return;
        }
        $this->queue[] = $task;
    }

    private function handleRejection(callable $task): void
    {
        switch ($this->rejectPolicy) {
            case 'abort':
                // টাস্ক বাতিল, এক্সেপশন থ্রো
                throw new \OverflowException('কিউ ফুল! টাস্ক প্রত্যাখ্যাত।');

            case 'caller_runs':
                // কলার থ্রেডেই কাজটি চালাও (ধীর করে দেয়, কিন্তু টাস্ক হারায় না)
                echo "⚠️ কিউ ফুল, কলার থ্রেডে চালানো হচ্ছে...\n";
                $task();
                break;

            case 'discard_oldest':
                // সবচেয়ে পুরাতন টাস্ক বাদ দিয়ে নতুনটি নাও
                array_shift($this->queue);
                $this->queue[] = $task;
                echo "⚠️ পুরাতন টাস্ক বাদ দেওয়া হলো।\n";
                break;
        }
    }
}
```

### স্ট্র্যাটেজি ৩: সার্কিট ব্রেকার সহ থ্রেড পুল

```javascript
/**
 * সার্কিট ব্রেকার — বারবার ব্যর্থ হলে পুরো পুল সাময়িক বন্ধ
 * bKash পেমেন্ট গেটওয়ে — ব্যাংক API ডাউন থাকলে অহেতুক চেষ্টা বন্ধ
 */
class CircuitBreakerPool {
    constructor(options = {}) {
        this.pool = new Piscina(options);
        this.failureCount = 0;
        this.failureThreshold = 5;   // 5 বার ব্যর্থ হলে সার্কিট ওপেন
        this.resetTimeout = 30_000;   // 30 সেকেন্ড পর আবার চেষ্টা
        this.state = 'CLOSED';        // CLOSED | OPEN | HALF_OPEN
        this.lastFailureTime = null;
    }

    async run(task) {
        // সার্কিট OPEN থাকলে সরাসরি প্রত্যাখ্যান
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime > this.resetTimeout) {
                this.state = 'HALF_OPEN';
                console.log('🔄 সার্কিট HALF_OPEN — পরীক্ষামূলক চেষ্টা...');
            } else {
                throw new Error('সার্কিট ওপেন! সেবা সাময়িকভাবে বন্ধ।');
            }
        }

        try {
            const result = await this.pool.run(task);
            // সফল হলে সার্কিট বন্ধ করো
            if (this.state === 'HALF_OPEN') {
                this.state = 'CLOSED';
                this.failureCount = 0;
                console.log('✅ সার্কিট CLOSED — সেবা পুনরুদ্ধার!');
            }
            return result;
        } catch (err) {
            this.failureCount++;
            this.lastFailureTime = Date.now();

            if (this.failureCount >= this.failureThreshold) {
                this.state = 'OPEN';
                console.error('🔴 সার্কিট OPEN! অনেক বার ব্যর্থ হয়েছে।');
            }
            throw err;
        }
    }
}
```

---

## 📖 সাধারণ প্যাটার্ন ও এন্টি-প্যাটার্ন

### ❌ এন্টি-প্যাটার্ন ১: আনবাউন্ডেড পুল

```javascript
// ❌ কোনো সীমা নেই — সার্ভার ধ্বংস হবে!
const pool = new Piscina({
    filename: './worker.js',
    // maxThreads সেট না করলে ডিফল্ট = CPU কোর × 1.5
    // maxQueue সেট না করলে Infinity!
    maxQueue: Infinity, // 💣 মেমোরি বোমা!
});

// ১ লক্ষ টাস্ক একসাথে সাবমিট
for (let i = 0; i < 100_000; i++) {
    pool.run({ id: i }); // সব কিউতে জমা হবে — RAM শেষ!
}
```

### ✅ প্যাটার্ন ১: বাউন্ডেড পুল + ব্যাচিং

```javascript
// ✅ সীমিত কিউ + ব্যাচে প্রসেস
const pool = new Piscina({
    filename: './worker.js',
    maxThreads: 4,
    maxQueue: 100,
});

async function processBatch(allTasks, batchSize = 50) {
    const results = [];

    for (let i = 0; i < allTasks.length; i += batchSize) {
        const batch = allTasks.slice(i, i + batchSize);
        const batchResults = await Promise.all(
            batch.map(task => pool.run(task))
        );
        results.push(...batchResults);
        console.log(`✅ ব্যাচ ${Math.floor(i / batchSize) + 1} সম্পন্ন`);
    }

    return results;
}
```

### ❌ এন্টি-প্যাটার্ন ২: পুল শাটডাউন ভুলে যাওয়া

```php
<?php
// ❌ প্রসেস পুল শাটডাউন না করা — জম্বি প্রসেস!
function bad_processPool(array $tasks): void
{
    $pids = [];
    foreach ($tasks as $task) {
        $pid = pcntl_fork();
        if ($pid === 0) {
            processTask($task);
            exit(0);
        }
        $pids[] = $pid;
    }
    // ⚠️ pcntl_waitpid() কল নেই!
    // চাইল্ড প্রসেস জম্বি হয়ে থাকবে!
}
```

### ✅ প্যাটার্ন ২: গ্রেসফুল শাটডাউন

```javascript
// ✅ গ্রেসফুল শাটডাউন — চলমান কাজ শেষ করে তারপর বন্ধ
const pool = new Piscina({ filename: './worker.js', maxThreads: 4 });

// SIGTERM সিগন্যাল পেলে
process.on('SIGTERM', async () => {
    console.log('⏳ শাটডাউন শুরু — চলমান কাজ শেষ হচ্ছে...');

    // নতুন টাস্ক বন্ধ, চলমান শেষ হওয়ার অপেক্ষা
    await pool.destroy();

    console.log('✅ সব কাজ শেষ, সার্ভার বন্ধ হচ্ছে।');
    process.exit(0);
});
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন থ্রেড পুল ব্যবহার করবেন

```
✅ ব্যবহার করুন যখন:
┌──────────────────────────────────────────────────────┐
│                                                      │
│  1. CPU-ভারী কাজ আছে                                │
│     • ইমেজ প্রসেসিং (Daraz প্রোডাক্ট ছবি রিসাইজ)   │
│     • ভিডিও এনকোডিং                                 │
│     • ক্রিপ্টোগ্রাফি (bKash ট্রানজ্যাকশন সাইনিং)    │
│     • ডেটা কমপ্রেশন                                  │
│                                                      │
│  2. অনেকগুলো স্বাধীন কাজ একসাথে করতে হবে            │
│     • বাল্ক ইমেইল/SMS পাঠানো (GP ক্যাম্পেইন)        │
│     • একাধিক API কল (Pathao রাইড ম্যাচিং)           │
│     • ব্যাচ ডেটা প্রসেসিং                            │
│                                                      │
│  3. রিসোর্স ব্যবহার নিয়ন্ত্রণ করতে চান              │
│     • সার্ভারে সর্বোচ্চ কতগুলো কাজ চলবে সেটা ঠিক    │
│     • মেমোরি/CPU সীমার মধ্যে থাকা                    │
│                                                      │
│  4. থ্রেড তৈরি/ধ্বংসের খরচ এড়াতে চান               │
│     • হাই-ট্রাফিক API সার্ভার                        │
│     • রিয়েল-টাইম ডেটা প্রসেসিং                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### ❌ কখন থ্রেড পুল ব্যবহার করবেন না

```
❌ ব্যবহার করবেন না যখন:
┌──────────────────────────────────────────────────────┐
│                                                      │
│  1. কাজ খুব সরল ও দ্রুত শেষ হয়                     │
│     • সাধারণ CRUD অপারেশন                            │
│     • ছোট JSON পার্সিং                               │
│     → ওভারহেড বেশি হবে, লাভ কম                      │
│                                                      │
│  2. কাজ একটি আরেকটির উপর নির্ভরশীল                 │
│     • A → B → C ক্রমানুসারে চলতেই হবে               │
│     → সমান্তরালে চালানোর সুযোগ নেই                  │
│                                                      │
│  3. শেয়ার্ড স্টেট বেশি আছে                          │
│     • একই ডেটায় অনেকে লিখবে                        │
│     → Lock contention এ পারফরম্যান্স কমবে            │
│                                                      │
│  4. সার্ভারলেস এনভায়রনমেন্ট                         │
│     • AWS Lambda, Vercel Functions                    │
│     → ফাংশন শর্ট-লিভড, পুল maintain অর্থহীন          │
│                                                      │
│  5. PHP তে সরাসরি (সাধারণ ওয়েব রিকোয়েস্ট)          │
│     • PHP-FPM নিজেই প্রসেস পুল হিসেবে কাজ করে       │
│     → আলাদা থ্রেড পুল দরকার নেই                     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 📖 মনে রাখার বিষয়

```
╔══════════════════════════════════════════════════════════════╗
║                থ্রেড পুল চেকলিস্ট                           ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  □ পুল সাইজ সঠিকভাবে নির্ধারণ করুন                         ║
║    • CPU-bound → কোর সংখ্যা                                ║
║    • I/O-bound → কোর × (1 + wait/compute)                   ║
║                                                              ║
║  □ কিউ সাইজে সীমা রাখুন (Bounded Queue)                    ║
║    • Infinity কিউ = মেমোরি বোমা 💣                          ║
║                                                              ║
║  □ রিজেক্ট পলিসি ঠিক করুন                                  ║
║    • abort / caller_runs / discard_oldest                    ║
║                                                              ║
║  □ গ্রেসফুল শাটডাউন নিশ্চিত করুন                            ║
║    • চলমান কাজ শেষ হতে দিন, তারপর বন্ধ করুন               ║
║                                                              ║
║  □ মনিটরিং যোগ করুন                                        ║
║    • কিউ সাইজ, অ্যাক্টিভ থ্রেড, wait time ট্র্যাক করুন     ║
║                                                              ║
║  □ হেলথ চেক রাখুন                                          ║
║    • স্ট্যাক ওয়ার্কার চেক, মেমোরি লিক ডিটেকশন              ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

```
থ্রেড পুল
  │
  ├── 🔗 কানেকশন পুল (connection-pool.md)
  │     └── ডেটাবেস কানেকশনের জন্য একই ধারণা
  │
  ├── 🔗 অবজেক্ট পুল (Object Pool)
  │     └── যেকোনো ব্যয়বহুল অবজেক্ট reuse
  │
  ├── 🔗 প্রডিউসার-কনজিউমার (Producer-Consumer)
  │     └── টাস্ক কিউ + ওয়ার্কার = প্রডিউসার-কনজিউমার প্যাটার্ন
  │
  ├── 🔗 সার্কিট ব্রেকার (Circuit Breaker)
  │     └── ব্যাকপ্রেশার হ্যান্ডলিংয়ে ব্যবহার
  │
  └── 🔗 বাল্কহেড (Bulkhead)
        └── আলাদা পুল আলাদা সার্ভিসের জন্য — এক সার্ভিস ক্র্যাশে অন্যটি প্রভাবিত না হয়
```

---

> **💡 মনে রাখুন:** থ্রেড পুল হলো "একবার তৈরি করো, বারবার ব্যবহার করো" — ঠিক bKash এর
> কাস্টমার সার্ভিস এজেন্টদের মতো। সীমিত রিসোর্স দিয়ে সর্বোচ্চ কাজ আদায় করাই এর লক্ষ্য!
