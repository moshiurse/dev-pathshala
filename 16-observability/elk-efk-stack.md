# 📌 ELK / EFK / EBK Stack — সেন্ট্রালাইজড লগিং ডিপ ডাইভ

> "তোমার ১০০টা পডে লগ ছড়িয়ে আছে — `kubectl logs` দিয়ে কয়টা দেখবে?
> সেন্ট্রালাইজড লগিং ছাড়া প্রোডাকশন চালানো অন্ধকারে দৌড়ানোর মতো।"

মাইক্রোসার্ভিস যুগে প্রতিটি সার্ভিস আলাদা পডে চলে, রিকোয়েস্ট ৫-৭টা সার্ভিস ঘুরে আসে,
কন্টেইনার রিস্টার্ট হলে stdout লগ মুছে যায়। **ELK / EFK Stack** সব লগ এক জায়গায় এনে
search, correlate, alert এবং visualize — চারটাই একসাথে দেয়।

এই ডকুমেন্টে আমরা ELK পরিবারের প্রতিটি কম্পোনেন্ট, প্রোডাকশন আর্কিটেকচার, K8s
ইন্টিগ্রেশন, এবং Daraz/Pathao/bKash-এর বাস্তব সেটআপ গভীরভাবে দেখব।

> 🔗 আগে পড়ুন: [`logging.md`](./logging.md), [`opentelemetry.md`](./opentelemetry.md),
> [`distributed-tracing.md`](./distributed-tracing.md), [`metrics-alerting.md`](./metrics-alerting.md)

---

## 📖 সূচিপত্র

1. [কেন সেন্ট্রালাইজড লগিং দরকার](#-কেন-সেন্ট্রালাইজড-লগিং-দরকার)
2. [Stack পরিবার — ELK, EFK, EBK, PLG](#-stack-পরিবার--elk-efk-ebk-plg)
3. [Elasticsearch — লগ ইঞ্জিন](#-elasticsearch--লগ-ইঞ্জিন)
4. [Logstash — পাইপলাইনের ভারী হাতি](#-logstash--পাইপলাইনের-ভারী-হাতি)
5. [Fluentd ও Fluent Bit](#-fluentd-ও-fluent-bit)
6. [Beats পরিবার (Filebeat ইত্যাদি)](#-beats-পরিবার-filebeat-ইত্যাদি)
7. [Kibana — UI ও ড্যাশবোর্ড](#-kibana--ui-ও-ড্যাশবোর্ড)
8. [প্রোডাকশন আর্কিটেকচার প্যাটার্ন](#-প্রোডাকশন-আর্কিটেকচার-প্যাটার্ন)
9. [Kubernetes-এ DaemonSet সেটআপ](#-kubernetes-এ-daemonset-সেটআপ)
10. [Index ডিজাইন ও ECS](#-index-ডিজাইন-ও-ecs)
11. [PHP / Node.js অ্যাপ ইন্টিগ্রেশন](#-php--nodejs-অ্যাপ-ইন্টিগ্রেশন)
12. [KQL Search ও Alerting](#-kql-search-ও-alerting)
13. [Performance, Cost, Sizing](#-performance-cost-sizing)
14. [Security ও Compliance](#-security-ও-compliance)
15. [Operations — ILM, Snapshot, DR](#-operations--ilm-snapshot-dr)
16. [বাস্তব সেটআপ — Daraz/Pathao/bKash/Startup](#-বাস্তব-সেটআপ--darazpathaobkashstartup)
17. [ELK vs Loki vs OpenSearch vs Datadog vs Splunk](#-elk-vs-loki-vs-opensearch-vs-datadog-vs-splunk)
18. [Anti-Patterns](#-anti-patterns)
19. [Production Checklist](#-production-checklist)

---

## 📊 কেন সেন্ট্রালাইজড লগিং দরকার

### সমস্যা — ছড়িয়ে থাকা লগ

```
  ❌ সেন্ট্রালাইজেশন ছাড়া (Daraz multi-region)

  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │  AWS Mumbai     │  │  AWS Singapore  │  │  On-prem DC      │
  │  ──────────────  │  │  ──────────────  │  │  ──────────────  │
  │  order-svc x40   │  │  search-svc x20 │  │  legacy ERP x5   │
  │  /var/log/*.log  │  │  stdout (k8s)   │  │  syslog          │
  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
           │                    │                    │
           ▼                    ▼                    ▼
        SSH+grep             kubectl logs          tail -f

  সমস্যা:
    ❌ ৬৫টা পডে কোন রিকোয়েস্টে এরর — খুঁজতে ৩০ মিনিট
    ❌ Pod রিস্টার্ট হলে stdout গায়েব
    ❌ trace_id দিয়ে cross-service correlation অসম্ভব
    ❌ অডিট লগ ৭ বছর রাখার প্রমাণ নেই (Bangladesh Bank issue)
    ❌ অ্যালার্ট সেট করা যায় না
```

### সমাধান — সব লগ এক জায়গায়

```
  ✅ Centralized (ELK/EFK)

   ┌────────────────────────────────────────────┐
   │           Elasticsearch Cluster             │
   │     (logs-app-*, logs-audit-*, ...)        │
   └────────────────▲───────────────────────────┘
                    │
        ┌───────────┼───────────┬──────────────┐
        │           │           │              │
   ┌────┴────┐ ┌────┴────┐ ┌───┴────┐  ┌──────┴────┐
   │Filebeat │ │Fluent   │ │Logstash│  │ App OTLP   │
   │ DS (k8s)│ │  Bit DS │ │ (Kafka)│  │ (direct)   │
   └────┬────┘ └────┬────┘ └───┬────┘  └────────────┘
        │           │          │
       Apps       Apps        Kafka
                              buffer
```

### Observability-র ৩ পিলার পুনরালোচনা

| পিলার | প্রশ্ন | টুল |
|-------|---------|------|
| **Logs** | "কী ঘটেছে?" — discrete event | ELK / EFK / Loki |
| **Metrics** | "কতটা / কত দ্রুত?" — aggregated number | Prometheus |
| **Traces** | "কোথায় গেল?" — request flow | Jaeger / Tempo |

ELK Stack মূলত **Logs** পিলারে কেন্দ্র, কিন্তু APM অ্যাড করলে metrics/traces-ও
ধরা যায়। আদর্শ হলো OTel দিয়ে তিনটাই পাঠানো — দেখুন
[`opentelemetry.md`](./opentelemetry.md)।

### Compliance — কেন বাধ্যতামূলক

bKash, Nagad, ব্যাংক — Bangladesh Bank-এর Guidelines on ICT Security অনুযায়ী:

- **অডিট লগ ৭ বছর** সংরক্ষণ (Anti-Money Laundering)
- **Tamper-proof storage** — লগ পরিবর্তন করা যাবে না (S3 Object Lock / WORM)
- **Access log** — কে কখন কোন রেকর্ড দেখল
- **Data residency** — কাস্টমার-সংবেদনশীল লগ Bangladesh-এ থাকতে হতে পারে

ELK Stack এই compliance-এর ভিত্তি। নিচে bKash সেটআপ-এ দেখব কীভাবে dual-write
(ES hot + S3 cold) দিয়ে এটা সমাধান করা হয়।

---

## 📊 Stack পরিবার — ELK, EFK, EBK, PLG

### পরিচিতি

```
  ┌───────────────────────────────────────────────────────────────┐
  │  ELK   = Elasticsearch + Logstash      + Kibana    (+Beats)  │
  │  EFK   = Elasticsearch + Fluentd/FBit  + Kibana              │
  │  EBK   = Elasticsearch + Beats only    + Kibana    (no LS)   │
  │  PLG   = Promtail      + Loki          + Grafana             │
  │  OSK   = OpenSearch    + ...           + OS Dashboards       │
  └───────────────────────────────────────────────────────────────┘
```

### তুলনা ম্যাট্রিক্স

| বৈশিষ্ট্য         | ELK            | EFK (Fluentd)  | EFK (Fluent Bit) | EBK            | PLG (Loki)         |
|-------------------|----------------|----------------|------------------|----------------|--------------------|
| Storage           | Elasticsearch  | Elasticsearch  | Elasticsearch    | Elasticsearch  | Loki (object store)|
| Shipper           | Logstash       | Fluentd (Ruby) | Fluent Bit (C)   | Beats direct   | Promtail           |
| Memory (shipper)  | 500MB-2GB JVM  | ~80MB          | **~10MB**        | ~30MB          | ~50MB              |
| Transformations   | খুব শক্তিশালী | শক্তিশালী      | মাঝারি           | সীমিত          | সীমিত (LogQL)      |
| K8s integration   | ভালো           | ভালো           | **চমৎকার**       | ভালো           | চমৎকার             |
| Cost              | মাঝারি         | মাঝারি         | কম               | কম             | **খুব কম**         |
| Query             | KQL/Lucene     | KQL/Lucene     | KQL/Lucene       | KQL/Lucene     | LogQL              |
| Full-text search  | ✅ চমৎকার     | ✅              | ✅                | ✅              | ❌ (label-only)    |
| BD startup ফিট    | মাঝারি         | মাঝারি         | ভালো             | ভালো           | চমৎকার ($)         |

### কখন কোনটা?

| পরিস্থিতি | পছন্দ |
|----------|--------|
| লেগেসি apps থেকে complex parsing দরকার | **ELK** (Logstash grok) |
| K8s ক্লাস্টার, structured JSON logs | **EFK Fluent Bit** |
| বাজেট কম, label-based search যথেষ্ট | **PLG (Loki)** |
| Elastic license সমস্যা / AWS managed | **OpenSearch** |
| ছোট সেটআপ, transformation কম | **EBK** (Filebeat → ES) |
| Enterprise + SLA + UEBA দরকার | **Splunk** (commercial) |

### Variants ব্যাখ্যা

- **OpenSearch** — Elastic 7.11-এ SSPL/Elastic License-এ গেলে AWS ফর্ক করল।
  Apache 2.0, free, AWS-এ managed হিসেবে available। API অনেকাংশে compatible।
- **Graylog** — MongoDB + Elasticsearch + Graylog server। সহজ UI, কিন্তু ES ছাড়া
  scale করে না।
- **Splunk** — ক্লাসিক, প্রচুর enterprise ফিচার, কিন্তু অসম্ভব দামি (~$2000/GB/yr)।
- **Datadog / New Relic** — SaaS, খুব সহজ, কিন্তু ingest cost ($0.10-$0.30/GB) +
  retention cost আলাদা। Bangladesh থেকে data egress + USD bill সমস্যা।

---

## 📊 Elasticsearch — লগ ইঞ্জিন

> 🔗 ES-এর ভিতরের ডিটেইল (inverted index, sharding, scoring) দেখুন
> [`05-database/elasticsearch.md`](../05-database/elasticsearch.md) এবং
> [`05-database/full-text-search.md`](../05-database/full-text-search.md)।
> এখানে আমরা শুধু **লগ-নির্দিষ্ট** বিষয়গুলো দেখব।

### Time-series Indices ও Data Streams

লগ মূলত append-only time-series ডেটা। Elasticsearch দুটি কৌশল দেয়:

**1. পুরনো ধাঁচ — Time-based indices**

```
logs-app-2024.01.15
logs-app-2024.01.16
logs-app-2024.01.17
...
```

প্রতিদিন নতুন index, একটা alias `logs-app` যেটা সবগুলোর উপর read করে।

**2. আধুনিক — Data Streams (ES 7.9+)**

```
PUT _data_stream/logs-order-prod
```

ES স্বয়ংক্রিয়ভাবে backing indices ম্যানেজ করে: `.ds-logs-order-prod-2024.01.15-000001`,
rollover, ILM, সব integrated। **নতুন সেটআপে এটাই recommended।**

### ILM Policy — Hot / Warm / Cold / Delete

```
   Hot (SSD, fresh)        Warm (HDD, older)      Cold (snapshot/S3)
   ┌───────────┐            ┌───────────┐          ┌──────────────┐
   │ 0-7 days  │ ──rollover│ 7-30 days │ ──move──│ 30-90 days    │
   │ index+    │  at 50GB │  read only│          │ searchable    │
   │ search    │  or 7d   │  fewer    │          │ snapshot (S3)  │
   │ replicas=1│           │ replicas=0│          │ replicas=0     │
   └───────────┘            └───────────┘          └──────────────┘
                                                          │
                                                          ▼
                                                    Delete @ 90d
```

**ILM Policy YAML:**

```json
PUT _ilm/policy/logs-app-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "7d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink":     { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "allocate":   { "include": { "data_tier": "data_warm" } },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "s3-mumbai-logs" },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

### Data Tiers (নোড রোল)

```
node.roles: [ data_hot ]    ─► NVMe SSD, high CPU,  fresh writes
node.roles: [ data_warm ]   ─► SATA SSD, medium CPU, older indices
node.roles: [ data_cold ]   ─► HDD or S3, low CPU,   read-mostly
node.roles: [ data_frozen ] ─► S3 only,  searchable snapshot
node.roles: [ master ]      ─► dedicated, 3 nodes recommended
node.roles: [ ingest ]      ─► ingest pipelines (alternative to LS)
node.roles: [ ml ]          ─► anomaly detection
node.roles: [ remote_cluster_client ] ─► cross-cluster search
```

প্রোডাকশনে **dedicated master + hot + warm** আলাদা রাখুন। ছোট ক্লাস্টারে
mixed roles চলে কিন্তু split-brain ঝুঁকি থাকে।

### Sharding — লগের জন্য Math

```
  rule of thumb:
    primary shard ~ 30-50 GB ideal
    total shards per node < 600 (ES 7+) / < 1000 (ES 8+)
    primary × (1 + replicas) ≤ data nodes (otherwise yellow)
```

উদাহরণ — Daraz, ১ TB/দিন ingest, ৭ দিন hot retention:

```
  Total hot = 7 × 1 TB × (1 + 1 replica) = 14 TB
  Primary shard target = 50 GB
  ⇒ primary shards needed = 14000 / 50 ≈ 280 (across 7 days)
  ⇒ per day ~ 40 primaries × 2 (replica) = 80 shards/day
  Data hot nodes (10 nodes × 2 TB SSD) = 20 TB capacity ✓
```

---

## 📊 Logstash — পাইপলাইনের ভারী হাতি

### Pipeline মডেল

```
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │  INPUT   │ ─►│  FILTER  │─►│  OUTPUT  │
  └─────────┘    └─────────┘    └─────────┘
   beats          grok           elasticsearch
   kafka          json           kafka
   tcp/udp        mutate         s3
   http           date           file
   syslog         geoip          stdout
   file           dissect        email
                  ruby
                  drop
                  if/else
```

### সম্পূর্ণ Production Pipeline — Apache Access + JSON Apps

`/etc/logstash/pipelines.yml`:

```yaml
- pipeline.id: apache-access
  path.config: "/etc/logstash/conf.d/apache.conf"
  pipeline.workers: 4
  pipeline.batch.size: 250
  queue.type: persisted
  queue.max_bytes: 4gb

- pipeline.id: app-json
  path.config: "/etc/logstash/conf.d/app.conf"
  pipeline.workers: 8
  pipeline.batch.size: 500
  queue.type: persisted
  dead_letter_queue.enable: true
```

`/etc/logstash/conf.d/apache.conf`:

```ruby
input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/logstash/certs/ls.crt"
    ssl_key         => "/etc/logstash/certs/ls.key"
  }
}

filter {
  if [fileset][module] == "apache" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
      tag_on_failure => ["_grokparsefailure_apache"]
    }

    date {
      match  => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
      target => "@timestamp"
      timezone => "Asia/Dhaka"
    }

    geoip {
      source => "clientip"
      target => "geo"
      fields => ["city_name","country_name","location","country_iso_code"]
    }

    useragent { source => "agent" target => "ua" }

    mutate {
      convert    => { "response" => "integer"  "bytes" => "integer" }
      remove_field => [ "message", "agent", "host" ]
      add_field    => { "[event][dataset]" => "apache.access" }
    }

    if [response] >= 500 {
      mutate { add_tag => [ "server_error" ] }
    }

    # PII redaction — credit card pattern
    mutate {
      gsub => [
        "request", "(?:\d[ -]*?){13,16}", "[REDACTED-CARD]"
      ]
    }
  }
}

output {
  if "_grokparsefailure_apache" in [tags] {
    file { path => "/var/log/logstash/apache-failed-%{+yyyy-MM-dd}.log" }
  } else {
    elasticsearch {
      hosts    => ["https://es-hot-01:9200","https://es-hot-02:9200"]
      user     => "${ES_USER}"
      password => "${ES_PASS}"
      ssl      => true
      cacert   => "/etc/logstash/certs/ca.crt"
      data_stream      => "true"
      data_stream_type => "logs"
      data_stream_dataset    => "apache.access"
      data_stream_namespace  => "prod"
    }
  }
}
```

### Persistent Queue ও DLQ

**Persistent Queue (PQ):** Logstash ক্র্যাশ করলে in-memory event হারায় না।
ডিস্কে WAL-এর মতো লেখে। `queue.type: persisted` enable করলেই হয়।

**Dead Letter Queue (DLQ):** যেসব event ES reject করে (mapping conflict ইত্যাদি)
সেগুলো DLQ-তে যায়, পরে আলাদা পাইপলাইনে fix করে replay করা যায়।

```ruby
input {
  dead_letter_queue {
    path             => "/var/lib/logstash/dead_letter_queue"
    pipeline_id      => "app-json"
    commit_offsets   => true
  }
}
filter {
  # mapping issue হলে field টাইপ ঠিক করুন
  mutate { convert => { "[user][id]" => "string" } }
}
output {
  elasticsearch { ... }
}
```

### Performance Tuning

| প্যারামিটার | ডিফল্ট | প্রোডাকশনে |
|------------|--------|-------------|
| `pipeline.workers` | CPU cores | 1×–2× cores |
| `pipeline.batch.size` | 125 | 500–1000 (heavy filter হলে কম) |
| `pipeline.batch.delay` | 50 ms | 50 ms (latency vs throughput) |
| JVM heap (`-Xms -Xmx`) | 1g | 4-8g (max 50% RAM, ≤ 31g) |
| `queue.max_bytes` | 1gb | 4-16gb |

### ⚠️ Logstash সতর্কতা

```
┌─────────────────────────────────────────────────────┐
│ Logstash একটা JVM process — RAM হাঙ্গরি!           │
│                                                      │
│  • Minimum  : 1 GB heap, 2 vCPU                     │
│  • Typical  : 4-8 GB heap, 4 vCPU                   │
│  • Heavy    : 8-16 GB heap, 8 vCPU                  │
│                                                      │
│ K8s sidecar হিসেবে Logstash চালাবেন না — overkill! │
│ Edge-এ Filebeat/Fluent Bit, কেন্দ্রে Logstash।     │
└─────────────────────────────────────────────────────┘
```

---

## 📊 Fluentd ও Fluent Bit

### Fluentd — Ruby-Based, Plugin-Heavy

Fluentd CNCF-graduated প্রজেক্ট। 1000+ plugin, কিন্তু Ruby — তাই Fluent Bit-এর
চেয়ে ভারী।

**Config syntax:**

```
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_key time
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<filter kubernetes.**>
  @type kubernetes_metadata
  kubernetes_url https://kubernetes.default.svc
  bearer_token_file /var/run/secrets/kubernetes.io/serviceaccount/token
</filter>

<filter kubernetes.**>
  @type record_transformer
  enable_ruby true
  <record>
    service ${record.dig("kubernetes","labels","app") || "unknown"}
    env     ${record.dig("kubernetes","namespace_name")}
    host    "#{Socket.gethostname}"
  </record>
</filter>

# PII redaction
<filter kubernetes.var.log.containers.payment**>
  @type record_modifier
  <replace>
    key     msisdn
    expression /(\d{3})\d{5}(\d{3})/
    replace  \1*****\2
  </replace>
</filter>

<match kubernetes.**>
  @type copy

  <store>
    @type elasticsearch
    host es-hot.elastic.svc
    port 9200
    scheme https
    user "#{ENV['ES_USER']}"
    password "#{ENV['ES_PASS']}"
    logstash_format true
    logstash_prefix logs-k8s
    logstash_dateformat %Y.%m.%d
    include_tag_key true
    type_name _doc
    <buffer>
      @type file
      path /var/log/fluentd-buffers/es.buffer
      flush_thread_count 4
      flush_interval 5s
      chunk_limit_size 8MB
      total_limit_size 8GB
      retry_max_interval 30
      retry_forever false
      overflow_action drop_oldest_chunk
    </buffer>
  </store>

  <store>
    @type kafka2
    brokers kafka1:9092,kafka2:9092
    default_topic raw-logs
    use_event_time true
    <format> @type json </format>
  </store>
</match>
```

**Buffer**: Fluentd-এর সবচেয়ে গুরুত্বপূর্ণ অংশ। `memory` (দ্রুত, restart-এ যায়)
বা `file` (slow, persistent)। প্রোডাকশনে `file` recommended।

### Fluent Bit — C-Based, Lightweight

K8s edge / sidecar / DaemonSet-এর জন্য ideal। **~10 MB RAM**, single binary।

**`fluent-bit.conf`:**

```ini
[SERVICE]
    Flush         1
    Daemon        Off
    Log_Level     info
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Port     2020
    Health_Check  On
    storage.path  /var/log/flb-storage/
    storage.sync  normal
    storage.backlog.mem_limit 50M

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    DB                /var/log/flb_kube.db
    storage.type      filesystem

[INPUT]
    Name              systemd
    Tag               host.*
    Systemd_Filter    _SYSTEMD_UNIT=kubelet.service
    Read_From_Tail    On

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log           On
    Merge_Log_Key       log_processed
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On
    Annotations         Off
    Labels              On

# Multi-line stack trace merge (Java/Node)
[FILTER]
    Name              multiline
    Match             kube.*
    multiline.parser  java,go,python
    multiline.key_content  log

# Drop noisy health-check logs
[FILTER]
    Name    grep
    Match   kube.*
    Exclude log /healthz|/metrics|GET /ping/

# Add ECS-style fields
[FILTER]
    Name    modify
    Match   kube.*
    Rename  kubernetes.namespace_name  service.namespace
    Rename  kubernetes.labels.app      service.name
    Add     event.dataset              k8s.app

[OUTPUT]
    Name             es
    Match            kube.*
    Host             es-hot.elastic.svc
    Port             9200
    HTTP_User        ${ES_USER}
    HTTP_Passwd      ${ES_PASS}
    tls              On
    tls.verify       On
    Logstash_Format  On
    Logstash_Prefix  logs-k8s
    Time_Key         @timestamp
    Replace_Dots     On
    Suppress_Type_Name On
    Retry_Limit      False
    storage.total_limit_size 5G
```

### Fluentd vs Fluent Bit

| | Fluentd | Fluent Bit |
|--|---------|-------------|
| ভাষা | Ruby + C | **C** |
| RAM | ~80 MB | **~10 MB** |
| Plugin | 1000+ | ~100 |
| K8s DS উপযোগী | ঠিকঠাক | **চমৎকার** |
| Aggregator | **চমৎকার** | মাঝারি |
| Pattern | Edge **+** Aggregator | শুধু Edge |

> **আদর্শ p পদ্ধতি**: K8s নোডে **Fluent Bit DaemonSet**, কেন্দ্রে যদি ভারী
> transformation দরকার তবে **Fluentd Aggregator** বা **Logstash**।

---

## 📊 Beats পরিবার (Filebeat ইত্যাদি)

Beats হলো হালকা single-purpose shipper। Go-তে লেখা।

| Beat | কাজ |
|------|-----|
| **Filebeat** | লগ ফাইল tail করে |
| **Metricbeat** | system/app metrics |
| **Packetbeat** | network packet inspection |
| **Heartbeat** | uptime monitoring (ICMP/HTTP/TCP) |
| **Auditbeat** | OS audit events (auditd, file integrity) |
| **Winlogbeat** | Windows event log |
| **Functionbeat** | AWS Lambda/serverless logs |

### Filebeat — Production Config

`/etc/filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx/access.log
    parsers:
      - ndjson:
          target: ""
          add_error_key: true
    fields:
      service.name: nginx
      event.dataset: nginx.access
      env: prod
    fields_under_root: true

  - type: filestream
    id: laravel-app
    paths:
      - /var/www/storage/logs/laravel-*.log
    parsers:
      - multiline:
          type: pattern
          pattern: '^\['
          negate: true
          match: after
          max_lines: 500
          timeout: 5s
    fields:
      service.name: order-svc

filebeat.modules:
  - module: nginx
    access:  { enabled: true }
    error:   { enabled: true }
  - module: mysql
    slowlog: { enabled: true }
  - module: system
    syslog:  { enabled: true }
    auth:    { enabled: true }

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
  - drop_event:
      when:
        contains:
          message: "GET /healthz"
  # PII redaction
  - script:
      lang: javascript
      source: >
        function process(event) {
          var msg = event.Get("message");
          if (msg) event.Put("message", msg.replace(/\b\d{16}\b/g, "[CARD]"));
        }

# Ship to Logstash (preferred) — buffered + transformations
output.logstash:
  hosts: ["logstash-01:5044","logstash-02:5044"]
  loadbalance: true
  worker: 2
  bulk_max_size: 2048
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  ssl.certificate: "/etc/filebeat/client.crt"
  ssl.key: "/etc/filebeat/client.key"

# Backpressure-aware queue
queue.disk:
  max_size: 2GB
  path: /var/lib/filebeat/queue

logging.level: info
logging.to_files: true
monitoring.enabled: true
```

### Multiline Pattern — Java Stack Trace মার্জ

```yaml
parsers:
  - multiline:
      type: pattern
      pattern: '^[[:space:]]+(at|\.{3})[[:space:]]+|^Caused by:'
      negate: false
      match: after
```

PHP/Laravel-এর জন্য `^\[` (timestamp bracket দিয়ে শুরু), Node.js এর জন্য
`^\s+at\s` (stack frame indent)।

---

## 📊 Kibana — UI ও ড্যাশবোর্ড

### মূল অ্যাপস

```
  ┌───────────────────────────────────────────────────────────┐
  │  Kibana                                                   │
  ├───────────────────────────────────────────────────────────┤
  │  Discover    — raw log explore + KQL filter              │
  │  Lens        — drag-drop visualization                   │
  │  Dashboard   — saved viz combine                         │
  │  Maps        — geoip-based map                           │
  │  Canvas      — pixel-perfect report                      │
  │  Alerting    — rule + connector (slack/email/webhook)    │
  │  Stack Mgmt  — index pattern, ILM, role, space           │
  │  ML          — anomaly detection (paid)                  │
  │  APM         — application performance                   │
  │  Observability — logs + metrics + traces unified         │
  └───────────────────────────────────────────────────────────┘
```

### KQL — Kibana Query Language

```
# সব 5xx error order সার্ভিসে
service.name : "order-svc" and http.response.status_code >= 500

# trace correlation (Jaeger trace_id)
trace.id : "4bf92f3577b34da6a3ce929d0e0e4736"

# slow query
event.dataset : "mysql.slowlog" and event.duration > 1000000000

# bKash পেমেন্ট fail, বিগত ১ ঘণ্টা
service.name : "payment-svc" and event.outcome : "failure"
  and gateway : "bkash"

# wildcard + range + negation
log.level : (ERROR or FATAL)
  and not message : "*health*"
  and @timestamp >= "now-15m"

# geo filter (Maps app)
geo.country_iso_code : "BD" and http.response.status_code : 500
```

### Alert Rule উদাহরণ — Error Rate Spike

```
Rule type    : Elasticsearch query
Index        : logs-app-*
Query (KQL)  : log.level: ERROR and service.name: "checkout-svc"
Threshold    : count > 50 in last 5 minutes
Schedule     : every 1 minute
Action       : Slack #incidents + PagerDuty
Throttle     : 10 minutes (avoid spam)
```

### Spaces ও RBAC

```
  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
  │ Space: payments │ │ Space: logistics│ │ Space: audit    │
  │ ─────────────── │ │ ─────────────── │ │ ─────────────── │
  │ idx: logs-pay-* │ │ idx: logs-log-* │ │ idx: logs-aud-* │
  │ users: pay-team │ │ users: ops      │ │ users: compl.   │
  └─────────────────┘ └─────────────────┘ └─────────────────┘

  Field-level security:
    role "support" → masking on user.email, user.phone, card.*
  Document-level security:
    role "vendor-x" → query: vendor.id == "x"
```

---

## 📊 প্রোডাকশন আর্কিটেকচার প্যাটার্ন

### প্যাটার্ন ১ — Single Node Lab

```
  App → Filebeat → Elasticsearch (1 node) → Kibana
```

ডেভেলপমেন্ট/POC-তে ঠিক, **প্রোডাকশনে কখনোই না**।

### প্যাটার্ন ২ — Multi-Node Hot/Warm

```
                      ┌──────────────────────┐
                      │  3× dedicated master  │
                      └──────────┬───────────┘
                                 │
              ┌──────────────────┼─────────────────┐
              ▼                  ▼                 ▼
       ┌────────────┐     ┌────────────┐    ┌────────────┐
       │ Hot ×6      │     │ Warm ×3    │    │ Cold ×2    │
       │ NVMe SSD    │     │ SATA SSD   │    │ HDD / S3   │
       │ replicas=1  │     │ replicas=0 │    │ snapshot   │
       └────────────┘     └────────────┘    └────────────┘
              ▲                  ▲                 ▲
              └──── ILM rollover────────┘──── searchable───┘
                                                snapshot
```

### প্যাটার্ন ৩ — Kafka Buffer (Spike Absorption)

```
  App → Filebeat → Kafka → Logstash → Elasticsearch
                    ▲          │
                    │          └─► (parallel) Spark/Flink for analytics
                    │
                trafiic spike হলে Kafka শোষণ করে
```

🔗 Kafka details: `18-kafka/`. প্রতি Black Friday-তে Daraz এই প্যাটার্ন ব্যবহার
করে — Logstash কখনই overwhelm হয় না।

### প্যাটার্ন ৪ — Multi-Cluster Fan-out

```
  Region A (Mumbai)        Region B (Singapore)
  ┌──────────────┐         ┌──────────────┐
  │ ES cluster 1 │         │ ES cluster 2 │
  └──────┬───────┘         └──────┬───────┘
         └─────────┬───────────────┘
                   ▼
            ┌────────────┐
            │ Cross-Cluster│
            │   Search    │  ← Kibana single pane of glass
            └────────────┘
```

### প্যাটার্ন ৫ — Hot/Cold Dual Write

```
          ┌─► ES Hot (search 90d)
   App ──►│
          └─► S3 (archive 7 years, WORM/Object Lock)
```

bKash/ব্যাংকের জন্য বাধ্যতামূলক। নিচে পূর্ণ সেটআপ।

---

## 📊 Kubernetes-এ DaemonSet সেটআপ

DaemonSet মানে প্রতিটি নোডে একটা পড — সেই নোডের সব কন্টেইনারের stdout সংগ্রহ করে।
Sidecar প্যাটার্ন ব্যবহার করুন **শুধু** যখন অ্যাপ লগ ফাইলে লেখে এবং stdout-এ
আনা সম্ভব না।

### পূর্ণ Fluent Bit DaemonSet (Pathao সেটআপ)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
  - apiGroups: [""]
    resources: ["namespaces","pods"]
    verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: logging
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush 1
        Log_Level info
        Daemon off
        Parsers_File parsers.conf
        HTTP_Server On
        HTTP_Port 2020
        storage.path /var/log/flb-storage/

    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser cri
        Tag kube.*
        Mem_Buf_Limit 50MB
        Skip_Long_Lines On
        DB /var/log/flb_kube.db

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [FILTER]
        Name multiline
        Match kube.*
        multiline.parser java,go,python

    [FILTER]
        Name modify
        Match kube.*
        Rename kubernetes.namespace_name service.namespace
        Rename kubernetes.labels.app     service.name
        Rename kubernetes.pod_name       host.name

    [OUTPUT]
        Name es
        Match kube.*
        Host elasticsearch.logging.svc
        Port 9200
        HTTP_User ${ES_USER}
        HTTP_Passwd ${ES_PASS}
        tls On
        tls.verify On
        Logstash_Format On
        Logstash_Prefix logs-k8s
        Replace_Dots On
        Retry_Limit False
        storage.total_limit_size 5G

  parsers.conf: |
    [PARSER]
        Name cri
        Format regex
        Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector: { matchLabels: { k8s-app: fluent-bit } }
  template:
    metadata:
      labels: { k8s-app: fluent-bit }
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists
          effect: NoSchedule
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          imagePullPolicy: IfNotPresent
          ports:
            - { containerPort: 2020, name: http }
          env:
            - name: ES_USER
              valueFrom: { secretKeyRef: { name: es-creds, key: user } }
            - name: ES_PASS
              valueFrom: { secretKeyRef: { name: es-creds, key: pass } }
          resources:
            requests: { cpu: 100m, memory: 64Mi }
            limits:   { cpu: 500m, memory: 256Mi }
          volumeMounts:
            - name: varlog;     mountPath: /var/log
            - name: varlibdocker; mountPath: /var/lib/docker/containers; readOnly: true
            - name: config;     mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog;       hostPath: { path: /var/log }
        - name: varlibdocker; hostPath: { path: /var/lib/docker/containers }
        - name: config;       configMap: { name: fluent-bit-config }
```

> 🔗 K8s বেসিকস: [`10-devops/kubernetes.md`](../10-devops/kubernetes.md)

### Sidecar প্যাটার্ন (যখন stdout নয়)

```yaml
spec:
  containers:
    - name: app
      image: legacy-php:7.4
      volumeMounts:
        - { name: app-logs, mountPath: /var/www/storage/logs }
    - name: log-shipper
      image: docker.elastic.co/beats/filebeat:8.12.0
      args: ["-e","-c","/etc/filebeat/filebeat.yml"]
      volumeMounts:
        - { name: app-logs, mountPath: /logs, readOnly: true }
        - { name: fb-config, mountPath: /etc/filebeat }
  volumes:
    - { name: app-logs, emptyDir: {} }
    - { name: fb-config, configMap: { name: fb-sidecar-config } }
```

---

## 📊 Index ডিজাইন ও ECS

### Component Templates + Index Templates

```json
PUT _component_template/logs-mappings
{
  "template": {
    "mappings": {
      "dynamic": "false",
      "properties": {
        "@timestamp":              { "type": "date" },
        "log.level":               { "type": "keyword" },
        "service.name":            { "type": "keyword" },
        "service.namespace":       { "type": "keyword" },
        "host.name":               { "type": "keyword" },
        "trace.id":                { "type": "keyword" },
        "span.id":                 { "type": "keyword" },
        "event.dataset":           { "type": "keyword" },
        "event.outcome":           { "type": "keyword" },
        "event.duration":          { "type": "long" },
        "http.request.method":     { "type": "keyword" },
        "http.response.status_code": { "type": "short" },
        "url.path":                { "type": "keyword" },
        "user.id":                 { "type": "keyword" },
        "geo.country_iso_code":    { "type": "keyword" },
        "message":                 { "type": "text" },
        "error.stack_trace":       { "type": "text", "index": false }
      }
    }
  }
}

PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "number_of_shards":   1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-app-policy",
      "index.codec": "best_compression"
    }
  }
}

PUT _index_template/logs-app
{
  "index_patterns": ["logs-app-*"],
  "data_stream": {},
  "composed_of": ["logs-settings","logs-mappings"],
  "priority": 200
}
```

### ECS — Elastic Common Schema

ECS হলো একটা **স্ট্যান্ডার্ড field naming convention**। বিভিন্ন সোর্সের লগে
যেন একই অর্থে একই field থাকে।

| পুরনো | ECS |
|------|-----|
| `app`, `application` | `service.name` |
| `level`, `severity` | `log.level` |
| `userid`, `uid` | `user.id` |
| `ip`, `client_ip` | `client.ip` / `source.ip` |
| `traceid` | `trace.id` |

ECS অনুসরণ করলে — Filebeat module, APM, OTel — সবাই same field-এ লেখে। Kibana
dashboards reusable হয়।

### Mapping Explosion এড়ানো

বিপদ: dynamic JSON-এ random keys (e.g. `details.{userId}`) → প্রতিটা key
আলাদা mapping field → মেমরি ফেটে যায়।

সমাধান:

```json
"mappings": {
  "dynamic": "strict",        // অজানা field reject
  "_source": { "enabled": true }
}
```

বা runtime field — write-time parse না করে read-time parse:

```json
PUT logs-app/_mapping
{
  "runtime": {
    "user_country": {
      "type": "keyword",
      "script": "emit(doc['client.ip'].value.substring(0,3))"
    }
  }
}
```

---

## 📊 PHP / Node.js অ্যাপ ইন্টিগ্রেশন

### Laravel — Monolog JSON to stdout

`config/logging.php`:

```php
'channels' => [
    'json_stdout' => [
        'driver'    => 'monolog',
        'level'     => env('LOG_LEVEL', 'info'),
        'handler'   => Monolog\Handler\StreamHandler::class,
        'with'      => [ 'stream' => 'php://stdout' ],
        'formatter' => Monolog\Formatter\JsonFormatter::class,
        'tap'       => [ App\Logging\AddEcsContext::class ],
    ],
],
```

`app/Logging/AddEcsContext.php`:

```php
<?php
namespace App\Logging;

use Monolog\LogRecord;
use Monolog\Processor\ProcessorInterface;
use OpenTelemetry\API\Trace\TraceContextValidator;
use OpenTelemetry\API\Globals;

class AddEcsContext
{
    public function __invoke($logger): void
    {
        $logger->pushProcessor(new class implements ProcessorInterface {
            public function __invoke(LogRecord $record): LogRecord
            {
                $span = Globals::tracerProvider()
                    ->getTracer('app')->getCurrentSpan();
                $ctx  = $span ? $span->getContext() : null;

                $extra = [
                    'service.name'      => config('app.name'),
                    'service.namespace' => config('app.env'),
                    'host.name'         => gethostname(),
                    'log.level'         => strtolower($record->level->name),
                ];
                if ($ctx && $ctx->isValid()) {
                    $extra['trace.id'] = $ctx->getTraceId();
                    $extra['span.id']  = $ctx->getSpanId();
                }
                return $record->with(extra: array_merge($record->extra, $extra));
            }
        });
    }
}
```

ব্যবহার:

```php
Log::channel('json_stdout')->info('Payment captured', [
    'event.dataset'    => 'payment.capture',
    'event.outcome'    => 'success',
    'event.duration'   => 1_234_000_000,  // ns
    'user.id'          => $user->id,
    'transaction.id'   => $txn->id,
    'gateway'          => 'bkash',
    'amount'           => 500.00,
    'currency'         => 'BDT',
]);
```

stdout-এ JSON → Filebeat/Fluent Bit ধরে ES-এ পাঠাবে।

### Node.js — pino + pino-elasticsearch

```js
// logger.js
const pino = require('pino');
const ecsFormat = require('@elastic/ecs-pino-format');
const { trace } = require('@opentelemetry/api');

const logger = pino({
  ...ecsFormat({ convertReqRes: true }),
  level: process.env.LOG_LEVEL || 'info',
  base: {
    'service.name':      process.env.SERVICE_NAME,
    'service.namespace': process.env.NODE_ENV,
    'host.name':         require('os').hostname(),
  },
  mixin() {
    const span = trace.getActiveSpan();
    if (!span) return {};
    const { traceId, spanId } = span.spanContext();
    return { 'trace.id': traceId, 'span.id': spanId };
  },
  redact: {
    paths: ['*.password', '*.card_number', '*.cvv', '*.otp'],
    censor: '[REDACTED]',
  },
});

module.exports = logger;
```

ব্যবহার:

```js
logger.info({
  'event.dataset': 'order.create',
  'event.outcome': 'success',
  'user.id':       userId,
  'order.id':      orderId,
  'order.amount':  amount,
}, 'Order created');

logger.error({
  err,
  'event.dataset': 'order.create',
  'event.outcome': 'failure',
}, 'Order failed');
```

### Multi-line Stack Trace Strategy

| ভাষা | শিপারে multiline pattern |
|------|---------------------------|
| Java/Kotlin | `^\s+at\s` (continuation) negate=false |
| Python | `^\s+File\s` বা `^Traceback` start |
| PHP/Laravel | `^\[\d{4}-` (timestamp) negate=true match=after |
| Node.js | `^\s+at\s` continuation |

**আদর্শ**: অ্যাপেই JSON-এ stack trace single field হিসেবে লিখুন (`error.stack_trace`)
— তাহলে multiline parser লাগবে না।

### Sampling — Noisy Logs

```js
// pino — DEBUG লগে 1% sampling
const sampled = pino({
  hooks: {
    logMethod(args, method, level) {
      if (level === 20 /* debug */ && Math.random() > 0.01) return;
      return method.apply(this, args);
    }
  }
});
```

বা Fluent Bit-এ:

```ini
[FILTER]
    Name    throttle
    Match   kube.*
    Rate    100
    Window  10
    Interval 1s
```

---

## 📊 KQL Search ও Alerting

### প্রোডাকশন Triage Queries

```
# 1. order সার্ভিসে শেষ ১ ঘণ্টায় সব 5xx
service.name : "order-svc"
  and http.response.status_code >= 500
  and @timestamp >= "now-1h"

# 2. ১ সেকেন্ডের বেশি query, user-ভিত্তিক
event.dataset : "mysql.slowlog"
  and event.duration > 1000000000
  | stats count by user.id    # (Lens aggregation)

# 3. trace_id দিয়ে cross-service journey
trace.id : "4bf92f3577b34da6a3ce929d0e0e4736"
  | sort @timestamp asc

# 4. Failed payment, geo মানচিত্রে
event.dataset : "payment.capture"
  and event.outcome : "failure"
  and gateway : ("bkash" or "nagad")
  → Maps app: cluster by geo.location

# 5. অস্বাভাবিক login (KQL + transform)
event.dataset : "auth.login"
  and event.outcome : "failure"
  and source.ip.geo.country_iso_code != "BD"
```

### Alerting — Spike Detection

```
Rule: Error rate spike (checkout-svc)
─────────────────────────────────────
Type     : Threshold
Index    : logs-app-*
Filter   : log.level: ERROR and service.name: "checkout-svc"
Group by : service.name
Window   : last 5 minutes
Threshold: count > 100
Schedule : every 1 min
Actions  :
  - Slack #alerts-payments
  - PagerDuty (severity=high)
  - Webhook → ChatOps bot

Rule: Specific exception spike
──────────────────────────────
Filter   : error.type : "DeadlockException"
Threshold: count > 5 in 1 min
Actions  : page DBA on-call

Rule: Missing heartbeat
──────────────────────
Filter   : event.dataset : "heartbeat.summary" and monitor.id: "bkash-prod"
Window   : last 3 minutes
Threshold: count == 0    (aha — কোনো heartbeat আসেনি!)
Actions  : page SRE
```

### Anomaly Detection (X-Pack)

ML জব — প্রতি service-এ baseline শিখে, বিচ্যুতি হলে বলে। লাইসেন্স paid (basic
trial available)। উদাহরণ: order-svc-এর গড় error rate প্রতি 3 PM-এ 2%, হঠাৎ
8 PM-এ 15% — ML alert।

---

## 📊 Performance, Cost, Sizing

### Sizing Rules of Thumb

```
  Daily ingest  : G GB/day  (compressed ~ 0.4×G after _source compression)
  Retention hot : H days
  Replicas      : R   (1 typical)
  Disk overhead : ~25% (translog, segments merge)

  Hot disk needed = G × H × (1 + R) × 1.25
  e.g. 100 GB/day × 7d × 2 × 1.25 = 1750 GB ≈ 1.75 TB
```

```
  RAM rule:
    Heap = min(50% RAM, 31 GB)
    OS file cache = remaining RAM (very important!)
    Total docs/node ≤ ~600M (heap-bound)
```

### Compression

```json
"index.codec": "best_compression"   // Deflate, ~50% smaller, ~10% slower search
```

লগের জন্য `best_compression` সাধারণত win — search rare, write/store expensive।

### Sampling — High-Volume Service

```
  ❌ সব access log ES-এ ⇒ ৫০০ GB/day, $5000/mo
  ✅ 100% error + 1% success ⇒ ~10 GB/day, $100/mo

  Filebeat dissect + drop_event:
    if response_code < 400 and rand() > 0.01 → drop
```

### Cold Tier — S3 Searchable Snapshot

```json
PUT _snapshot/s3-mumbai-logs
{
  "type": "s3",
  "settings": {
    "bucket": "darza-logs-cold",
    "region": "ap-south-1",
    "base_path": "es-snapshots",
    "compress": true,
    "server_side_encryption": true
  }
}
```

ILM `cold` phase-এ `searchable_snapshot` action ⇒ index S3-এ shift, কিন্তু
Kibana থেকে query করা যায় (slow)।

### SaaS vs Self-hosted — BD স্টার্টআপ পার্সপেক্টিভ

| | Self-hosted ELK | Datadog Logs | New Relic |
|--|----------------|--------------|-----------|
| Setup time | 1-2 সপ্তাহ | ১ ঘণ্টা | ১ ঘণ্টা |
| 100 GB/day cost | ~$300/mo (DO) | ~$1500/mo + retention | ~$1200/mo |
| BD bandwidth | ✅ Mumbai region | ❌ US/EU egress | ❌ US egress |
| Operational pain | মিড-হাই | শূন্য | শূন্য |
| Compliance customization | ✅ পূর্ণ control | সীমিত | সীমিত |
| ভাল কেস | seed/scale-up | well-funded SaaS | well-funded SaaS |

> Bangladesh-এ USD পেমেন্ট + bandwidth-এর কারণে self-hosted (DO/AWS Mumbai)
> প্রায় সবসময় ৫×–১০× সস্তা।

---

## 📊 Security ও Compliance

### Field-level Redaction (PII)

**Logstash:**

```ruby
filter {
  mutate {
    gsub => [
      "message", "(?:\d[ -]*?){13,16}",          "[CARD]",
      "message", "01[3-9]\d{8}",                  "[MSISDN]",
      "message", "\b\d{10,17}\b",                 "[ACCOUNT]"
    ]
  }
  if [user][nid] {
    ruby {
      code => "event.set('[user][nid]', event.get('[user][nid]').to_s.gsub(/(\d{3})\d+(\d{3})/,'\1****\2'))"
    }
  }
}
```

**Fluent Bit:**

```ini
[FILTER]
    Name    modify
    Match   *
    Condition Key_value_matches  card_number  ^\d{13,16}$
    Set     card_number          [REDACTED]
```

### Encrypted Shipping (TLS)

```yaml
# Filebeat → Logstash
output.logstash:
  hosts: ["ls:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  ssl.certificate: "/etc/filebeat/client.crt"
  ssl.key: "/etc/filebeat/client.key"
  ssl.verification_mode: full
```

ES-এ:

```yaml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.certificate: certs/http.crt
xpack.security.http.ssl.key: certs/http.key
```

### RBAC — Field & Document Level

```json
PUT _security/role/support_pii_redacted
{
  "indices": [{
    "names": ["logs-app-*"],
    "privileges": ["read"],
    "field_security": {
      "grant":  ["*"],
      "except": ["user.email","user.phone","card.*","auth.*"]
    },
    "query": {
      "term": { "service.namespace": "prod" }
    }
  }]
}
```

### Bangladesh Bank — অডিট লগ Retention

ফিনটেক (bKash, Nagad, ব্যাংক)-এর জন্য:

```
  Audit log retention:    7 years (BFIU/AML)
  Access log:             3 years
  Tamper-proof:           cryptographically signed or WORM
  Data residency:         কাস্টমার PII Bangladesh-এ থাকতে পারে (recommended)
```

ব্যবস্থা:

```
  ES Hot   (90d)  ── searchable
  ES Cold  (1y)   ── searchable_snapshot S3
  S3 WORM  (7y)   ── Object Lock + KMS encryption
                     daily SHA-256 manifest, signed
```

---

## 📊 Operations — ILM, Snapshot, DR

### Cluster Health Monitoring

```bash
# Cluster status
GET _cluster/health
# yellow = কোনো replica unassigned, red = primary missing, green = সব ভাল

# কেন unassigned?
GET _cluster/allocation/explain

# Shards প্রতি নোডে
GET _cat/shards?v&h=index,shard,prirep,state,node,unassigned.reason

# Disk watermark
GET _cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation.disk*
# low: 85%, high: 90%, flood_stage: 95% — read-only হয়ে যায়!
```

### JVM Heap Tuning

```
  -Xms24g -Xmx24g            # min == max, ≤ 50% RAM, ≤ 31g (compressed oops)
  -XX:+UseG1GC               # ES 7+ default
  -XX:G1HeapRegionSize=16m
  -XX:InitiatingHeapOccupancyPercent=30
```

### Snapshot to S3

```json
PUT _snapshot/s3-backup/daily-2024-01-15?wait_for_completion=false
{
  "indices": "logs-*,audit-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "metadata": { "taken_by": "cron", "reason": "daily" }
}
```

SLM (Snapshot Lifecycle Management):

```json
PUT _slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-{now/d}>",
  "repository": "s3-backup",
  "config": { "indices": ["logs-*"], "include_global_state": false },
  "retention": { "expire_after": "30d", "min_count": 7, "max_count": 60 }
}
```

### Common Failure Modes

| সমস্যা | কারণ | সমাধান |
|--------|------|---------|
| Cluster red | primary shard unassigned | `_cluster/allocation/explain`, replica/disk fix |
| Mapping conflict | একই field-এ different type | reindex বা DLQ → fix → replay |
| Mapping explosion | dynamic JSON random keys | `dynamic: strict` বা `flattened` type |
| Disk watermark hit | retention/ILM ছাড়া | ILM enable, watermark adjust, scale |
| Logstash slow | filter heavy / single worker | workers বাড়ান, batch size বাড়ান, persistent queue |
| Filebeat lag | output back-pressure | queue.disk বড়, multiple Logstash, Kafka buffer |
| ES OOM | heap > 31g বা mapping explosion | heap কমান, mapping audit |
| Kibana slow | বড় time range, no index pattern | data view limit, "rollups" use |

### Upgrade Paths

```
  ES 7.x → 8.x: rolling upgrade, kibana same version, breaking:
    type removal, security default ON, system index protection
  Logstash: backward compatible, plugins recheck
  OpenSearch fork: ES 7.10 ↔ OS 1.x compatible-ish; ES 8.x and OS 2.x diverged
```

### লাইসেন্স পরিবর্তন (গুরুত্বপূর্ণ!)

```
  ES ≤ 7.10  : Apache 2.0 (free open source)
  ES 7.11+   : Elastic License v2 / SSPL — **AWS managed banned**
                X-Pack basic features (security) এখন free
  AWS এর response : OpenSearch ফর্ক (ES 7.10 base, Apache 2.0)

  Bangladesh consideration:
    - DigitalOcean managed Elasticsearch ছিল না, এখনো সীমিত
    - AWS Mumbai-এ OpenSearch managed available
    - self-hosted ELK তে Elastic License-ও free (commercial use ঠিক), শুধু
      "as-a-service offering" বানানো নিষেধ
```

---

## 📊 বাস্তব সেটআপ — Daraz/Pathao/bKash/Startup

### Setup 1 — Daraz Scale (Multi-region, Black Friday)

```
                  ┌─────────────────────────────────────────┐
                  │  AWS Mumbai (primary)                    │
                  │                                          │
   K8s nodes ───► │  Filebeat DS ─► Kafka (3-broker, RF=3)  │
   ~600 pods      │                  │                       │
                  │                  ▼                       │
                  │              Logstash ×6 (consumer group)│
                  │                  │                       │
                  │   ┌──────────────┼─────────────────┐    │
                  │   ▼              ▼                 ▼    │
                  │ Hot ES ×10    Warm ES ×4      Cold/S3   │
                  │ (NVMe 4TB)    (SSD 8TB)      (snapshot) │
                  │   ▲                                      │
                  │   └── Kibana (SSO via Okta) ────────────┘
                  └─────────────────────────────────────────┘
                                  ▲
                                  │ CCS
                  ┌───────────────┴──────────────┐
                  │  AWS Singapore (DR + region) │
                  │  similar stack, replicated   │
                  └──────────────────────────────┘

   Volume: ~2 TB/day, 7d hot, 30d warm, 90d cold, ILM auto.
   Black Friday spike: ~5×, Kafka শোষণ করে, ES না।
```

### Setup 2 — Pathao K8s (ECS + Trace Correlation)

```
   ┌──────────────────────────────────────────────────┐
   │  EKS cluster (Mumbai) — ride/food/parcel svc     │
   │                                                   │
   │  Pod stdout ──► Fluent Bit DS ──► OTel Collector │
   │                                        │          │
   │                ECS-formatted JSON      │          │
   │                trace.id / span.id      ▼          │
   │                              ┌──────────────────┐ │
   │                              │ ES (logs-k8s-*)   │ │
   │                              │ Tempo (traces)    │ │
   │                              │ Prometheus(metric)│ │
   │                              └──────────────────┘ │
   │                                        ▲          │
   │                                        │          │
   │                Kibana Discover ◄───────┘          │
   │                  + "View trace in Jaeger" link     │
   └──────────────────────────────────────────────────┘

   ECS adoption: log.level, service.name, trace.id সব standardized।
   Kibana → trace.id ক্লিক ⇒ Jaeger UI খোলে (data-link)।
```

### Setup 3 — bKash Audit (Compliance, 7-year)

```
   App ──► structured audit log (signed JSON) ──► Filebeat
                                                     │
                            ┌────────────────────────┤
                            ▼                        ▼
                    ES (hot 90d)              S3 Object Lock
                    audit-* index             7-year retention
                    BFIU search               WORM, KMS
                            │                        │
                            ▼                        ▼
                    Kibana (RBAC)             Daily SHA-256 manifest
                    field-level               signed by HSM
                    redaction
                    audit team only
```

প্রতিটি event-এ `audit.signature` (HMAC-SHA256), যাতে tamper detect হয়।

### Setup 4 — Small BD Startup ($200/mo)

```
   3× DigitalOcean droplet (4 vCPU, 8 GB, 160 GB SSD) — Mumbai region
   Total: ~$144/mo + $50 backup space

   ┌────────────────────────────────────────┐
   │  ES cluster (3-node, master+data, RF=1)│
   │  retention: 14 days, 1 shard/index     │
   │  ILM: hot only → delete                 │
   │  ~30 GB/day ingest                      │
   └──────────────┬─────────────────────────┘
                  ▲
            Filebeat (each app server)
                  │
            stdout JSON from PHP/Node apps

   Kibana on same cluster, basic auth.
   Daily snapshot to DO Spaces ($5/mo).
```

৩-৪ জনের startup-এর জন্য যথেষ্ট। গ্রোথ হলে Kafka + Warm tier যোগ করুন।

---

## 📊 ELK vs Loki vs OpenSearch vs Datadog vs Splunk

### Pros / Cons Matrix

| | Pros | Cons |
|--|------|------|
| **ELK** | Powerful search, mature, ECS, APM | RAM-হাঙ্গরি ES, license complexity |
| **OpenSearch** | Apache 2.0, AWS managed, ES API compat | innovation lag, plugin gap |
| **Loki (PLG)** | Cheap (object store), label-based, Grafana native | full-text limited, কম mature |
| **Datadog** | Zero ops, integrated APM/RUM | very expensive, vendor lock-in, egress |
| **Splunk** | UEBA, SIEM, enterprise polish | $$$$, steep learning, on-prem complex |
| **Graylog** | সহজ UI, alert built-in | ES still needed, less ecosystem |
| **New Relic** | All-in-one, easy | similar Datadog issues |

### Decision Framework

```
  shall I self-host?
  ├─ team size < 5, no SRE → SaaS (DD/NR)
  ├─ budget tight, BD-based → self-host
  └─ compliance heavy (BFIU)→ self-host (data residency)

  full-text search critical?
  ├─ yes → ELK / OpenSearch
  └─ label-based enough → Loki

  Elastic license OK?
  ├─ yes → ELK
  └─ no (resell as SaaS)→ OpenSearch

  K8s-only logs?
  ├─ yes → Fluent Bit DS + ES (or Loki)
  └─ mixed VM/legacy → Filebeat / Logstash
```

---

## 📊 Anti-Patterns

```
❌ ১. সব verbatim index — মাসিক bill ফেটে যায়
   ✅ structured fields, drop low-value, sample success

❌ ২. লগ parse না করে message field-এ raw রাখা
   ✅ JSON at source, ECS fields

❌ ৩. কোনো ILM/retention policy না — disk full → cluster red
   ✅ ILM মাস্ট, watermark alert

❌ ৪. PII (card, NID, OTP) plain log-এ
   ✅ shipper-এ redact, rotate keys

❌ ৫. প্রোডাকশনে single-node ES
   ✅ 3+ master, replicas=1 minimum

❌ ৬. Logstash কে K8s sidecar — ১ GB heap × ১০০ pod = ১০০ GB!
   ✅ Fluent Bit edge, Logstash কেন্দ্রে

❌ ৭. ES সরাসরি ইন্টারনেটে expose
   ✅ TLS + auth + private VPC, Kibana-only public

❌ ৮. dynamic JSON-এ unbounded keys (e.g. metadata.{userId})
   ✅ flattened type বা schema enforce

❌ ৯. trace_id ছাড়া লগ — distributed debug অসম্ভব
   ✅ OTel context propagation, log-trace correlation

❌ ১০. সব service একই index-এ লেখে
   ✅ data stream per service/dataset, ILM per pattern
```

---

## ✅ Production Checklist

### Architecture & Deployment
- [ ] ES cluster ≥ 3 master-eligible nodes (split-brain prevention)
- [ ] Hot/Warm/Cold tier আলাদা node role
- [ ] Replica count ≥ 1 প্রোডাকশন index-এ
- [ ] Dedicated coordinating node বড় Kibana traffic-এ
- [ ] Multi-AZ / multi-region DR plan
- [ ] Kafka buffer যেখানে spike আছে (Black Friday)

### Index Design
- [ ] Data streams ব্যবহৃত (rollover automatic)
- [ ] Component + index template ECS-aligned
- [ ] `dynamic: strict` বা `flattened` mapping explosion রুখতে
- [ ] Primary shard ~30-50 GB target
- [ ] `index.codec: best_compression` লগ index-এ
- [ ] Index pattern per service/dataset

### Lifecycle & Retention
- [ ] ILM policy hot/warm/cold/delete configured
- [ ] Snapshot Lifecycle Management (SLM) daily, S3 backed
- [ ] Cold tier → searchable snapshot বা S3 archive
- [ ] Audit/financial logs 7-year WORM (bKash/bank)
- [ ] Disk watermark alerts (85/90/95%)

### Shippers & Pipelines
- [ ] Filebeat/Fluent Bit DaemonSet K8s-এ
- [ ] Persistent queue / disk buffer enable
- [ ] Multiline parser stack-trace-এর জন্য
- [ ] Sampling/drop নিকটেস্ট logs (healthz, metrics endpoints)
- [ ] Logstash dead-letter queue enable
- [ ] Pipeline workers / batch size tuned

### Application Side
- [ ] JSON structured logs (not text)
- [ ] ECS field naming
- [ ] trace.id / span.id correlation (OTel)
- [ ] log.level proper (no INFO spam)
- [ ] Stack trace as `error.stack_trace` field
- [ ] Sensitive fields redacted client-side too

### Security
- [ ] X-Pack security enabled, TLS everywhere
- [ ] Filebeat → Logstash → ES TLS mutual auth
- [ ] Kibana SSO (Okta/Google/Azure AD)
- [ ] Role-based field/document level security
- [ ] Audit log enable on ES itself
- [ ] PII redaction at shipper (card, NID, OTP, phone)
- [ ] No Elasticsearch on public IP

### Operations
- [ ] Cluster health monitoring (Prometheus exporter)
- [ ] JVM heap ≤ 50% RAM, ≤ 31 GB
- [ ] GC log enabled
- [ ] Slow log threshold set (search/indexing)
- [ ] Snapshot restore drill কোয়ার্টারলি
- [ ] Mapping conflict alerts → DLQ replay process documented
- [ ] License/version upgrade plan
- [ ] Capacity dashboard (GB/day trend)

### Alerting
- [ ] Error rate spike rule per critical service
- [ ] Specific exception alerts (DB deadlock, OOM)
- [ ] Heartbeat missing alert
- [ ] PII leak detector (regex on logs)
- [ ] Disk/heap/queue depth alert
- [ ] PagerDuty / Slack integration tested

### Cost & BD-Specific
- [ ] Sampling enabled noisy services
- [ ] Cold tier S3 (Mumbai region)
- [ ] Self-hosted vs SaaS TCO doc
- [ ] BFIU/Bangladesh Bank retention compliance signed off
- [ ] Disaster recovery RTO/RPO documented

---

## 🎯 সারসংক্ষেপ

ELK/EFK/EBK Stack হলো **production-grade observability-এর লগ স্তম্ভ**।

- **ELK** = ভারী transformation, mature, কিন্তু RAM-হাঙ্গরি Logstash।
- **EFK Fluent Bit** = K8s-এর জন্য ideal, lightweight।
- **EBK** = Beats সরাসরি ES-এ, simplest for small setups।
- **Loki/PLG** = সবচেয়ে সস্তা, কিন্তু label-based — full-text limited।
- **OpenSearch** = AWS-managed, license-free বিকল্প।

Bangladesh-এর কনটেক্সটে — Daraz scale-এ Kafka-buffered ELK, Pathao-তে ECS+OTel
correlation, bKash-এ dual-write S3 WORM, ছোট startup-এ DigitalOcean 3-node ES।

মূল মন্ত্র: **structured logs at source → ECS schema → ILM tiered storage →
RBAC + redaction → KQL search + alert**। এই পাঁচটা ঠিক থাকলে প্রোডাকশন ঘুমাতে দেবে।

> 🔗 পরবর্তী পঠন:
> [`metrics-alerting.md`](./metrics-alerting.md) — Prometheus/Grafana,
> [`distributed-tracing.md`](./distributed-tracing.md) — Jaeger/Tempo,
> [`opentelemetry.md`](./opentelemetry.md) — তিন পিলার একত্রে।
