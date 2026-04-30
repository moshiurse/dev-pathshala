# 🔄 Async/Await — অ্যাসিঙ্ক্রোনাস প্রোগ্রামিংয়ের আধুনিক রূপ

> **"অ্যাসিঙ্ক কোডকে সিঙ্ক্রোনাস কোডের মতো পড়তে ও লিখতে পারার শিল্প"**

অ্যাসিঙ্ক্রোনাস প্রোগ্রামিং হলো আধুনিক সফটওয়্যার ইঞ্জিনিয়ারিংয়ের একটি অপরিহার্য দক্ষতা। bKash-এ পেমেন্ট
প্রসেস করা, Daraz-এ হাজার হাজার প্রোডাক্ট লোড করা, বা Pathao-তে রিয়েল-টাইম রাইডার ট্র্যাক করা — সবকিছুতেই
অ্যাসিঙ্ক প্যাটার্ন ব্যবহৃত হয়। এই ডকুমেন্টে আমরা Callbacks থেকে শুরু করে Async/Await পর্যন্ত পুরো বিবর্তন,
JavaScript এবং PHP উভয় ভাষায় গভীরভাবে শিখব।

---

## 📑 সূচিপত্র

1. [বিবর্তন: Callbacks → Promises → Async/Await](#-বিবর্তন-callbacks--promises--asyncawait)
2. [JavaScript Async/Await গভীর বিশ্লেষণ](#-javascript-asyncawait-গভীর-বিশ্লেষণ)
3. [PHP Fibers (PHP 8.1+)](#-php-fibers-php-81)
4. [PHP অ্যাসিঙ্ক ফ্রেমওয়ার্ক](#-php-অ্যাসিঙ্ক-ফ্রেমওয়ার্ক)
5. [সাধারণ ভুল ও সমস্যা](#️-সাধারণ-ভুল-ও-সমস্যা)
6. [অ্যাসিঙ্ক এরর হ্যান্ডলিং](#️-অ্যাসিঙ্ক-এরর-হ্যান্ডলিং)
7. [Promise.all, Promise.race, Promise.allSettled](#-promiseall-promiserace-promiseallsettled)
8. [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 বিবর্তন: Callbacks → Promises → Async/Await

অ্যাসিঙ্ক্রোনাস প্রোগ্রামিং তিনটি প্রধান যুগের মধ্য দিয়ে গেছে। প্রতিটি যুগ আগেরটির সমস্যা সমাধান করেছে:

```
বিবর্তনের টাইমলাইন:
══════════════════════════════════════════════════════════════════

  ২০০৯              ২০১৫              ২০১৭              আজ
   │                 │                 │                 │
   ▼                 ▼                 ▼                 ▼
┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌───────────┐
│ Callbacks │──▶│ Promises │──▶│ Async/Await  │──▶│ Top-level │
│ (Node.js │   │ (ES2015) │   │ (ES2017)     │   │   await   │
│  style)  │   │          │   │              │   │ (ES2022)  │
└──────────┘   └──────────┘   └──────────────┘   └───────────┘
     │              │                │
     │              │                │
  Callback       .then()        try/catch
   Hell          Chaining        ব্লকে
  সমস্যা         সমাধান       সুন্দর সিনট্যাক্স

══════════════════════════════════════════════════════════════════
```

### ❌ Callback Hell (Pyramid of Doom)

ধরুন Daraz-এর একটি অর্ডার ট্র্যাকিং সিস্টেম তৈরি করছেন। প্রতিটি ধাপ আগেরটির উপর নির্ভরশীল:

```javascript
// ❌ Callback Hell — "Pyramid of Doom"
// Daraz অর্ডার ট্র্যাকিং
getUser(userId, (err, user) => {
    if (err) { console.error(err); return; }
    getOrders(user.id, (err, orders) => {
        if (err) { console.error(err); return; }
        getOrderDetails(orders[0].id, (err, details) => {
            if (err) { console.error(err); return; }
            getPayment(details.paymentId, (err, payment) => {
                if (err) { console.error(err); return; }
                getShipping(payment.orderId, (err, shipping) => {
                    if (err) { console.error(err); return; }
                    console.log(shipping); // ৫ স্তর গভীরে!
                });
            });
        });
    });
});
```

**সমস্যাগুলো:**
- কোড ডানদিকে ক্রমাগত সরে যায় (indentation hell)
- এরর হ্যান্ডলিং প্রতিটি স্তরে আলাদাভাবে করতে হয়
- কোড পড়া ও রক্ষণাবেক্ষণ অত্যন্ত কঠিন

### 🔄 Promise Chain

Promises কলব্যাক হেলের সমস্যার সমাধান এনেছে `.then()` চেইনিংয়ের মাধ্যমে:

```javascript
// 🔄 Promise Chain — ফ্ল্যাট ও পরিষ্কার
getUser(userId)
    .then(user => getOrders(user.id))
    .then(orders => getOrderDetails(orders[0].id))
    .then(details => getPayment(details.paymentId))
    .then(payment => getShipping(payment.orderId))
    .then(shipping => console.log(shipping))
    .catch(err => console.error(err)); // একবারেই সব এরর হ্যান্ডেল!
```

**উন্নতি:** ফ্ল্যাট চেইন, কেন্দ্রীয় এরর হ্যান্ডলিং। তবে জটিল লজিকে `.then()` চেইন দীর্ঘ হয়ে যায়।

### ✅ Async/Await — আধুনিক সমাধান

Async/Await প্রোমিসের উপর নির্মিত সিনট্যাক্টিক সুগার। কোড দেখতে সিঙ্ক্রোনাসের মতো, কিন্তু চলে অ্যাসিঙ্ক্রোনাসভাবে:

```javascript
// ✅ Async/Await — সবচেয়ে পরিষ্কার ও পাঠযোগ্য
async function trackDarazOrder(userId) {
    try {
        const user = await getUser(userId);
        const orders = await getOrders(user.id);
        const details = await getOrderDetails(orders[0].id);
        const payment = await getPayment(details.paymentId);
        const shipping = await getShipping(payment.orderId);

        return {
            orderId: details.id,
            status: shipping.status,
            eta: shipping.estimatedDelivery
        };
    } catch (err) {
        console.error('অর্ডার ট্র্যাকিং ব্যর্থ:', err.message);
        throw err; // উপরে প্রপাগেট
    }
}

// ব্যবহার
const tracking = await trackDarazOrder('user_12345');
console.log(`আপনার অর্ডার ${tracking.status} অবস্থায় আছে`);
```

**মূল নিয়ম:**
- `async` ফাংশন সবসময় একটি Promise রিটার্ন করে
- `await` শুধুমাত্র `async` ফাংশনের ভেতরে ব্যবহার করা যায় (অথবা top-level module-এ)
- `await` Promise resolve না হওয়া পর্যন্ত এক্সিকিউশন পজ করে

---

## ⚡ JavaScript Async/Await গভীর বিশ্লেষণ

Async/Await বুঝতে হলে JavaScript-এর Event Loop বুঝতে হবে। নিচের ডায়াগ্রামটি মনোযোগ দিয়ে দেখুন:

```
JavaScript Runtime আর্কিটেকচার:
══════════════════════════════════════════════════════════════════

+───────────────────+    +───────────────────────+
│    Call Stack      │    │    Microtask Queue     │
│  (কল স্ট্যাক)     │    │  (মাইক্রোটাস্ক কিউ)    │
│                    │    │                        │
│  [current fn()]    │    │  [.then() callback]    │
│  [outer fn()]      │    │  [await resume]        │
│                    │    │  [queueMicrotask()]    │
+───────────────────+    +───────────────────────+
          │                         │
          ▼                         ▼
+─────────────────────────────────────────────────+
│                  Event Loop                      │
│  (ইভেন্ট লুপ — JavaScript এর হৃদপিণ্ড)          │
│                                                  │
│  ১. Call Stack সম্পূর্ণ খালি করো                   │
│  ২. সব Microtask চালাও (Queue খালি না হওয়া পর্যন্ত)│
│  ৩. একটি Macrotask চালাও                         │
│  ৪. আবার ধাপ ১ থেকে শুরু করো                      │
+─────────────────────────────────────────────────+
          ▲
          │
+───────────────────────+
│   Macrotask Queue      │
│  (ম্যাক্রোটাস্ক কিউ)   │
│                        │
│  [setTimeout cb]       │
│  [setInterval cb]      │
│  [I/O callback]        │
│  [UI rendering]        │
+───────────────────────+

══════════════════════════════════════════════════════════════════
```

### 🔑 Microtask বনাম Macrotask — কে আগে চলে?

```javascript
console.log('১. শুরু'); // Call Stack — সবার আগে

setTimeout(() => {
    console.log('৪. setTimeout'); // Macrotask — সবার শেষে
}, 0);

Promise.resolve().then(() => {
    console.log('৩. Promise'); // Microtask — setTimeout এর আগে
});

console.log('২. শেষ'); // Call Stack — সিঙ্ক্রোনাস

// আউটপুট:
// ১. শুরু
// ২. শেষ
// ৩. Promise      ← Microtask আগে চলে!
// ৪. setTimeout   ← Macrotask পরে চলে!
```

### 🔍 await কী করে — ভেতরের কথা

যখন JavaScript ইঞ্জিন একটি `await` দেখে, এটি আসলে নিচের কাজগুলো করে:

```
await এর অভ্যন্তরীণ কাজ:
══════════════════════════════════════════════════

async function fetchBkashBalance(userId) {
    console.log("A");                    // ① সিঙ্ক্রোনাস
    const balance = await getBalance();  // ② এখানে পজ
    console.log("B: " + balance);        // ③ রেজিউম
}

কল করলে যা ঘটে:

   fetchBkashBalance('user1')
          │
          ▼
   ┌──────────────────────────────┐
   │  ① "A" প্রিন্ট হয়            │  ← সিঙ্ক্রোনাস
   │  ② getBalance() কল হয়         │  ← Promise তৈরি হয়
   │  ③ ফাংশন PAUSE হয়            │  ← Call Stack থেকে বের হয়
   │     বাকি কোড Microtask        │
   │     Queue-তে যায়              │
   └──────────────────────────────┘
          │
          ▼   (Promise resolve হলে)
   ┌──────────────────────────────┐
   │  ④ ফাংশন RESUME হয়           │  ← Microtask হিসেবে
   │  ⑤ balance = resolved value  │
   │  ⑥ "B: ৫০০০" প্রিন্ট হয়      │
   └──────────────────────────────┘

══════════════════════════════════════════════════
```

### 🏠 বাস্তব উদাহরণ: Grameenphone MyGP অ্যাপ

```javascript
// Grameenphone MyGP Dashboard — সব ডেটা একসাথে লোড
async function loadMyGPDashboard(phoneNumber) {
    // ✅ সমান্তরাল (parallel) — তিনটি API একসাথে কল
    const [balance, dataUsage, offers] = await Promise.all([
        fetchBalance(phoneNumber),      // ব্যালেন্স চেক
        fetchDataUsage(phoneNumber),    // ডেটা ব্যবহার
        fetchExclusiveOffers(phoneNumber) // বিশেষ অফার
    ]);

    // এগুলো আগেরটির উপর নির্ভরশীল — ক্রমানুসারে (sequential)
    const recommendations = await getRecommendations(dataUsage.pattern);
    const nearbyRecharge = await findNearbyRechargePoints(balance.location);

    return {
        balance: balance.amount,
        dataRemaining: dataUsage.remaining,
        topOffers: offers.slice(0, 5),
        recommendations,
        nearbyRecharge
    };
}
```

---

## 🐘 PHP Fibers (PHP 8.1+)

PHP ঐতিহ্যগতভাবে সিঙ্ক্রোনাস ভাষা। কিন্তু PHP 8.1-এ Fibers এসেছে, যা অ্যাসিঙ্ক্রোনাস প্রোগ্রামিংয়ের ভিত্তি তৈরি করেছে।

### Fiber কী?

Fiber হলো একটি লাইটওয়েট থ্রেড যা নিজেকে pause (`suspend`) এবং resume করতে পারে। এটি সাধারণ থ্রেডের মতো OS-level নয়, বরং userland-level, তাই অনেক হালকা।

```
PHP Fiber এর কাজের ধারা:
══════════════════════════════════════════════════════

  Main Thread                    Fiber
  ───────────                    ─────
       │
       │  $fiber = new Fiber(fn)
       │  $fiber->start()
       │─────────────────────────▶│
       │                          │  কাজ শুরু...
       │                          │  Fiber::suspend($value)
       │◀─────────────────────────│
       │  $value পাওয়া গেছে       │  (পজ অবস্থায়)
       │  কিছু কাজ করো            │
       │  $fiber->resume($data)   │
       │─────────────────────────▶│
       │                          │  $data পাওয়া গেছে
       │                          │  বাকি কাজ শেষ
       │                          │  return $result
       │◀─────────────────────────│
       │  $fiber->getReturn()     │
       ▼                          ▼

══════════════════════════════════════════════════════
```

### মৌলিক Fiber উদাহরণ

```php
<?php
// PHP Fiber — মৌলিক উদাহরণ
$fiber = new Fiber(function (): void {
    // ১ম পজ — Main thread-কে একটি মান পাঠায়
    $receivedValue = Fiber::suspend('fiber started');
    echo "Fiber-এ পাওয়া মান: " . $receivedValue . "\n";

    // ২য় পজ — আরেকটি মান পাঠায়
    $anotherValue = Fiber::suspend('second pause');
    echo "আবার পাওয়া মান: " . $anotherValue . "\n";
});

// Fiber শুরু করি — প্রথম suspend পর্যন্ত চলবে
$result = $fiber->start();
echo $result . "\n"; // আউটপুট: "fiber started"

// Fiber রেজিউম করি — দ্বিতীয় suspend পর্যন্ত চলবে
$result2 = $fiber->resume('hello from main');
// আউটপুট: "Fiber-এ পাওয়া মান: hello from main"
echo $result2 . "\n"; // আউটপুট: "second pause"

// আবার রেজিউম — শেষ পর্যন্ত চলবে
$fiber->resume('final value');
// আউটপুট: "আবার পাওয়া মান: final value"
```

### 🏠 বাস্তব উদাহরণ: Nagad পেমেন্ট ভেরিফিকেশন

```php
<?php
// Nagad-এর পেমেন্ট ভেরিফিকেশন — Fiber ব্যবহার করে
function createPaymentVerifier(string $transactionId): Fiber
{
    return new Fiber(function () use ($transactionId): array {
        // ধাপ ১: ট্রানজাকশন খুঁজুন
        $transaction = findTransaction($transactionId);
        $status = Fiber::suspend(['step' => 'found', 'data' => $transaction]);

        // ধাপ ২: ব্যালেন্স চেক করুন
        $balanceOk = checkSufficientBalance($transaction['sender'], $transaction['amount']);
        $status = Fiber::suspend(['step' => 'balance_checked', 'sufficient' => $balanceOk]);

        // ধাপ ৩: ফ্রড চেক চালান
        $fraudResult = runFraudDetection($transaction);
        Fiber::suspend(['step' => 'fraud_checked', 'safe' => $fraudResult['isSafe']]);

        // ধাপ ৪: চূড়ান্ত ভেরিফিকেশন
        return [
            'verified' => true,
            'transactionId' => $transactionId,
            'timestamp' => time()
        ];
    });
}

// ব্যবহার — প্রতিটি ধাপে লগ করা যায়
$verifier = createPaymentVerifier('TXN_98765');
$step1 = $verifier->start();
logStep($step1); // "found" লগ করো

$step2 = $verifier->resume(null);
logStep($step2); // "balance_checked" লগ করো

$step3 = $verifier->resume(null);
logStep($step3); // "fraud_checked" লগ করো

$verifier->resume(null);
$finalResult = $verifier->getReturn();
// $finalResult = ['verified' => true, ...]
```

---

## 🐘 PHP অ্যাসিঙ্ক ফ্রেমওয়ার্ক

PHP-তে সত্যিকারের অ্যাসিঙ্ক্রোনাস কাজের জন্য তিনটি প্রধান ফ্রেমওয়ার্ক আছে:

```
PHP অ্যাসিঙ্ক ফ্রেমওয়ার্ক তুলনা:
══════════════════════════════════════════════════════════════

  ┌───────────┐    ┌───────────┐    ┌───────────┐
  │  ReactPHP  │    │   Amphp    │    │  Swoole   │
  │            │    │            │    │           │
  │ Event Loop │    │  Fibers +  │    │ C Ext +   │
  │  ভিত্তিক   │    │ Event Loop │    │ Coroutine │
  │            │    │            │    │           │
  │ পুরনো ও    │    │ আধুনিক ও   │    │ সবচেয়ে   │
  │ প্রমাণিত   │    │ সহজ API   │    │ দ্রুত     │
  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
        │                │                │
        ▼                ▼                ▼
  Callback-based    await-style      Go-style
  প্রোগ্রামিং       প্রোগ্রামিং       coroutine

══════════════════════════════════════════════════════════════
```

### ReactPHP উদাহরণ

```php
<?php
// ReactPHP — Daraz-এর প্রোডাক্ট সার্চ API
use React\Http\Browser;
use React\EventLoop\Loop;

$browser = new Browser();

// অ্যাসিঙ্ক HTTP রিকোয়েস্ট — Callback-based
$browser->get('https://api.daraz.com.bd/products?q=mobile')
    ->then(function (Psr\Http\Message\ResponseInterface $response) {
        $products = json_decode($response->getBody(), true);
        echo "পাওয়া গেছে: " . count($products) . " পণ্য\n";
    }, function (Exception $e) {
        echo "এরর: " . $e->getMessage() . "\n";
    });

// একাধিক সমান্তরাল রিকোয়েস্ট
$promises = [
    $browser->get('https://api.daraz.com.bd/products?category=phones'),
    $browser->get('https://api.daraz.com.bd/products?category=laptops'),
    $browser->get('https://api.daraz.com.bd/products?category=tablets'),
];

React\Promise\all($promises)->then(function (array $responses) {
    foreach ($responses as $response) {
        echo "স্ট্যাটাস: " . $response->getStatusCode() . "\n";
    }
});
```

### Amphp উদাহরণ (আধুনিক — await-style)

```php
<?php
// Amphp — Daraz ক্যাটালগ সার্ভিস
use Amp\Http\Client\HttpClientBuilder;
use Amp\Http\Client\Request;
use function Amp\async;
use function Amp\Future\await;

$client = HttpClientBuilder::buildDefault();

// ✅ সমান্তরাল — সব রিকোয়েস্ট একসাথে
$futures = [];
$futures['products']   = async(fn () => $client->request(
    new Request('https://api.daraz.com.bd/products')
));
$futures['categories'] = async(fn () => $client->request(
    new Request('https://api.daraz.com.bd/categories')
));
$futures['deals']      = async(fn () => $client->request(
    new Request('https://api.daraz.com.bd/flash-deals')
));

// সব Future একসাথে resolve হবে — সমান্তরাল!
$responses = await($futures);

$products   = json_decode($responses['products']->getBody()->buffer(), true);
$categories = json_decode($responses['categories']->getBody()->buffer(), true);
$deals      = json_decode($responses['deals']->getBody()->buffer(), true);

echo "প্রোডাক্ট: " . count($products) . "\n";
echo "ক্যাটাগরি: " . count($categories) . "\n";
echo "ডিল: " . count($deals) . "\n";
```

### Swoole উদাহরণ

```php
<?php
// Swoole — Pathao রাইডার ম্যাচিং সার্ভিস
use Swoole\Coroutine;
use function Swoole\Coroutine\run;

run(function () {
    // সমান্তরাল coroutine — একসাথে কাছের রাইডার খোঁজা
    $results = [];

    // তিনটি coroutine একসাথে চলবে
    Coroutine::join([
        Coroutine::create(function () use (&$results) {
            $results['zone_a'] = searchRidersInZone('zone_a', ['lat' => 23.8103, 'lng' => 90.4125]);
        }),
        Coroutine::create(function () use (&$results) {
            $results['zone_b'] = searchRidersInZone('zone_b', ['lat' => 23.8110, 'lng' => 90.4130]);
        }),
        Coroutine::create(function () use (&$results) {
            $results['zone_c'] = searchRidersInZone('zone_c', ['lat' => 23.8095, 'lng' => 90.4120]);
        }),
    ]);

    // সবচেয়ে কাছের রাইডার বাছাই
    $allRiders = array_merge(...array_values($results));
    usort($allRiders, fn ($a, $b) => $a['distance'] <=> $b['distance']);

    echo "সবচেয়ে কাছের রাইডার: " . $allRiders[0]['name'] . "\n";
});
```

---

## ⚠️ সাধারণ ভুল ও সমস্যা

### ভুল ১: Sequential await যেখানে Parallel সম্ভব

এটি সবচেয়ে কমন পারফরম্যান্স ভুল:

```javascript
// ❌ Sequential (ধীর) — একটার পর একটা অপেক্ষা
async function loadDarazHomepage() {
    const users = await getUsers();         // ১ সেকেন্ড
    const products = await getProducts();   // ১ সেকেন্ড
    const orders = await getOrders();       // ১ সেকেন্ড
    // মোট সময়: ৩ সেকেন্ড (1s + 1s + 1s) 🐌
    return { users, products, orders };
}

// ✅ Parallel (দ্রুত) — সব একসাথে শুরু
async function loadDarazHomepage() {
    const [users, products, orders] = await Promise.all([
        getUsers(),       // ─┐
        getProducts(),    // ─┤ তিনটি একসাথে চলে
        getOrders()       // ─┘
    ]);
    // মোট সময়: ১ সেকেন্ড (তিনটির মধ্যে সর্বোচ্চ) ⚡
    return { users, products, orders };
}
```

```
সময় তুলনা:
══════════════════════════════════════════

❌ Sequential:
  getUsers()  ████████░░░░░░░░░░░░░░░░  1s
  getProducts()        ████████░░░░░░░░  1s
  getOrders()                   ████████ 1s
  ─────────────────────────────────────
  মোট:                              3s  🐌

✅ Parallel:
  getUsers()  ████████                   1s
  getProducts()████████                  1s
  getOrders() ████████                   1s
  ─────────────────────────────────────
  মোট:        ████████                   1s  ⚡

══════════════════════════════════════════
```

### ভুল ২: Unhandled Promise Rejection

```javascript
// ❌ catch ছাড়া — অ্যাপ ক্র্যাশ হতে পারে
async function sendBkashMoney(amount) {
    const result = await transferMoney(amount);
    return result;
}
sendBkashMoney(500); // reject হলে UnhandledPromiseRejection!

// ✅ সঠিক — try/catch ব্যবহার
async function sendBkashMoney(amount) {
    try {
        const result = await transferMoney(amount);
        return result;
    } catch (err) {
        console.error('bKash ট্রান্সফার ব্যর্থ:', err.message);
        await logFailedTransaction(amount, err);
        throw new TransferError('টাকা পাঠানো যায়নি', { cause: err });
    }
}

// ✅ বিকল্প — .catch() দিয়ে কল করা
sendBkashMoney(500).catch(err => handleError(err));
```

### ভুল ৩: async forEach কাজ করে না যেভাবে মনে হয়

```javascript
// ❌ ভুল — forEach async/await সমর্থন করে না
const orderIds = ['ORD-001', 'ORD-002', 'ORD-003'];

orderIds.forEach(async (id) => {
    const order = await fetchOrder(id);  // এগুলো সমান্তরালে চলে!
    await processOrder(order);            // ক্রম ঠিক থাকে না!
});
console.log('সব শেষ!'); // ❌ আসলে শেষ হয়নি!

// ✅ সঠিক — for...of (ক্রমানুসারে)
for (const id of orderIds) {
    const order = await fetchOrder(id);
    await processOrder(order);
}
console.log('সব শেষ!'); // ✅ সত্যিই শেষ

// ✅ সঠিক — Promise.all + map (সমান্তরালে)
await Promise.all(
    orderIds.map(async (id) => {
        const order = await fetchOrder(id);
        await processOrder(order);
    })
);
console.log('সব শেষ!'); // ✅ সত্যিই শেষ
```

### ভুল ৪: await ছাড়া async ফাংশন কল

```javascript
// ❌ await ভুলে গেছি — ফলাফল Promise অবজেক্ট!
async function getBkashBalance(userId) {
    return await fetchBalance(userId);
}

const balance = getBkashBalance('user1'); // Promise { <pending> }
console.log(balance); // Promise { <pending> } — সংখ্যা নয়!

// ✅ সঠিক
const balance = await getBkashBalance('user1');
console.log(balance); // 5000 ✅
```

---

## 🛡️ অ্যাসিঙ্ক এরর হ্যান্ডলিং

এরর হ্যান্ডলিং অ্যাসিঙ্ক কোডের সবচেয়ে গুরুত্বপূর্ণ অংশ। ভুল হ্যান্ডলিং করলে সিস্টেম ক্র্যাশ করতে পারে।

### প্যাটার্ন ১: try/catch ব্লক

```javascript
// bKash ট্রান্সফার — স্তরভিত্তিক এরর হ্যান্ডলিং
async function bkashTransfer(senderId, receiverId, amount) {
    try {
        // ধাপ ১: সেন্ডার ভেরিফাই
        const sender = await verifySender(senderId);

        try {
            // ধাপ ২: ট্রান্সফার চালানো
            const txn = await executeTransfer(sender, receiverId, amount);
            return { success: true, transactionId: txn.id };
        } catch (transferErr) {
            // ট্রান্সফার ব্যর্থ হলে rollback
            await rollbackTransaction(senderId, amount);
            throw new TransferFailedError(transferErr.message);
        }

    } catch (err) {
        if (err instanceof SenderNotFoundError) {
            return { success: false, error: 'প্রেরক পাওয়া যায়নি' };
        }
        if (err instanceof InsufficientBalanceError) {
            return { success: false, error: 'পর্যাপ্ত ব্যালেন্স নেই' };
        }
        throw err; // অজানা এরর উপরে পাঠাও
    }
}
```

### প্যাটার্ন ২: গ্লোবাল এরর হ্যান্ডলার

```javascript
// Node.js — গ্লোবাল unhandled rejection হ্যান্ডলার
process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection:', reason);
    // প্রোডাকশনে: লগ করো, অ্যালার্ট পাঠাও, graceful shutdown
    notifyDevTeam(reason);
    process.exit(1);
});

// ব্রাউজার — গ্লোবাল হ্যান্ডলার
window.addEventListener('unhandledrejection', (event) => {
    console.error('Unhandled Rejection:', event.reason);
    event.preventDefault(); // ডিফল্ট console error বন্ধ
    sendErrorToSentry(event.reason);
});
```

### প্যাটার্ন ৩: Result Wrapper (Go-style)

```javascript
// Go-style এরর হ্যান্ডলিং — try/catch ছাড়া
async function safeAsync(asyncFn) {
    try {
        const data = await asyncFn();
        return [data, null];
    } catch (err) {
        return [null, err];
    }
}

// ব্যবহার — অনেক পরিষ্কার!
const [user, userErr] = await safeAsync(() => getUser(userId));
if (userErr) {
    console.error('ইউজার পাওয়া যায়নি:', userErr.message);
    return;
}

const [balance, balErr] = await safeAsync(() => getBalance(user.id));
if (balErr) {
    console.error('ব্যালেন্স আনতে ব্যর্থ:', balErr.message);
    return;
}

console.log(`${user.name}-এর ব্যালেন্স: ৳${balance}`);
```

### PHP-তে এরর হ্যান্ডলিং (Amphp)

```php
<?php
// Amphp-তে Async এরর হ্যান্ডলিং
use Amp\Http\Client\HttpClientBuilder;
use Amp\Http\Client\HttpException;
use function Amp\async;
use function Amp\Future\await;

async function fetchWithRetry(string $url, int $maxRetries = 3): string
{
    $client = HttpClientBuilder::buildDefault();
    $lastException = null;

    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            $response = $client->request(new Request($url));
            return $response->getBody()->buffer();
        } catch (HttpException $e) {
            $lastException = $e;
            echo "প্রচেষ্টা {$attempt}/{$maxRetries} ব্যর্থ: {$e->getMessage()}\n";

            if ($attempt < $maxRetries) {
                // Exponential backoff
                Amp\delay($attempt * 1.0);
            }
        }
    }

    throw new RuntimeException(
        "সর্বোচ্চ {$maxRetries} বার চেষ্টার পরও ব্যর্থ",
        previous: $lastException
    );
}
```

---

## 🔧 Promise.all, Promise.race, Promise.allSettled

তিনটি শক্তিশালী Promise combinator — প্রতিটির আলাদা ব্যবহার ক্ষেত্র আছে।

```
Promise Combinators তুলনা:
══════════════════════════════════════════════════════════════════

┌─────────────────┬──────────────────┬─────────────────────────┐
│   Combinator    │  কখন resolve?    │  কখন reject?            │
├─────────────────┼──────────────────┼─────────────────────────┤
│ Promise.all     │ সব সফল হলে       │ যেকোনো একটি fail হলে    │
│ Promise.race    │ প্রথমটি শেষ হলে   │ প্রথমটি fail হলে        │
│ Promise.allSettled│ সব শেষ হলে      │ কখনোই reject হয় না     │
│ Promise.any     │ প্রথম সফলটি      │ সব fail হলে            │
└─────────────────┴──────────────────┴─────────────────────────┘

══════════════════════════════════════════════════════════════════
```

### Promise.all — সব সফল হতে হবে

bKash-এর টাকা পাঠানোর আগে তিনটি চেক — সবগুলো পাস করতে হবে:

```javascript
// bKash: টাকা পাঠানোর পূর্ববর্তী চেক — সব সফল হতে হবে
async function validateBkashTransfer(senderId, receiverId, amount) {
    try {
        const [balance, receiver, limit] = await Promise.all([
            checkBalance(senderId),           // ব্যালেন্স চেক
            verifyReceiver(receiverId),       // রিসিভার ভেরিফাই
            checkDailyLimit(senderId, amount) // দৈনিক লিমিট চেক
        ]);

        // তিনটিই সফল — এগিয়ে যাও
        if (balance.amount < amount) {
            throw new Error('অপর্যাপ্ত ব্যালেন্স');
        }

        return { canTransfer: true, balance, receiver, limit };
    } catch (err) {
        // যেকোনো একটি ব্যর্থ হলে সবই বাতিল
        return { canTransfer: false, reason: err.message };
    }
}
```

### Promise.race — প্রথমে যে শেষ হয় সেটা জিতে

Pathao-তে সবচেয়ে দ্রুত রাইডার খোঁজা:

```javascript
// Pathao: প্রথম উপলব্ধ রাইডার জিতে
async function findNearestRider(pickupLocation) {
    const timeoutPromise = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('রাইডার খুঁজতে সময় শেষ')), 30000)
    );

    try {
        const rider = await Promise.race([
            findRiderInZoneA(pickupLocation),
            findRiderInZoneB(pickupLocation),
            findRiderInZoneC(pickupLocation),
            timeoutPromise // ৩০ সেকেন্ড টাইমআউট
        ]);

        return { found: true, rider };
    } catch (err) {
        return { found: false, error: err.message };
    }
}
```

### Promise.allSettled — সবগুলো শেষ হওয়া পর্যন্ত অপেক্ষা

Daraz-এ সব সেলারদের নোটিফিকেশন পাঠানো — কেউ কেউ ব্যর্থ হতে পারে:

```javascript
// Daraz: সব সেলারকে নোটিফিকেশন — কিছু ব্যর্থ হলেও বাকিরা চলবে
async function notifyAllSellers(sellerIds, message) {
    const results = await Promise.allSettled(
        sellerIds.map(id => sendNotification(id, message))
    );

    const summary = {
        total: results.length,
        successful: results.filter(r => r.status === 'fulfilled').length,
        failed: results.filter(r => r.status === 'rejected').length,
        failures: results
            .filter(r => r.status === 'rejected')
            .map(r => r.reason.message)
    };

    console.log(`নোটিফিকেশন: ${summary.successful}/${summary.total} সফল`);
    if (summary.failed > 0) {
        console.warn(`ব্যর্থ: ${summary.failures.join(', ')}`);
    }

    return summary;
}

// ব্যবহার
await notifyAllSellers(
    ['seller_01', 'seller_02', 'seller_03', 'seller_04'],
    'ফ্ল্যাশ সেল শুরু হয়েছে! আপনার পণ্য প্রস্তুত করুন।'
);
// আউটপুট: নোটিফিকেশন: 3/4 সফল
// ব্যর্থ: seller_03 অফলাইনে আছে
```

### Promise.any — যেকোনো একটি সফল হলেই হবে

```javascript
// Grameenphone: যেকোনো একটি পেমেন্ট গেটওয়ে কাজ করলেই হবে
async function processRecharge(phoneNumber, amount) {
    try {
        const result = await Promise.any([
            chargeViaBkash(phoneNumber, amount),
            chargeViaNagad(phoneNumber, amount),
            chargeViaRocket(phoneNumber, amount)
        ]);
        return { success: true, gateway: result.gateway, txnId: result.id };
    } catch (err) {
        // AggregateError — সব গেটওয়ে ব্যর্থ!
        return { success: false, error: 'সব পেমেন্ট গেটওয়ে ব্যর্থ' };
    }
}
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Async/Await ব্যবহার করবেন

| পরিস্থিতি | উদাহরণ | কেন |
|-----------|---------|-----|
| API কল | Daraz প্রোডাক্ট ফেচ | নেটওয়ার্ক I/O — ব্লক করলে পুরো অ্যাপ আটকে যায় |
| ডাটাবেস কুয়েরি | bKash ট্রানজাকশন লুকআপ | ডিস্ক I/O — সময়সাপেক্ষ অপারেশন |
| ফাইল অপারেশন | লগ ফাইল লেখা/পড়া | ডিস্ক I/O |
| একাধিক স্বাধীন কাজ | Pathao-তে রাইডার + রুট + ভাড়া একসাথে | Promise.all দিয়ে সমান্তরালে চালাও |
| টাইমআউট সহ অপেক্ষা | রিকোয়েস্ট ৫ সেকেন্ডে শেষ না হলে বাতিল | Promise.race দিয়ে টাইমআউট |
| Retry লজিক | ব্যর্থ রিকোয়েস্ট পুনরায় চেষ্টা | async loop + delay |

### ❌ কখন Async/Await ব্যবহার করবেন না

| পরিস্থিতি | কেন না | বিকল্প |
|-----------|--------|--------|
| CPU-intensive কাজ | await CPU ব্লক করে, ইভেন্ট লুপ আটকায় | Worker Thread / Child Process |
| সিম্পল সিঙ্ক্রোনাস লজিক | অপ্রয়োজনীয় জটিলতা যোগ হয় | সাধারণ ফাংশন |
| Real-time streaming | WebSocket বা Stream ভালো কাজ করে | EventEmitter / Stream API |
| সিম্পল ক্যালকুলেশন | `2 + 2`-এর জন্য async দরকার নেই | সাধারণ রিটার্ন |

### 📊 সিদ্ধান্ত গ্রহণ ফ্লোচার্ট

```
অ্যাসিঙ্ক প্যাটার্ন বাছাই ফ্লোচার্ট:
══════════════════════════════════════════════════════

           কাজটি কি I/O-bound?
           (নেটওয়ার্ক / ডিস্ক / DB)
                  │
          ┌───────┴───────┐
          │               │
         হ্যাঁ            না
          │               │
          ▼               ▼
   একাধিক স্বাধীন     CPU-intensive?
   কাজ আছে?          │
   │                  ├─ হ্যাঁ → Worker Thread
   ├─ হ্যাঁ             │          / Child Process
   │  │               └─ না  → সাধারণ ফাংশন
   │  ▼
   │  সব সফল হতে হবে?
   │  │
   │  ├─ হ্যাঁ → Promise.all
   │  │
   │  ├─ না, কিছু ব্যর্থ হলেও চলবে
   │  │       → Promise.allSettled
   │  │
   │  └─ প্রথমটি নিলেই হবে
   │          → Promise.race / Promise.any
   │
   └─ না
      │
      ▼
   ক্রমানুসারে চলবে?
   │
   ├─ হ্যাঁ → async/await (sequential)
   │
   └─ না  → Promise.all (parallel)

══════════════════════════════════════════════════════
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

| প্যাটার্ন | সম্পর্ক |
|-----------|---------|
| [Event Loop](./event-loop.md) | Async/Await যে ভিত্তির উপর কাজ করে |
| [Concurrency Patterns](./concurrency-patterns.md) | বৃহত্তর concurrency কৌশল |
| [Race Conditions](./race-conditions.md) | Async কোডে race condition সমস্যা |
| [Saga Pattern](./saga-pattern.md) | বিতরণকৃত ট্রানজাকশন ম্যানেজমেন্ট |

---

## 📖 সারসংক্ষেপ

```
মূল বিষয়গুলো মনে রাখুন:
══════════════════════════════════════════════════════

✅ async ফাংশন সবসময় Promise রিটার্ন করে
✅ await শুধু async ফাংশনের ভেতরে কাজ করে
✅ স্বাধীন কাজে Promise.all ব্যবহার করুন (parallel)
✅ নির্ভরশীল কাজে sequential await ব্যবহার করুন
✅ সবসময় try/catch দিয়ে এরর হ্যান্ডেল করুন
✅ PHP 8.1+ এ Fiber অ্যাসিঙ্ক প্রোগ্রামিং সম্ভব করে

❌ forEach-এ async/await ব্যবহার করবেন না
❌ CPU-intensive কাজে async ব্যবহার করবেন না
❌ await ছাড়া async ফাংশন কল করবেন না
❌ Unhandled Promise rejection রাখবেন না

══════════════════════════════════════════════════════
```

> **"যেকোনো পর্যাপ্ত উন্নত প্রযুক্তি জাদু থেকে আলাদা করা যায় না।"**
> — আর্থার সি. ক্লার্ক
>
> Async/Await হলো সেই জাদু যা জটিল অ্যাসিঙ্ক্রোনাস কোডকে সহজ ও পাঠযোগ্য করে তোলে। 🪄
