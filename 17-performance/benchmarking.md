# 📌 বেঞ্চমার্কিং (Benchmarking)

> "যদি তুমি মাপতে না পারো, তাহলে তুমি উন্নতিও করতে পারবে না।" — Peter Drucker

বেঞ্চমার্কিং হলো সফটওয়্যারের পারফরম্যান্স পরিমাপ করার একটি পদ্ধতিগত প্রক্রিয়া। এটি আমাদের জানায় কোড কতটুকু
দ্রুত চলছে, কতটুকু রিসোর্স ব্যবহার করছে, এবং কোথায় bottleneck আছে।

## 🏠 বাস্তব উদাহরণ

ধরুন **Daraz Bangladesh** তাদের 11.11 সেলের জন্য প্রস্তুতি নিচ্ছে। গত বছর সেলের সময় সাইট ক্র্যাশ করেছিল।
এবার তারা জানতে চায়:

```
প্রশ্ন:
├── প্রতি সেকেন্ডে কতটি অর্ডার হ্যান্ডেল করতে পারে?
├── ১০ লাখ ইউজার একসাথে ঢুকলে কী হবে?
├── প্রোডাক্ট সার্চ কত মিলিসেকেন্ডে রেসপন্স দেয়?
└── পেমেন্ট গেটওয়ে (bKash/Nagad) কতক্ষণ সময় নেয়?
```

এই প্রশ্নগুলোর উত্তর পেতেই **বেঞ্চমার্কিং** দরকার।

---

## 📊 মাইক্রো vs ম্যাক্রো বেঞ্চমার্ক

বেঞ্চমার্কিং দুই ধরনের — **মাইক্রো** এবং **ম্যাক্রো**। দুটোর উদ্দেশ্য এবং স্কোপ সম্পূর্ণ আলাদা।

### ASCII ডায়াগ্রাম — স্কোপ পার্থক্য

```
  মাইক্রো বেঞ্চমার্ক                    ম্যাক্রো বেঞ্চমার্ক
  ┌─────────────────┐                   ┌──────────────────────────────────┐
  │                  │                   │  ┌──────┐   ┌──────┐   ┌─────┐ │
  │   একটি ফাংশন    │                   │  │ HTTP │──▶│ App  │──▶│ DB  │ │
  │   বা অপারেশন    │                   │  │Server│   │Server│   │Query│ │
  │                  │                   │  └──────┘   └──────┘   └─────┘ │
  │  যেমন: sort()   │                   │       ▲                   │     │
  │  যেমন: json     │                   │       └───────────────────┘     │
  │    encode/decode │                   │     সম্পূর্ণ রিকোয়েস্ট সাইকেল  │
  └─────────────────┘                   └──────────────────────────────────┘
        ↕ ns/μs                                     ↕ ms/s
     (ন্যানো/মাইক্রো                            (মিলি সেকেন্ড/
       সেকেন্ড)                                    সেকেন্ড)
```

### তুলনা টেবিল

| বিষয়           | মাইক্রো বেঞ্চমার্ক              | ম্যাক্রো বেঞ্চমার্ক                  |
|-----------------|----------------------------------|---------------------------------------|
| স্কোপ           | একটি ফাংশন/মেথড                 | সম্পূর্ণ সিস্টেম                      |
| সময় একক        | ns / μs                          | ms / s                                |
| উদাহরণ          | `array_map` vs `foreach`         | API endpoint response time            |
| টুলস            | phpbench, Benchmark.js           | k6, Artillery, wrk                    |
| উদ্দেশ্য         | অ্যালগরিদম অপ্টিমাইজেশন          | সিস্টেম ক্যাপাসিটি প্ল্যানিং           |
| Daraz উদাহরণ    | প্রাইস ক্যালকুলেশন ফাংশন         | 11.11 সেলের সময় সম্পূর্ণ অর্ডার ফ্লো |

### কখন কোনটি ব্যবহার করবেন?

```
মাইক্রো বেঞ্চমার্ক ব্যবহার করুন যখন:
├── দুটি অ্যালগরিদমের মধ্যে তুলনা করতে চান
├── একটি ফাংশন অপ্টিমাইজ করছেন
└── Library A vs Library B তুলনা করছেন

ম্যাক্রো বেঞ্চমার্ক ব্যবহার করুন যখন:
├── প্রোডাকশনে deploy করার আগে লোড টেস্ট করতে চান
├── সিস্টেমের ক্যাপাসিটি জানতে চান
└── SLA (Service Level Agreement) নির্ধারণ করতে চান
```

---

## 🐘 PHP বেঞ্চমার্কিং

PHP তে বেঞ্চমার্কিং করতে আমরা `hrtime()` ফাংশন ব্যবহার করি যা ন্যানোসেকেন্ড পর্যন্ত সুনির্দিষ্ট সময় দেয়।

### সাধারণ বেঞ্চমার্ক ফাংশন

```php
<?php

/**
 * একটি callable ফাংশনকে নির্দিষ্ট সংখ্যকবার চালিয়ে
 * পারফরম্যান্স পরিসংখ্যান (avg, p50, p95, p99) রিটার্ন করে।
 */
function benchmark(callable $fn, int $iterations = 10000): array
{
    $times = [];

    for ($i = 0; $i < $iterations; $i++) {
        $start = hrtime(true);
        $fn();
        $times[] = hrtime(true) - $start;
    }

    sort($times);

    return [
        'avg' => array_sum($times) / count($times) / 1e6,   // মিলিসেকেন্ডে গড়
        'p50' => $times[(int)(count($times) * 0.5)] / 1e6,   // ৫০তম পার্সেন্টাইল
        'p95' => $times[(int)(count($times) * 0.95)] / 1e6,  // ৯৫তম পার্সেন্টাইল
        'p99' => $times[(int)(count($times) * 0.99)] / 1e6,  // ৯৯তম পার্সেন্টাইল
    ];
}
```

### অ্যারে ফাংশন তুলনা — `array_map` vs `foreach`

```php
<?php

// পদ্ধতি ১: array_map ব্যবহার করে
$result1 = benchmark(fn() => array_map(fn($x) => $x * 2, range(1, 1000)));

// পদ্ধতি ২: foreach ব্যবহার করে (reference দিয়ে)
$result2 = benchmark(function() {
    $arr = range(1, 1000);
    foreach ($arr as &$x) $x *= 2;
});

echo "=== array_map vs foreach (রেফারেন্স) ===\n";
echo sprintf("array_map → avg: %.4fms | p95: %.4fms | p99: %.4fms\n",
    $result1['avg'], $result1['p95'], $result1['p99']);
echo sprintf("foreach   → avg: %.4fms | p95: %.4fms | p99: %.4fms\n",
    $result2['avg'], $result2['p95'], $result2['p99']);
```

**সম্ভাব্য আউটপুট:**

```
=== array_map vs foreach (রেফারেন্স) ===
array_map → avg: 0.0892ms | p95: 0.1134ms | p99: 0.1567ms
foreach   → avg: 0.0534ms | p95: 0.0678ms | p99: 0.0912ms
```

> 💡 সাধারণত `foreach` reference সহ `array_map` এর চেয়ে দ্রুত কারণ এতে কোনো callback overhead নেই।

### JSON Encode/Decode বেঞ্চমার্ক — Daraz প্রোডাক্ট ডেটা

```php
<?php

// Daraz-এর মতো একটি প্রোডাক্ট অবজেক্ট
$product = [
    'id' => 12345,
    'name' => 'Samsung Galaxy S24',
    'price' => 89999,
    'discount' => 15,
    'seller' => 'Official Samsung Store',
    'ratings' => array_fill(0, 100, ['score' => rand(1, 5), 'review' => 'Good product']),
    'categories' => ['Electronics', 'Mobile Phones', 'Samsung'],
];

$jsonEncode = benchmark(fn() => json_encode($product));
$jsonString = json_encode($product);
$jsonDecode = benchmark(fn() => json_decode($jsonString, true));

echo "json_encode → avg: {$jsonEncode['avg']}ms\n";
echo "json_decode → avg: {$jsonDecode['avg']}ms\n";
```

### PHP বেঞ্চমার্কিং রেজাল্ট ফ্লো

```
 বেঞ্চমার্ক শুরু
       │
       ▼
 ┌─────────────┐
 │  Warm-up     │  ← প্রথম কিছু iteration ওয়ার্মআপ
 │  (100 runs)  │     OPcache সক্রিয় হয়
 └──────┬───────┘
        │
        ▼
 ┌─────────────┐
 │  Measure     │  ← প্রকৃত পরিমাপ
 │  (10000 runs)│     hrtime(true) দিয়ে
 └──────┬───────┘
        │
        ▼
 ┌─────────────┐
 │  Analyze     │  ← সর্ট, গড়, পার্সেন্টাইল
 │  Statistics  │     হিসাব করা
 └──────┬───────┘
        │
        ▼
 ┌─────────────┐
 │  Report      │  ← ফলাফল প্রদর্শন
 │  Results     │     avg, p50, p95, p99
 └─────────────┘
```

---

## ⚡ JavaScript বেঞ্চমার্কিং

JavaScript-এ বেঞ্চমার্কিং এর জন্য বেশ কয়েকটি জনপ্রিয় টুল আছে।

### Benchmark.js দিয়ে ফাংশন তুলনা

```javascript
const Benchmark = require('benchmark');
const suite = new Benchmark.Suite;

// Pathao-এর মতো ড্রাইভার ফিল্টারিং — দুটি পদ্ধতির তুলনা
const drivers = Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Driver ${i}`,
    rating: Math.random() * 5,
    available: Math.random() > 0.5
}));

suite
    .add('for loop', function() {
        let sum = 0;
        for (let i = 0; i < 1000; i++) sum += i;
    })
    .add('reduce', function() {
        Array.from({ length: 1000 }, (_, i) => i).reduce((a, b) => a + b, 0);
    })
    .add('filter available drivers', function() {
        drivers.filter(d => d.available && d.rating >= 4.0);
    })
    .add('for-loop available drivers', function() {
        const result = [];
        for (let i = 0; i < drivers.length; i++) {
            if (drivers[i].available && drivers[i].rating >= 4.0) {
                result.push(drivers[i]);
            }
        }
    })
    .on('cycle', function(event) {
        console.log(String(event.target));
    })
    .on('complete', function() {
        console.log('সবচেয়ে দ্রুত: ' + this.filter('fastest').map('name'));
    })
    .run({ async: true });
```

### performance.now() দিয়ে সরাসরি পরিমাপ

```javascript
// bKash ট্রানজাকশন প্রসেসিং সিমুলেশন
function measureExecution(fn, label, iterations = 10000) {
    const times = [];

    for (let i = 0; i < iterations; i++) {
        const start = performance.now();
        fn();
        times.push(performance.now() - start);
    }

    times.sort((a, b) => a - b);

    console.log(`\n=== ${label} ===`);
    console.log(`  গড়:  ${(times.reduce((a, b) => a + b) / times.length).toFixed(4)}ms`);
    console.log(`  p50:  ${times[Math.floor(times.length * 0.5)].toFixed(4)}ms`);
    console.log(`  p95:  ${times[Math.floor(times.length * 0.95)].toFixed(4)}ms`);
    console.log(`  p99:  ${times[Math.floor(times.length * 0.99)].toFixed(4)}ms`);
    console.log(`  min:  ${times[0].toFixed(4)}ms`);
    console.log(`  max:  ${times[times.length - 1].toFixed(4)}ms`);
}

// বিভিন্ন অবজেক্ট ক্লোনিং পদ্ধতি তুলনা
const obj = { user: 'Rahim', amount: 5000, type: 'send-money', ref: 'TXN123456' };

measureExecution(() => JSON.parse(JSON.stringify(obj)), 'JSON clone');
measureExecution(() => structuredClone(obj), 'structuredClone');
measureExecution(() => ({ ...obj }), 'Spread operator (shallow)');
```

### autocannon দিয়ে HTTP লোড টেস্ট

```bash
# Pathao API-তে লোড টেস্ট — ১০০ কানেকশন, ৩০ সেকেন্ড
npx autocannon -c 100 -d 30 http://localhost:3000/api/nearby-drivers

# bKash ট্রানজাকশন API — ৫০ কানেকশন, পাইপলাইন ১০
npx autocannon -c 50 -p 10 -d 60 http://localhost:3000/api/transactions

# Daraz প্রোডাক্ট সার্চ — কাস্টম হেডার সহ
npx autocannon -c 200 -d 30 \
  -H "Authorization=Bearer token123" \
  http://localhost:3000/api/products?search=phone
```

**autocannon আউটপুট উদাহরণ:**

```
┌─────────┬──────┬──────┬───────┬──────┬─────────┬─────────┬──────────┐
│ Stat    │ 2.5% │ 50%  │ 97.5% │ 99%  │ Avg     │ Stdev   │ Max      │
├─────────┼──────┼──────┼───────┼──────┼─────────┼─────────┼──────────┤
│ Latency │ 2 ms │ 5 ms │ 23 ms │ 45ms │ 7.34 ms │ 8.12 ms │ 234 ms   │
└─────────┴──────┴──────┴───────┴──────┴─────────┴─────────┴──────────┘
┌───────────┬─────────┬─────────┬────────┬─────────┬─────────┬────────┐
│ Stat      │ 1%      │ 2.5%    │ 50%    │ 97.5%   │ Avg     │ Max    │
├───────────┼─────────┼─────────┼────────┼─────────┼─────────┼────────┤
│ Req/Sec   │ 8,234   │ 9,123   │ 12,456 │ 14,321  │ 11,890  │ 15,234 │
└───────────┴─────────┴─────────┴────────┴─────────┴─────────┴────────┘
```

---

## 🔨 লোড টেস্টিং টুলস

প্রোডাকশনে deploy করার আগে সিস্টেমে লোড টেস্ট চালানো অত্যন্ত গুরুত্বপূর্ণ। নিচে জনপ্রিয় টুলগুলোর তুলনা:

### টুলস তুলনা টেবিল

| টুল        | ভাষা          | প্রোটোকল         | স্ক্রিপ্টিং   | বিশেষত্ব                        |
|------------|---------------|-------------------|---------------|----------------------------------|
| **k6**     | Go (JS API)   | HTTP, WS, gRPC   | JavaScript    | ডেভেলপার-বান্ধব, CI/CD ইন্টিগ্রেশন |
| **Artillery** | Node.js    | HTTP, WS, Socket  | YAML/JS       | সহজ কনফিগ, ক্লাউড রান সাপোর্ট    |
| **wrk**    | C             | HTTP              | Lua           | খুব হালকা, অত্যন্ত দ্রুত           |
| **ab**     | C             | HTTP              | CLI only      | সিম্পল, Apache সাথে আসে           |
| **JMeter** | Java          | HTTP, FTP, JDBC   | GUI/XML       | ফিচার-সমৃদ্ধ, ভারী                |
| **Locust** | Python        | HTTP              | Python        | ডিস্ট্রিবিউটেড টেস্ট সহজ           |

### k6 স্ক্রিপ্ট উদাহরণ — Grameenphone API লোড টেস্ট

```javascript
// load-test.js — k6 স্ক্রিপ্ট
import http from 'k6/http';
import { check, sleep } from 'k6';

// টেস্ট কনফিগারেশন — ধাপে ধাপে লোড বাড়ানো
export const options = {
    stages: [
        { duration: '30s', target: 50 },    // ৩০ সেকেন্ডে ৫০ ইউজার পর্যন্ত বাড়াও
        { duration: '1m',  target: 200 },   // ১ মিনিটে ২০০ ইউজার পর্যন্ত বাড়াও
        { duration: '2m',  target: 200 },   // ২ মিনিট ২০০ ইউজার ধরে রাখো
        { duration: '30s', target: 0 },     // ৩০ সেকেন্ডে ধীরে ধীরে কমাও
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],    // ৯৫% রিকোয়েস্ট ৫০০ms এর মধ্যে হতে হবে
        http_req_failed: ['rate<0.01'],      // ১% এর কম ফেইলিওর গ্রহণযোগ্য
    },
};

export default function () {
    // GP MyGP অ্যাপের মতো ব্যালেন্স চেক API
    const res = http.get('https://api.example.com/balance', {
        headers: {
            'Authorization': 'Bearer test-token-123',
            'X-Device-ID': 'android-device-001',
        },
    });

    // রেসপন্স ভ্যালিডেশন
    check(res, {
        'স্ট্যাটাস ২০০': (r) => r.status === 200,
        'রেসপন্স টাইম < ৫০০ms': (r) => r.timings.duration < 500,
        'ব্যালেন্স ফিল্ড আছে': (r) => JSON.parse(r.body).balance !== undefined,
    });

    sleep(1); // প্রতিটি virtual user ১ সেকেন্ড অপেক্ষা করবে
}
```

### k6 রান কমান্ড

```bash
# লোকাল রান
k6 run load-test.js

# কাস্টম VU এবং duration সহ
k6 run --vus 100 --duration 2m load-test.js

# JSON আউটপুট সহ (CI/CD এর জন্য)
k6 run --out json=results.json load-test.js
```

### লোড টেস্ট প্যাটার্ন

```
  VUs (ভার্চুয়াল ইউজার)
   │
200├─────────────────────────────────────────┐
   │                    ┌────────────────────┤
   │                   ╱                     │
   │                  ╱  স্থিতিশীল লোড        │
   │                 ╱   (Steady State)       ╲
50 ├────────────────╱                          ╲
   │              ╱                              ╲
   │            ╱  র‍্যাম্প আপ                     ╲ র‍্যাম্প ডাউন
   │          ╱   (Ramp Up)                        ╲
   │        ╱                                        ╲
   ├──────╱───────────────────────────────────────────╲──▶ সময়
   0     30s          1m30s          3m30s          4m
```

---

## 📈 রেজাল্ট ইন্টারপ্রেটেশন

বেঞ্চমার্কিং রেজাল্ট সঠিকভাবে পড়তে জানা খুব গুরুত্বপূর্ণ। শুধু "গড়" দেখলেই সব বোঝা যায় না।

### মূল পরিসংখ্যান (Key Metrics)

| মেট্রিক      | অর্থ                                              | উদাহরণ                                  |
|--------------|----------------------------------------------------|-----------------------------------------|
| **Latency**  | একটি রিকোয়েস্টের শুরু থেকে রেসপন্স পর্যন্ত সময়     | bKash Send Money → ২৫০ms                |
| **Throughput** | প্রতি সেকেন্ডে কতটি রিকোয়েস্ট হ্যান্ডেল করা হয়      | Daraz → ৫,০০০ RPS (Requests Per Second) |
| **p50**      | ৫০% রিকোয়েস্ট এই সময়ের মধ্যে সম্পন্ন হয়           | bKash p50 = ৫০ms                        |
| **p95**      | ৯৫% রিকোয়েস্ট এই সময়ের মধ্যে সম্পন্ন হয়           | bKash p95 = ২০০ms                       |
| **p99**      | ৯৯% রিকোয়েস্ট এই সময়ের মধ্যে সম্পন্ন হয়           | bKash p99 = ৮০০ms                       |
| **Error Rate** | মোট রিকোয়েস্টের কত শতাংশ ব্যর্থ হয়               | গ্রহণযোগ্য: < ০.১%                       |

### পার্সেন্টাইল ডিস্ট্রিবিউশন — bKash ইউজারদের জন্য কী অর্থ বহন করে

```
  রিকোয়েস্ট সংখ্যা
   ▲
   │
   │  ▓▓▓▓▓▓
   │  ▓▓▓▓▓▓▓▓
   │  ▓▓▓▓▓▓▓▓▓▓
   │  ▓▓▓▓▓▓▓▓▓▓▓▓
   │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
   │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
   │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░
   │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░▒▒▒▒▒▒
   └──────────────────────────────────────────────────▶ Latency (ms)
      │          │              │                │
     10ms      50ms           200ms            800ms
      ▲          ▲              ▲                ▲
    সর্বনিম্ন    p50            p95              p99

   ▓ = বেশিরভাগ bKash ইউজার (দ্রুত রেসপন্স পায়)
   ░ = কিছু ইউজার (একটু বেশি সময় লাগে)
   ▒ = অল্প কিছু ইউজার (সবচেয়ে বেশি সময় লাগে — tail latency)
```

### বাস্তব বিশ্লেষণ — bKash উদাহরণ

bKash-এ যদি প্রতিদিন **১ কোটি** ট্রানজাকশন হয়:

```
p50 = 50ms   → ৫০ লাখ ইউজার ৫০ms এর মধ্যে ট্রানজাকশন সম্পন্ন করে ✅
p95 = 200ms  → ৯৫ লাখ ইউজার ২০০ms এর মধ্যে সম্পন্ন করে           ✅
p99 = 800ms  → ৯৯ লাখ ইউজার ৮০০ms এর মধ্যে সম্পন্ন করে           ⚠️

কিন্তু বাকি ১% = ১ লাখ ইউজার!
এই ১ লাখ ইউজার ৮০০ms+ অপেক্ষা করছে — এটা অনেক বেশি!
```

> 💡 **শিক্ষা:** শুধু গড় (average) দেখবেন না। p95 এবং p99 দেখুন — কারণ সেখানেই
> ইউজার ড্রপ-অফ হয়। Nagad যদি p99 এ ভালো করে, bKash-এর ইউজার সেদিকে চলে যাবে!

### গড় (Average) কেন বিভ্রান্তিকর

```
  ❌ শুধু গড় দিয়ে বিচার

  রিকোয়েস্ট ১:  10ms  ─┐
  রিকোয়েস্ট ২:  12ms   │
  রিকোয়েস্ট ৩:  11ms   ├── গড় = 208.6ms ← বিভ্রান্তিকর!
  রিকোয়েস্ট ৪:  15ms   │
  রিকোয়েস্ট ৫:  9ms    │   বেশিরভাগ রিকোয়েস্ট ~10ms
  রিকোয়েস্ট ৬:  13ms   │   কিন্তু একটি outlier সব নষ্ট করেছে
  রিকোয়েস্ট ৭: 1390ms ─┘

  ✅ পার্সেন্টাইল দিয়ে বিচার

  p50 = 12ms    ← অর্ধেক রিকোয়েস্ট ১২ms এর মধ্যে (সত্যিকারের ছবি)
  p95 = 15ms    ← ৯৫% রিকোয়েস্ট ১৫ms এর মধ্যে
  p99 = 1390ms  ← কিছু outlier আছে — তদন্ত করুন!
```

---

## ❌ সাধারণ বেঞ্চমার্কিং ভুল

### ভুল ১: JIT/OPcache ওয়ার্মআপ না করা

```php
<?php
// ❌ Bad — প্রথম রান থেকেই মাপা শুরু
function badBenchmark(callable $fn, int $n = 1000): float {
    $start = hrtime(true);
    for ($i = 0; $i < $n; $i++) {
        $fn();
    }
    return (hrtime(true) - $start) / 1e6;
}

// ✅ Good — ওয়ার্মআপ দিয়ে OPcache/JIT সক্রিয় করা
function goodBenchmark(callable $fn, int $n = 1000, int $warmup = 100): float {
    // ওয়ার্মআপ — ফলাফল গণনায় আসবে না
    for ($i = 0; $i < $warmup; $i++) {
        $fn();
    }

    // এখন প্রকৃত পরিমাপ
    $start = hrtime(true);
    for ($i = 0; $i < $n; $i++) {
        $fn();
    }
    return (hrtime(true) - $start) / 1e6;
}
```

### ভুল ২: অবাস্তব ডেটা দিয়ে টেস্ট

```javascript
// ❌ Bad — ছোট ডেটা দিয়ে টেস্ট করে প্রোডাকশনের সিদ্ধান্ত নেওয়া
const testData = [1, 2, 3, 4, 5]; // মাত্র ৫টি আইটেম!
benchmark(() => testData.sort((a, b) => a - b));

// ✅ Good — প্রোডাকশনের কাছাকাছি ডেটা সাইজ ব্যবহার
// Daraz-এ সার্চ রেজাল্টে সাধারণত ১০,০০০+ প্রোডাক্ট থাকে
const realisticData = Array.from({ length: 10000 }, () => ({
    id: Math.random(),
    name: 'Product ' + Math.random().toString(36),
    price: Math.floor(Math.random() * 50000),
    rating: Math.random() * 5,
    reviews: Math.floor(Math.random() * 1000),
}));

benchmark(() => {
    realisticData
        .filter(p => p.price < 20000 && p.rating >= 4.0)
        .sort((a, b) => b.reviews - a.reviews)
        .slice(0, 20);
});
```

### ভুল ৩: GC (Garbage Collection) পজ ইগনোর করা

```javascript
// ❌ Bad — GC-এর প্রভাব বিবেচনা না করা
function badBench() {
    const start = performance.now();
    // প্রতি iteration-এ বড় অবজেক্ট তৈরি হচ্ছে
    for (let i = 0; i < 100000; i++) {
        const data = new Array(1000).fill({ key: 'value' });
    }
    return performance.now() - start;
}

// ✅ Good — একাধিকবার চালিয়ে GC স্পাইক শনাক্ত করা
function goodBench(iterations = 20) {
    const results = [];
    for (let run = 0; run < iterations; run++) {
        // প্রতি রানের আগে GC-কে সুযোগ দিন
        if (global.gc) global.gc();

        const start = performance.now();
        for (let i = 0; i < 100000; i++) {
            const data = new Array(1000).fill({ key: 'value' });
        }
        results.push(performance.now() - start);
    }

    results.sort((a, b) => a - b);
    // Outlier বাদ দিয়ে (trimmed mean) রিপোর্ট
    const trimmed = results.slice(2, -2);
    return trimmed.reduce((a, b) => a + b) / trimmed.length;
}

// চালাতে হবে: node --expose-gc bench.js
```

### ভুল ৪: টেস্ট এনভায়রনমেন্ট আইসোলেট না করা

```
❌ Bad                              ✅ Good
┌──────────────────────┐           ┌──────────────────────┐
│  বেঞ্চমার্ক চালানো     │           │  বেঞ্চমার্ক চালানো     │
│  + Chrome ওপেন        │           │                      │
│  + VS Code ওপেন       │           │  কোনো অন্য প্রসেস নেই  │
│  + Spotify বাজছে      │           │  CPU governor: perf  │
│  + Docker চলছে        │           │  Network isolated    │
│                       │           │  Fixed CPU frequency │
└───────────────────────┘           └──────────────────────┘
  ফলাফল: অসামঞ্জস্যপূর্ণ            ফলাফল: নির্ভরযোগ্য
```

### ভুল ৫: শুধু Wall Clock Time মাপা

```php
<?php
// ❌ Bad — শুধু সময় মাপা, মেমরি ইগনোর করা
$start = microtime(true);
$result = processLargeDataset($data);
$time = microtime(true) - $start;
echo "সময়: {$time}s\n";

// ✅ Good — সময়, মেমরি, এবং CPU সব মাপা
$memBefore = memory_get_usage(true);
$start = hrtime(true);

$result = processLargeDataset($data);

$time = (hrtime(true) - $start) / 1e9;
$memAfter = memory_get_usage(true);
$peakMem = memory_get_peak_usage(true);

echo sprintf(
    "সময়: %.4fs | মেমরি ব্যবহার: %s | পিক মেমরি: %s\n",
    $time,
    formatBytes($memAfter - $memBefore),
    formatBytes($peakMem)
);

function formatBytes(int $bytes): string {
    $units = ['B', 'KB', 'MB', 'GB'];
    $i = 0;
    while ($bytes >= 1024 && $i < count($units) - 1) {
        $bytes /= 1024;
        $i++;
    }
    return round($bytes, 2) . ' ' . $units[$i];
}
```

### সকল ভুলের সারসংক্ষেপ

```
  সাধারণ বেঞ্চমার্কিং ভুল              সমাধান
  ─────────────────────────────────────────────────────────
  ❌ ওয়ার্মআপ ছাড়াই মাপা        →  ✅ ১০০+ iteration ওয়ার্মআপ
  ❌ ছোট/ভুয়া ডেটা ব্যবহার       →  ✅ প্রোডাকশন-সদৃশ ডেটা
  ❌ GC পজ ইগনোর করা            →  ✅ একাধিক রান, outlier বাদ
  ❌ ব্যস্ত মেশিনে টেস্ট          →  ✅ আইসোলেটেড এনভায়রনমেন্ট
  ❌ শুধু wall clock মাপা         →  ✅ সময় + মেমরি + CPU মাপা
  ❌ একবার চালিয়ে সিদ্ধান্ত নেওয়া  →  ✅ পরিসংখ্যানগতভাবে বিশ্লেষণ
  ❌ শুধু গড় দেখা               →  ✅ p50, p95, p99 দেখা
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন বেঞ্চমার্কিং করবেন

```
১. নতুন ফিচার deploy করার আগে
   └── Daraz: নতুন সার্চ অ্যালগরিদম লাইভ করার আগে পারফরম্যান্স যাচাই

২. পারফরম্যান্স রিগ্রেশন ধরতে
   └── Pathao: নতুন রিলিজে ড্রাইভার ম্যাচিং ধীর হয়েছে কিনা

৩. টেকনোলজি সিদ্ধান্তে
   └── bKash: Redis vs Memcached — কোনটি ক্যাশিং-এ ভালো?

৪. ক্যাপাসিটি প্ল্যানিং-এ
   └── Nagad: ঈদের সময় কত সার্ভার লাগবে?

৫. SLA নির্ধারণে
   └── Grameenphone: "৯৯.৯% রিকোয়েস্ট ৫০০ms এর মধ্যে রেসপন্স দেবে"

৬. অপ্টিমাইজেশনের আগে ও পরে
   └── প্রমাণ করুন যে আপনার অপ্টিমাইজেশন আসলেই কাজ করেছে

৭. CI/CD পাইপলাইনে
   └── প্রতিটি PR-এ অটোমেটেড পারফরম্যান্স টেস্ট চালান
```

### ❌ কখন বেঞ্চমার্কিং করবেন না

```
১. Premature Optimization হিসেবে
   └── "এই ফাংশন হয়তো ধীর হতে পারে" — আগে প্রোফাইল করুন, তারপর বেঞ্চমার্ক

২. তুচ্ছ পার্থক্যের জন্য
   └── ❌ "for loop নাকি forEach — কোনটি ১ ns দ্রুত?"
       (readability বেশি গুরুত্বপূর্ণ)

৩. অবাস্তব পরিবেশে
   └── ❌ লোকাল ল্যাপটপে প্রোডাকশন লোড সিমুলেট করা
       (ফলাফল বিভ্রান্তিকর হবে)

৪. শুধু একবার চালিয়ে সিদ্ধান্ত নিতে
   └── পরিসংখ্যানগতভাবে (statistically) significant ফলাফল দরকার

৫. অন্যের বেঞ্চমার্ক অন্ধভাবে বিশ্বাস করতে
   └── ❌ "ইন্টারনেটে দেখলাম Library X দ্রুত"
       (আপনার workload আলাদা হতে পারে)
```

---

## 📖 বেঞ্চমার্কিং চেকলিস্ট

প্রোডাকশন বেঞ্চমার্কিং-এর আগে এই চেকলিস্ট অনুসরণ করুন:

```
বেঞ্চমার্কিং চেকলিস্ট
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□  লক্ষ্য নির্ধারণ করেছি (কী মাপতে চাই?)
□  প্রোডাকশন-সদৃশ ডেটা প্রস্তুত করেছি
□  এনভায়রনমেন্ট আইসোলেট করেছি
□  ওয়ার্মআপ পিরিয়ড রেখেছি
□  পর্যাপ্ত iteration চালাচ্ছি (>১০০০)
□  পার্সেন্টাইল রিপোর্ট করছি (শুধু গড় নয়)
□  মেমরি ও CPU-ও মাপছি
□  একাধিকবার চালিয়ে ফলাফল যাচাই করেছি
□  ফলাফল ডকুমেন্ট ও সংরক্ষণ করেছি
□  বেসলাইনের সাথে তুলনা করেছি
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 🔗 সম্পর্কিত প্যাটার্ন ও রিসোর্স

```
বেঞ্চমার্কিং
├── সম্পর্কিত প্যাটার্ন
│   ├── Profiling → বেঞ্চমার্কিং-এর আগে কোথায় সমস্যা তা জানুন
│   ├── Caching → বেঞ্চমার্ক দিয়ে ক্যাশ হিট/মিস রেশিও মাপুন
│   ├── Load Balancing → কতটুকু লোড সামলাতে পারে তা বেঞ্চমার্ক করুন
│   └── Auto Scaling → বেঞ্চমার্ক থেকে স্কেলিং threshold ঠিক করুন
│
├── টুলস ও লাইব্রেরি
│   ├── PHP → phpbench, blackfire.io
│   ├── JS → Benchmark.js, autocannon, Clinic.js
│   ├── লোড টেস্ট → k6, Artillery, wrk, ab, JMeter, Locust
│   └── মনিটরিং → Grafana, Datadog, New Relic
│
└── আরও পড়ুন
    ├── "Systems Performance" — Brendan Gregg
    ├── k6 ডকুমেন্টেশন → https://k6.io/docs
    └── Benchmark.js → https://benchmarkjs.com
```

---

> 💡 **মনে রাখবেন:** বেঞ্চমার্কিং একটি চলমান প্রক্রিয়া, এককালীন কাজ নয়। প্রতিটি রিলিজে
> বেঞ্চমার্ক চালান এবং পূর্ববর্তী ফলাফলের সাথে তুলনা করুন। Daraz, bKash, Pathao —
> সবাই প্রোডাকশনে continuous benchmarking করে যাতে ইউজার এক্সপেরিয়েন্স কখনো খারাপ না হয়।
