# 📊 Metrics & Alerting — পর্যবেক্ষণ মেট্রিক্স ও সতর্কতা ব্যবস্থা

> "যা পরিমাপ করা যায় না, তা উন্নত করা যায় না।" — Lord Kelvin

সিস্টেম অবজার্ভেবিলিটির তিনটি স্তম্ভের মধ্যে **Metrics** সবচেয়ে গুরুত্বপূর্ণ। লগ আপনাকে বলে
"কী ঘটেছে", ট্রেস বলে "কোথায় ঘটেছে", আর মেট্রিক্স বলে **"কতটুকু ঘটছে"**।

bKash-এ যখন প্রতি সেকেন্ডে হাজার হাজার ট্রানজ্যাকশন হয়, তখন প্রতিটি রিকোয়েস্ট লগ করা
সম্ভব না — কিন্তু "প্রতি সেকেন্ডে কতটি ট্রানজ্যাকশন হচ্ছে" এই সংখ্যাটি ট্র্যাক করা সহজ এবং
অত্যন্ত মূল্যবান।

---

## 📖 সূচিপত্র

1. [মেট্রিক্সের প্রকারভেদ](#-মেট্রিক্সের-প্রকারভেদ)
2. [RED Method](#-red-method)
3. [USE Method](#-use-method)
4. [Four Golden Signals](#-four-golden-signals)
5. [Prometheus + Grafana আর্কিটেকচার](#-prometheus--grafana-আর্কিটেকচার)
6. [কাস্টম বিজনেস মেট্রিক্স](#-কাস্টম-বিজনেস-মেট্রিক্স)
7. [PHP মেট্রিক্স উদাহরণ](#-php-মেট্রিক্স-উদাহরণ)
8. [JavaScript মেট্রিক্স উদাহরণ](#-javascript-মেট্রিক্স-উদাহরণ)
9. [Alerting কৌশল](#-alerting-কৌশল)
10. [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 মেট্রিক্সের প্রকারভেদ

মেট্রিক্স মূলত চার প্রকার। প্রতিটির আলাদা ব্যবহার ও বৈশিষ্ট্য আছে। নিচের ডায়াগ্রামটি দেখুন:

```
┌─────────────────────────────────────────────────────────┐
│              মেট্রিক্সের চার প্রকার                     │
├──────────────┬──────────────┬────────────┬──────────────┤
│   Counter    │    Gauge     │ Histogram  │   Summary    │
│  (গণনাকারী)  │  (পরিমাপক)   │ (বন্টন)    │  (সারাংশ)    │
│              │              │            │              │
│  শুধু বাড়ে   │  বাড়ে-কমে    │ বালতিতে    │ quantile     │
│  ↗ ↗ ↗ ↗    │  ↗ ↘ ↗ ↘    │  ভাগ করে   │  গণনা করে   │
│              │              │            │              │
│  মোট অর্ডার  │ CPU ব্যবহার  │  latency   │  p50, p99    │
│  মোট error   │ মেমরি ব্যবহার │  বন্টন     │  response    │
│  মোট request │ queue size   │            │  time        │
└──────────────┴──────────────┴────────────┴──────────────┘
```

### 1️⃣ Counter (ক্রমবর্ধমান গণনা)

Counter শুধুমাত্র বাড়তে পারে, কখনো কমে না (রিসেট ছাড়া)। এটি ব্যবহার করুন যখন
আপনি **মোট সংখ্যা** গণনা করতে চান।

```
Counter এর আচরণ (bKash ট্রানজ্যাকশন গণনা):

মান
 │
50k│                                          ●
   │                                      ●
40k│                                  ●
   │                             ●
30k│                        ●
   │                   ●
20k│              ●
   │         ●
10k│    ●
   │●
  0├──┬──┬──┬──┬──┬──┬──┬──┬──┬──→ সময়
   0  1  2  3  4  5  6  7  8  9  (ঘণ্টা)

   ↑ সবসময় উপরের দিকে — কখনো নিচে নামে না!
   (restart হলে 0 থেকে শুরু হয়)
```

**বাস্তব উদাহরণ:**
- `bkash_transactions_total` — মোট ট্রানজ্যাকশন সংখ্যা
- `daraz_orders_total` — মোট অর্ডার সংখ্যা
- `pathao_rides_completed_total` — সম্পন্ন রাইড সংখ্যা
- `http_requests_total` — মোট HTTP রিকোয়েস্ট

**Rate বের করা:** Counter থেকে `rate()` ফাংশন দিয়ে "প্রতি সেকেন্ডে কতটি" বের করা যায়:
```
rate(bkash_transactions_total[5m])  →  প্রতি সেকেন্ডে ট্রানজ্যাকশন হার
```

### 2️⃣ Gauge (ওঠানামা করা মান)

Gauge বাড়তেও পারে, কমতেও পারে। এটি ব্যবহার করুন যখন আপনি **বর্তমান অবস্থা**
জানতে চান।

```
Gauge এর আচরণ (Pathao সার্ভার CPU ব্যবহার):

 %
100│       ●
   │      ● ●
 80│     ●   ●          ●
   │    ●     ●        ● ●
 60│   ●       ●      ●   ●
   │  ●         ●    ●     ●
 40│ ●           ●  ●       ●
   │●             ●●         ●
 20│                          ●
   │
  0├──┬──┬──┬──┬──┬──┬──┬──┬──→ সময়
   0  1  2  3  4  5  6  7  8  (ঘণ্টা)

   ↑ উপরে-নিচে ওঠানামা করে — বর্তমান মান দেখায়
```

**বাস্তব উদাহরণ:**
- `server_cpu_usage_percent` — বর্তমান CPU ব্যবহার
- `server_memory_usage_bytes` — বর্তমান মেমরি ব্যবহার
- `pathao_active_drivers` — বর্তমানে সক্রিয় ড্রাইভার সংখ্যা
- `daraz_cart_items_current` — কার্টে থাকা আইটেম সংখ্যা
- `nagad_pending_transactions` — পেন্ডিং ট্রানজ্যাকশন সংখ্যা

### 3️⃣ Histogram (বন্টন পরিমাপ)

Histogram মানগুলোকে নির্দিষ্ট bucket-এ ভাগ করে রাখে। এটি ব্যবহার করুন যখন আপনি
**latency বা response time এর বন্টন** জানতে চান।

```
Histogram এর আচরণ (bKash API Response Time বন্টন):

রিকোয়েস্ট
সংখ্যা
   │
800│  ████
   │  ████
600│  ████  ████
   │  ████  ████
400│  ████  ████  ████
   │  ████  ████  ████
200│  ████  ████  ████  ████
   │  ████  ████  ████  ████  ████
  0├──────┬──────┬──────┬──────┬──────→ Response Time
   0-50ms 50-100 100-250 250-500 500ms+
          ms      ms      ms

   bucket: le="0.05"  le="0.1"  le="0.25"  le="0.5"  le="+Inf"
   count:    800        600       400        200        50
```

**কেন Histogram দরকার?** গড় (average) সবসময় সত্য বলে না:
```
❌ ভুল ধারণা: "গড় response time ২০০ms, তাই সব ঠিক আছে"

বাস্তবতা:
    ৯০% রিকোয়েস্ট = ৫০ms   (দ্রুত)
    ১০% রিকোয়েস্ট = ১৫৫০ms (অনেক ধীর!)
    গড় = (৯০×৫০ + ১০×১৫৫০) / ১০০ = ২০০ms

    → গড় ঠিক থাকলেও ১০% ব্যবহারকারী কষ্ট পাচ্ছে!
    → Histogram দিয়ে p99 দেখলে ধরা পড়বে
```

### 4️⃣ Summary (সারাংশ)

Summary হলো Histogram-এর মতো, তবে এটি সরাসরি quantile (p50, p90, p99) গণনা করে
ক্লায়েন্ট সাইডে। Histogram সার্ভার সাইডে bucket count রাখে।

```
Summary vs Histogram তুলনা:

┌──────────────────┬───────────────────┬───────────────────┐
│     বৈশিষ্ট্য     │    Histogram      │     Summary       │
├──────────────────┼───────────────────┼───────────────────┤
│ quantile গণনা    │ সার্ভারে (PromQL)  │ ক্লায়েন্টে        │
│ Aggregation      │ ✅ করা যায়        │ ❌ করা যায় না     │
│ Accuracy         │ bucket এর উপর     │ configurable      │
│                  │  নির্ভর করে       │  error             │
│ সুপারিশ          │ ✅ বেশিরভাগ ক্ষেত্রে │ নির্দিষ্ট ক্ষেত্রে   │
└──────────────────┴───────────────────┴───────────────────┘
```

---

## 📊 RED Method

RED Method হলো **রিকোয়েস্ট-চালিত সার্ভিসের** জন্য মেট্রিক্স কৌশল। Tom Wilkie (Grafana Labs)
এটি প্রবর্তন করেন। প্রতিটি মাইক্রোসার্ভিসের জন্য তিনটি জিনিস ট্র্যাক করুন:

```
┌─────────────────────────────────────────────────────────┐
│                    RED Method                            │
│         (Request-driven service এর জন্য)                 │
│                                                         │
│  ┌─────────┐    ┌─────────┐    ┌──────────┐            │
│  │    R    │    │    E    │    │    D     │            │
│  │  Rate   │    │ Errors  │    │Duration  │            │
│  │  হার    │    │  ত্রুটি  │    │ সময়কাল  │            │
│  └────┬────┘    └────┬────┘    └────┬─────┘            │
│       │              │              │                   │
│       ▼              ▼              ▼                   │
│  প্রতি সেকেন্ডে    ব্যর্থ         প্রতিটি              │
│  কতটি রিকোয়েস্ট  রিকোয়েস্টের    রিকোয়েস্টে           │
│  আসছে?           হার কত?        কত সময়                │
│                                  লাগছে?                │
│                                                         │
│  bKash উদাহরণ:                                          │
│  R: ৫০০০ req/sec   E: ০.১%      D: p99 = ২০০ms        │
└─────────────────────────────────────────────────────────┘
```

**Daraz উদাহরণ — প্রোডাক্ট সার্চ সার্ভিস:**
```
Rate:     rate(daraz_search_requests_total[5m])
Errors:   rate(daraz_search_errors_total[5m]) / rate(daraz_search_requests_total[5m])
Duration: histogram_quantile(0.99, rate(daraz_search_duration_seconds_bucket[5m]))
```

---

## 📊 USE Method

USE Method হলো **রিসোর্স মনিটরিং**-এর জন্য। Brendan Gregg এটি প্রবর্তন করেন।
প্রতিটি রিসোর্সের (CPU, মেমরি, ডিস্ক, নেটওয়ার্ক) জন্য তিনটি জিনিস ট্র্যাক করুন:

```
┌─────────────────────────────────────────────────────────┐
│                    USE Method                            │
│           (Resource monitoring এর জন্য)                  │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │     U      │  │     S      │  │     E      │        │
│  │Utilization │  │ Saturation │  │   Errors   │        │
│  │ ব্যবহার    │  │  পরিপূর্ণতা │  │   ত্রুটি    │        │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘        │
│        │               │               │                │
│        ▼               ▼               ▼                │
│   রিসোর্স কতটুকু   রিকোয়েস্ট      রিসোর্সে           │
│   ব্যবহৃত হচ্ছে?   queue-তে       কতগুলো error        │
│   (% বা ratio)    জমা হচ্ছে?      হচ্ছে?              │
│                                                         │
│  Grameenphone সার্ভার উদাহরণ:                            │
│  U: CPU ৭৫%        S: Load avg ১২   E: disk err ০      │
│  U: Memory ৮৫%     S: swap ব্যবহৃত   E: NIC err ২      │
│  U: Disk I/O ৬০%   S: iowait ১৫%   E: SMART warn ১    │
└─────────────────────────────────────────────────────────┘
```

```
প্রতিটি রিসোর্সের জন্য USE চেকলিস্ট:

┌────────────┬──────────────────┬────────────────────┬──────────────┐
│  Resource  │  Utilization     │  Saturation        │  Errors      │
├────────────┼──────────────────┼────────────────────┼──────────────┤
│  CPU       │  cpu_usage_%     │  load_average      │  -           │
│  Memory    │  mem_used/total  │  swap_usage        │  OOM kills   │
│  Disk      │  disk_io_%       │  io_queue_length   │  SMART errs  │
│  Network   │  bandwidth_%     │  tcp_retransmits   │  NIC errors  │
│  DB conn   │  active/max      │  waiting_queries   │  conn errors │
└────────────┴──────────────────┴────────────────────┴──────────────┘
```

---

## 🏠 Four Golden Signals

Google SRE বই থেকে আসা এই চারটি সিগন্যাল হলো যেকোনো সিস্টেম মনিটর করার ভিত্তি।
RED ও USE এর একটি সমন্বিত রূপ হিসেবে এটিকে ভাবা যায়:

```
┌───────────────────────────────────────────────────────────┐
│              Four Golden Signals (Google SRE)              │
│                                                           │
│    ┌──────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐  │
│    │ Latency  │  │ Traffic  │  │ Errors │  │Saturation│  │
│    │ বিলম্ব   │  │ ট্রাফিক  │  │ ত্রুটি  │  │পরিপূর্ণতা│  │
│    └────┬─────┘  └────┬─────┘  └───┬────┘  └────┬─────┘  │
│         │             │            │             │         │
│         ▼             ▼            ▼             ▼         │
│    রিকোয়েস্টে     সিস্টেমে     ব্যর্থ        সিস্টেম     │
│    কত সময়        কতটুকু       রিকোয়েস্ট     কতটুকু      │
│    লাগছে?        demand       এর হার        ভর্তি?       │
│                  আছে?                                     │
│                                                           │
│    ⚠️ সফল ও       📈 req/sec    📊 5xx rate   💾 CPU,     │
│    ব্যর্থ আলাদা    📈 QPS        📊 timeout    মেমরি,      │
│    করে মাপুন!     📈 bandwidth   📊 rate       queue       │
│                                                           │
│  bKash পেমেন্ট সার্ভিসে প্রয়োগ:                            │
│  Latency:    p50=৫০ms, p99=৩০০ms (সফল রিকোয়েস্ট)         │
│  Traffic:    ৫,০০০ transactions/sec (পিক আওয়ারে)          │
│  Errors:     ০.০৫% failure rate                           │
│  Saturation: DB connection pool ৮০% ব্যবহৃত               │
└───────────────────────────────────────────────────────────┘
```

```
তিনটি পদ্ধতির সম্পর্ক:

     Four Golden Signals
    ┌──────────────────────┐
    │  Latency  = Duration │──── RED
    │  Traffic  = Rate     │──── RED
    │  Errors   = Errors   │──── RED + USE
    │  Saturation          │──── USE
    └──────────────────────┘

    RED  → সার্ভিস মেট্রিক্স (user-facing)
    USE  → রিসোর্স মেট্রিক্স (infrastructure)
    4 Golden Signals → দুটোর সমন্বয়
```

---

## 📌 Prometheus + Grafana আর্কিটেকচার

Prometheus হলো একটি **pull-based** মনিটরিং সিস্টেম। এটি নির্দিষ্ট সময় পর পর আপনার
সার্ভিস থেকে মেট্রিক্স **scrape** (টেনে আনে) করে।

```
Prometheus Architecture (Pull/Scrape Model):

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐          │
│   │ bKash    │     │ Pathao   │     │ Nagad    │          │
│   │ Payment  │     │ Ride     │     │ Wallet   │          │
│   │ Service  │     │ Service  │     │ Service  │          │
│   │          │     │          │     │          │          │
│   │ :9090/   │     │ :9090/   │     │ :9090/   │          │
│   │ metrics  │     │ metrics  │     │ metrics  │          │
│   └────┬─────┘     └────┬─────┘     └────┬─────┘          │
│        │                │                │                  │
│        │    scrape      │    scrape      │    scrape        │
│        │  (every 15s)   │  (every 15s)   │  (every 15s)    │
│        ▼                ▼                ▼                  │
│   ┌─────────────────────────────────────────┐              │
│   │              Prometheus                  │              │
│   │                                         │              │
│   │  ┌───────────┐  ┌──────────────────┐   │              │
│   │  │ Scraper   │  │   TSDB           │   │              │
│   │  │ (টেনে আনে)│  │ (Time Series DB) │   │              │
│   │  └───────────┘  └──────────────────┘   │              │
│   │  ┌───────────┐  ┌──────────────────┐   │              │
│   │  │ PromQL    │  │  Alert Rules     │   │              │
│   │  │ (Query)   │  │  (সতর্কতা নিয়ম)  │   │              │
│   │  └─────┬─────┘  └────────┬─────────┘   │              │
│   └────────┼─────────────────┼─────────────┘              │
│            │                 │                              │
│            ▼                 ▼                              │
│   ┌──────────────┐  ┌───────────────┐                      │
│   │   Grafana    │  │ Alertmanager  │                      │
│   │  (Dashboard) │  │  (সতর্কতা     │                      │
│   │  ┌────────┐  │  │   পাঠানো)     │                      │
│   │  │ Graph  │  │  │               │                      │
│   │  │ Panel  │  │  │  ┌─────────┐  │                      │
│   │  └────────┘  │  │  │ Slack   │  │                      │
│   │  ┌────────┐  │  │  │ Email   │  │                      │
│   │  │ Table  │  │  │  │PagerDuty│  │                      │
│   │  │ Panel  │  │  │  └─────────┘  │                      │
│   │  └────────┘  │  └───────────────┘                      │
│   └──────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

### Docker Compose সেটআপ

```yaml
# docker-compose.yml — Prometheus + Grafana স্ট্যাক
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=secret
    volumes:
      - grafana_data:/var/lib/grafana

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # bKash Payment Service
  - job_name: 'bkash-payment'
    static_configs:
      - targets: ['payment-service:9090']
    metrics_path: /metrics

  # Pathao Ride Service
  - job_name: 'pathao-ride'
    static_configs:
      - targets: ['ride-service:9090']

  # Daraz Product Service
  - job_name: 'daraz-product'
    static_configs:
      - targets: ['product-service:8080']
    metrics_path: /api/metrics
```

### PromQL (Prometheus Query Language) উদাহরণ

```
┌──────────────────────────────────────────────────────────────┐
│                    PromQL চিট শিট                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ১. তাৎক্ষণিক মান (Instant vector):                          │
│     http_requests_total                                      │
│     → বর্তমান মোট রিকোয়েস্ট সংখ্যা                          │
│                                                              │
│  ২. হার (Rate):                                              │
│     rate(http_requests_total[5m])                            │
│     → শেষ ৫ মিনিটে প্রতি সেকেন্ডে রিকোয়েস্ট হার             │
│                                                              │
│  ৩. Error Rate (শতাংশ):                                      │
│     rate(http_requests_total{status=~"5.."}[5m])             │
│     /                                                        │
│     rate(http_requests_total[5m])                            │
│     → 5xx error এর শতকরা হার                                 │
│                                                              │
│  ৪. Percentile (Histogram থেকে):                             │
│     histogram_quantile(0.99,                                 │
│       rate(http_request_duration_seconds_bucket[5m]))        │
│     → p99 latency                                            │
│                                                              │
│  ৫. বৃদ্ধি (Increase):                                       │
│     increase(bkash_transactions_total[1h])                   │
│     → শেষ ১ ঘণ্টায় কতটি নতুন ট্রানজ্যাকশন হয়েছে             │
│                                                              │
│  ৬. গড় (Average):                                            │
│     avg(rate(cpu_usage_percent[5m])) by (instance)           │
│     → প্রতিটি ইন্সট্যান্সের গড় CPU ব্যবহার                   │
│                                                              │
│  ৭. Top K:                                                   │
│     topk(5, rate(http_requests_total[5m]))                   │
│     → সবচেয়ে বেশি রিকোয়েস্ট পাচ্ছে এমন ৫টি endpoint         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🎯 কাস্টম বিজনেস মেট্রিক্স

শুধু টেকনিক্যাল মেট্রিক্স (CPU, মেমরি) যথেষ্ট নয়। বিজনেসের ভাষায়ও মেট্রিক্স দরকার।
bKash-এর CTO শুধু CPU usage দেখে বুঝবেন না সিস্টেম ভালো আছে কিনা — তাকে দেখতে
হবে **"প্রতি মিনিটে কতটি সফল ট্রানজ্যাকশন হচ্ছে"**।

```
বিজনেস মেট্রিক্স উদাহরণ:

┌─────────────────────────────────────────────────────────┐
│  bKash                                                  │
│  ├── bkash_send_money_total (counter)                   │
│  ├── bkash_cash_out_total (counter)                     │
│  ├── bkash_payment_total (counter)                      │
│  ├── bkash_transaction_amount_bdt (histogram)           │
│  ├── bkash_active_users (gauge)                         │
│  └── bkash_failed_transactions_total (counter)          │
│                                                         │
│  Pathao                                                 │
│  ├── pathao_ride_requests_total (counter)                │
│  ├── pathao_ride_accept_duration_seconds (histogram)     │
│  ├── pathao_active_drivers (gauge)                       │
│  ├── pathao_ride_cancellations_total (counter)           │
│  └── pathao_driver_rating_average (gauge)                │
│                                                         │
│  Daraz                                                  │
│  ├── daraz_orders_total (counter)                        │
│  ├── daraz_order_value_bdt (histogram)                   │
│  ├── daraz_cart_abandonment_total (counter)              │
│  ├── daraz_search_results_count (histogram)              │
│  └── daraz_inventory_items (gauge)                       │
│                                                         │
│  Grameenphone                                           │
│  ├── gp_sms_sent_total (counter)                         │
│  ├── gp_call_duration_seconds (histogram)                │
│  ├── gp_data_usage_bytes (counter)                       │
│  ├── gp_active_subscribers (gauge)                       │
│  └── gp_network_errors_total (counter)                   │
└─────────────────────────────────────────────────────────┘
```

---

## 📌 PHP মেট্রিক্স উদাহরণ

`prometheus_client_php` লাইব্রেরি ব্যবহার করে Laravel/PHP অ্যাপ্লিকেশনে মেট্রিক্স
যোগ করা যায়।

```bash
# ইনস্টলেশন
composer require promphp/prometheus_client_php
```

### Counter উদাহরণ (bKash ট্রানজ্যাকশন ট্র্যাকিং)

```php
<?php
// app/Metrics/TransactionMetrics.php

use Prometheus\CollectorRegistry;
use Prometheus\Storage\Redis;

class TransactionMetrics
{
    private CollectorRegistry $registry;

    public function __construct()
    {
        // Redis adapter ব্যবহার করুন — মাল্টি-প্রসেস PHP-তে
        // InMemory adapter কাজ করবে না কারণ প্রতিটি PHP
        // রিকোয়েস্ট আলাদা প্রসেসে চলে
        $adapter = new Redis(['host' => '127.0.0.1']);
        $this->registry = new CollectorRegistry($adapter);
    }

    public function recordTransaction(
        string $type,
        string $status,
        float $amount
    ): void {
        // Counter — মোট ট্রানজ্যাকশন গণনা
        $counter = $this->registry->getOrRegisterCounter(
            'bkash',                          // namespace
            'transactions_total',             // name
            'Total number of transactions',   // help
            ['type', 'status']                // labels
        );
        $counter->inc(['send_money', 'success']);

        // Histogram — ট্রানজ্যাকশন পরিমাণের বন্টন
        $histogram = $this->registry->getOrRegisterHistogram(
            'bkash',
            'transaction_amount_bdt',
            'Transaction amount in BDT',
            ['type'],
            [100, 500, 1000, 5000, 10000, 50000] // buckets (টাকায়)
        );
        $histogram->observe($amount, [$type]);
    }
}
```

### Gauge উদাহরণ (সক্রিয় ব্যবহারকারী ট্র্যাকিং)

```php
<?php
// app/Metrics/UserMetrics.php

use Prometheus\CollectorRegistry;
use Prometheus\Storage\Redis;

class UserMetrics
{
    private CollectorRegistry $registry;

    public function __construct(CollectorRegistry $registry)
    {
        $this->registry = $registry;
    }

    public function updateActiveUsers(): void
    {
        // Gauge — বর্তমান সক্রিয় ব্যবহারকারী সংখ্যা
        $gauge = $this->registry->getOrRegisterGauge(
            'bkash',
            'active_users',
            'Currently active users',
            ['platform']
        );

        // ডাটাবেস থেকে গণনা করে সেট করুন
        $mobileCount = DB::table('sessions')
            ->where('platform', 'mobile')
            ->where('last_active', '>', now()->subMinutes(5))
            ->count();

        $webCount = DB::table('sessions')
            ->where('platform', 'web')
            ->where('last_active', '>', now()->subMinutes(5))
            ->count();

        $gauge->set($mobileCount, ['mobile']);
        $gauge->set($webCount, ['web']);
    }
}
```

### Histogram উদাহরণ (API Response Time)

```php
<?php
// app/Http/Middleware/MetricsMiddleware.php

use Prometheus\CollectorRegistry;

class MetricsMiddleware
{
    private CollectorRegistry $registry;

    public function __construct(CollectorRegistry $registry)
    {
        $this->registry = $registry;
    }

    public function handle($request, Closure $next)
    {
        $startTime = microtime(true);

        $response = $next($request);

        $duration = microtime(true) - $startTime;

        // Histogram — প্রতিটি রিকোয়েস্টের সময়কাল রেকর্ড
        $histogram = $this->registry->getOrRegisterHistogram(
            'http',
            'request_duration_seconds',
            'HTTP request duration in seconds',
            ['method', 'route', 'status'],
            [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]
        );

        $histogram->observe($duration, [
            $request->method(),
            $request->route()?->getName() ?? 'unknown',
            (string) $response->getStatusCode(),
        ]);

        // Counter — মোট রিকোয়েস্ট সংখ্যা
        $counter = $this->registry->getOrRegisterCounter(
            'http',
            'requests_total',
            'Total HTTP requests',
            ['method', 'status']
        );
        $counter->inc([
            $request->method(),
            (string) $response->getStatusCode(),
        ]);

        return $response;
    }
}
```

### মেট্রিক্স Endpoint

```php
<?php
// routes/web.php

use Prometheus\CollectorRegistry;
use Prometheus\RenderTextFormat;

Route::get('/metrics', function (CollectorRegistry $registry) {
    $renderer = new RenderTextFormat();
    $result = $renderer->render($registry->getMetricFamilySamples());

    return response($result, 200, [
        'Content-Type' => RenderTextFormat::MIME_TYPE,
    ]);
});

// আউটপুট উদাহরণ:
// # HELP bkash_transactions_total Total number of transactions
// # TYPE bkash_transactions_total counter
// bkash_transactions_total{type="send_money",status="success"} 15234
// bkash_transactions_total{type="send_money",status="failed"} 12
// bkash_transactions_total{type="cash_out",status="success"} 8901
//
// # HELP http_request_duration_seconds HTTP request duration
// # TYPE http_request_duration_seconds histogram
// http_request_duration_seconds_bucket{method="GET",route="home",le="0.05"} 890
// http_request_duration_seconds_bucket{method="GET",route="home",le="0.1"} 950
// http_request_duration_seconds_bucket{method="GET",route="home",le="+Inf"} 1000
```

---

## 📌 JavaScript মেট্রিক্স উদাহরণ

`prom-client` হলো Node.js-এর জন্য সবচেয়ে জনপ্রিয় Prometheus ক্লায়েন্ট লাইব্রেরি।

```bash
# ইনস্টলেশন
npm install prom-client express
```

### সম্পূর্ণ সেটআপ (Pathao Ride Service)

```javascript
// metrics/setup.js
const promClient = require('prom-client');

// ডিফল্ট মেট্রিক্স সংগ্রহ (CPU, মেমরি, event loop ইত্যাদি)
promClient.collectDefaultMetrics({
  prefix: 'pathao_ride_',
  gcDurationBuckets: [0.001, 0.01, 0.1, 1, 2, 5],
});

// ──────────────────────────────────────────────
// Counter — রাইড রিকোয়েস্ট গণনা
// ──────────────────────────────────────────────
const rideRequestsTotal = new promClient.Counter({
  name: 'pathao_ride_requests_total',
  help: 'Total number of ride requests',
  labelNames: ['vehicle_type', 'status', 'city'],
});

// ──────────────────────────────────────────────
// Gauge — সক্রিয় ড্রাইভার সংখ্যা
// ──────────────────────────────────────────────
const activeDrivers = new promClient.Gauge({
  name: 'pathao_active_drivers',
  help: 'Number of currently active drivers',
  labelNames: ['vehicle_type', 'city'],
});

// ──────────────────────────────────────────────
// Histogram — রাইড accept হতে কত সময় লাগছে
// ──────────────────────────────────────────────
const rideAcceptDuration = new promClient.Histogram({
  name: 'pathao_ride_accept_duration_seconds',
  help: 'Time taken for a driver to accept a ride',
  labelNames: ['vehicle_type', 'city'],
  buckets: [5, 10, 15, 30, 45, 60, 90, 120, 180, 300],
});

// ──────────────────────────────────────────────
// Summary — রাইডের দূরত্ব সারাংশ
// ──────────────────────────────────────────────
const rideDistanceSummary = new promClient.Summary({
  name: 'pathao_ride_distance_km',
  help: 'Ride distance in kilometers',
  labelNames: ['vehicle_type'],
  percentiles: [0.5, 0.9, 0.95, 0.99],
  maxAgeSeconds: 600,
  ageBuckets: 5,
});

module.exports = {
  promClient,
  rideRequestsTotal,
  activeDrivers,
  rideAcceptDuration,
  rideDistanceSummary,
};
```

### Express অ্যাপে ব্যবহার

```javascript
// app.js
const express = require('express');
const {
  promClient,
  rideRequestsTotal,
  activeDrivers,
  rideAcceptDuration,
  rideDistanceSummary,
} = require('./metrics/setup');

const app = express();

// মেট্রিক্স endpoint — Prometheus এখান থেকে scrape করবে
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});

// রাইড রিকোয়েস্ট API
app.post('/api/rides/request', async (req, res) => {
  const { vehicleType, city } = req.body;

  try {
    const ride = await createRideRequest(vehicleType, city);

    // ✅ সফল রিকোয়েস্ট কাউন্ট
    rideRequestsTotal.inc({
      vehicle_type: vehicleType,
      status: 'created',
      city: city,
    });

    res.json({ rideId: ride.id, status: 'searching_driver' });
  } catch (error) {
    // ❌ ব্যর্থ রিকোয়েস্ট কাউন্ট
    rideRequestsTotal.inc({
      vehicle_type: vehicleType,
      status: 'failed',
      city: city,
    });

    res.status(500).json({ error: 'Ride request failed' });
  }
});

// ড্রাইভার রাইড accept করলে
app.post('/api/rides/:id/accept', async (req, res) => {
  const ride = await getRide(req.params.id);
  const acceptTime = (Date.now() - ride.createdAt) / 1000;

  // Histogram — accept এ কত সময় লেগেছে রেকর্ড
  rideAcceptDuration.observe(
    { vehicle_type: ride.vehicleType, city: ride.city },
    acceptTime
  );

  res.json({ status: 'accepted' });
});

// ড্রাইভার অনলাইন/অফলাইন ট্র্যাক
app.post('/api/drivers/online', (req, res) => {
  const { vehicleType, city } = req.body;
  activeDrivers.inc({ vehicle_type: vehicleType, city: city });
  res.json({ status: 'online' });
});

app.post('/api/drivers/offline', (req, res) => {
  const { vehicleType, city } = req.body;
  activeDrivers.dec({ vehicle_type: vehicleType, city: city });
  res.json({ status: 'offline' });
});

// রাইড শেষ হলে দূরত্ব রেকর্ড
app.post('/api/rides/:id/complete', async (req, res) => {
  const ride = await getRide(req.params.id);

  // Summary — রাইডের দূরত্ব রেকর্ড
  rideDistanceSummary.observe(
    { vehicle_type: ride.vehicleType },
    ride.distanceKm
  );

  res.json({ status: 'completed', fare: ride.fare });
});

app.listen(3000, () => {
  console.log('Pathao Ride Service running on port 3000');
});
```

### HTTP Middleware (সব রিকোয়েস্ট ট্র্যাকিং)

```javascript
// middleware/httpMetrics.js
const promClient = require('prom-client');

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

function metricsMiddleware(req, res, next) {
  const end = httpRequestDuration.startTimer();

  res.on('finish', () => {
    const route = req.route?.path || req.path;
    const labels = {
      method: req.method,
      route: route,
      status_code: res.statusCode,
    };

    end(labels);
    httpRequestsTotal.inc(labels);
  });

  next();
}

module.exports = { metricsMiddleware };
```

---

## 📌 Grafana Dashboard

### Dashboard JSON স্নিপেট

```json
{
  "dashboard": {
    "title": "bKash Transaction Dashboard",
    "panels": [
      {
        "title": "Transactions per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(bkash_transactions_total[5m])",
            "legendFormat": "{{type}} - {{status}}"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{route}}"
          }
        ]
      },
      {
        "title": "Error Rate %",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~'5..'}[5m]) / rate(http_requests_total[5m]) * 100",
            "legendFormat": "Error %"
          }
        ]
      },
      {
        "title": "Active Users",
        "type": "gauge",
        "targets": [
          {
            "expr": "bkash_active_users",
            "legendFormat": "{{platform}}"
          }
        ]
      }
    ]
  }
}
```

```
Grafana Dashboard Layout:

┌──────────────────────────────────────────────────────────────┐
│  bKash Transaction Dashboard                     🔄 Last 6h │
├─────────────────────────┬────────────────────────────────────┤
│  Transactions/sec       │  Error Rate %                      │
│  ┌───────────────────┐  │  ┌───────────────────────────┐    │
│  │   ╱╲    ╱╲        │  │  │                           │    │
│  │  ╱  ╲╱╱  ╲╱╲     │  │  │  ___                      │    │
│  │ ╱          ╲╱╲   │  │  │_╱   ╲_____╱╲__            │    │
│  │╱              ╲  │  │  │                ╲___        │    │
│  └───────────────────┘  │  └───────────────────────────┘    │
│  [5,234 req/s]          │  [0.05%]                           │
├─────────────────────────┼────────────────────────────────────┤
│  P99 Response Time      │  Active Users                      │
│  ┌───────────────────┐  │  ┌───────────────────────────┐    │
│  │      ╱╲           │  │  │       ████                │    │
│  │  ___╱  ╲___       │  │  │  ████ ████                │    │
│  │ ╱          ╲___   │  │  │  ████ ████ ████           │    │
│  │╱               ╲  │  │  │  ████ ████ ████ ████      │    │
│  └───────────────────┘  │  │  Mobile Web  API  Total   │    │
│  [p99: 180ms]           │  └───────────────────────────┘    │
├─────────────────────────┴────────────────────────────────────┤
│  Transaction Amount Distribution (Histogram)                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ████                                                │    │
│  │  ████ ████                                           │    │
│  │  ████ ████ ████                                      │    │
│  │  ████ ████ ████ ████ ████                            │    │
│  │  <500 500-1k 1k-5k 5k-10k >10k  (BDT)              │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 🚨 Alerting কৌশল

Alerting হলো মেট্রিক্সের সবচেয়ে গুরুত্বপূর্ণ প্রয়োগ। কিন্তু ভুল alerting হলে **alert fatigue**
(সতর্কতা ক্লান্তি) হয় — টিম এত alert পায় যে আসল সমস্যাও ignore করে ফেলে।

### Alert Severity Levels

```
┌──────────┬────────────────────┬──────────────────────┬──────────────┐
│  Level   │  অর্থ               │  Action              │  Response    │
│          │                    │                      │  Time        │
├──────────┼────────────────────┼──────────────────────┼──────────────┤
│ CRITICAL │ সিস্টেম ডাউন /     │ তাৎক্ষণিক on-call    │ ৫ মিনিটের    │
│  (P1)    │ ব্যবহারকারীরা      │ ইঞ্জিনিয়ার জাগাও    │ মধ্যে        │
│          │ সেবা পাচ্ছে না     │ PagerDuty / ফোন     │              │
├──────────┼────────────────────┼──────────────────────┼──────────────┤
│ WARNING  │ সমস্যার লক্ষণ /    │ অফিস আওয়ারে Slack  │ ৩০ মিনিটের   │
│  (P2)    │ degraded           │ নোটিফিকেশন          │ মধ্যে        │
│          │ performance        │                      │              │
├──────────┼────────────────────┼──────────────────────┼──────────────┤
│  INFO    │ জানার জন্য /       │ Dashboard-এ দেখা    │ পরবর্তী      │
│  (P3)    │ trend পর্যবেক্ষণ   │ যথেষ্ট              │ business day │
├──────────┼────────────────────┼──────────────────────┼──────────────┤
│  TICKET  │ দীর্ঘমেয়াদী       │ JIRA ticket তৈরি    │ sprint-এ     │
│  (P4)    │ উন্নতির সুযোগ      │ করো                  │ অন্তর্ভুক্ত   │
└──────────┴────────────────────┴──────────────────────┴──────────────┘
```

### Prometheus Alert Rules

```yaml
# prometheus/alert-rules.yml
groups:
  - name: bkash_payment_alerts
    rules:
      # CRITICAL — ট্রানজ্যাকশন সম্পূর্ণ বন্ধ
      - alert: BkashTransactionsDown
        expr: rate(bkash_transactions_total[5m]) == 0
        for: 2m
        labels:
          severity: critical
          team: payment
        annotations:
          summary: "bKash ট্রানজ্যাকশন সম্পূর্ণ বন্ধ!"
          description: "শেষ ২ মিনিটে কোনো ট্রানজ্যাকশন হয়নি।"

      # CRITICAL — উচ্চ error rate
      - alert: BkashHighErrorRate
        expr: >
          rate(bkash_transactions_total{status="failed"}[5m])
          /
          rate(bkash_transactions_total[5m])
          > 0.05
        for: 3m
        labels:
          severity: critical
          team: payment
        annotations:
          summary: "bKash error rate {{ $value | humanizePercentage }} (৫% এর বেশি)"
          description: "ব্যর্থ ট্রানজ্যাকশনের হার বিপদসীমার উপরে।"

      # WARNING — latency বেড়ে গেছে
      - alert: BkashHighLatency
        expr: >
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket{job="bkash-payment"}[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
          team: payment
        annotations:
          summary: "bKash API p99 latency {{ $value }}s (১ সেকেন্ডের বেশি)"

  - name: infrastructure_alerts
    rules:
      # WARNING — CPU বেশি
      - alert: HighCPUUsage
        expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance) > 0.85
        for: 10m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "CPU usage ৮৫% এর বেশি on {{ $labels.instance }}"

      # CRITICAL — ডিস্ক ফুরিয়ে যাচ্ছে
      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.10
        for: 5m
        labels:
          severity: critical
          team: infra
        annotations:
          summary: "ডিস্ক ১০% এর কম বাকি on {{ $labels.instance }}"

      # WARNING — মেমরি বেশি
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.90
        for: 10m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "মেমরি ৯০% এর বেশি ব্যবহৃত on {{ $labels.instance }}"
```

### Alertmanager Configuration

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'default-slack'
  group_by: ['alertname', 'team']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # CRITICAL alerts → PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'critical-pagerduty'
      repeat_interval: 15m

    # WARNING alerts → Slack only
    - match:
        severity: warning
      receiver: 'warning-slack'
      repeat_interval: 2h

    # Payment team specific routing
    - match:
        team: payment
      receiver: 'payment-team-slack'

receivers:
  - name: 'default-slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts-general'

  - name: 'critical-pagerduty'
    pagerduty_configs:
      - service_key: '<PAGERDUTY_SERVICE_KEY>'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts-critical'

  - name: 'warning-slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts-warning'

  - name: 'payment-team-slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#payment-alerts'
```

### ❌ Bad Alerting vs ✅ Good Alerting

```
❌ BAD: Alert Fatigue তৈরি করে এমন প্যাটার্ন
─────────────────────────────────────────────

❌ CPU > 50% হলেই alert
   → সমস্যা: CPU ৫০% স্বাভাবিক! প্রতিদিন শত শত alert আসবে
   → সবাই ignore করবে → আসল সমস্যাও মিস হবে

❌ প্রতিটি 404 error-এ alert
   → সমস্যা: 404 হলো স্বাভাবিক ব্যাপার (ভুল URL)
   → noise তৈরি করে

❌ কোনো threshold ছাড়া "কিছু একটা বেড়েছে" alert
   → সমস্যা: "বেড়েছে" মানে কতটুকু? কতক্ষণ ধরে?
   → actionable না

❌ `for: 0s` (তাৎক্ষণিক alert, কোনো buffer নেই)
   → সমস্যা: একটি spike-এও alert আসবে
   → flapping alerts তৈরি হবে

❌ সব alert একই চ্যানেলে, একই severity তে
   → সমস্যা: CRITICAL আর INFO একসাথে
   → কোনটা আগে দেখবে বোঝা যায় না
```

```
✅ GOOD: কার্যকর Alerting প্যাটার্ন
──────────────────────────────────────

✅ লক্ষণ-ভিত্তিক (symptom-based) alert
   → "ব্যবহারকারীরা 5xx error পাচ্ছে" (symptom)
   → "CPU বেড়েছে" নয় (cause) — cause dashboard-এ দেখুন

✅ `for: 5m` — ৫ মিনিট ধরে সমস্যা থাকলে তবেই alert
   → spike filter হয়ে যায়
   → শুধু আসল সমস্যায় alert আসে

✅ Severity আলাদা করুন
   → CRITICAL: রাত ৩টায় ফোন করে জাগাবে
   → WARNING: অফিস আওয়ারে Slack-এ জানাবে
   → INFO: শুধু dashboard-এ দেখাবে

✅ Runbook লিংক দিন
   → alert-এর সাথে "কী করতে হবে" এর লিংক
   → on-call ইঞ্জিনিয়ার দ্রুত সমাধান করতে পারবে

✅ প্রতিটি alert-এ জিজ্ঞেস করুন:
   "এই alert আসলে কি কেউ কিছু করবে?"
   যদি না → alert দরকার নেই, dashboard-এ রাখুন
```

```
Alert তৈরির আগে এই চেকলিস্ট অনুসরণ করুন:

┌──────────────────────────────────────────────────┐
│  ☐ এই alert কি user-facing সমস্যা নির্দেশ করে?  │
│  ☐ এই alert আসলে কি কেউ action নেবে?           │
│  ☐ Severity সঠিক? (রাত ৩টায় জাগানোর যোগ্য?)    │
│  ☐ `for` duration যথেষ্ট? (flapping হবে না?)     │
│  ☐ Runbook আছে? (কী করতে হবে জানা আছে?)        │
│  ☐ সপ্তাহে কতবার fire হবে? (>২ বার = tune করুন) │
└──────────────────────────────────────────────────┘
```

---

## 📌 বাস্তব পরিস্থিতি: bKash ট্রানজ্যাকশন মনিটরিং

```
bKash এর একটি সম্পূর্ণ মনিটরিং সেটআপ:

                    ব্যবহারকারী
                        │
                        ▼
                 ┌──────────────┐
                 │   API GW     │
                 │   (nginx)    │──→ nginx_http_requests_total
                 └──────┬───────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
    ┌────────────┐ ┌──────────┐ ┌──────────┐
    │  Payment   │ │ Account  │ │  Fraud   │
    │  Service   │ │ Service  │ │  Check   │
    │            │ │          │ │  Service │
    │ Metrics:   │ │ Metrics: │ │ Metrics: │
    │ • txn/sec  │ │ • balance│ │ • scans  │
    │ • latency  │ │  queries │ │ • blocks │
    │ • errors   │ │ • errors │ │ • false+ │
    └─────┬──────┘ └────┬─────┘ └────┬─────┘
          │             │            │
          └─────────────┼────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  PostgreSQL  │──→ pg_stat_activity
                 │  Database    │──→ pg_connections
                 └──────────────┘

  Alert Examples:
  ┌─────────────────────────────────────────────────────┐
  │ CRITICAL: txn rate == 0 for 2min                    │
  │ CRITICAL: error rate > 5% for 3min                  │
  │ WARNING:  p99 latency > 1s for 5min                 │
  │ WARNING:  DB connections > 80% for 10min            │
  │ INFO:     txn rate dropped 50% vs last week         │
  └─────────────────────────────────────────────────────┘
```

---

## 📌 বাস্তব পরিস্থিতি: Pathao রাইড রিকোয়েস্ট মনিটরিং

```
Pathao Ride Matching Pipeline Metrics:

  রাইডার রিকোয়েস্ট
        │
        ▼
  ┌──────────────┐   pathao_ride_requests_total{status="created"}
  │ Ride Request │
  │   Created    │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐   pathao_driver_search_duration_seconds
  │   Driver     │   pathao_nearby_drivers_count
  │   Matching   │
  └──────┬───────┘
    ╱         ╲
   ▼           ▼
┌────────┐ ┌────────┐
│ Match  │ │  No    │  pathao_ride_requests_total{status="no_driver"}
│ Found  │ │ Match  │
└───┬────┘ └────────┘
    │
    ▼
┌────────────┐   pathao_ride_accept_duration_seconds
│  Driver    │
│  Accepts   │
└─────┬──────┘
      │
      ▼
┌────────────┐   pathao_ride_duration_seconds
│   Ride     │   pathao_ride_distance_km
│ In Progress│   pathao_ride_fare_bdt
└─────┬──────┘
      │
      ▼
┌────────────┐   pathao_ride_requests_total{status="completed"}
│ Completed  │   pathao_ride_rating
└────────────┘

  Key PromQL Queries:
  ─────────────────────────────────────────────────
  # প্রতি মিনিটে কত রাইড রিকোয়েস্ট আসছে?
  rate(pathao_ride_requests_total[5m]) * 60

  # ড্রাইভার না পাওয়ার হার কত?
  rate(pathao_ride_requests_total{status="no_driver"}[5m])
  / rate(pathao_ride_requests_total[5m])

  # ড্রাইভার accept করতে গড়ে কত সময় নিচ্ছে?
  histogram_quantile(0.5, rate(pathao_ride_accept_duration_seconds_bucket[5m]))

  # ঢাকায় কত ড্রাইভার সক্রিয়?
  pathao_active_drivers{city="dhaka"}
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### কখন ব্যবহার করবেন ✅

| পরিস্থিতি | কারণ |
|-----------|------|
| প্রোডাকশন সার্ভিস মনিটরিং | সিস্টেমের স্বাস্থ্য বোঝার জন্য অপরিহার্য |
| SLA/SLO ট্র্যাকিং | "৯৯.৯% আপটাইম" প্রমাণ করতে মেট্রিক্স লাগবে |
| Capacity planning | ভবিষ্যতে কত রিসোর্স লাগবে বুঝতে |
| বিজনেস KPI | ট্রানজ্যাকশন/সেকেন্ড, অর্ডার সংখ্যা ইত্যাদি |
| Incident response | সমস্যার কারণ দ্রুত খুঁজে বের করতে |
| A/B testing | নতুন ফিচারের performance তুলনা করতে |

### কখন ব্যবহার করবেন না ❌

| পরিস্থিতি | কারণ |
|-----------|------|
| ডিবাগিং-এর জন্য একা | লগ ও ট্রেস সাথে লাগবে |
| সব কিছু মেট্রিক্স করা | cardinality বিস্ফোরণ হবে — শুধু গুরুত্বপূর্ণ জিনিস মাপুন |
| User ID দিয়ে label | প্রতিটি ইউনিক user-এর জন্য আলাদা time series = Prometheus crash |
| Request body মেট্রিক্সে | সংবেদনশীল তথ্য মেট্রিক্সে রাখবেন না, লগে রাখুন |

### ⚠️ Label Cardinality সতর্কতা

```
❌ খারাপ (High Cardinality — Prometheus ধ্বংস করবে):

  http_requests_total{
    user_id="12345",          // লক্ষ লক্ষ ইউনিক মান!
    request_id="abc-def-ghi", // প্রতিটি রিকোয়েস্ট ইউনিক!
    ip_address="1.2.3.4"     // হাজার হাজার ইউনিক মান!
  }
  → প্রতিটি ইউনিক label combination = একটি আলাদা time series
  → ১ মিলিয়ন user × ১০ endpoint = ১ কোটি time series = 💥

✅ ভালো (Low Cardinality — সীমিত সংখ্যক মান):

  http_requests_total{
    method="POST",            // GET, POST, PUT, DELETE (৪টি মান)
    route="/api/payment",     // সীমিত সংখ্যক route
    status="200"              // 200, 400, 500 (সীমিত মান)
  }
  → ৪ method × ২০ route × ৫ status = ৪০০ time series ✅
```

---

## 🔗 সারসংক্ষেপ

```
Metrics & Alerting মনে রাখার সূত্র:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  ১. মেট্রিক্স টাইপ মনে রাখুন:                              │
│     Counter → "মোট কত?" (শুধু বাড়ে)                        │
│     Gauge   → "এখন কত?" (বাড়ে-কমে)                        │
│     Histogram → "বন্টন কেমন?" (buckets)                    │
│     Summary → "percentile কত?" (quantiles)                │
│                                                            │
│  ২. RED → সার্ভিসের জন্য (Rate, Errors, Duration)          │
│     USE → রিসোর্সের জন্য (Utilization, Saturation, Errors) │
│     4 Golden Signals → দুটোর সমন্বয়                        │
│                                                            │
│  ৩. Alert তৈরির নিয়ম:                                     │
│     • লক্ষণ-ভিত্তিক (symptom), কারণ-ভিত্তিক নয়            │
│     • Severity আলাদা করুন (P1 ≠ P4)                       │
│     • "এই alert এ কি কেউ action নেবে?" — না হলে বাদ       │
│     • Runbook লিংক দিন                                    │
│     • সপ্তাহে ২ বারের বেশি fire → tune করুন                 │
│                                                            │
│  ৪. Label cardinality সীমিত রাখুন                          │
│     user_id, request_id কখনো label হিসেবে নয়!             │
│                                                            │
│  ৫. গড় (average) বিশ্বাস করবেন না — percentile দেখুন       │
│     p50 = সাধারণ অভিজ্ঞতা, p99 = সবচেয়ে খারাপ অভিজ্ঞতা   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

> **পরবর্তী:** [Distributed Tracing](./distributed-tracing.md) — মাইক্রোসার্ভিস জুড়ে রিকোয়েস্ট ট্র্যাক করা
