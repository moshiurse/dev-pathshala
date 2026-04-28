# 🔁 ইভেন্ট লুপ ও নন-ব্লকিং I/O

## ধারণা

ইভেন্ট লুপ হলো একটি প্রোগ্রামিং মডেল যেখানে একটি সিঙ্গেল থ্রেড ক্রমাগত ইভেন্ট/কলব্যাক চেক করে এবং এক্সিকিউট করে। এটি নন-ব্লকিং I/O অপারেশনের মাধ্যমে একসাথে অনেক কাজ পরিচালনা করতে পারে — মাল্টিপল থ্রেড ছাড়াই।

### মূল ধারণাসমূহ

- **Blocking I/O**: ডেটা পড়া/লেখা শেষ না হওয়া পর্যন্ত থ্রেড আটকে থাকে
- **Non-blocking I/O**: অপারেশন শুরু করে অন্য কাজ করে, ফলাফল প্রস্তুত হলে কলব্যাক পায়
- **Event Loop**: একটি অসীম লুপ যা ইভেন্ট কিউ থেকে কলব্যাক তুলে এক্সিকিউট করে
- **Callback Queue**: সম্পন্ন I/O অপারেশনের কলব্যাকগুলো এখানে জমা হয়

---

## 🏠 বাস্তব জীবনের উদাহরণ

### রেস্তোরাঁর ওয়েটার

**Blocking মডেল (Traditional PHP)**: প্রতিটি টেবিলের জন্য একজন ওয়েটার। অর্ডার নিয়ে রান্নাঘরে দাঁড়িয়ে থাকে খাবার তৈরি হওয়া পর্যন্ত। ১০০ টেবিল = ১০০ ওয়েটার দরকার!

**Non-blocking মডেল (Node.js/Event Loop)**: একজন ওয়েটার সব টেবিলে যায়। অর্ডার নিয়ে রান্নাঘরে দেয়, আবার অন্য টেবিলে যায়। খাবার তৈরি হলে ঘণ্টা বাজে (callback), ওয়েটার এসে পরিবেশন করে। একজন ওয়েটারই যথেষ্ট!

---

## 🔍 JavaScript ইভেন্ট লুপ — বিস্তারিত

### ইভেন্ট লুপের অংশসমূহ

```
┌─────────────────────────────────────┐
│           Call Stack                │  ← বর্তমানে যা চলছে
├─────────────────────────────────────┤
│         Web APIs / Node APIs        │  ← setTimeout, fetch, fs
├──────────┬──────────────────────────┤
│ Microtask│     Macrotask Queue      │
│  Queue   │  (setTimeout, setInterval│
│(Promise) │   I/O callbacks)         │
└──────────┴──────────────────────────┘
```

### এক্সিকিউশন অর্ডার

1. **Call Stack** — সিঙ্ক্রোনাস কোড সবার আগে চলে
2. **Microtask Queue** — `Promise.then()`, `queueMicrotask()`, `MutationObserver`
3. **Macrotask Queue** — `setTimeout`, `setInterval`, `setImmediate`, I/O callbacks

---

## 💻 JavaScript কোড উদাহরণ

### ইভেন্ট লুপ বোঝার উদাহরণ

```javascript
console.log('1️⃣ শুরু (Call Stack)');

setTimeout(() => {
    console.log('4️⃣ setTimeout (Macrotask Queue)');
}, 0);

Promise.resolve().then(() => {
    console.log('3️⃣ Promise (Microtask Queue)');
});

console.log('2️⃣ শেষ (Call Stack)');

// আউটপুট ক্রম:
// 1️⃣ শুরু (Call Stack)
// 2️⃣ শেষ (Call Stack)
// 3️⃣ Promise (Microtask Queue)
// 4️⃣ setTimeout (Macrotask Queue)
```

### নন-ব্লকিং I/O — বাস্তব উদাহরণ

```javascript
const fs = require('fs');
const http = require('http');

// ❌ ব্লকিং — সার্ভার আটকে যাবে
function blockingRead() {
    const data = fs.readFileSync('/large-file.csv', 'utf-8');
    return data.length;
}

// ✅ নন-ব্লকিং — সার্ভার অন্য রিকোয়েস্ট হ্যান্ডেল করতে পারবে
function nonBlockingRead() {
    return new Promise((resolve, reject) => {
        fs.readFile('/large-file.csv', 'utf-8', (err, data) => {
            if (err) reject(err);
            else resolve(data.length);
        });
    });
}

// নন-ব্লকিং HTTP সার্ভার
const server = http.createServer(async (req, res) => {
    if (req.url === '/data') {
        // এই I/O চলাকালে অন্য রিকোয়েস্ট আসতে পারে
        const length = await nonBlockingRead();
        res.end(`ডেটা সাইজ: ${length} বাইট`);
    }
});

server.listen(3000, () => console.log('সার্ভার চালু: পোর্ট ৩০০০'));
```

### Async/Await দিয়ে কনকারেন্সি

```javascript
// ❌ সিরিয়াল — মোট সময়: ৩ সেকেন্ড
async function fetchSequential() {
    const user = await fetchUser();       // ১ সেকেন্ড
    const orders = await fetchOrders();    // ১ সেকেন্ড
    const products = await fetchProducts(); // ১ সেকেন্ড
    return { user, orders, products };
}

// ✅ প্যারালেল — মোট সময়: ১ সেকেন্ড
async function fetchParallel() {
    const [user, orders, products] = await Promise.all([
        fetchUser(),     // সবগুলো একসাথে শুরু
        fetchOrders(),
        fetchProducts(),
    ]);
    return { user, orders, products };
}

// ✅ Promise.allSettled — কোনোটি ব্যর্থ হলেও বাকিগুলো পাওয়া যায়
async function fetchWithFallback() {
    const results = await Promise.allSettled([
        fetchUser(),
        fetchOrders(),
        fetchProducts(),
    ]);

    return results.map((r, i) => {
        if (r.status === 'fulfilled') return r.value;
        console.log(`⚠️ API ${i + 1} ব্যর্থ: ${r.reason.message}`);
        return null;
    });
}

function fetchUser() {
    return new Promise(r => setTimeout(() => r({ name: 'রহিম' }), 1000));
}
function fetchOrders() {
    return new Promise(r => setTimeout(() => r([{ id: 1 }]), 1000));
}
function fetchProducts() {
    return new Promise(r => setTimeout(() => r([{ id: 101 }]), 1000));
}
```

---

## 💻 PHP Async — ReactPHP ও Swoole

### ReactPHP (ইভেন্ট-ড্রিভেন PHP)

```php
<?php
// ReactPHP — নন-ব্লকিং HTTP সার্ভার
require 'vendor/autoload.php';

$loop = React\EventLoop\Loop::get();

// নন-ব্লকিং HTTP সার্ভার
$http = new React\Http\HttpServer(function (Psr\Http\Message\ServerRequestInterface $request) {
    $path = $request->getUri()->getPath();

    if ($path === '/api/data') {
        // নন-ব্লকিং ডিলে
        return new React\Promise\Promise(function ($resolve) {
            React\EventLoop\Loop::get()->addTimer(1.0, function () use ($resolve) {
                $resolve(new React\Http\Message\Response(
                    200,
                    ['Content-Type' => 'application/json'],
                    json_encode(['message' => 'ডেটা প্রস্তুত', 'timestamp' => time()])
                ));
            });
        });
    }

    return new React\Http\Message\Response(
        200,
        ['Content-Type' => 'text/plain'],
        "স্বাগতম ReactPHP সার্ভারে!\n"
    );
});

$socket = new React\Socket\SocketServer('0.0.0.0:8080');
$http->listen($socket);
echo "🚀 ReactPHP সার্ভার চালু: http://localhost:8080\n";
```

### Swoole (PHP Coroutine)

```php
<?php
// Swoole — কোরাউটিন-ভিত্তিক কনকারেন্সি

$server = new Swoole\Http\Server('0.0.0.0', 9501);

$server->on('request', function ($request, $response) {
    // প্রতিটি রিকোয়েস্ট আলাদা কোরাউটিনে চলে
    $result = [];

    // সমান্তরাল ডাটাবেজ কোয়েরি (নন-ব্লকিং)
    Swoole\Coroutine\run(function () use (&$result) {
        $wg = new Swoole\Coroutine\WaitGroup();

        // কোরাউটিন ১ — ইউজার ডেটা
        $wg->add();
        go(function () use ($wg, &$result) {
            Swoole\Coroutine::sleep(0.5); // DB কোয়েরি সিমুলেশন
            $result['user'] = ['name' => 'করিম', 'age' => 28];
            $wg->done();
        });

        // কোরাউটিন ২ — অর্ডার ডেটা
        $wg->add();
        go(function () use ($wg, &$result) {
            Swoole\Coroutine::sleep(0.3);
            $result['orders'] = [['id' => 1, 'total' => 5000]];
            $wg->done();
        });

        $wg->wait(); // সব কোরাউটিন শেষ হওয়ার অপেক্ষা
    });

    $response->header('Content-Type', 'application/json');
    $response->end(json_encode($result));
});

echo "🚀 Swoole সার্ভার চালু: http://localhost:9501\n";
$server->start();
```

---

## ⚖️ তুলনা: Blocking vs Non-blocking

| বৈশিষ্ট্য | Blocking (PHP-FPM) | Non-blocking (Node.js/Swoole) |
|-----------|--------------------|-----------------------------|
| মডেল | প্রতি রিকোয়েস্টে একটি প্রসেস/থ্রেড | সিঙ্গেল থ্রেড + ইভেন্ট লুপ |
| মেমরি | বেশি (প্রতি প্রসেসে ~2-10MB) | কম (শেয়ারড মেমরি) |
| কনকারেন্সি | সীমিত (প্রসেস সংখ্যা অনুযায়ী) | হাজার হাজার কানেকশন |
| CPU-bound | ভালো | ব্লক করতে পারে |
| I/O-bound | অপচয় (অপেক্ষায় থাকে) | দক্ষ (অন্য কাজ করে) |
| জটিলতা | সহজ (সিকোয়েন্শিয়াল কোড) | কলব্যাক/Promise বুঝতে হবে |

---

## ⚠️ সাধারণ ভুল

### ১. ইভেন্ট লুপ ব্লক করা

```javascript
// ❌ এটি ইভেন্ট লুপ ব্লক করবে — অন্য কোনো রিকোয়েস্ট হ্যান্ডেল হবে না
app.get('/heavy', (req, res) => {
    let sum = 0;
    for (let i = 0; i < 1_000_000_000; i++) {
        sum += i; // CPU-intensive — ইভেন্ট লুপ আটকে যাবে
    }
    res.json({ sum });
});

// ✅ ভারী কাজ Worker Thread এ পাঠান
const { Worker } = require('worker_threads');

app.get('/heavy', (req, res) => {
    const worker = new Worker('./heavy-computation.js');
    worker.on('message', (result) => res.json({ sum: result }));
    worker.on('error', (err) => res.status(500).json({ error: err.message }));
});
```

### ২. Unhandled Promise Rejection

```javascript
// ❌ এরর হ্যান্ডেল না করলে প্রসেস ক্র্যাশ করতে পারে
async function riskyOperation() {
    const data = await fetch('https://api.example.com/data');
    return data.json();
}

// ✅ সবসময় try-catch ব্যবহার করুন
async function safeOperation() {
    try {
        const data = await fetch('https://api.example.com/data');
        return await data.json();
    } catch (err) {
        console.error('API কল ব্যর্থ:', err.message);
        return null;
    }
}
```

---

## ✅ কখন ব্যবহার করবেন

- I/O-heavy অ্যাপ্লিকেশনে (API গেটওয়ে, চ্যাট সার্ভার, রিয়েল-টাইম অ্যাপ)
- হাজার হাজার সিমালটেনিয়াস কানেকশন হ্যান্ডেল করতে
- মাইক্রোসার্ভিস যেখানে বেশিরভাগ সময় অন্য সার্ভিসের রেসপন্সের অপেক্ষায় থাকে
- WebSocket ভিত্তিক রিয়েল-টাইম অ্যাপ্লিকেশনে

## ❌ কখন ব্যবহার করবেন না

- CPU-intensive কাজে (ইমেজ প্রসেসিং, ক্রিপ্টোগ্রাফি, মেশিন লার্নিং) — Worker Thread ব্যবহার করুন
- সাধারণ CRUD অ্যাপে যেখানে PHP-FPM যথেষ্ট
- যখন টিমের Non-blocking প্রোগ্রামিং অভিজ্ঞতা নেই
- ছোট প্রজেক্টে যেখানে সহজতা বেশি গুরুত্বপূর্ণ
