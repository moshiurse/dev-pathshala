# 🔒 ডেডলক (Deadlocks)

## ধারণা

ডেডলক হলো এমন একটি পরিস্থিতি যেখানে দুই বা ততোধিক প্রসেস একে অপরের দখলে থাকা রিসোর্সের জন্য অপেক্ষা করে এবং কেউই আগাতে পারে না। প্রতিটি প্রসেস এমন একটি রিসোর্স ধরে রাখে যা অন্য প্রসেসের প্রয়োজন, ফলে একটি চক্রাকার অপেক্ষা তৈরি হয়।

### ডেডলকের চারটি শর্ত (Coffman Conditions)

ডেডলক ঘটতে হলে চারটি শর্ত একসাথে পূরণ হতে হবে:

1. **Mutual Exclusion**: রিসোর্স একবারে শুধু একটি প্রসেস ব্যবহার করতে পারে
2. **Hold and Wait**: একটি প্রসেস রিসোর্স ধরে রেখে অন্য রিসোর্সের জন্য অপেক্ষা করে
3. **No Preemption**: জোর করে রিসোর্স কেড়ে নেওয়া যায় না
4. **Circular Wait**: প্রসেসগুলো চক্রাকারে একে অপরের রিসোর্সের জন্য অপেক্ষা করে

---

## 🏠 বাস্তব জীবনের উদাহরণ

### সরু রাস্তার উদাহরণ

একটি সরু রাস্তা যেখানে একবারে একটি গাড়ি যেতে পারে। দুই দিক থেকে দুটি গাড়ি এসে মুখোমুখি দাঁড়িয়ে গেছে। কেউ পিছু হটতে রাজি নয়। ফলে কেউই আগাতে পারছে না — এটাই ডেডলক।

### ডাইনিং ফিলোসফারস সমস্যা

৫ জন দার্শনিক একটি গোল টেবিলে বসে আছেন। প্রত্যেকের মাঝে একটি করে কাঁটাচামচ। খেতে হলে দুটি কাঁটাচামচ লাগবে। সবাই যদি একসাথে বাম পাশের কাঁটাচামচ তুলে নেয়, তাহলে কেউই ডান পাশেরটি পাবে না — ডেডলক!

---

## 💻 PHP কোড উদাহরণ

### ডেডলক সমস্যা (ডাটাবেজ)

```php
<?php
// ❌ ডেডলক পরিস্থিতি — দুটি ট্রানজ্যাকশন বিপরীত ক্রমে লক করছে

// ট্রানজ্যাকশন ১ (প্রসেস A)
function transferMoneyA(PDO $pdo): void
{
    $pdo->beginTransaction();
    // প্রথমে Account 1 লক করে
    $pdo->exec("SELECT * FROM accounts WHERE id = 1 FOR UPDATE");
    sleep(1); // কিছু কাজ
    // তারপর Account 2 লক করতে চায় — কিন্তু B ধরে রেখেছে!
    $pdo->exec("SELECT * FROM accounts WHERE id = 2 FOR UPDATE");
    $pdo->exec("UPDATE accounts SET balance = balance - 500 WHERE id = 1");
    $pdo->exec("UPDATE accounts SET balance = balance + 500 WHERE id = 2");
    $pdo->commit();
}

// ট্রানজ্যাকশন ২ (প্রসেস B)
function transferMoneyB(PDO $pdo): void
{
    $pdo->beginTransaction();
    // প্রথমে Account 2 লক করে
    $pdo->exec("SELECT * FROM accounts WHERE id = 2 FOR UPDATE");
    sleep(1);
    // তারপর Account 1 লক করতে চায় — কিন্তু A ধরে রেখেছে!
    $pdo->exec("SELECT * FROM accounts WHERE id = 1 FOR UPDATE");
    $pdo->exec("UPDATE accounts SET balance = balance - 300 WHERE id = 2");
    $pdo->exec("UPDATE accounts SET balance = balance + 300 WHERE id = 1");
    $pdo->commit();
}
```

### সমাধান: সুশৃঙ্খল লকিং ক্রম (Lock Ordering)

```php
<?php
// ✅ সবসময় ছোট ID আগে লক করে — ডেডলক এড়ানো যায়
function safeTransfer(PDO $pdo, int $fromId, int $toId, float $amount): void
{
    $pdo->beginTransaction();

    try {
        // সবসময় ছোট ID আগে লক করো
        $firstId = min($fromId, $toId);
        $secondId = max($fromId, $toId);

        $pdo->exec("SELECT * FROM accounts WHERE id = {$firstId} FOR UPDATE");
        $pdo->exec("SELECT * FROM accounts WHERE id = {$secondId} FOR UPDATE");

        $pdo->exec("UPDATE accounts SET balance = balance - {$amount} WHERE id = {$fromId}");
        $pdo->exec("UPDATE accounts SET balance = balance + {$amount} WHERE id = {$toId}");

        $pdo->commit();
        echo "ট্রান্সফার সফল: {$fromId} → {$toId}, পরিমাণ: {$amount}\n";
    } catch (PDOException $e) {
        $pdo->rollBack();
        echo "ট্রান্সফার ব্যর্থ: " . $e->getMessage() . "\n";
    }
}
```

### টাইমআউট ভিত্তিক ডেডলক প্রতিরোধ

```php
<?php
// ✅ টাইমআউট দিয়ে ডেডলক ডিটেকশন
function transferWithTimeout(PDO $pdo, int $fromId, int $toId, float $amount): void
{
    $pdo->exec("SET innodb_lock_wait_timeout = 5"); // ৫ সেকেন্ড টাইমআউট
    $maxRetries = 3;

    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            $pdo->beginTransaction();
            $pdo->exec("SELECT * FROM accounts WHERE id = {$fromId} FOR UPDATE");
            $pdo->exec("SELECT * FROM accounts WHERE id = {$toId} FOR UPDATE");
            $pdo->exec("UPDATE accounts SET balance = balance - {$amount} WHERE id = {$fromId}");
            $pdo->exec("UPDATE accounts SET balance = balance + {$amount} WHERE id = {$toId}");
            $pdo->commit();
            echo "সফল (প্রচেষ্টা #{$attempt})\n";
            return;
        } catch (PDOException $e) {
            $pdo->rollBack();
            echo "প্রচেষ্টা #{$attempt} ব্যর্থ, পুনরায় চেষ্টা...\n";
            usleep(rand(100000, 500000)); // র‍্যান্ডম ব্যাকঅফ
        }
    }
    echo "সর্বোচ্চ প্রচেষ্টা শেষ — ট্রান্সফার ব্যর্থ\n";
}
```

---

## 💻 JavaScript কোড উদাহরণ

### ডেডলক সিমুলেশন (Promise ভিত্তিক)

```javascript
// ❌ ডেডলক সিমুলেশন — দুটি ফাংশন বিপরীত ক্রমে লক চায়
class Resource {
    constructor(name) {
        this.name = name;
        this._locked = false;
        this._queue = [];
    }

    async acquire() {
        return new Promise(resolve => {
            if (!this._locked) {
                this._locked = true;
                resolve();
            } else {
                this._queue.push(resolve);
            }
        });
    }

    release() {
        if (this._queue.length > 0) {
            this._queue.shift()();
        } else {
            this._locked = false;
        }
    }
}

const resourceA = new Resource('A');
const resourceB = new Resource('B');

// ডেডলক!
async function process1() {
    await resourceA.acquire();
    console.log('Process 1: A পেয়েছে');
    await new Promise(r => setTimeout(r, 100));
    await resourceB.acquire(); // B এর জন্য অপেক্ষা — কিন্তু Process 2 ধরে রেখেছে
    console.log('Process 1: B পেয়েছে');
    resourceB.release();
    resourceA.release();
}

async function process2() {
    await resourceB.acquire();
    console.log('Process 2: B পেয়েছে');
    await new Promise(r => setTimeout(r, 100));
    await resourceA.acquire(); // A এর জন্য অপেক্ষা — কিন্তু Process 1 ধরে রেখেছে
    console.log('Process 2: A পেয়েছে');
    resourceA.release();
    resourceB.release();
}
```

### সমাধান: টাইমআউট সহ লক অর্জন

```javascript
// ✅ টাইমআউট দিয়ে ডেডলক প্রতিরোধ
class SafeResource {
    constructor(name) {
        this.name = name;
        this._locked = false;
        this._queue = [];
    }

    async acquire(timeoutMs = 3000) {
        return new Promise((resolve, reject) => {
            const timer = setTimeout(() => {
                const idx = this._queue.indexOf(cb);
                if (idx > -1) this._queue.splice(idx, 1);
                reject(new Error(`টাইমআউট: ${this.name} লক পাওয়া যায়নি`));
            }, timeoutMs);

            const cb = () => {
                clearTimeout(timer);
                resolve();
            };

            if (!this._locked) {
                this._locked = true;
                clearTimeout(timer);
                resolve();
            } else {
                this._queue.push(cb);
            }
        });
    }

    release() {
        if (this._queue.length > 0) {
            this._queue.shift()();
        } else {
            this._locked = false;
        }
    }
}

async function safeProcess(id, res1, res2) {
    try {
        await res1.acquire(2000);
        console.log(`Process ${id}: ${res1.name} পেয়েছে`);
        await new Promise(r => setTimeout(r, 100));
        await res2.acquire(2000);
        console.log(`Process ${id}: ${res2.name} পেয়েছে`);
        res2.release();
        res1.release();
    } catch (err) {
        console.log(`Process ${id}: ${err.message} — রিট্রাই করা হবে`);
        res1.release();
    }
}
```

---

## 🛡️ ডেডলক প্রতিরোধ কৌশল

| কৌশল | বিবরণ |
|-------|--------|
| **Lock Ordering** | সবসময় একই ক্রমে রিসোর্স লক করুন |
| **Timeout** | নির্দিষ্ট সময়ের মধ্যে লক না পেলে ছেড়ে দিন |
| **Try Lock** | লক পাওয়া না গেলে সাথে সাথে ফিরে আসুন |
| **Deadlock Detection** | Wait-for Graph দিয়ে চক্র শনাক্ত করুন |
| **Resource Hierarchy** | রিসোর্সে ক্রমিক নম্বর দিন, ছোট থেকে বড় ক্রমে লক করুন |

---

## ✅ কখন ব্যবহার করবেন (ডেডলক প্রতিরোধ)

- ডাটাবেজ ট্রানজ্যাকশনে একাধিক টেবিল/রো লক করার সময়
- ডিস্ট্রিবিউটেড সিস্টেমে একাধিক সার্ভিসে লক নেওয়ার সময়
- ফাইল সিস্টেমে একাধিক ফাইল একসাথে অ্যাক্সেস করার সময়
- মাইক্রোসার্ভিস আর্কিটেকচারে ক্রস-সার্ভিস ট্রানজ্যাকশনে

## ❌ কখন ব্যবহার করবেন না

- সিঙ্গেল রিসোর্স অপারেশনে (ডেডলকের সম্ভাবনা নেই)
- রিড-অনলি অপারেশনে
- Stateless সিস্টেমে যেখানে শেয়ারড স্টেট নেই
- যখন ডাটাবেজ ইঞ্জিন নিজেই ডেডলক ডিটেকশন করে (যেমন InnoDB)
