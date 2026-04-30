# ⏱️ টাইম-সিরিজ ডাটাবেস (Time-Series Database)

> টাইম-সিরিজ ডাটাবেস সময়ের সাথে সম্পর্কিত ডেটা সংরক্ষণ ও বিশ্লেষণে বিশেষায়িত।
> সার্ভার মনিটরিং, IoT সেন্সর ডেটা, আর্থিক ডেটা — এসব ক্ষেত্রে RDBMS এর চেয়ে অনেক বেশি কার্যকর।

---

## 📖 সূচিপত্র

- [টাইম-সিরিজ ডাটাবেস কী?](#-টাইম-সিরিজ-ডাটাবেস-কী)
- [RDBMS vs TSDB](#-rdbms-vs-tsdb)
- [ডেটা মডেল](#-ডেটা-মডেল)
- [প্রধান TSDB সমূহ](#-প্রধান-tsdb-সমূহ)
- [Retention Policy ও Downsampling](#-retention-policy-ও-downsampling)
- [Use Cases](#-use-cases)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 টাইম-সিরিজ ডাটাবেস কী?

টাইম-সিরিজ ডেটা হলো সময়ের সাথে সংগৃহীত ক্রমাগত ডেটা পয়েন্ট। TSDB এই ধরনের ডেটা সংরক্ষণ ও কোয়েরির জন্য অপটিমাইজড।

```
টাইম-সিরিজ ডেটা কেমন দেখায়:
──────────────────────────────

সময়             │ মান
─────────────────┼──────
2024-01-01 00:00 │ 45.2°C  ← bKash সার্ভার CPU তাপমাত্রা
2024-01-01 00:01 │ 46.1°C
2024-01-01 00:02 │ 44.8°C
2024-01-01 00:03 │ 50.3°C  ← স্পাইক! 📈
2024-01-01 00:04 │ 48.7°C
2024-01-01 00:05 │ 47.2°C
...
2024-12-31 23:59 │ 43.1°C

প্রতি মিনিটে = ৫২৫,৬০০ ডেটা পয়েন্ট/বছর (একটি সেন্সর!)
১০০ সার্ভার = ৫২.৫ মিলিয়ন ডেটা পয়েন্ট/বছর!

গ্রাফে:
  Temperature
  50°C │         ╱╲
  48°C │   ╱╲  ╱    ╲    ╱╲
  46°C │  ╱  ╲╱      ╲  ╱  ╲
  44°C │╱              ╲╱    ╲
  42°C │                      ╲
       └───────────────────────────► সময়
       00:00  00:01  00:02  00:03
```

### টাইম-সিরিজ ডেটার বৈশিষ্ট্য

```
┌─────────────────────────────────────────────────────────┐
│           টাইম-সিরিজ ডেটার বিশেষ বৈশিষ্ট্য               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ Time-ordered: সবসময় সময়ের ক্রমানুসারে                │
│  ✅ Append-only: নতুন ডেটা যোগ হয়, পুরানোটা পরিবর্তন হয় না│
│  ✅ High write throughput: প্রতি সেকেন্ডে হাজার হাজার write│
│  ✅ Recent data bias: সাম্প্রতিক ডেটা বেশি query হয়       │
│  ✅ Time-range queries: "গত ১ ঘণ্টায় CPU usage?"          │
│  ✅ Aggregation heavy: AVG, MAX, MIN, SUM, Percentile    │
│  ❌ Rarely updated: ডেটা insert হয়, আপডেট হয় না          │
│  ❌ Rarely deleted: পুরানো ডেটা automatically expire হয়    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📊 RDBMS vs TSDB

```
┌────────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য       │     RDBMS        │     TSDB         │
│                    │   (MySQL/PG)     │  (InfluxDB/etc)  │
├────────────────────┼──────────────────┼──────────────────┤
│ Write Speed        │ ~10K/sec         │ ~1M+/sec         │
│ (insert rate)      │                  │ 100x faster!     │
├────────────────────┼──────────────────┼──────────────────┤
│ Storage            │ Row-based        │ Column-based     │
│                    │ (বেশি জায়গা)     │ (10x কম্প্রেশন)   │
├────────────────────┼──────────────────┼──────────────────┤
│ Time-range Query   │ Slow (B-Tree     │ Ultra Fast       │
│                    │ scan)            │ (Time-indexed)   │
├────────────────────┼──────────────────┼──────────────────┤
│ Aggregation        │ GROUP BY (slow)  │ Built-in (fast)  │
│ (AVG, MAX, MIN)    │                  │                  │
├────────────────────┼──────────────────┼──────────────────┤
│ Data Retention     │ Manual DELETE    │ Auto-expiry      │
├────────────────────┼──────────────────┼──────────────────┤
│ Downsampling       │ Manual ETL       │ Built-in         │
├────────────────────┼──────────────────┼──────────────────┤
│ UPDATE/DELETE      │ ✅ দ্রুত          │ ❌ ধীর/সীমিত      │
├────────────────────┼──────────────────┼──────────────────┤
│ JOIN               │ ✅ আছে           │ ❌ নেই/সীমিত      │
├────────────────────┼──────────────────┼──────────────────┤
│ Transaction        │ ✅ ACID          │ ❌ সীমিত          │
├────────────────────┼──────────────────┼──────────────────┤
│ Use Case           │ Business Data,   │ Metrics, Logs,   │
│                    │ Users, Orders    │ IoT, Finance     │
└────────────────────┴──────────────────┴──────────────────┘
```

### কেন RDBMS তে টাইম-সিরিজ ডেটা সমস্যা?

```
❌ MySQL এ সার্ভার মেট্রিক্স রাখার সমস্যা:
──────────────────────────────────────────

CREATE TABLE server_metrics (
    id BIGINT AUTO_INCREMENT,
    server_name VARCHAR(50),
    metric_name VARCHAR(50),
    value DOUBLE,
    timestamp DATETIME,
    PRIMARY KEY (id),
    INDEX idx_time (timestamp),
    INDEX idx_server (server_name, timestamp)
);

সমস্যা ১: INSERT পারফরম্যান্স
  ১০০ সার্ভার × ১০ মেট্রিক × ১/সেকেন্ড = ১,০০০ INSERT/sec
  ❌ B-Tree index আপডেট ধীর হয়ে যায়

সমস্যা ২: Storage
  ১ বছরে ~৩১.৫ বিলিয়ন rows
  ❌ ডিস্ক স্পেস ও ব্যাকআপ সমস্যা

সমস্যা ৩: Query পারফরম্যান্স
  SELECT AVG(value) FROM server_metrics
  WHERE server_name = 'bkash-api-01'
  AND metric_name = 'cpu_usage'
  AND timestamp BETWEEN '2024-01-01' AND '2024-01-31';
  ❌ ৩১ দিনের ডেটা scan = ধীর!

✅ TSDB তে এই সমস্যা নেই:
  - Columnar storage (10x কম জায়গা)
  - Time-based indexing (দ্রুত range query)
  - Auto-retention (পুরানো ডেটা automatically মুছে যায়)
```

---

## 📖 ডেটা মডেল

### InfluxDB Data Model

```
┌─────────────────────────────────────────────────────────┐
│               InfluxDB ডেটা মডেল                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Measurement (টেবিলের মতো): "cpu_usage"                 │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ time          │ tags           │ fields           │  │
│  │ (timestamp)   │ (indexed)      │ (not indexed)    │  │
│  ├───────────────┼────────────────┼──────────────────┤  │
│  │ 2024-01-01    │ server=bkash1  │ usage=45.2       │  │
│  │ 00:00:00      │ region=dhaka   │ idle=54.8        │  │
│  ├───────────────┼────────────────┼──────────────────┤  │
│  │ 2024-01-01    │ server=bkash2  │ usage=62.1       │  │
│  │ 00:00:00      │ region=ctg     │ idle=37.9        │  │
│  ├───────────────┼────────────────┼──────────────────┤  │
│  │ 2024-01-01    │ server=bkash1  │ usage=47.8       │  │
│  │ 00:01:00      │ region=dhaka   │ idle=52.2        │  │
│  └───────────────┴────────────────┴──────────────────┘  │
│                                                         │
│  Tags: মেটাডেটা, indexed, query filter এ ব্যবহৃত        │
│  Fields: আসল মান, not indexed, aggregation এ ব্যবহৃত    │
│  Series: Measurement + Tag set এর ইউনিক combination     │
│                                                         │
└─────────────────────────────────────────────────────────┘

Line Protocol (InfluxDB তে ডেটা লেখার ফরম্যাট):
─────────────────────────────────────────────────
cpu_usage,server=bkash1,region=dhaka usage=45.2,idle=54.8 1704067200000000000
│         │                         │                     │
│         └─ Tags                   └─ Fields             └─ Timestamp (ns)
└─ Measurement
```

### Prometheus Data Model

```
┌─────────────────────────────────────────────────────────┐
│              Prometheus ডেটা মডেল                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Metric Types:                                          │
│                                                         │
│  1. Counter (শুধু বাড়ে):                                 │
│     http_requests_total{method="GET", status="200"} 1234│
│     ▲                                                   │
│     │    ╱─────────                                     │
│     │   ╱                                               │
│     │  ╱                                                │
│     │ ╱                                                 │
│     └──────────────► সময়                                │
│                                                         │
│  2. Gauge (উপরে-নিচে যায়):                               │
│     cpu_temperature{server="bkash-api-01"} 45.2         │
│     ▲                                                   │
│     │    ╱╲    ╱╲                                       │
│     │   ╱  ╲  ╱  ╲                                     │
│     │  ╱    ╲╱    ╲                                     │
│     └──────────────► সময়                                │
│                                                         │
│  3. Histogram (বিন্যাস):                                  │
│     http_request_duration_seconds_bucket{le="0.1"} 500  │
│     http_request_duration_seconds_bucket{le="0.5"} 800  │
│     http_request_duration_seconds_bucket{le="1.0"} 950  │
│                                                         │
│  4. Summary (Percentile):                               │
│     http_request_duration{quantile="0.5"} 0.123         │
│     http_request_duration{quantile="0.99"} 0.456        │
│                                                         │
└─────────────────────────────────────────────────────────┘

Labels (Tags এর মতো):
  metric_name{label1="value1", label2="value2"} value

উদাহরণ:
  bkash_transactions_total{type="send_money", status="success"} 1523456
  bkash_transactions_total{type="send_money", status="failed"} 234
  bkash_transactions_total{type="cash_out", status="success"} 987654
```

---

## 📊 প্রধান TSDB সমূহ

```
┌──────────────┬────────────────┬───────────────┬──────────────────┐
│              │   InfluxDB     │  TimescaleDB  │   Prometheus     │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ ভিত্তি        │ Custom Engine  │ PostgreSQL    │ Custom Engine    │
│              │                │ Extension     │                  │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ Query Lang.  │ Flux /         │ SQL!          │ PromQL           │
│              │ InfluxQL       │ (পরিচিত!)     │                  │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ Storage      │ TSM/TSI        │ Hypertables   │ Local TSDB       │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ Best For     │ IoT, Metrics   │ SQL + TS      │ Monitoring       │
│              │                │ Hybrid        │ & Alerting       │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ Clustering   │ ✅ (Enterprise)│ ✅            │ ✅ (Thanos/      │
│              │                │               │  Cortex)         │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ Pull/Push    │ Push           │ Push          │ Pull (scrape)    │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ সুবিধা       │ দ্রুত write,    │ SQL জানলেই    │ Kubernetes       │
│              │ ফ্লেক্সিবল     │ পারবেন        │ native, alert    │
├──────────────┼────────────────┼───────────────┼──────────────────┤
│ সীমাবদ্ধতা   │ High cardin.   │ PG overhead   │ Long-term        │
│              │ সমস্যা         │               │ storage সীমিত    │
└──────────────┴────────────────┴───────────────┴──────────────────┘
```

### আর্কিটেকচার তুলনা

```
Prometheus (Pull Model):
────────────────────────
┌──────────┐     Scrape     ┌──────────────┐
│ bKash    │ ◄───────────── │  Prometheus  │
│ API      │  /metrics      │              │
│ Server   │  (প্রতি ১৫s)   │  ┌─────────┐ │
└──────────┘                │  │  TSDB   │ │
                            │  └─────────┘ │
┌──────────┐     Scrape     │       │      │
│ Pathao   │ ◄───────────── │  ┌────▼────┐ │
│ Ride     │  /metrics      │  │ AlertMgr│ │
│ Server   │                │  └─────────┘ │
└──────────┘                └──────┬───────┘
                                   │ PromQL
                            ┌──────▼───────┐
                            │   Grafana    │
                            │  Dashboard   │
                            └──────────────┘

InfluxDB (Push Model):
──────────────────────
┌──────────┐   Push Data    ┌──────────────┐
│ bKash    │ ──────────────►│  InfluxDB    │
│ API      │  HTTP POST     │              │
│ Server   │                │  ┌─────────┐ │
└──────────┘                │  │  TSM    │ │
                            │  │ Storage │ │
┌──────────┐   Push Data    │  └─────────┘ │
│ IoT      │ ──────────────►│       │      │
│ Sensors  │  Telegraf      │  ┌────▼────┐ │
│          │                │  │  Tasks  │ │
└──────────┘                │  │(Alerts) │ │
                            │  └─────────┘ │
                            └──────┬───────┘
                                   │ Flux
                            ┌──────▼───────┐
                            │   Grafana    │
                            └──────────────┘
```

---

## 📊 Retention Policy ও Downsampling

### Retention Policy

```
ডেটা কতদিন রাখবেন:
──────────────────

┌──────────────────────────────────────────────────────┐
│             Retention Policy Strategy                 │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Raw Data (প্রতি সেকেন্ড):                             │
│  ├── রাখুন: ৭ দিন                                     │
│  └── সাইজ: ~100GB                                    │
│                                                      │
│  1-Minute Average:                                   │
│  ├── রাখুন: ৩০ দিন                                    │
│  └── সাইজ: ~2GB (60x কম!)                            │
│                                                      │
│  1-Hour Average:                                     │
│  ├── রাখুন: ১ বছর                                     │
│  └── সাইজ: ~50MB                                     │
│                                                      │
│  1-Day Average:                                      │
│  ├── রাখুন: ৫ বছর                                     │
│  └── সাইজ: ~5MB                                      │
│                                                      │
└──────────────────────────────────────────────────────┘

সময়ের সাথে ডেটা "ঘনত্ব" কমে:

  Resolution  ▲
  (ডেটা ঘনত্ব) │  ■■■■■■■
              │  ■■■■■■■
              │  ■■■■■■■   ▓▓▓▓▓
              │  ■■■■■■■   ▓▓▓▓▓
              │  ■■■■■■■   ▓▓▓▓▓   ░░░░
              │  ■■■■■■■   ▓▓▓▓▓   ░░░░   ○○
              └──────────────────────────────► সময়
                 ৭ দিন      ৩০ দিন   ১ বছর   ৫ বছর
                 Raw(1s)   1min avg  1hr avg  1day avg
```

### Downsampling

```
Downsampling = উচ্চ-রেজোলিউশন ডেটা থেকে নিম্ন-রেজোলিউশন ডেটা তৈরি

Raw Data (প্রতি সেকেন্ড):
time     │ cpu_usage
─────────┼──────────
00:00:00 │ 45.2
00:00:01 │ 46.1
00:00:02 │ 44.8
00:00:03 │ 47.5
...      │ ...
00:00:59 │ 43.9

     ▼ Downsample (1-minute average)

1-Minute Average:
time     │ cpu_avg │ cpu_max │ cpu_min
─────────┼─────────┼─────────┼────────
00:00:00 │ 45.5    │ 47.5    │ 43.9

     ▼ Downsample (1-hour average)

1-Hour Average:
time     │ cpu_avg │ cpu_max │ cpu_min
─────────┼─────────┼─────────┼────────
00:00:00 │ 46.2    │ 62.1    │ 40.3
```

---

## 🎯 Use Cases

```
┌──────────────────┬───────────────────────────────────────┐
│    Use Case      │       বাংলাদেশী উদাহরণ                 │
├──────────────────┼───────────────────────────────────────┤
│ সার্ভার মনিটরিং  │ bKash API সার্ভারের CPU, RAM,          │
│                  │ Network throughput ট্র্যাকিং           │
├──────────────────┼───────────────────────────────────────┤
│ অ্যাপ পারফরম্যান্স│ Pathao অ্যাপের response time,          │
│ (APM)            │ error rate, request count             │
├──────────────────┼───────────────────────────────────────┤
│ IoT/সেন্সর       │ বাংলাদেশ আবহাওয়া অধিদপ্তরের           │
│                  │ তাপমাত্রা, বৃষ্টিপাত, আর্দ্রতা ডেটা     │
├──────────────────┼───────────────────────────────────────┤
│ আর্থিক ডেটা      │ Nagad/bKash ট্রানজ্যাকশন volume,       │
│                  │ BDT/USD exchange rate                 │
├──────────────────┼───────────────────────────────────────┤
│ নেটওয়ার্ক         │ Grameenphone নেটওয়ার্ক latency,        │
│ মনিটরিং          │ bandwidth, packet loss                │
├──────────────────┼───────────────────────────────────────┤
│ ব্যবসা মেট্রিক্স   │ Daraz এ প্রতি ঘণ্টায় অর্ডার সংখ্যা,     │
│                  │ GMV (Gross Merchandise Value)         │
├──────────────────┼───────────────────────────────────────┤
│ ডিভাইস           │ GP 4G/5G টাওয়ার signal strength,      │
│ টেলিমেট্রি        │ connected users                      │
└──────────────────┴───────────────────────────────────────┘
```

---

## 💻 কোড উদাহরণ

### InfluxDB (PHP)

```php
<?php
// composer require influxdata/influxdb-client-php

use InfluxDB2\Client;
use InfluxDB2\Model\WritePrecision;
use InfluxDB2\Point;

// =============================================
// InfluxDB PHP Client
// বাস্তব উদাহরণ: bKash সার্ভার মনিটরিং
// =============================================

$client = new Client([
    'url'    => 'http://localhost:8086',
    'token'  => 'my-super-secret-token',
    'org'    => 'bkash',
    'bucket' => 'server_metrics',
]);

// --- ডেটা লেখা (Write) ---
$writeApi = $client->createWriteApi();

// Point builder ব্যবহার করে
$point = Point::measurement('cpu_usage')
    ->addTag('server', 'bkash-api-01')
    ->addTag('region', 'dhaka')
    ->addTag('env', 'production')
    ->addField('usage_percent', 45.2)
    ->addField('idle_percent', 54.8)
    ->addField('load_avg_1m', 2.34)
    ->time(microtime(true));

$writeApi->write($point);

// Line Protocol ব্যবহার করে (bulk write)
$lines = [
    'cpu_usage,server=bkash-api-01,region=dhaka usage_percent=45.2,idle_percent=54.8',
    'memory_usage,server=bkash-api-01,region=dhaka used_gb=12.5,total_gb=32.0',
    'http_requests,server=bkash-api-01,region=dhaka count=1523,error_count=12',
    'transaction_rate,service=send_money success=450,failed=3,avg_ms=120.5',
];

foreach ($lines as $line) {
    $writeApi->write($line);
}

$writeApi->close();
echo "✅ ডেটা লেখা সফল!\n";

// --- ডেটা পড়া (Query) ---
$queryApi = $client->createQueryApi();

// গত ১ ঘণ্টায় গড় CPU usage
$query = '
from(bucket: "server_metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "cpu_usage")
  |> filter(fn: (r) => r["server"] == "bkash-api-01")
  |> filter(fn: (r) => r["_field"] == "usage_percent")
  |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
  |> yield(name: "mean")
';

$result = $queryApi->query($query);

foreach ($result as $table) {
    foreach ($table->records as $record) {
        echo sprintf(
            "সময়: %s | গড় CPU: %.1f%%\n",
            $record->getTime(),
            $record->getValue()
        );
    }
}

// সর্বোচ্চ CPU usage (গত ২৪ ঘণ্টায়)
$maxQuery = '
from(bucket: "server_metrics")
  |> range(start: -24h)
  |> filter(fn: (r) => r["_measurement"] == "cpu_usage")
  |> filter(fn: (r) => r["_field"] == "usage_percent")
  |> group(columns: ["server"])
  |> max()
';

$maxResult = $queryApi->query($maxQuery);
foreach ($maxResult as $table) {
    foreach ($table->records as $record) {
        $server = $record->values['server'];
        echo "সর্বোচ্চ CPU: $server = {$record->getValue()}%\n";
    }
}

$client->close();
```

### InfluxDB (Node.js)

```javascript
// npm install @influxdata/influxdb-client

// =============================================
// InfluxDB Node.js Client
// বাস্তব উদাহরণ: Pathao রাইড মেট্রিক্স
// =============================================

const { InfluxDB, Point } = require('@influxdata/influxdb-client');

const client = new InfluxDB({
    url: 'http://localhost:8086',
    token: 'my-super-secret-token'
});

const org = 'pathao';
const bucket = 'ride_metrics';

// --- ডেটা লেখা ---
const writeApi = client.getWriteApi(org, bucket, 'ms');

// রাইড মেট্রিক্স পাঠানো
function recordRideMetric(rideId, riderId, data) {
    const point = new Point('ride_metrics')
        .tag('ride_id', rideId)
        .tag('rider_id', riderId)
        .tag('city', data.city)
        .tag('vehicle_type', data.vehicleType)
        .floatField('distance_km', data.distanceKm)
        .floatField('duration_min', data.durationMin)
        .floatField('fare_bdt', data.fareBdt)
        .floatField('rider_rating', data.riderRating)
        .intField('surge_multiplier', data.surgeMultiplier || 1);
    
    writeApi.writePoint(point);
}

// উদাহরণ রাইড ডেটা
recordRideMetric('RIDE-001', 'RIDER-42', {
    city: 'dhaka',
    vehicleType: 'bike',
    distanceKm: 5.2,
    durationMin: 18.5,
    fareBdt: 120,
    riderRating: 4.8,
    surgeMultiplier: 1
});

// সিস্টেম মেট্রিক্স (প্রতি ১০ সেকেন্ড)
setInterval(() => {
    const sysPoint = new Point('system_metrics')
        .tag('service', 'ride-service')
        .tag('instance', 'ride-01')
        .floatField('cpu_usage', Math.random() * 80 + 10)
        .floatField('memory_mb', Math.random() * 2048 + 512)
        .intField('active_connections', Math.floor(Math.random() * 500))
        .intField('requests_per_sec', Math.floor(Math.random() * 1000));
    
    writeApi.writePoint(sysPoint);
}, 10000);

// Flush
writeApi.flush().then(() => {
    console.log('✅ ডেটা লেখা সফল!');
});

// --- ডেটা পড়া ---
const queryApi = client.getQueryApi(org);

// গত ১ ঘণ্টায় শহর অনুযায়ী রাইড সংখ্যা ও গড় fare
const rideQuery = `
from(bucket: "${bucket}")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "ride_metrics")
  |> filter(fn: (r) => r["_field"] == "fare_bdt")
  |> group(columns: ["city"])
  |> aggregateWindow(every: 15m, fn: mean, createEmpty: false)
`;

console.log('\n📊 শহর অনুযায়ী গড় ভাড়া (গত ১ ঘণ্টা):');
queryApi.queryRows(rideQuery, {
    next(row, tableMeta) {
        const o = tableMeta.toObject(row);
        console.log(`  ${o.city}: ${o._time} - গড় ভাড়া: ৳${o._value.toFixed(0)}`);
    },
    error(error) {
        console.error('Query error:', error);
    },
    complete() {
        console.log('Query সম্পন্ন');
    }
});

// Prometheus-style PromQL (via Prometheus)
// rate(http_requests_total{service="ride"}[5m])
```

### TimescaleDB (SQL!)

```sql
-- =============================================
-- TimescaleDB (PostgreSQL Extension)
-- বাস্তব উদাহরণ: Daraz সার্ভার মনিটরিং
-- =============================================

-- Extension enable
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- সাধারণ PostgreSQL টেবিল তৈরি
CREATE TABLE server_metrics (
    time        TIMESTAMPTZ NOT NULL,
    server_name TEXT NOT NULL,
    region      TEXT NOT NULL,
    cpu_usage   DOUBLE PRECISION,
    memory_mb   DOUBLE PRECISION,
    disk_io     DOUBLE PRECISION,
    net_in_mb   DOUBLE PRECISION,
    net_out_mb  DOUBLE PRECISION
);

-- Hypertable এ রূপান্তর (TimescaleDB ম্যাজিক! ✨)
SELECT create_hypertable('server_metrics', 'time');

-- ডেটা INSERT (সাধারণ SQL!)
INSERT INTO server_metrics VALUES
    (NOW(), 'daraz-web-01', 'dhaka', 45.2, 8192, 120.5, 50.3, 25.1),
    (NOW(), 'daraz-web-02', 'dhaka', 62.1, 12288, 200.8, 80.5, 40.2),
    (NOW(), 'daraz-api-01', 'ctg',   38.5, 6144, 90.2, 30.1, 15.5);

-- গত ১ ঘণ্টায় ৫ মিনিট interval এ গড় CPU
SELECT
    time_bucket('5 minutes', time) AS interval,
    server_name,
    AVG(cpu_usage) AS avg_cpu,
    MAX(cpu_usage) AS max_cpu,
    MIN(cpu_usage) AS min_cpu
FROM server_metrics
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY interval, server_name
ORDER BY interval DESC;

-- Continuous Aggregate (Auto-downsampling!)
CREATE MATERIALIZED VIEW cpu_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    server_name,
    AVG(cpu_usage) AS avg_cpu,
    MAX(cpu_usage) AS max_cpu,
    COUNT(*) AS sample_count
FROM server_metrics
GROUP BY hour, server_name;

-- Retention Policy (৩০ দিন পর ডেটা মুছে ফেলো)
SELECT add_retention_policy('server_metrics', INTERVAL '30 days');

-- Compression Policy (৭ দিন পুরানো ডেটা compress করো)
ALTER TABLE server_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'server_name'
);
SELECT add_compression_policy('server_metrics', INTERVAL '7 days');
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: উচ্চ-কার্ডিনালিটি ট্যাগ ব্যবহার
──────────────────────────────────────
// InfluxDB তে user_id কে tag হিসেবে ব্যবহার
api_requests,user_id=123456 count=1
api_requests,user_id=789012 count=1
// ১ মিলিয়ন ইউজার = ১ মিলিয়ন series! 💥 OOM crash!

✅ সঠিক: user_id কে field হিসেবে ব্যবহার করুন
api_requests,endpoint=/api/orders user_id="123456",count=1

──────────────────────────────────────────

❌ ভুল: MySQL এ সেকেন্ড-বাই-সেকেন্ড মেট্রিক্স রাখা
   → টেবিল বিলিয়ন rows, query ধীর, storage বিশাল

✅ সঠিক: TSDB ব্যবহার করুন + Retention Policy সেট করুন

──────────────────────────────────────────

❌ ভুল: সব ডেটা একই resolution এ রাখা
   → ৫ বছরের সেকেন্ড-বাই-সেকেন্ড ডেটা = পেটাবাইট!

✅ সঠিক: Downsampling ব্যবহার করুন
   Raw → 7 দিন, 1-min avg → 30 দিন, 1-hr avg → 1 বছর
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### TSDB ব্যবহার করুন:

```
✅ সার্ভার/ইনফ্রাস্ট্রাকচার মনিটরিং
   (CPU, RAM, Disk, Network)

✅ অ্যাপ্লিকেশন পারফরম্যান্স মনিটরিং (APM)
   (Response time, Error rate, Throughput)

✅ IoT সেন্সর ডেটা
   (তাপমাত্রা, আর্দ্রতা, চাপ)

✅ আর্থিক ডেটা / ট্রেডিং
   (স্টক প্রাইস, Exchange rate, Transaction volume)

✅ নেটওয়ার্ক মনিটরিং
   (Bandwidth, Latency, Packet loss)

✅ বিজনেস মেট্রিক্স (Counters)
   (Daily Active Users, Order count, Revenue)
```

### TSDB ব্যবহার করবেন না:

```
❌ ইউজার প্রোফাইল / অ্যাকাউন্ট ডেটা → RDBMS
❌ ই-কমার্স অর্ডার ডেটা → RDBMS
❌ কনটেন্ট ম্যানেজমেন্ট → RDBMS / Document DB
❌ Session ডেটা → Redis
❌ ফুল-টেক্সট সার্চ → Elasticsearch
❌ গ্রাফ রিলেশনশিপ → Graph DB
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│          টাইম-সিরিজ ডাটাবেস সারসংক্ষেপ                 │
├───────────────────────────────────────────────────────┤
│                                                       │
│  TSDB = সময়ের সাথে সম্পর্কিত ডেটার জন্য বিশেষায়িত DB  │
│                                                       │
│  প্রধান TSDB:                                          │
│  - InfluxDB: IoT, জেনারেল পারপাস TSDB                  │
│  - TimescaleDB: PostgreSQL + Time-series (SQL!)       │
│  - Prometheus: মনিটরিং ও অ্যালার্টিং                    │
│                                                       │
│  মূল কনসেপ্ট:                                          │
│  - Tags (indexed metadata) vs Fields (values)        │
│  - Retention Policy (কতদিন ডেটা রাখবেন)               │
│  - Downsampling (ডেটা ঘনত্ব কমানো)                     │
│  - High write throughput                              │
│                                                       │
│  RDBMS এর চেয়ে TSDB:                                  │
│  - 100x দ্রুত write                                    │
│  - 10x কম storage                                     │
│  - Time-range query optimized                        │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [গ্রাফ ডাটাবেস](./graph-database.md) — নোড ও এজ ভিত্তিক ডেটা মডেলিং
