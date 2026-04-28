# 🚀 পারফরম্যান্স টেস্টিং (Performance Testing) — সম্পূর্ণ গাইড

## 📌 সংজ্ঞা ও মূল ধারণা

**পারফরম্যান্স টেস্টিং** হলো সফটওয়্যার টেস্টিং-এর একটি গুরুত্বপূর্ণ শাখা যেখানে সিস্টেমের গতি (speed), স্থিতিশীলতা (stability), স্কেলেবিলিটি (scalability) এবং রিসোর্স ব্যবহার (resource utilization) পরিমাপ করা হয়। এটি functional correctness যাচাই করে না — বরং নিশ্চিত করে যে সিস্টেম প্রত্যাশিত লোডে সঠিকভাবে কাজ করবে।

### পারফরম্যান্স টেস্টিং-এর প্রকারভেদ

```
╔══════════════════════════════════════════════════════════════════════════╗
║                  Performance Testing Types                              ║
╠══════════════════════════════════════════════════════════════════════════╣
║  1. Load Testing (লোড টেস্টিং)                                         ║
║     • প্রত্যাশিত লোডে সিস্টেম কেমন আচরণ করে তা পরীক্ষা               ║
║     • উদাহরণ: ১০০০ concurrent user handle করতে পারে কিনা               ║
║                                                                          ║
║  2. Stress Testing (স্ট্রেস টেস্টিং)                                    ║
║     • সিস্টেমের সীমার বাইরে লোড দিয়ে ভাঙনের পয়েন্ট খোঁজা             ║
║     • উদাহরণ: ধীরে ধীরে লোড বাড়িয়ে দেখা কোথায় ক্র্যাশ করে            ║
║                                                                          ║
║  3. Spike Testing (স্পাইক টেস্টিং)                                      ║
║     • হঠাৎ করে বিপুল পরিমাণ লোড আসলে কী হয়                             ║
║     • উদাহরণ: ফ্ল্যাশ সেলে হঠাৎ ট্রাফিক বেড়ে গেলে                     ║
║                                                                          ║
║  4. Soak/Endurance Testing (সোক টেস্টিং)                                ║
║     • দীর্ঘ সময় ধরে স্বাভাবিক লোডে চালিয়ে memory leak খোঁজা           ║
║     • উদাহরণ: ২৪ ঘণ্টা ধরে সিস্টেম চালিয়ে দেখা                        ║
║                                                                          ║
║  5. Benchmarking (বেঞ্চমার্কিং)                                          ║
║     • নির্দিষ্ট মেট্রিক্স পরিমাপ করে baseline তৈরি করা                  ║
║     • উদাহরণ: API response time, throughput পরিমাপ                       ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### 🔑 মূল Metrics (মেট্রিক্স)

| মেট্রিক | বর্ণনা | আদর্শ মান (উদাহরণ) |
|----------|--------|---------------------|
| **Response Time** | রিকোয়েস্ট পাঠানো থেকে রেসপন্স পাওয়া পর্যন্ত সময় | < 200ms (API), < 2s (page load) |
| **Throughput** | প্রতি সেকেন্ডে কতটি রিকোয়েস্ট handle হচ্ছে (RPS) | অ্যাপভেদে পরিবর্তনশীল |
| **Latency (p50, p95, p99)** | percentile ভিত্তিক response time | p99 < 500ms |
| **Error Rate** | মোট রিকোয়েস্টের মধ্যে কত শতাংশ ব্যর্থ | < 1% |
| **Concurrent Users** | একসাথে কতজন ব্যবহারকারী সিস্টেম ব্যবহার করছে | টার্গেট অনুযায়ী |
| **CPU/Memory Usage** | সার্ভারের রিসোর্স ব্যবহার | < 80% sustained |
| **Apdex Score** | Application Performance Index (0-1) | > 0.9 |

---

## 🏗️ বাস্তব উদাহরণ (Real-life Analogy)

একটি **রেস্তোরাঁ**-র কথা ভাবুন:

- **Load Testing** = সন্ধ্যায় ৫০ জন কাস্টমার একসাথে আসলে রান্নাঘর ও ওয়েটার কতটা দ্রুত সার্ভ করতে পারে।
- **Stress Testing** = ২০০ জন কাস্টমার একসাথে ঢুকলে কী হবে? রান্নাঘর কি ভেঙে পড়বে?
- **Spike Testing** = হঠাৎ বিয়ের অনুষ্ঠানের ১০০ জন গেস্ট এসে পড়লে কী হবে?
- **Soak Testing** = সারাদিন ক্রমাগত কাস্টমার আসতে থাকলে সন্ধ্যায় কি সার্ভিস কোয়ালিটি কমে যাবে? শেফ ক্লান্ত হয়ে পড়বে? (memory leak)
- **Benchmarking** = প্রতিটি অর্ডার গড়ে কত মিনিটে সার্ভ হচ্ছে তার পরিমাপ।

---

## 🔧 জনপ্রিয় টুলস

```
╔═══════════════════════════════════════════════════════════════╗
║  টুল          ║  ভাষা        ║  বৈশিষ্ট্য                    ║
╠═══════════════════════════════════════════════════════════════╣
║  k6            ║  JavaScript  ║  Developer-friendly, CI যোগ্য ║
║  JMeter        ║  Java/GUI    ║  পুরনো, শক্তিশালী, GUI আছে  ║
║  Artillery     ║  YAML/JS     ║  সহজ config, cloud-ready     ║
║  Locust        ║  Python      ║  Python-ভিত্তিক, distributed ║
║  Gatling       ║  Scala       ║  High performance, reports    ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## 🐘 PHP Benchmarking উদাহরণ

### সাধারণ Benchmarking ক্লাস

```php
<?php

declare(strict_types=1);

/**
 * একটি সিম্পল benchmarking ইউটিলিটি যা execution time ও memory usage ট্র্যাক করে
 */
class Benchmark
{
    private array $markers = [];
    private array $results = [];

    public function start(string $label): void
    {
        $this->markers[$label] = [
            'start_time'   => hrtime(true),
            'start_memory' => memory_get_usage(true),
        ];
    }

    public function end(string $label): array
    {
        if (!isset($this->markers[$label])) {
            throw new RuntimeException("Benchmark '{$label}' was never started.");
        }

        $elapsed  = (hrtime(true) - $this->markers[$label]['start_time']) / 1e6; // ms
        $memoryDelta = memory_get_usage(true) - $this->markers[$label]['start_memory'];

        $this->results[$label] = [
            'time_ms'    => round($elapsed, 4),
            'memory_bytes' => $memoryDelta,
            'memory_mb'  => round($memoryDelta / 1048576, 4),
        ];

        return $this->results[$label];
    }

    public function getResults(): array
    {
        return $this->results;
    }

    public function printReport(): void
    {
        echo str_repeat('=', 60) . PHP_EOL;
        echo "BENCHMARK REPORT" . PHP_EOL;
        echo str_repeat('=', 60) . PHP_EOL;

        foreach ($this->results as $label => $data) {
            printf(
                "%-30s | Time: %10.4f ms | Memory: %.4f MB\n",
                $label,
                $data['time_ms'],
                $data['memory_mb']
            );
        }
    }
}

// ব্যবহার উদাহরণ — দুইটি sorting algorithm-এর তুলনা
$bench = new Benchmark();
$data = range(1, 10000);
shuffle($data);

// Bubble Sort Benchmark
$arr = $data;
$bench->start('bubble_sort');
for ($i = 0; $i < count($arr); $i++) {
    for ($j = 0; $j < count($arr) - $i - 1; $j++) {
        if ($arr[$j] > $arr[$j + 1]) {
            [$arr[$j], $arr[$j + 1]] = [$arr[$j + 1], $arr[$j]];
        }
    }
}
$bench->end('bubble_sort');

// Built-in Sort Benchmark
$arr = $data;
$bench->start('builtin_sort');
sort($arr);
$bench->end('builtin_sort');

$bench->printReport();
```

### PHPBench দিয়ে Benchmarking

```php
<?php

use PhpBench\Attributes as Bench;

#[Bench\Iterations(5)]
#[Bench\Revs(1000)]
class JsonSerializationBench
{
    private array $payload;

    public function __construct()
    {
        $this->payload = [
            'users' => array_map(fn($i) => [
                'id'    => $i,
                'name'  => "User {$i}",
                'email' => "user{$i}@example.com",
                'roles' => ['admin', 'editor'],
            ], range(1, 100)),
        ];
    }

    #[Bench\Subject]
    public function benchJsonEncode(): void
    {
        json_encode($this->payload);
    }

    #[Bench\Subject]
    public function benchSerialize(): void
    {
        serialize($this->payload);
    }

    #[Bench\Subject]
    public function benchMsgpack(): void
    {
        if (function_exists('msgpack_pack')) {
            msgpack_pack($this->payload);
        }
    }
}

// চালানোর কমান্ড: vendor/bin/phpbench run --report=default
```

---

## 🟨 JavaScript / k6 স্ক্রিপ্ট উদাহরণ

### k6 দিয়ে Load Testing

```javascript
// load-test.js — k6 দিয়ে একটি API-তে load test
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// কাস্টম মেট্রিক্স
const errorRate = new Rate('errors');
const apiDuration = new Trend('api_duration', true);

// টেস্ট কনফিগারেশন — বিভিন্ন stage-এ লোড বাড়ানো/কমানো
export const options = {
    stages: [
        { duration: '2m', target: 50 },   // ধীরে ধীরে ৫০ user-এ উঠানো (ramp-up)
        { duration: '5m', target: 50 },   // ৫ মিনিট ৫০ user-এ ধরে রাখা
        { duration: '2m', target: 100 },  // ১০০ user-এ বাড়ানো
        { duration: '5m', target: 100 },  // ৫ মিনিট ১০০ user-এ ধরে রাখা
        { duration: '2m', target: 0 },    // ধীরে ধীরে শূন্যে নামানো (ramp-down)
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // ৯৫% রিকোয়েস্ট ৫০০ms-এর মধ্যে
        http_req_failed: ['rate<0.01'],    // error rate ১%-এর কম
        errors: ['rate<0.05'],             // কাস্টম error rate ৫%-এর কম
    },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';

export default function () {
    group('API Endpoints Load Test', () => {
        // GET — ব্যবহারকারীদের তালিকা আনা
        group('GET /users', () => {
            const listRes = http.get(`${BASE_URL}/api/users?page=1&limit=20`, {
                headers: { 'Accept': 'application/json' },
            });

            check(listRes, {
                'status is 200': (r) => r.status === 200,
                'response time < 300ms': (r) => r.timings.duration < 300,
                'has users array': (r) => JSON.parse(r.body).data?.length > 0,
            });

            errorRate.add(listRes.status !== 200);
            apiDuration.add(listRes.timings.duration);
        });

        // POST — নতুন ব্যবহারকারী তৈরি
        group('POST /users', () => {
            const payload = JSON.stringify({
                name: `LoadTestUser_${Date.now()}`,
                email: `test_${Date.now()}@example.com`,
            });

            const createRes = http.post(`${BASE_URL}/api/users`, payload, {
                headers: { 'Content-Type': 'application/json' },
            });

            check(createRes, {
                'created successfully': (r) => r.status === 201,
                'response time < 500ms': (r) => r.timings.duration < 500,
            });

            errorRate.add(createRes.status !== 201);
        });
    });

    sleep(1); // প্রতিটি virtual user প্রতি iteration-এ ১ সেকেন্ড অপেক্ষা
}

// চালানোর কমান্ড:
// k6 run --env BASE_URL=https://api.staging.example.com load-test.js
// k6 run --out json=results.json load-test.js
```

### Artillery দিয়ে Load Test (YAML Config)

```yaml
# artillery-config.yml
config:
  target: "https://api.example.com"
  phases:
    - duration: 120
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      name: "Sustained load"
    - duration: 60
      arrivalRate: 100
      name: "Spike"
  defaults:
    headers:
      Content-Type: "application/json"
  plugins:
    expect: {}

scenarios:
  - name: "Browse and Create"
    flow:
      - get:
          url: "/api/products"
          expect:
            - statusCode: 200
            - hasProperty: "data"
      - think: 2
      - post:
          url: "/api/orders"
          json:
            productId: 1
            quantity: 2
          expect:
            - statusCode: 201
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন দরকার |
|-----------|----------|
| প্রোডাকশনে যাওয়ার আগে | baseline performance নিশ্চিত করতে |
| নতুন ফিচার deploy-এর পর | performance regression ধরতে |
| ইনফ্রাস্ট্রাকচার পরিবর্তন করলে | নতুন সার্ভার/DB ঠিকমতো কাজ করছে কিনা |
| ব্ল্যাক ফ্রাইডে/বড় ইভেন্টের আগে | spike traffic সামলাতে পারবে কিনা |
| SLA নির্ধারণ করতে | সিস্টেমের সক্ষমতা সম্পর্কে সঠিক তথ্য পেতে |
| CI/CD pipeline-এ | প্রতিটি deploy-এ automated performance check |

## ❌ কখন করবেন না / সীমাবদ্ধতা

| পরিস্থিতি | কারণ |
|-----------|------|
| প্রোডাকশন environment-এ সরাসরি | আসল ব্যবহারকারীদের সমস্যা হবে |
| Functional bug থাকা অবস্থায় | আগে সঠিকতা নিশ্চিত করুন, তারপর গতি |
| অবাস্তব লোড scenario দিয়ে | বাস্তবসম্মত ট্রাফিক প্যাটার্ন ব্যবহার করুন |
| শুধু response time দিয়ে বিচার করলে | সব মেট্রিক্স একসাথে বিশ্লেষণ করুন |
| একবার চালিয়ে ভুলে গেলে | নিয়মিত চালাতে হবে, CI-তে যুক্ত করুন |
| Shared dev environment-এ | অন্যদের কাজে ব্যাঘাত ঘটাবে |

---

## 🧠 সিনিয়র ইঞ্জিনিয়ারদের জন্য পরামর্শ

১. **Percentile-ভিত্তিক চিন্তা করুন** — average response time প্রায়ই বিভ্রান্তিকর। p95 এবং p99 দেখুন কারণ সেগুলো worst-case user experience দেখায়।

২. **Production-like environment-এ টেস্ট করুন** — staging server যদি production-এর চেয়ে অনেক ছোট হয় তাহলে ফলাফল অর্থহীন।

৩. **Think time ব্যবহার করুন** — বাস্তব ব্যবহারকারীরা প্রতিটি ক্লিকের মাঝে কিছু সময় ব্যয় করে। এটি ছাড়া টেস্ট অবাস্তব হবে।

৪. **Correlation ID ব্যবহার করুন** — load test-এর সময় প্রতিটি request track করতে correlation/trace ID পাঠান। এতে bottleneck খুঁজে বের করা সহজ হয়।

৫. **Baseline তৈরি করুন** — প্রথমে বর্তমান performance baseline রেকর্ড করুন, তারপর প্রতিটি পরিবর্তনের পর তুলনা করুন।

৬. **Database-ও মনিটর করুন** — শুধু API response time না, DB query time, connection pool, slow queries — সব মনিটর করুন।

> **মনে রাখবেন**: "Performance testing শুধু ভাঙার জন্য নয় — সিস্টেমকে ভালোভাবে চেনার জন্য।" 🎯
