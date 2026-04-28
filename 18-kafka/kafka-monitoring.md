# 📘 Kafka মনিটরিং (Monitoring & Observability)

> **"যেটা মাপা যায় না, সেটা উন্নত করা যায় না।"** — Peter Drucker
>
> Production-এ Kafka চালানো মানে শুধু ব্রোকার চালু রাখা না — সবকিছু সুস্থ আছে কিনা ক্রমাগত নজর রাখা।

---

## 📖 কেন মনিটরিং গুরুত্বপূর্ণ?

### 🏠 বাস্তব জীবনের উপমা

মনিটরিং হলো আপনার সিস্টেমের **স্বাস্থ্য পরীক্ষা (Health Checkup)**। আপনি ডাক্তারের কাছে যান,
ব্লাড প্রেশার মাপেন, সুগার চেক করেন — একইভাবে সিস্টেমেরও নিয়মিত চেকআপ দরকার।

ধরুন bKash-এর Kafka cluster মনিটর করা হচ্ছে না:
- পেমেন্ট ইভেন্ট প্রসেস হচ্ছে না → গ্রাহক টাকা পাঠালো কিন্তু রিসিভার পেলো না
- Consumer lag বাড়ছে → নোটিফিকেশন দেরিতে যাচ্ছে
- ব্রোকার ডিস্ক ভরে গেছে → পুরো ক্লাস্টার ক্র্যাশ

### 😱 Production Incident Scenarios

```
🔴 ঘটনা ১: Daraz Flash Sale
─────────────────────────────
সমস্যা: Order events-এর consumer lag ১ লক্ষ ছাড়িয়ে গেছে
কারণ:  Consumer group rebalance লুপে আটকে গেছে
ফল:    ৩০ মিনিট অর্ডার প্রসেস বন্ধ, কোটি টাকার ক্ষতি
সমাধান: মনিটরিং থাকলে ৫ মিনিটেই ধরা পড়তো

🔴 ঘটনা ২: bKash পেমেন্ট সিস্টেম
────────────────────────────────────
সমস্যা: UnderReplicatedPartitions বাড়তে থাকলো
কারণ:  একটা ব্রোকারের ডিস্ক ৯৮% ভরে গেছে
ফল:    ডেটা লস-এর ঝুঁকি তৈরি হলো
সমাধান: ডিস্ক usage অ্যালার্ট থাকলে আগেই সতর্ক হওয়া যেতো

🔴 ঘটনা ৩: Pathao রাইড ম্যাচিং
─────────────────────────────────
সমস্যা: Producer latency ৫ সেকেন্ড ছাড়িয়ে গেছে
কারণ:  acks=all সেটিং-এ একটা ব্রোকার ধীর ছিলো
ফল:    রাইডার আর প্যাসেঞ্জার ম্যাচ হচ্ছে না
সমাধান: latency মেট্রিক্স মনিটর করলে আগেই ব্যবস্থা নেওয়া যেতো
```

---

## 📊 মূল মেট্রিক্স (Key Metrics)

Kafka মনিটরিং-এ চারটি প্রধান ক্যাটাগরি আছে: **Broker, Producer, Consumer, এবং Topic**।

---

### 🖥️ Broker মেট্রিক্স

ব্রোকার হলো Kafka-র হৃদপিণ্ড। এগুলো সবচেয়ে গুরুত্বপূর্ণ মেট্রিক্স:

| মেট্রিক নাম | বিবরণ | অ্যালার্ট থ্রেশহোল্ড | গুরুত্ব |
|---|---|---|---|
| `UnderReplicatedPartitions` | যেসব পার্টিশনের রেপ্লিকা সিঙ্কে নেই | > 0 (১০ মিনিট ধরে) | 🔴 Critical |
| `ActiveControllerCount` | ক্লাস্টারে কয়টা কন্ট্রোলার আছে | != 1 | 🔴 Critical |
| `OfflinePartitionsCount` | অফলাইন পার্টিশন সংখ্যা | > 0 | 🔴 Critical |
| `RequestHandlerAvgIdlePercent` | রিকোয়েস্ট হ্যান্ডলার থ্রেড কতটা ফ্রি | < 0.3 (30%) | 🟡 Warning |
| `NetworkProcessorAvgIdlePercent` | নেটওয়ার্ক প্রসেসর থ্রেড কতটা ফ্রি | < 0.3 (30%) | 🟡 Warning |
| `UncleanLeaderElectionsPerSec` | ISR-এর বাইরে থেকে লিডার নির্বাচন | > 0 | 🔴 Critical |
| `TotalTimeMs (Produce)` | Produce রিকোয়েস্টের মোট সময় | > 100ms (p99) | 🟡 Warning |
| `TotalTimeMs (FetchConsumer)` | Consumer fetch-এর মোট সময় | > 200ms (p99) | 🟡 Warning |
| `TotalTimeMs (FetchFollower)` | Follower fetch-এর মোট সময় | > 200ms (p99) | 🟡 Warning |
| `BytesInPerSec` | প্রতি সেকেন্ডে আসা ডেটা | ক্যাপাসিটি অনুযায়ী | 📊 Info |
| `BytesOutPerSec` | প্রতি সেকেন্ডে বের হওয়া ডেটা | ক্যাপাসিটি অনুযায়ী | 📊 Info |

#### 🔎 গুরুত্বপূর্ণ মেট্রিকের ব্যাখ্যা

- **UnderReplicatedPartitions** — > 0 হলে ডেটা সব রেপ্লিকায় কপি হয়নি। ব্রোকার ক্র্যাশে ডেটা হারানোর ঝুঁকি।
- **ActiveControllerCount** — পুরো ক্লাস্টারে ঠিক ১টা কন্ট্রোলার থাকতে হবে। ০ = বিপদ, ২ = split-brain।
- **UncleanLeaderElectionsPerSec** — > 0 মানে ISR-বহির্ভূত রেপ্লিকা লিডার হয়েছে, ডেটা লসের সম্ভাবনা।

---

### 📤 Producer মেট্রিক্স

| মেট্রিক নাম | বিবরণ | অ্যালার্ট থ্রেশহোল্ড | গুরুত্ব |
|---|---|---|---|
| `record-send-rate` | প্রতি সেকেন্ডে পাঠানো রেকর্ড সংখ্যা | বেসলাইন থেকে ৫০% কমলে | 🟡 Warning |
| `record-error-rate` | প্রতি সেকেন্ডে ব্যর্থ রেকর্ড সংখ্যা | > 0 (sustained) | 🔴 Critical |
| `request-latency-avg` | গড় রিকোয়েস্ট লেটেন্সি | > 100ms | 🟡 Warning |
| `batch-size-avg` | গড় ব্যাচ সাইজ (বাইটে) | খুব ছোট হলে ইনএফিশিয়েন্ট | 📊 Info |
| `compression-rate-avg` | কম্প্রেশন রেশিও | > 0.8 হলে কম্প্রেশন কাজ করছে না | 📊 Info |
| `buffer-available-bytes` | Producer বাফারে ফাঁকা জায়গা | < 10% মোট বাফার | 🔴 Critical |
| `waiting-threads` | ব্লক হওয়া থ্রেড সংখ্যা | > 0 (sustained) | 🟡 Warning |

> 💡 **Nagad উদাহরণ:** Nagad-এর পেমেন্ট সার্ভিস যদি `record-error-rate` > 0 দেখায়,
> তাহলে পেমেন্ট ইভেন্ট Kafka-তে যাচ্ছে না — অবিলম্বে তদন্ত করা দরকার।

---

### 📥 Consumer মেট্রিক্স

| মেট্রিক নাম | বিবরণ | অ্যালার্ট থ্রেশহোল্ড | গুরুত্ব |
|---|---|---|---|
| `records-consumed-rate` | প্রতি সেকেন্ডে প্রসেস করা রেকর্ড | বেসলাইনের ৫০% কমলে | 🟡 Warning |
| `records-lag-max` | সর্বোচ্চ consumer lag | অ্যাপ অনুযায়ী (যেমন > 10000) | 🔴 Critical |
| `fetch-latency-avg` | গড় fetch লেটেন্সি | > 500ms | 🟡 Warning |
| `commit-latency-avg` | গড় offset commit লেটেন্সি | > 200ms | 🟡 Warning |
| `rebalance-latency-avg` | গড় রিব্যালেন্স সময় | > 60s | 🟡 Warning |
| `assigned-partitions` | এই consumer-এ অ্যাসাইন করা পার্টিশন | আকস্মিক পরিবর্তন | 📊 Info |

---

### 📋 Topic মেট্রিক্স

| মেট্রিক নাম | বিবরণ | কখন দেখবেন |
|---|---|---|
| `MessagesInPerSec` | প্রতি সেকেন্ডে আসা মেসেজ সংখ্যা | ট্রাফিক প্যাটার্ন বুঝতে |
| `BytesInPerSec` | প্রতি সেকেন্ডে আসা ডেটা (বাইট) | ডিস্ক ক্যাপাসিটি প্ল্যানিং |
| `BytesOutPerSec` | প্রতি সেকেন্ডে বের হওয়া ডেটা (বাইট) | নেটওয়ার্ক ব্যান্ডউইথ চেক |
| `PartitionCount` | মোট পার্টিশন সংখ্যা | ক্লাস্টার ব্যালেন্স দেখতে |

---

## 📈 কনজিউমার ল্যাগ মনিটরিং (Consumer Lag)

### ল্যাগ কী এবং কেন গুরুত্বপূর্ণ?

Consumer lag হলো **Producer যতটুকু লিখেছে আর Consumer যতটুকু পড়েছে তার পার্থক্য**।
এটা দিয়ে বোঝা যায় আপনার সিস্টেম কতটা পিছিয়ে আছে।

> 🏠 **বাংলাদেশি উপমা:** ধরুন Grameenphone-এর কল সেন্টারে কল আসছে প্রতি মিনিটে ১০০টা,
> কিন্তু এজেন্টরা প্রসেস করতে পারছে ৮০টা। তাহলে প্রতি মিনিটে ২০টা কল "ল্যাগ" হচ্ছে।
> এই ল্যাগ বাড়তে থাকলে একসময় সিস্টেম ভেঙে পড়বে।

### 📊 ল্যাগ ভিজুয়ালাইজেশন

```
Topic: payment-events, Partition 0

Offset Timeline:
0   10   20   30   40   50   60   70   80   90  100
|────|────|────|────|────|────|────|────|────|────|
                              ↑                    ↑
                        Consumer              Log End
                        Offset: 55           Offset: 100
                        LAG = 45 messages ⚠️

সময়    | Consumer Offset | Log End Offset | Lag
--------|-----------------|----------------|--------
10:00   |     40          |      50        |  10 ✅
10:05   |     45          |      65        |  20 🟡
10:10   |     50          |      85        |  35 🟠
10:15   |     55          |     100        |  45 🔴  ← বিপদ!
```

### 🔧 kafka-consumer-groups.sh দিয়ে ল্যাগ চেক

```bash
# সব consumer group দেখুন
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# নির্দিষ্ট group-এর ল্যাগ দেখুন
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group payment-processor

# আউটপুট:
# GROUP              TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# payment-processor  payment-events   0          55              100             45
# payment-processor  payment-events   1          80              95              15
# payment-processor  payment-events   2          120             125             5
```

### 🦅 Burrow দিয়ে ল্যাগ মনিটরিং

Burrow হলো LinkedIn-এর তৈরি একটা টুল যা consumer lag-এর **ট্রেন্ড** বিশ্লেষণ করে:

```
Burrow Lag Status:
─────────────────
OK     → ল্যাগ স্থিতিশীল বা কমছে
WARN   → ল্যাগ ধীরে ধীরে বাড়ছে
ERR    → ল্যাগ দ্রুত বাড়ছে, consumer আটকে গেছে
STOP   → Consumer পুরোপুরি বন্ধ হয়ে গেছে
STALL  → Consumer চলছে কিন্তু offset এগোচ্ছে না
```

### 🚨 ল্যাগ অ্যালার্ট কৌশল

```
✅ ভালো কৌশল:
  - ল্যাগ threshold + সময়কাল ভিত্তিক (যেমন: lag > 10000 for 5 min)
  - ল্যাগ বৃদ্ধির হার ভিত্তিক (যেমন: lag/sec > 100)

❌ খারাপ কৌশল:
  - শুধু নির্দিষ্ট সংখ্যা দিয়ে alert (spike-এ false alarm হয়)
  - ল্যাগ শূন্য না হলেই alert (কিছুটা ল্যাগ স্বাভাবিক)
```

---

## 🛠️ মনিটরিং টুলস

| টুল | ধরন | মূল সুবিধা | সেটআপ |
|---|---|---|---|
| **CMAK** | ওয়েব UI (Yahoo) | ক্লাস্টার ম্যানেজমেন্ট, টপিক তৈরি/মোছা | Docker image |
| **Conduktor** | ডেস্কটপ/ওয়েব UI | সুন্দর UI, Schema Registry সাপোর্ট, মেসেজ ব্রাউজিং | ডেস্কটপ অ্যাপ/Docker |
| **Burrow** | Consumer lag টুল | ল্যাগ ট্রেন্ড বিশ্লেষণ, HTTP API (LinkedIn তৈরি) | Go binary / Docker |
| **Prometheus+Grafana** | মেট্রিক্স + ড্যাশবোর্ড | কাস্টম মেট্রিক্স, PromQL, সুন্দর ড্যাশবোর্ড | JMX Exporter পাইপলাইন |
| **Kafka UI** | ওপেন সোর্স ওয়েব UI | হালকা, সহজ সেটআপ, মাল্টি-ক্লাস্টার | Docker + env vars |
| **JMX Exporter** | মেট্রিক্স রূপান্তর | Kafka JMX → Prometheus ফরম্যাট | Java agent |

### 📋 টুল তুলনা সারণী

```
টুল              | বিনামূল্যে | ল্যাগ   | ড্যাশবোর্ড | অ্যালার্ট | সেটআপ
─────────────────|───────────|────────|───────────|──────────|──────────
CMAK             |    ✅     |  ❌    |    ❌     |    ❌    | সহজ
Conduktor        |    ⚠️     |  ✅    |    ✅     |    ✅    | সহজ
Burrow           |    ✅     |  ✅    |    ❌     |    ✅    | মাঝারি
Prometheus+Grafana|   ✅     |  ✅    |    ✅     |    ✅    | জটিল
Kafka UI         |    ✅     |  ✅    |    ⚠️     |    ❌    | সহজ
```

> 💡 **সুপারিশ:** Production-এ **Prometheus + Grafana** ব্যবহার করুন। Development-এ **Kafka UI** যথেষ্ট।

---

## 📊 JMX মেট্রিক্স

### Kafka কিভাবে মেট্রিক্স এক্সপোজ করে?

Kafka সব মেট্রিক্স **JMX (Java Management Extensions)** এর মাধ্যমে প্রকাশ করে।
Prometheus দিয়ে এগুলো সংগ্রহ করতে **JMX Exporter** লাগে।

### গুরুত্বপূর্ণ JMX MBeans

```
# Broker মেট্রিক্স
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
kafka.controller:type=KafkaController,name=ActiveControllerCount
kafka.controller:type=KafkaController,name=OfflinePartitionsCount

# Request মেট্রিক্স
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer

# Topic ও Log মেট্রিক্স
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.server:type=ReplicaManager,name=IsrShrinksPerSec
```

### JMX Exporter কনফিগারেশন

```yaml
# jmx-exporter-config.yml
lowercaseOutputName: true
rules:
  - pattern: kafka.server<type=ReplicaManager, name=(.+)><>Value
    name: kafka_server_replicamanager_$1
    type: GAUGE
  - pattern: kafka.network<type=RequestMetrics, name=(.+), request=(.+)><>(\w+)
    name: kafka_network_requestmetrics_$1_$3
    labels:
      request: "$2"
  - pattern: kafka.server<type=BrokerTopicMetrics, name=(.+), topic=(.+)><>Count
    name: kafka_server_brokertopicmetrics_$1_total
    type: COUNTER
    labels:
      topic: "$2"
```

Kafka Broker-এ যোগ করতে:

```bash
export KAFKA_OPTS="-javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent.jar=7071:/opt/jmx-exporter/config.yml"
```

---

## 🔔 অ্যালার্টিং কৌশল (Alerting Strategy)

### 🔴 Critical Alerts — অবিলম্বে অন-কল ইঞ্জিনিয়ারকে জাগাতে হবে

```yaml
# Prometheus Alerting Rules

groups:
  - name: kafka-critical
    rules:
      # অফলাইন পার্টিশন — ডেটা অ্যাক্সেসযোগ্য নয়
      - alert: KafkaOfflinePartitions
        expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "🔴 Kafka-তে অফলাইন পার্টিশন পাওয়া গেছে"
          description: "{{ $value }}টা পার্টিশন অফলাইন। অবিলম্বে চেক করুন!"

      # কন্ট্রোলার সমস্যা — ক্লাস্টার ম্যানেজমেন্ট ব্যাহত
      - alert: KafkaNoActiveController
        expr: sum(kafka_controller_kafkacontroller_activecontrollercount) != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "🔴 Kafka ক্লাস্টারে কন্ট্রোলার সমস্যা"

      # আন্ডার-রেপ্লিকেটেড — ডেটা লসের ঝুঁকি
      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "🔴 Under-replicated partitions সনাক্ত হয়েছে"
```

### 🟡 Warning Alerts — কাজের সময়ে দেখতে হবে

```yaml
      # Consumer ল্যাগ বেশি
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumergroup_lag_sum > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "🟡 Consumer group {{ $labels.consumergroup }}-এর ল্যাগ বেশি"

      # Request লেটেন্সি বেশি
      - alert: KafkaProduceLatencyHigh
        expr: kafka_network_requestmetrics_totaltimems_mean{request="Produce"} > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "🟡 Produce লেটেন্সি {{ $value }}ms"

      # ডিস্ক ব্যবহার বেশি
      - alert: KafkaDiskUsageHigh
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "🟡 ডিস্ক ব্যবহার ৮০% ছাড়িয়ে গেছে"
```

### 📊 Info Alerts — ড্যাশবোর্ডে দেখা যথেষ্ট

```yaml
      - alert: KafkaIsrShrink
        expr: rate(kafka_server_replicamanager_isrshrinkspersec[5m]) > 0
        for: 5m
        labels: { severity: info }
        annotations:
          summary: "📊 ISR সংকোচন হচ্ছে, ব্রোকার চেক করুন"

      - alert: KafkaLeaderElection
        expr: rate(kafka_controller_controllerstats_leaderelectionrateandtimems_count[5m]) > 0
        for: 5m
        labels: { severity: info }
        annotations:
          summary: "📊 লিডার নির্বাচন ঘটেছে"
```

---

## 🏥 হেলথ চেক (Health Checks)

### ব্রোকার হেলথ চেক

```bash
#!/bin/bash
BOOTSTRAP_SERVER="localhost:9092"
echo "=== 🏥 Kafka Broker Health Check ==="

# ব্রোকার রিচেবল কিনা
kafka-broker-api-versions.sh --bootstrap-server $BOOTSTRAP_SERVER > /dev/null 2>&1 \
  && echo "✅ Broker reachable" || echo "❌ Broker unreachable"

# আন্ডার-রেপ্লিকেটেড পার্টিশন
URP=$(kafka-topics.sh --bootstrap-server $BOOTSTRAP_SERVER \
    --describe --under-replicated-partitions 2>/dev/null | wc -l)
[ "$URP" -eq 0 ] && echo "✅ No under-replicated partitions" \
  || echo "⚠️ $URP under-replicated partitions found"
```

### টপিক হেলথ চেক

```bash
#!/bin/bash
TOPIC="payment-events"
echo "=== 📋 Topic Health: $TOPIC ==="
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic $TOPIC

OFFLINE=$(kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --topic $TOPIC --unavailable-partitions 2>/dev/null | wc -l)
[ "$OFFLINE" -eq 0 ] && echo "✅ সব পার্টিশন অনলাইন" \
  || echo "❌ $OFFLINE পার্টিশন অফলাইন!"
```

### Consumer Group হেলথ চেক

```bash
#!/bin/bash
GROUP="payment-processor"; LAG_THRESHOLD=5000
echo "=== 👥 Consumer Group Health: $GROUP ==="

kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group $GROUP

TOTAL_LAG=$(kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group $GROUP 2>/dev/null | awk 'NR>1 {sum+=$6} END {print sum}')
echo "মোট ল্যাগ: $TOTAL_LAG"
[ "$TOTAL_LAG" -gt "$LAG_THRESHOLD" ] \
  && echo "🔴 ল্যাগ থ্রেশহোল্ড ছাড়িয়ে গেছে!" || echo "✅ ল্যাগ স্বাভাবিক"
```

---

## 📊 Grafana ড্যাশবোর্ড

### ড্যাশবোর্ড লেআউট

```
┌──────────────────────────────────────────────────────┐
│              Kafka Cluster Dashboard                  │
├────────────────┬────────────────┬────────────────────┤
│ Brokers: 3 ✅  │ Topics: 25     │ Partitions: 75     │
├────────────────┼────────────────┼────────────────────┤
│ Messages/sec   │ Bytes In/sec   │ Bytes Out/sec      │
│  ████ 50K      │  ████ 100MB    │  ████ 200MB        │
├────────────────┴────────────────┴────────────────────┤
│ Consumer Lag (Top 5 Groups)                           │
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ payment-svc: 1200                │
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓  notification-svc: 800                │
│ ▓▓▓▓▓▓▓▓▓  analytics-svc: 500                       │
│ ▓▓▓▓▓  sms-svc: 300                                 │
│ ▓▓▓  email-svc: 150                                 │
├──────────────────────────────────────────────────────┤
│ Under-Replicated Partitions    │ ISR Shrinks/Expands │
│  ──────── 0 ✅                 │  ────── stable ✅   │
├────────────────────────────────┴─────────────────────┤
│ Request Latency (p99)                                │
│ Produce: ██ 15ms  │ Fetch: ███ 25ms                  │
├──────────────────────────────────────────────────────┤
│ Disk Usage per Broker                                │
│ Broker-1: ████████░░ 65%                             │
│ Broker-2: ██████░░░░ 52%                             │
│ Broker-3: ███████░░░ 58%                             │
└──────────────────────────────────────────────────────┘
```

### মূল প্যানেল তালিকা

| প্যানেল | PromQL কোয়েরি | ধরন |
|---|---|---|
| Messages/sec | `sum(rate(kafka_server_brokertopicmetrics_messagesinpersec_count[5m]))` | Graph |
| Bytes In | `sum(rate(kafka_server_brokertopicmetrics_bytesinpersec_count[5m]))` | Graph |
| Consumer Lag | `sum by (consumergroup)(kafka_consumergroup_lag_sum)` | Bar Gauge |
| Under-replicated | `kafka_server_replicamanager_underreplicatedpartitions` | Stat |
| Produce Latency | `kafka_network_requestmetrics_totaltimems{request="Produce",quantile="0.99"}` | Graph |
| Disk Usage | `(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes` | Gauge |

> 💡 Grafana ড্যাশবোর্ড JSON রেফারেন্স: Grafana.com-এ **Dashboard ID 7589** (Kafka Overview)
> ইমপোর্ট করলেই রেডিমেড ড্যাশবোর্ড পেয়ে যাবেন।

---

## 💻 মনিটরিং সেটআপ উদাহরণ

### Docker Compose — Prometheus + Grafana + JMX Exporter

```yaml
# docker-compose.monitoring.yml
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_OPTS: >-
        -javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent-0.19.0.jar=7071:/opt/jmx-exporter/config.yml
    volumes:
      - ./jmx-exporter:/opt/jmx-exporter
    ports: ["9092:9092", "7071:7071"]

  prometheus:
    image: prom/prometheus:v2.47.0
    volumes: ["./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml"]
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:10.1.0
    environment: { GF_SECURITY_ADMIN_PASSWORD: admin }
    ports: ["3000:3000"]
    depends_on: [prometheus]
```

### Prometheus কনফিগারেশন

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
rule_files: ["kafka-alerts.yml"]

scrape_configs:
  - job_name: 'kafka-broker'
    static_configs:
      - targets: ['kafka-1:7071', 'kafka-2:7071', 'kafka-3:7071']
        labels: { cluster: 'production' }
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### মূল PromQL কোয়েরি

```promql
# মোট মেসেজ রেট
sum(rate(kafka_server_brokertopicmetrics_messagesinpersec_count[5m]))

# নির্দিষ্ট টপিকের মেসেজ রেট
sum(rate(kafka_server_brokertopicmetrics_messagesinpersec_count{topic="payment-events"}[5m]))

# Consumer group ল্যাগ
sum by (consumergroup, topic)(kafka_consumergroup_lag_sum)

# Produce লেটেন্সি p99
histogram_quantile(0.99, rate(kafka_network_requestmetrics_totaltimems_bucket{request="Produce"}[5m]))

# আন্ডার-রেপ্লিকেটেড পার্টিশন ও ডিস্ক usage
sum(kafka_server_replicamanager_underreplicatedpartitions)
100 * (1 - node_filesystem_avail_bytes{mountpoint="/var/kafka-logs"} / node_filesystem_size_bytes)
```

---

## 🎯 মনিটরিং বেস্ট প্র্যাকটিস

### ✅ Production মনিটরিং চেকলিস্ট

```
প্রাথমিক সেটআপ:
  ☐ JMX Exporter সব ব্রোকারে ইনস্টল করা হয়েছে
  ☐ Prometheus কনফিগার করে মেট্রিক্স সংগ্রহ চালু করা হয়েছে
  ☐ Grafana ড্যাশবোর্ড তৈরি করা হয়েছে
  ☐ Alertmanager কনফিগার করে নোটিফিকেশন চ্যানেল সেট করা হয়েছে

Critical অ্যালার্ট:
  ☐ OfflinePartitions > 0
  ☐ UnderReplicatedPartitions > 0 (১০ মিনিট ধরে)
  ☐ ActiveControllerCount != 1
  ☐ UncleanLeaderElections > 0
  ☐ Producer error rate > 0

Warning অ্যালার্ট:
  ☐ Consumer lag > threshold
  ☐ Produce/Fetch latency > threshold
  ☐ Disk usage > 80%
  ☐ Network thread idle < 30%
  ☐ Request handler idle < 30%

ড্যাশবোর্ড:
  ☐ ক্লাস্টার ওভারভিউ (broker status, topic count)
  ☐ থ্রুপুট (messages/sec, bytes/sec)
  ☐ Consumer lag (গ্রুপ অনুযায়ী)
  ☐ লেটেন্সি (produce, fetch)
  ☐ রিসোর্স ব্যবহার (CPU, মেমরি, ডিস্ক, নেটওয়ার্ক)

অপারেশনাল:
  ☐ লগ মনিটরিং ও নিয়মিত হেলথ চেক স্ক্রিপ্ট চালানো
  ☐ ক্যাপাসিটি প্ল্যানিং রিভিউ (মাসিক)
  ☐ অ্যালার্ট রুল রিভিউ ও রানবুক আপডেট (ত্রৈমাসিক)
```

### 🏠 বাংলাদেশি কোম্পানির জন্য পরামর্শ

| স্কেল | সুপারিশ |
|---|---|
| 🟢 ছোট স্টার্টআপ (< ৫ ব্রোকার) | Kafka UI + শেল স্ক্রিপ্ট, CLI দিয়ে lag চেক |
| 🟡 মাঝারি (৫-২০ ব্রোকার) | Prometheus + Grafana, Critical অ্যালার্ট, Slack নোটিফিকেশন |
| 🔴 বড় — bKash, Pathao, Daraz (২০+) | পূর্ণাঙ্গ Prometheus + Grafana + Alertmanager + Burrow + PagerDuty + রানবুক |

---

## 🔗 পরবর্তী টপিক

| টপিক | ফাইল |
|---|---|
| ⬅️ আগের | `kafka-security.md` |
| ➡️ পরের | `kafka-design-patterns.md` |

---

> **💡 মনে রাখুন:** "মনিটরিং ছাড়া Production-এ Kafka চালানো, চোখ বন্ধ করে ঢাকার ট্রাফিকে গাড়ি চালানোর মতো!" 🚗💥
