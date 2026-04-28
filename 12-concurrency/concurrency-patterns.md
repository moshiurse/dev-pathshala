# ⚙️ কনকারেন্সি প্যাটার্নস (Concurrency Patterns)

## ধারণা

কনকারেন্সি প্যাটার্ন হলো সুপরিচিত সমাধান কৌশল যা একাধিক থ্রেড বা প্রসেসের সমন্বয় ও যোগাযোগ পরিচালনা করে। এগুলো সঠিকভাবে ব্যবহার করলে সিস্টেমের কার্যকারিতা, স্কেলেবিলিটি এবং নির্ভরযোগ্যতা বাড়ে।

### প্রধান প্যাটার্নসমূহ

1. **Producer-Consumer**: একদল ডেটা তৈরি করে, অন্যদল প্রসেস করে
2. **Reader-Writer**: একাধিক Reader একসাথে পড়তে পারে, Writer একচেটিয়া অ্যাক্সেস পায়
3. **Thread Pool**: পুনরায় ব্যবহারযোগ্য থ্রেডের সংগ্রহ যা কাজের সারি থেকে টাস্ক নেয়

---

## 🏠 বাস্তব জীবনের উদাহরণ

### Producer-Consumer
রেস্তোরাঁর রান্নাঘর: বাবুর্চি (Producer) খাবার তৈরি করে কাউন্টারে রাখে, ওয়েটার (Consumer) কাউন্টার থেকে নিয়ে টেবিলে পরিবেশন করে। কাউন্টার হলো Buffer/Queue।

### Reader-Writer
লাইব্রেরির বই: অনেকে একসাথে একটি বই পড়তে পারে, কিন্তু কেউ যখন বইয়ে নোট লিখছে (Writer), তখন অন্যরা পড়তে পারবে না — নাহলে অসম্পূর্ণ তথ্য দেখবে।

### Thread Pool
ব্যাংকের কাউন্টার: ১০টি কাউন্টার আছে (Thread Pool)। গ্রাহকরা (Tasks) লাইনে দাঁড়ায়। কোনো কাউন্টার খালি হলে পরবর্তী গ্রাহক সেখানে যায়। প্রতিবার নতুন কাউন্টার তৈরি করার দরকার নেই।

---

## 💻 Producer-Consumer — PHP

```php
<?php

class MessageQueue
{
    private array $queue = [];
    private int $maxSize;

    public function __construct(int $maxSize = 10)
    {
        $this->maxSize = $maxSize;
    }

    public function produce(string $item): bool
    {
        if (count($this->queue) >= $this->maxSize) {
            echo "⚠️ কিউ পূর্ণ! অপেক্ষা করুন...\n";
            return false;
        }
        $this->queue[] = $item;
        echo "📥 প্রডিউস করা হয়েছে: {$item} (কিউ সাইজ: " . count($this->queue) . ")\n";
        return true;
    }

    public function consume(): ?string
    {
        if (empty($this->queue)) {
            echo "⚠️ কিউ খালি! কোনো আইটেম নেই\n";
            return null;
        }
        $item = array_shift($this->queue);
        echo "📤 কনজিউম করা হয়েছে: {$item} (কিউ সাইজ: " . count($this->queue) . ")\n";
        return $item;
    }

    public function size(): int
    {
        return count($this->queue);
    }
}

// Redis-ভিত্তিক Producer-Consumer (প্রোডাকশন উদাহরণ)
class RedisProducerConsumer
{
    private \Redis $redis;
    private string $queueName;

    public function __construct(string $queueName = 'task_queue')
    {
        $this->redis = new \Redis();
        $this->redis->connect('127.0.0.1', 6379);
        $this->queueName = $queueName;
    }

    public function produce(array $task): void
    {
        $payload = json_encode($task);
        $this->redis->lPush($this->queueName, $payload);
        echo "📥 টাস্ক কিউতে যোগ হয়েছে: {$task['type']}\n";
    }

    public function consume(int $timeout = 30): ?array
    {
        // BRPOP ব্লকিং — নতুন আইটেম না আসা পর্যন্ত অপেক্ষা করে
        $result = $this->redis->brPop([$this->queueName], $timeout);
        if ($result) {
            $task = json_decode($result[1], true);
            echo "📤 টাস্ক প্রসেস হচ্ছে: {$task['type']}\n";
            return $task;
        }
        return null;
    }
}
```

---

## 💻 Producer-Consumer — JavaScript

```javascript
class AsyncQueue {
    constructor(maxSize = 10) {
        this.queue = [];
        this.maxSize = maxSize;
        this.waitingConsumers = [];
        this.waitingProducers = [];
    }

    async produce(item) {
        // কিউ পূর্ণ হলে অপেক্ষা
        while (this.queue.length >= this.maxSize) {
            await new Promise(resolve => this.waitingProducers.push(resolve));
        }
        this.queue.push(item);
        console.log(`📥 প্রডিউস: ${item} (সাইজ: ${this.queue.length})`);

        // অপেক্ষারত Consumer থাকলে জাগাও
        if (this.waitingConsumers.length > 0) {
            this.waitingConsumers.shift()();
        }
    }

    async consume() {
        // কিউ খালি হলে অপেক্ষা
        while (this.queue.length === 0) {
            await new Promise(resolve => this.waitingConsumers.push(resolve));
        }
        const item = this.queue.shift();
        console.log(`📤 কনজিউম: ${item} (সাইজ: ${this.queue.length})`);

        // অপেক্ষারত Producer থাকলে জাগাও
        if (this.waitingProducers.length > 0) {
            this.waitingProducers.shift()();
        }
        return item;
    }
}

// ব্যবহার
async function demo() {
    const queue = new AsyncQueue(3);

    // Producer
    const producer = async () => {
        for (let i = 1; i <= 5; i++) {
            await queue.produce(`টাস্ক-${i}`);
            await new Promise(r => setTimeout(r, 200));
        }
    };

    // Consumer
    const consumer = async () => {
        for (let i = 0; i < 5; i++) {
            const item = await queue.consume();
            await new Promise(r => setTimeout(r, 500)); // প্রসেসিং সময়
        }
    };

    await Promise.all([producer(), consumer()]);
}

demo();
```

---

## 💻 Reader-Writer — PHP

```php
<?php

class ReadWriteLock
{
    private int $readers = 0;
    private bool $writing = false;
    private string $lockFile;

    public function __construct(string $lockFile = 'rw.lock')
    {
        $this->lockFile = $lockFile;
    }

    public function readLock(): void
    {
        $fp = fopen($this->lockFile, 'c');
        flock($fp, LOCK_SH); // শেয়ারড লক — একাধিক Reader অনুমোদিত
        $this->readers++;
        echo "📖 Reader যোগ হয়েছে (মোট: {$this->readers})\n";
    }

    public function readUnlock(): void
    {
        $this->readers--;
        echo "📖 Reader চলে গেছে (মোট: {$this->readers})\n";
    }

    public function writeLock(): void
    {
        $fp = fopen($this->lockFile, 'c');
        flock($fp, LOCK_EX); // এক্সক্লুসিভ লক — একজনই Write করতে পারবে
        $this->writing = true;
        echo "✍️ Writer লক পেয়েছে\n";
    }

    public function writeUnlock(): void
    {
        $this->writing = false;
        echo "✍️ Writer লক ছেড়ে দিয়েছে\n";
    }
}

// ক্যাশে সিস্টেমে Reader-Writer প্যাটার্ন
class CachedConfig
{
    private array $config = [];
    private ReadWriteLock $lock;

    public function __construct()
    {
        $this->lock = new ReadWriteLock();
    }

    public function get(string $key): mixed
    {
        $this->lock->readLock();
        $value = $this->config[$key] ?? null;
        $this->lock->readUnlock();
        return $value;
    }

    public function set(string $key, mixed $value): void
    {
        $this->lock->writeLock();
        $this->config[$key] = $value;
        echo "✅ কনফিগ আপডেট: {$key} = {$value}\n";
        $this->lock->writeUnlock();
    }
}
```

---

## 💻 Thread Pool — JavaScript

```javascript
class WorkerPool {
    constructor(maxWorkers = 4) {
        this.maxWorkers = maxWorkers;
        this.activeWorkers = 0;
        this.taskQueue = [];
    }

    async submit(taskFn, taskName = 'unnamed') {
        return new Promise((resolve, reject) => {
            const task = { fn: taskFn, name: taskName, resolve, reject };

            if (this.activeWorkers < this.maxWorkers) {
                this._runTask(task);
            } else {
                console.log(`⏳ ${taskName} কিউতে যোগ হয়েছে (কিউ: ${this.taskQueue.length + 1})`);
                this.taskQueue.push(task);
            }
        });
    }

    async _runTask(task) {
        this.activeWorkers++;
        console.log(`▶️ ${task.name} শুরু (অ্যাক্টিভ: ${this.activeWorkers}/${this.maxWorkers})`);

        try {
            const result = await task.fn();
            task.resolve(result);
        } catch (err) {
            task.reject(err);
        } finally {
            this.activeWorkers--;
            console.log(`✅ ${task.name} সম্পন্ন (অ্যাক্টিভ: ${this.activeWorkers}/${this.maxWorkers})`);

            if (this.taskQueue.length > 0) {
                const next = this.taskQueue.shift();
                this._runTask(next);
            }
        }
    }

    get pendingCount() {
        return this.taskQueue.length;
    }
}

// ব্যবহার — ইমেজ প্রসেসিং সিমুলেশন
async function processImage(name, duration) {
    return new Promise(resolve => {
        setTimeout(() => resolve(`${name} প্রসেস সম্পন্ন`), duration);
    });
}

async function demo() {
    const pool = new WorkerPool(3); // সর্বোচ্চ ৩টি সমান্তরাল কাজ

    const tasks = [
        pool.submit(() => processImage('ছবি-১', 1000), 'ছবি-১ রিসাইজ'),
        pool.submit(() => processImage('ছবি-২', 800), 'ছবি-২ রিসাইজ'),
        pool.submit(() => processImage('ছবি-৩', 1200), 'ছবি-৩ রিসাইজ'),
        pool.submit(() => processImage('ছবি-৪', 600), 'ছবি-৪ রিসাইজ'),
        pool.submit(() => processImage('ছবি-৫', 900), 'ছবি-৫ রিসাইজ'),
    ];

    const results = await Promise.all(tasks);
    results.forEach(r => console.log(`  📷 ${r}`));
}

demo();
```

---

## 📊 প্যাটার্ন তুলনা

| প্যাটার্ন | সমস্যা | সমাধান | ব্যবহার |
|----------|--------|--------|---------|
| Producer-Consumer | ডেটা তৈরি ও প্রসেসিংয়ের গতি ভিন্ন | Buffer/Queue দিয়ে ডিকাপলিং | মেসেজ কিউ, লগ প্রসেসিং |
| Reader-Writer | একাধিক Reader ও Writer এর সংঘাত | শেয়ারড রিড, এক্সক্লুসিভ রাইট | ক্যাশে, কনফিগ ম্যানেজমেন্ট |
| Thread Pool | থ্রেড তৈরি/ধ্বংসের ওভারহেড | পুনরায় ব্যবহারযোগ্য থ্রেড পুল | ওয়েব সার্ভার, ব্যাচ প্রসেসিং |

---

## ✅ কখন ব্যবহার করবেন

- **Producer-Consumer**: ইমেইল কিউ, নোটিফিকেশন সিস্টেম, ডেটা পাইপলাইন, লগ অ্যাগ্রিগেশন
- **Reader-Writer**: ক্যাশে সিস্টেম, কনফিগারেশন ম্যানেজমেন্ট, ডাটাবেজ কানেকশন পুল
- **Thread Pool**: HTTP সার্ভার (Nginx, Apache), ইমেজ প্রসেসিং, ব্যাচ জব এক্সিকিউশন

## ❌ কখন ব্যবহার করবেন না

- **Producer-Consumer**: যখন তাৎক্ষণিক রেসপন্স দরকার (সিঙ্ক্রোনাস কল ভালো)
- **Reader-Writer**: যখন Write অপারেশন অনেক বেশি (সবসময় এক্সক্লুসিভ লক ভালো)
- **Thread Pool**: যখন প্রতিটি টাস্ক খুব দ্রুত শেষ হয় (পুলের ওভারহেড অপ্রয়োজনীয়)
- সিঙ্গেল-থ্রেডেড অ্যাপ্লিকেশনে এগুলোর প্রয়োজন নেই
