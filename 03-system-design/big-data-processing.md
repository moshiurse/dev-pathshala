# 📊 Big Data Processing Strategies — সম্পূর্ণ গভীর বিশ্লেষণ

> **Advanced Guide** — Batch, Stream, Micro-batch, MapReduce, Hadoop, Spark, Kafka, Flink, Data Lake, Lakehouse, ETL/ELT, Partitioning, Monitoring ও Cost Optimization-এর গভীর আলোচনা।
>
> **বাংলাদেশ context**: bKash, Daraz, Pathao, GP, Prothom Alo, Chaldal, Foodpanda, Nagad-এর মতো high-scale platform কল্পনা করে বাস্তব উদাহরণ।
>
> **Tech angle**: PHP + Node.js/JavaScript code examples, extensive ASCII diagrams, decision matrix, architecture trade-off, production lessons.

---

## 🗺️ ডকুমেন্ট রোডম্যাপ

1. Big Data Processing — সংজ্ঞা ও ভূমিকা
2. Big Data Processing Paradigms
3. MapReduce — The Foundation
4. Apache Spark
5. Data Partitioning Strategies
6. Data Storage Strategies
7. ETL vs ELT
8. Real-time vs Batch — When to Use What
9. Distributed Computing Challenges
10. Big Data Pipeline Architecture
11. Scaling Strategies
12. Bangladesh Real-World Examples
13. ASCII Diagrams — এক নজরে পুরো ecosystem
14. PHP Code Examples
15. Node.js Code Examples
16. Monitoring Big Data Systems
17. Cost Optimization
18. When to Use — Decision Guide
19. Key Takeaways
20. Appendix — Glossary, Checklist, Design Questions

---

## 1. 📌 Big Data Processing — সংজ্ঞা ও ভূমিকা

### Big Data কী?

**Big Data** বলতে এমন ডেটাসেট, event stream, log stream, document collection, media stream, telemetry feed এবং transactional exhaust বোঝায় যেগুলো এত বড়, এত দ্রুত, এত বৈচিত্র্যময় এবং এত operationally complex যে traditional single-node system দিয়ে এগুলো efficiently store, process, query, monitor এবং derive-value করা কঠিন হয়ে যায়।

সহজ ভাষায়:

- ডেটা অনেক বেশি
- ডেটা খুব দ্রুত আসছে
- ডেটার ধরন অনেক রকম
- ডেটার quality সবসময় perfect নয়
- ডেটা থেকে value বের করতে distributed processing দরকার

### 3V → 5V Evolution

| V | অর্থ | কেন গুরুত্বপূর্ণ |
|---|---|---|
| **Volume** | বিপুল পরিমাণ ডেটা | TB, PB, এমনকি EB লেভেলের storage ও processing দরকার হতে পারে |
| **Velocity** | ডেটা আসার গতি | প্রতি সেকেন্ডে হাজার, লাখ, বা কোটি event ingest হতে পারে |
| **Variety** | ডেটার বৈচিত্র্য | relational rows, JSON, logs, image metadata, clickstream, GPS, CDR, IoT telemetry |
| **Veracity** | ডেটার নির্ভরযোগ্যতা | duplicate, missing, late, corrupt, skewed, inconsistent data থাকতে পারে |
| **Value** | ডেটা থেকে ব্যবসায়িক লাভ | শুধু storage না, actionable insight বের করাই আসল উদ্দেশ্য |

### 3V-এর ছোট উদাহরণ

```text
Volume   = ৩০ দিনের call detail record + recharge events + app logs
Velocity = প্রতি সেকেন্ডে নতুন call event, payment event, click event, GPS update
Variety  = SQL row + JSON + CSV + image metadata + text log + Kafka event
```

### বাংলাদেশ context-এ Big Data চিন্তা

#### GP / Telecom scale

ধরুন একটি telecom operator-এর কাছে আছে:

- 170M subscriber profile
- প্রতিদিন tower location update
- call detail record (CDR)
- data session usage
- recharge event
- offer subscription event
- app clickstream
- network quality telemetry

যদি average 20 event/subscriber/day ধরা হয়, তাহলে:

- `170M × 20 = 3.4B events/day`
- প্রতি event 500 bytes হলেও ≈ `1.7 TB/day raw event payload`
- replication, enrichment, checkpoint, derived dataset ধরলে বাস্তব storage footprint আরও অনেক বেশি

#### bKash / Fintech scale

একটি massive digital payment ecosystem-এ থাকতে পারে:

- send money event
- cash in / cash out event
- merchant payment event
- QR payment scan event
- fraud rule engine signal
- AML monitoring feed
- ledger write
- notification log
- app session event

যদি পুরো ecosystem-এ 500M+ transaction-related events/day ধরা হয়, তাহলে:

- low-latency fraud detection দরকার
- end-of-day reconciliation দরকার
- regulator-facing audit trail দরকার
- exactly-once-like financial correctness দরকার

#### Daraz / E-commerce scale

একটি বড় e-commerce platform-এ থাকে:

- millions of products
- inventory updates
- order placements
- payment attempts
- clickstream
- search queries
- recommendation impressions
- cart events
- delivery status events
- seller analytics logs

এখানে একই platform-এ **batch analytics**, **real-time dashboard**, **recommendation training**, **fraud screening**, **ad-hoc SQL exploration** — সব একসাথে দরকার হতে পারে।

### Traditional RDBMS কেন struggle করে?

খুব গুরুত্বপূর্ণ কথা: **RDBMS useless না**। কিন্তু traditional single-node row-store RDBMS কিছু পরিস্থিতিতে painful হয়ে যায়।

#### সীমাবদ্ধতা

- **Single node bottleneck**: storage, CPU, RAM, IOPS এক সার্ভারের সীমার মধ্যে আটকে যায়
- **Vertical scaling expensive**: bigger box মানে দ্রুত cost explosion
- **Write-heavy workload pain**: প্রতি সেকেন্ডে massive append/write এ bottleneck তৈরি হয়
- **Large scan expensive**: petabyte-scale scan row-store-এ খুব costly
- **Semi-structured data**: changing JSON schema, logs, nested data কঠিন হয়
- **Historical retention**: ২-৫ বছরের raw event রিলেশনাল টেবিলে রাখা operationally expensive
- **Streaming ingestion mismatch**: continuous event feed + real-time analytics + archival একই engine-এ সবসময় ভালো fit নাও হতে পারে
- **Distributed fault tolerance**: node failure, partition rebalancing, speculative execution, lakehouse semantics — এগুলো traditional DB-এর মূল design goal ছিল না

### OLTP বনাম Big Data Analytical Workload

| বৈশিষ্ট্য | OLTP Database | Big Data Analytical System |
|---|---|---|
| Typical query | একটি order lookup | গত ৯০ দিনের কোটি event scan করে trend বের করা |
| Row count | তুলনামূলক কম থেকে medium | massive |
| Write pattern | point update / transaction | append-heavy, event-heavy |
| Schema | relatively stable | evolving, semi-structured |
| Latency expectation | milliseconds | seconds to minutes (analytics), milliseconds (stream) |
| Scale strategy | primary + replica | distributed storage + distributed compute |

### Data at Rest vs Data in Motion

#### Data at Rest

এমন ডেটা যা ইতিমধ্যে সংরক্ষিত আছে:

- HDFS/S3-এ log file
- Parquet data lake table
- historical CSV archive
- daily order snapshot
- clickstream partitioned file

#### Data in Motion

এমন ডেটা যা এখনও চলমান:

- Kafka topic-এ flowing events
- payment authorization stream
- GPS location updates
- app telemetry
- fraud signal stream

```text
Data at Rest                               Data in Motion
====================                       ====================

[S3 / HDFS / Parquet]                      [Kafka / Pulsar / Kinesis]
        │                                              │
        │ historical scan                              │ live consume
        ▼                                              ▼
 [Batch job / SQL engine]                       [Stream processor / CEP]
        │                                              │
        ▼                                              ▼
[Daily report / ML training]                 [Fraud alert / dashboard / trigger]
```

### Big Data Processing-এর মূল উদ্দেশ্য

- অনেক বড় ডেটাসেট efficiently process করা
- distributed compute ব্যবহার করে parallelism পাওয়া
- data locality কাজে লাগানো
- raw event থেকে business insight তৈরি করা
- historical + real-time দুই ধরনের data flow handle করা
- storage cost কমিয়ে analysis power বাড়ানো
- data quality ও auditability ধরে রাখা

### এক লাইনের সারাংশ

> **Big Data Processing** হলো distributed storage + distributed compute + scalable ingestion + fault-tolerant orchestration ব্যবহার করে massive data থেকে reliable insight, automation, এবং business value বের করার প্রক্রিয়া।

---

## 🇧🇩 বাস্তব উদাহরণ — এক নজরে Bangladesh Context

| প্রতিষ্ঠান/ডোমেইন | ডেটার ধরন | কোন Processing দরকার | কেন |
|---|---|---|---|
| **GP / Robi / Banglalink** | CDR, tower ping, recharge, offer usage | Batch + Stream | network analytics + churn model + anomaly alert |
| **bKash / Nagad** | payment event, ledger event, fraud signal | Stream + Batch | fraud detection + reconciliation |
| **Daraz** | clickstream, search, order, inventory | Batch + Interactive + Stream | GMV, recommendation, live seller dashboard |
| **Pathao** | ride request, driver GPS, surge signal | Stream + Micro-batch | dispatch + ETA + surge pricing |
| **Prothom Alo** | article view, scroll depth, referrer | Stream + Batch | trending story + next day editorial analytics |
| **Chaldal** | warehouse movement, order fulfillment | Batch + Interactive | stock planning + fulfillment analytics |
| **Foodpanda** | order stream, restaurant events, rider events | Stream + Batch | order orchestration + marketplace performance |

### Real-world data explosion কেমন দেখায়?

```text
User Action Layer
=================

App Open
Search
Scroll
Click
Add To Cart
Pay
Track Order
Rate Ride
Read Article
Share Link

       ▼ each action becomes one or many events

Event Layer
===========

web_event
mobile_event
api_log
db_binlog
payment_event
fraud_signal
notification_event
location_ping
metrics_point

       ▼ ingest to distributed platform

Big Data Layer
==============

Kafka Topics → Data Lake → Batch Jobs → Warehouse → Dashboard / ML / Alerting
```

### Big Data সব জায়গায় লাগবে?

না।

যদি আপনার সিস্টেমে:

- দিনে ১০ হাজার order
- ১GB log/day
- ১টি PostgreSQL replica যথেষ্ট
- ৫ মিনিট রিপোর্ট delay acceptable

তাহলে full Hadoop/Spark ecosystem দরকার নাও হতে পারে।

Big Data stack তখনই meaningful যখন:

- scale খুব বড়
- velocity খুব বেশি
- data variety উচ্চ
- retention দীর্ঘ
- analytics/use case জটিল
- business value যথেষ্ট বড়

---

## 2. 🧠 Big Data Processing Paradigms

Big Data-এর জগতে সব workload একরকম নয়। একেক business question-এর জন্য একেক processing model বেশি উপযোগী।

### 2.1 Batch Processing (ব্যাচ প্রসেসিং)

**সংজ্ঞা**: accumulated data-কে নির্দিষ্ট interval-এ একসাথে process করা।

উদাহরণ:

- রাত ২টায় আগের দিনের সব order process করা
- মাস শেষে subscriber churn analysis করা
- daily sales report generate করা
- historical logs থেকে ETL চালিয়ে warehouse update করা

#### Batch Processing-এর মূল বৈশিষ্ট্য

- large volume friendly
- latency বেশি acceptable
- throughput খুব বেশি হতে পারে
- retry/reprocessing সহজ
- deterministic output তৈরি করা সহজ

#### Batch Flow

```text
[Raw Files of Whole Day]
          │
          ▼
   [Batch Scheduler]
          │
          ▼
   [Distributed Compute]
          │
          ├── Aggregate revenue
          ├── Build daily features
          ├── Join dimensions
          └── Produce warehouse tables
          ▼
     [Reports / ML / Finance]
```

#### Batch-এর classic technology

- Hadoop MapReduce
- Apache Spark batch jobs
- Hive
- Pig (historical)
- Airflow-triggered ETL jobs

#### বাংলাদেশ example

- **GP monthly subscriber analytics**
- **Daraz monthly GMV calculation**
- **Chaldal warehouse replenishment analysis**
- **Prothom Alo weekly content consumption report**

#### Batch use cases

- daily/weekly/monthly reports
- backfill job
- historical trend analysis
- ML feature generation
- reconciliation and settlement
- data lake compaction

### 2.2 Stream Processing (স্ট্রিম প্রসেসিং)

**সংজ্ঞা**: event আসার সাথে সাথে record-by-record বা very low-latency window-এ process করা।

উদাহরণ:

- payment fraud detect করা ২০০ms-এর মধ্যে
- rider demand spike দেখে surge update করা
- live dashboard প্রতি সেকেন্ডে refresh করা
- suspicious login detect করা

#### Stream-এর মূল বৈশিষ্ট্য

- low latency
- continuously running jobs
- event time গুরুত্বপূর্ণ
- late data handle করতে হয়
- stateful computation লাগতে পারে

#### সাধারণ concepts

- event-time vs processing-time
- windowing
- watermark
- checkpointing
- exactly-once / effectively-once semantics
- state store

#### Technology choices

- Apache Kafka Streams
- Apache Flink
- Spark Structured Streaming
- Apache Pulsar Functions
- Redpanda/Kafka ecosystem

#### বাংলাদেশ example

- **bKash real-time fraud detection**
- **Pathao surge pricing**
- **Foodpanda delivery ETA refresh**
- **live news trending detection for Prothom Alo**

#### Stream Flow

```text
[Live Events] → [Kafka Topic] → [Stream Processor] → [Alert / Cache / OLAP / ML Score]
```

### 2.3 Micro-batch Processing

**সংজ্ঞা**: events-কে খুব ছোট interval-এ (যেমন প্রতি 1s, 5s, 10s) group করে process করা।

#### এটি কেন দরকার?

Pure streaming implementation complex হতে পারে। আবার hourly batch অনেক slow। Micro-batch দুই জগতের মাঝে compromise দেয়।

#### বৈশিষ্ট্য

- latency seconds-level
- throughput pure record-by-record stream-এর চেয়ে বেশি হতে পারে
- operational complexity pure stream-এর চেয়ে কিছুটা কম হতে পারে
- Spark Streaming (classic DStream) এ micro-batch model জনপ্রিয় ছিল

#### use case

- near-real-time dashboard
- ad impression summary every 5 seconds
- order status rollup every 10 seconds
- app activity count update প্রতি 15 সেকেন্ডে

#### বাংলাদেশ example

- **Daraz seller dashboard প্রতি ৫ সেকেন্ডে refresh**
- **Pathao dispatch overview প্রতি ২ সেকেন্ডে refresh**
- **Chaldal warehouse operations monitoring**

### 2.4 Interactive / Ad-hoc Processing

**সংজ্ঞা**: analysts বা business team on-demand query চালায়, প্রায়শই huge dataset-এর উপর।

উদাহরণ প্রশ্ন:

- শেষ ৩০ দিনে সিলেট অঞ্চলে card payment failure rate কত?
- কোন seller category-তে return ratio বেশি?
- Prothom Alo-তে mobile users-এর average scroll depth কত?

#### বৈশিষ্ট্য

- SQL-first interface
- ad-hoc exploration
- fast scan over data lake বা warehouse
- BI tools integration

#### জনপ্রিয় tools

- Trino / Presto
- Apache Drill
- BigQuery
- Amazon Athena
- ClickHouse (serving/interactive analytics)
- Druid (sub-second OLAP use cases)

#### বাংলাদেশ example

- **Daraz business analyst**: promotion campaign uplift measure করছে
- **bKash risk team**: sudden device cluster দেখছে
- **Prothom Alo editorial analyst**: কোন article category most engaged তা বের করছে

### চারটি paradigm একসাথে কিভাবে coexist করে?

```text
                    Same Business, Multiple Processing Models
                    ========================================

Raw Events
   │
   ├──▶ Stream Processing     ──▶ Fraud alert / live KPI / surge update
   │
   ├──▶ Micro-batch           ──▶ 5-sec dashboard / rolling counters
   │
   ├──▶ Batch Processing      ──▶ Daily finance / monthly analytics / ML training
   │
   └──▶ Interactive SQL       ──▶ Analyst exploration / ad-hoc BI query
```

### Paradigm Comparison Table

| বিষয় | Batch | Stream | Micro-batch | Interactive |
|---|---|---|---|---|
| Latency | minutes-hours | milliseconds-seconds | seconds | seconds-minutes |
| Throughput | খুব বেশি | medium থেকে high | high | query-dependent |
| Complexity | তুলনামূলক কম | বেশি | medium | medium |
| Best for | reports, backfills | fraud, alerting | near-real-time | analyst exploration |
| Cost profile | cheaper | higher always-on cost | medium | scan/query cost |
| Determinism | high | state/late-data complexity | moderate | query-specific |

---

## 3. 🧱 MapReduce — The Foundation

### 3.1 Concept

MapReduce distributed data processing-এর earliest mainstream modelগুলোর একটি। এর মূল philosophy হলো **divide and conquer**।

- input data-কে chunk করা হয়
- প্রতিটি chunk parallel worker-এ map করা হয়
- intermediate key-value pair তৈরি হয়
- shuffle stage-এ same key-র data একসাথে আনা হয়
- reduce stage aggregate output তৈরি করে

### Word Count Example — ধারণার সবচেয়ে ক্লাসিক উদাহরণ

ধরুন Prothom Alo-এর headline dataset থেকে word frequency বের করতে হবে।

#### Input

```text
Line 1: "bangladesh wins match"
Line 2: "bangladesh market grows"
Line 3: "match analysis today"
```

#### Map Stage

প্রতিটি line independently process হবে।

```text
Mapper 1 input: "bangladesh wins match"
Mapper 1 output:
  (bangladesh, 1)
  (wins, 1)
  (match, 1)

Mapper 2 input: "bangladesh market grows"
Mapper 2 output:
  (bangladesh, 1)
  (market, 1)
  (grows, 1)

Mapper 3 input: "match analysis today"
Mapper 3 output:
  (match, 1)
  (analysis, 1)
  (today, 1)
```

#### Shuffle Stage

```text
(bangladesh, [1, 1])
(wins,       [1])
(match,      [1, 1])
(market,     [1])
(grows,      [1])
(analysis,   [1])
(today,      [1])
```

#### Reduce Stage

```text
Reducer(bangladesh, [1,1]) => (bangladesh, 2)
Reducer(match, [1,1])      => (match, 2)
Reducer(wins, [1])         => (wins, 1)
Reducer(market, [1])       => (market, 1)
Reducer(grows, [1])        => (grows, 1)
Reducer(analysis, [1])     => (analysis, 1)
Reducer(today, [1])        => (today, 1)
```

### 3.2 Detailed ASCII Diagram — Map → Shuffle → Reduce

```text
                         MAPREDUCE DATA FLOW
                         ===================

Input Files on HDFS
────────────────────────────────────────────────────────────
Block A          Block B             Block C
"bangladesh..."  "market..."        "today..."
     │                │                  │
     ▼                ▼                  ▼
 ┌────────┐      ┌────────┐         ┌────────┐
 │Mapper A│      │Mapper B│         │Mapper C│
 └────┬───┘      └────┬───┘         └────┬───┘
      │               │                   │
      │ emits         │ emits             │ emits
      ▼               ▼                   ▼
 (bangladesh,1)   (bangladesh,1)      (match,1)
 (wins,1)         (market,1)          (analysis,1)
 (match,1)        (grows,1)           (today,1)
      └───────────────┬──────────────────┘
                      ▼
             ┌─────────────────┐
             │ Shuffle / Sort  │
             │ Group by key    │
             └────────┬────────┘
                      │
      ┌───────────────┼──────────────────────┐
      ▼               ▼                      ▼
┌────────────┐  ┌────────────┐        ┌────────────┐
│Reducer #1  │  │Reducer #2  │        │Reducer #3  │
│bangladesh  │  │match       │        │others      │
│[1,1] → 2   │  │[1,1] → 2   │        │sum → count │
└──────┬─────┘  └──────┬─────┘        └──────┬─────┘
       │               │                     │
       └───────────────┴─────────────────────┘
                       ▼
                Final Output Files
```

### Parallel execution across nodes

```text
Cluster Nodes
=============

Node 1          Node 2          Node 3          Node 4
------          ------          ------          ------
Map Task 1      Map Task 2      Map Task 3      Map Task 4
Map Task 5      Map Task 6      Map Task 7      idle / retry

After map completes:

Node 1          Node 2          Node 3
------          ------          ------
Reduce A        Reduce B        Reduce C
```

### Hadoop Ecosystem-এ MapReduce কোথায় বসে?

```text
                 Hadoop Ecosystem (Classic View)
                 ===============================

           ┌─────────────────────────────────────┐
           │               YARN                  │
           │  Resource Management + Scheduling   │
           └─────────────────────────────────────┘
                     ▲                   ▲
                     │                   │
          schedules map/reduce     allocates containers
                     │                   │
           ┌─────────────────────────────────────┐
           │            MapReduce                │
           │     Distributed Compute Engine      │
           └─────────────────────────────────────┘
                           ▲
                           │ reads/writes
                           ▼
           ┌─────────────────────────────────────┐
           │                HDFS                 │
           │ Distributed Storage (blocks/files)  │
           └─────────────────────────────────────┘
```

### 3.3 Limitations of MapReduce

MapReduce revolutionary ছিল, কিন্তু painful-ও ছিল।

#### ১. Disk I/O between stages

- প্রতিটি stage-এর পরে data disk-এ লেখা হয়
- iterative job হলে বারবার read/write হয়
- SSD/HDD I/O bottleneck performance কমায়

#### ২. Programming model limited

- Map
- Shuffle
- Reduce

Complex DAG, in-memory iterative algorithm, interactive query — সবকিছু MapReduce model-এ elegant না।

#### ৩. Iterative algorithms painful

Machine learning, graph traversal, PageRank, repeated joins — বারবার data reread করতে হয়।

#### ৪. Latency high

Small interactive workload-এর জন্য MapReduce overkill।

#### ৫. Developer ergonomics weak

Lots of boilerplate, slower iteration, debugging কঠিন।

### কেন Spark এল?

> MapReduce দেখিয়েছিল distributed processing সম্ভব। Spark দেখাল distributed processing **দ্রুত**, **general-purpose**, এবং **developer-friendly** করা সম্ভব।

---

## 4. ⚡ Apache Spark

Apache Spark modern big data processing-এর সবচেয়ে প্রভাবশালী engineগুলোর একটি। এর মূল শক্তি হলো **in-memory computation**, **DAG execution**, **multiple APIs**, এবং **batch + SQL + streaming + ML** এক ecosystem-এ পাওয়া।

### 4.1 RDD (Resilient Distributed Dataset)

RDD হলো Spark-এর foundational abstraction।

RDD-এর বৈশিষ্ট্য:

- **Immutable** — একবার তৈরি হলে change হয় না
- **Partitioned** — data টুকরো টুকরো partition-এ ভাগ থাকে
- **Fault-tolerant** — lineage থেকে recompute করা যায়
- **Distributed** — cluster-wide partition ছড়িয়ে থাকে
- **Lazy** — transformation সঙ্গে সঙ্গে execute হয় না

#### RDD operations

- **Transformations**: map, flatMap, filter, union, reduceByKey
- **Actions**: collect, count, save, reduce, take

#### Lazy evaluation কী?

```text
val raw      = spark.read.text("s3://logs/")
val filtered = raw.filter(...)
val mapped   = filtered.map(...)
val grouped  = mapped.groupBy(...)

// এখনো actual execution হয়নি

grouped.count()   // action trigger করলেই execution plan run হবে
```

#### Spark lineage diagram

```text
Input File
   │
   ▼
RDD-1 (raw lines)
   │  filter
   ▼
RDD-2 (valid lines)
   │  map
   ▼
RDD-3 (key/value)
   │  reduceByKey
   ▼
RDD-4 (aggregated)
   │  save
   ▼
Output

If RDD-3 partition lost:
Spark can recompute from RDD-2 lineage.
```

#### কেন RDD fault-tolerant?

Spark সব data copy করে রাখে বলেই fault tolerance না; বরং transformation lineage track রাখে। কোনো partition হারালে শুধু lost partition recompute হয়।

### 4.2 Spark SQL & DataFrames

Spark-এর evolution-এ DataFrame API huge improvement এনেছে।

#### DataFrame কী?

- structured data abstraction
- column name + type আছে
- SQL-like operations possible
- optimizer-friendly

#### কেন DataFrame RDD-এর উপর preferable?

- declarative style
- optimizer কাজ করতে পারে
- column pruning possible
- predicate pushdown কাজে লাগে
- whole-stage codegen performance বাড়ায়

#### Catalyst Optimizer

Catalyst logical plan optimize করে:

- filter pushdown
- projection pruning
- join reordering
- constant folding
- expression simplification

#### Example flow

```text
SQL / DataFrame API
        │
        ▼
   Logical Plan
        │
        ▼
 Optimized Logical Plan
        │
        ▼
  Physical Plan Candidates
        │
        ▼
   Selected Physical Plan
        │
        ▼
      Execution
```

#### Spark SQL example চিন্তা

- Daraz order table
- customer dimension
- product category dimension
- daily GMV query

Spark SQL optimizer join/filter/predicate pushdown apply করে huge scan cost কমাতে পারে।

### 4.3 Spark Streaming

Spark streaming-এর দুই যুগ:

#### DStream (classic)

- micro-batch model
- RDD-based streaming abstraction
- প্রতি batch interval-এ data process হয়

#### Structured Streaming (modern)

- DataFrame/Dataset API ভিত্তিক
- event-time windowing better
- watermark support
- Kafka integration smooth
- batch ও stream mental model unify করে

#### DStream vs Structured Streaming

| বিষয় | DStream | Structured Streaming |
|---|---|---|
| Model | micro-batch on RDD | streaming table abstraction |
| API | older | modern DataFrame API |
| Optimizer | limited | Spark SQL optimizer সুবিধা |
| Window support | আছে | better and richer |
| Future-proof | legacy | preferred |

#### Kafka Integration ধারণা

```text
Kafka Topic → Spark Structured Streaming → Aggregation → Sink
```

#### Example use cases

- bKash transaction stream → suspicious pattern detection
- Prothom Alo article view stream → top stories rolling window
- Pathao ride events → per-area demand counter

### Spark কেন জনপ্রিয়?

- batch + SQL + stream + ML এক ecosystem-এ
- developer productivity ভালো
- large community
- cloud compatibility ভালো
- data lake architecture-এর সাথে strong fit

### Spark limitations-ও আছে

- cluster tuning complex
- memory pressure হতে পারে
- very low latency (<100ms) pure streaming use case-এ Flink/Kafka Streams better হতে পারে
- small files problem performance খারাপ করতে পারে

---

## 5. 🧩 Data Partitioning Strategies

Distributed system-এ partitioning হলো survival mechanism। data partition ঠিক না হলে scale, cost, latency — সব নষ্ট হতে পারে।

### Partitioning কেন দরকার?

- parallel processing enable করতে
- data distribute করতে
- storage pressure ভাগ করতে
- local aggregation সম্ভব করতে
- join cost কমাতে
- scalability বাড়াতে

### 5.1 Hash Partitioning

Formula:

```text
partition = hash(key) % numPartitions
```

#### ASCII diagram

```text
Keys:   U1   U2   U3   U4   U5   U6   U7
Hash:   12   91   44   03   67   18   34
Mod 4:   0    3    0    3    3    2    2

P0: U1, U3
P1: -
P2: U6, U7
P3: U2, U4, U5
```

#### সুবিধা

- implement করা সহজ
- average uniform distribution দেয়
- scalable
- key-based aggregation-এ useful

#### সমস্যা

- perfect uniformity guaranteed না
- hot key থাকলে skew হবে
- partition count change করলে reshuffle লাগে

#### বাংলাদেশ example

- bKash transaction by account_id
- Daraz clickstream by user_id
- Prothom Alo page view by article_id

### 5.2 Range Partitioning

Ordered range অনুযায়ী partition assign করা হয়।

#### ASCII diagram

```text
Order Amount Range Partitioning
===============================

P0:    0   -   999
P1: 1000   -  4999
P2: 5000   - 19999
P3: 20000+
```

#### ভালো কোথায়?

- range query efficient
- sort-heavy workload friendly
- time-series data-তে useful

#### সমস্যা

যদি ৮০% data `0-999` range-এ পড়ে, তাহলে P0 hot হয়ে যাবে।

#### বাংলাদেশ example

- order price range analysis
- telecom recharge amount buckets
- daily partitions by event date

### 5.3 Key-based Partitioning

All records with same key must go to same partition.

#### কেন দরকার?

- local joins
- local aggregation
- same-entity ordering
- stateful stream processing

#### উদাহরণ

```text
All events of rider_123 → same partition
All transactions of wallet_456 → same partition
All article views of article_789 → same partition
```

#### Celebrity / Hot Key Problem

```text
Normal Keys                     Hot Key
===========                     =======
A → P1                          celebrity_seller → P9
B → P2                          celebrity_seller → P9
C → P3                          celebrity_seller → P9
D → P4                          celebrity_seller → P9
E → P1                          celebrity_seller → P9

Result: P9 overloaded
```

#### বাংলাদেশ example

- Daraz 11.11 sale-এ popular seller বা flash-sale SKU hot key হতে পারে
- Prothom Alo breaking news article viral হলে সব traffic এক article_id-তে জমা হতে পারে

### 5.4 Custom Partitioning

Business logic অনুযায়ী partition design করা হয়।

#### Bangladesh district-based example

```text
Dhaka + Gazipur + Narayanganj         → Urban-Central partition set
Chattogram + Cox's Bazar              → South-East partition set
Rajshahi + Rangpur                    → North-West partition set
Khulna + Barishal                     → South-West partition set
Sylhet + Moulvibazar + Habiganj       → North-East partition set
```

#### use cases

- geo-based workload
- compliance boundary
- data sovereignty-like rule
- district/region-based operational analytics

#### সুবিধা

- business-friendly
- locality improve হতে পারে
- reporting alignment ভালো হয়

#### ঝুঁকি

- future growth ভুল predict হলে imbalance হবে
- hand-crafted partition design maintain করা কঠিন

### Partition Skew Mitigation Techniques

#### ১. Salting

```text
original key: seller_999
salted keys:
  seller_999#0
  seller_999#1
  seller_999#2
  seller_999#3
```

#### ২. Two-stage aggregation

- first local partial aggregation
- then global aggregation

#### ৩. Dedicated hot partitions

- top 1% hot keys আলাদা treatment পায়
- flash-sale seller আলাদা queue পেতে পারে
- viral article গণনার জন্য dedicated shard থাকতে পারে

#### ৪. Adaptive Query Execution

- Spark skewed join partition split করতে পারে
- dynamic plan optimization skew impact কমায়

#### ৫. Monitoring-based repartition

- hot partition detect করে layout change করা হয়
- partition health metrics operationally জরুরি

### Rule of thumb

> partitioning design সবসময় query pattern + write pattern + growth pattern + hot-key profile বুঝে করতে হবে।

---

## 6. 🗄️ Data Storage Strategies

Big data শুধু processing engine নয়; storage format ও layout equally গুরুত্বপূর্ণ।

### 6.1 HDFS (Hadoop Distributed File System)

HDFS distributed file system যেখানে huge file-কে block-এ ভেঙে multiple data node-এ রাখা হয়।

#### মূল ধারণা

- files are split into blocks (commonly 128MB বা 256MB)
- blocks multiple data node-এ replicate হয়
- NameNode metadata ধরে
- DataNode actual block store করে

#### ASCII diagram

```text
                   HDFS Architecture
                   =================

                  ┌────────────────┐
                  │    NameNode    │
                  │ file metadata  │
                  │ block mapping  │
                  └───────┬────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │DataNode1 │      │DataNode2 │      │DataNode3 │
   │B1, B3    │      │B1, B2    │      │B2, B3    │
   └──────────┘      └──────────┘      └──────────┘

File X => Block1, Block2, Block3
Replication factor = 3
```

#### HDFS-এর শক্তি

- huge sequential reads/writes
- cheap commodity hardware
- data locality-friendly
- append/analytics workload ভালো fit

#### সীমাবদ্ধতা

- small files সমস্যা
- low-latency point lookup-এর জন্য ideal না
- NameNode metadata pressure

### 6.2 Columnar Storage

Modern analytics-এ Parquet/ORC enormous advantage দেয়।

#### Row store vs Column store

```text
Row Store:
[order_id, customer_id, amount, city]
[1, 100, 500, Dhaka]
[2, 101, 700, Chattogram]
[3, 102, 300, Dhaka]

Column Store:
order_id:    [1,2,3]
customer_id: [100,101,102]
amount:      [500,700,300]
city:        [Dhaka,Chattogram,Dhaka]
```

#### কেন analytics-এ columnar format powerful?

- query যদি শুধু `amount` ও `city` পড়ে, অন্য column read করতে হয় না
- same-type values কাছাকাছি থাকায় compression ভালো হয়
- scan efficiency বাড়ে

#### Key features

- **Column pruning**: দরকারি column-ই পড়া হয়
- **Predicate pushdown**: file/row-group level filtering আগে হয়
- **Compression**: dictionary, run-length, bit packing, etc.

#### Bangladesh example

Daraz order analytics query যদি শুধু চায়:

- order_date
- category_id
- amount
- district

তাহলে Parquet পুরো raw JSON read না করে targeted scan করতে পারে।

### 6.3 Data Lake

Data lake হলো raw, semi-structured, structured সব ধরনের data-র বড় storage zone।

#### সাধারণ storage

- Amazon S3
- Azure Data Lake Storage (ADLS)
- Google Cloud Storage (GCS)
- on-prem object store / HDFS

#### Schema-on-read বনাম Schema-on-write

| ধারণা | ব্যাখ্যা |
|---|---|
| **Schema-on-write** | load করার আগেই strict schema enforce করা হয় |
| **Schema-on-read** | raw data আগে store করা হয়, query/read-এর সময় schema apply হয় |

#### Data lake-এর সুবিধা

- raw data retain করা যায়
- future use case-এর জন্য data preserve থাকে
- cheap storage
- multiple engines একই data পড়তে পারে

#### সমস্যা

যদি governance না থাকে, data lake দ্রুত **data swamp** হয়ে যায়।

সমস্যাগুলো:

- unclear ownership
- inconsistent schema
- duplicate files
- no quality checks
- poor discoverability

### 6.4 Data Lakehouse

Lakehouse data lake-এর cheap storage + warehouse-like reliability combine করে।

#### জনপ্রিয় technologies

- Delta Lake
- Apache Iceberg
- Apache Hudi

#### Lakehouse কী দেয়?

- ACID transaction
- schema evolution
- time travel
- merge/upsert/delete support
- concurrent read/write safety
- better table metadata

#### কেন এটা game changer?

কারণ pure data lake-এ huge file collections manage করা operationally painful। Lakehouse table format metadata layer দিয়ে data lake-কে manageable করে।

#### Medallion architecture

```text
Bronze  → raw ingest
Silver  → cleaned / validated / standardized
Gold    → business-ready aggregated tables
```

#### ASCII diagram

```text
                Lakehouse Medallion Flow
                ========================

Kafka / CDC / Logs / Files
            │
            ▼
     ┌────────────┐
     │ Bronze     │  raw append-only
     └─────┬──────┘
           │ clean / dedup / parse
           ▼
     ┌────────────┐
     │ Silver     │  curated canonical tables
     └─────┬──────┘
           │ business metrics / marts
           ▼
     ┌────────────┐
     │ Gold       │  finance, dashboard, ML features
     └────────────┘
```

### Storage design best practices

- small files avoid করুন
- partition by query pattern, not by habit
- Parquet/ORC prefer করুন analytics-এ
- retention + lifecycle policy define করুন
- metadata catalog maintain করুন
- quality checks ছাড়া raw data gold table-এ তুলবেন না

---

## 7. 🔄 ETL vs ELT

### 7.1 ETL (Extract-Transform-Load)

Traditional approach:

1. source থেকে data extract
2. staging-এ transform
3. final warehouse-এ load

#### কোথায় useful?

- strong governance দরকার
- target schema fixed
- expensive warehouse-এ raw junk তুলতে চাই না
- heavy transformation আগে করতে চাই

#### Tools

- Apache NiFi
- Informatica
- Talend
- SSIS (legacy world)
- custom Spark job

#### ASCII flow

```text
Sources → Extract → Transform Engine → Curated Output → Warehouse
```

#### বাংলাদেশ example

- GP monthly revenue reporting-এর জন্য finance-approved transformation pipeline
- regulator-facing report যেখানে deterministic cleaned data দরকার

### 7.2 ELT (Extract-Load-Transform)

Modern cloud-native approach:

1. source data raw/near-raw অবস্থায় warehouse/lakehouse-এ load
2. warehouse/lakehouse-এর compute ব্যবহার করে transform

#### কেন modern world-এ ELT জনপ্রিয়?

- storage cheap
- compute elastic
- raw data preserved থাকে
- business rule change হলে transform পুনরায় চালানো সহজ
- analytics team দ্রুত iterate করতে পারে

#### Tools

- dbt
- Fivetran + BigQuery / Snowflake
- Airbyte
- custom lakehouse SQL jobs

#### ASCII flow

```text
Sources → Load Raw → Warehouse/Lakehouse → SQL/dbt Transform → Mart / BI / ML
```

### ETL vs ELT Comparison

| বিষয় | ETL | ELT |
|---|---|---|
| Transform timing | load-এর আগে | load-এর পরে |
| Raw preservation | কম | বেশি |
| Flexibility | কম | বেশি |
| Governance | strong upfront | downstream controlled |
| Cloud friendliness | medium | high |
| Historical replay | কঠিন হতে পারে | সহজ |

### কোনটা বেছে নেবেন?

- **Strict compliance + fixed schema** → ETL
- **Exploratory analytics + lakehouse/warehouse** → ELT
- **Most modern stack** → raw ingestion + ELT + curated marts

### Practical pattern

আজকের অনেক mature company hybrid ব্যবহার করে:

- raw data lake-এ load
- critical compliance pipeline-এ ETL-style strict cleansing
- analyst use case-এ ELT/dbt model

---

## 8. ⏱️ Real-time vs Batch — When to Use What

নিচের table production decision নেওয়ার সময় খুব useful:

| Requirement | Batch | Stream | Micro-batch |
|-------------|-------|--------|-------------|
| Latency | Hours | Milliseconds | Seconds |
| Throughput | Very High | Medium | High |
| Complexity | Lower | Higher | Medium |
| Cost | Lower | Higher | Medium |
| Data Completeness | Complete | Approximate / event-driven | Near-complete |

### Table-এর বাস্তব ব্যাখ্যা

#### Latency

- batch: end-of-day finance report okay
- stream: fraud alert ২০০ms delay acceptable না
- micro-batch: dashboard ৫ সেকেন্ড delay acceptable

#### Throughput

- batch jobs huge historical data efficiently scan করতে পারে
- stream systems always-on, stateful; raw throughput designভেদে কম/বেশি হতে পারে
- micro-batch throughput ভালো, latency stream-এর তুলনায় একটু বেশি

#### Completeness

- batch often complete picture দেখে
- stream late event, out-of-order event, watermark impact handle করে
- micro-batch near-complete, but exact snapshot কিছুটা delay হতে পারে

### Examples

| Use case | Best fit | কারণ |
|---|---|---|
| bKash fraud detection | Stream | instant decision দরকার |
| bKash end-of-day reconciliation | Batch | full ledger comparison দরকার |
| Daraz seller dashboard every 10s | Micro-batch | near-real-time enough |
| Prothom Alo monthly readership report | Batch | historical complete data দরকার |
| Pathao surge pricing | Stream | live supply-demand imbalance |
| Analyst ad-hoc query | Interactive | on-demand SQL দরকার |

---

## 9. 🌐 Distributed Computing Challenges

Distributed system সুন্দর শোনালেও production-এ reality কঠিন।

### 9.1 Network Partitions এবং CAP relevance

Network partition মানে cluster-এর এক অংশ আরেক অংশের সাথে reliable communication হারায়।

#### CAP relevance simplified

partition হলে system-কে trade-off নিতে হয়:

- **Consistency**: সবাই একই truth দেখবে
- **Availability**: request reject না করে কিছু response দেবে

Analytics pipeline-এ অনেক সময় temporary inconsistency tolerate করা যায়।
Financial ledger-এ সেটা করা যায় না।

#### Bangladesh example

- **wallet balance debit**: consistency must win
- **live dashboard**: availability often wins

### 9.2 Data Skew and Hot Partitions

সব partition equal workload পায় না। কিছু key খুব hot হতে পারে।

#### উদাহরণ

- viral article
- celebrity seller flash sale
- one district under sudden ride demand
- one merchant under campaign spike

#### ফলাফল

- one executor overloaded
- shuffle spill
- longer task runtime
- SLA miss

#### mitigation

- salting
- skew-aware join
- pre-aggregation
- heavy hitter isolation
- AQE

### 9.3 Stragglers (Slow Nodes)

কিছু task অন্যগুলোর তুলনায় অনেক slow হয়। কারণ হতে পারে:

- noisy neighbor
- bad disk
- skewed partition
- GC pause
- network issue

#### কেন সমস্যা?

Spark/Hadoop job-এর total completion often slowest task-এর উপর নির্ভর করে।

#### mitigation

- speculative execution
- skew mitigation
- better partition sizing
- executor tuning

### 9.4 Exactly-once Semantics

Big data world-এ exact একবার process করা বলা সহজ, করা কঠিন।

Reality:

- producer retry করতে পারে
- consumer retry করতে পারে
- job crash হতে পারে
- sink partial write করতে পারে

#### practical approach

- idempotent writes
- transactional sink where possible
- deduplication keys
- checkpoint + offset management
- deterministic merge logic

#### finance use case

bKash/Nagad-like system-এ duplicate event handling critical। যদি একই payment event দুবার aggregate হয়, report ভুল হবে; যদি ledger side effect দুবার হয়, disaster।

### 9.5 Checkpointing and Fault Recovery

Stream processor stateful হলে periodic checkpoint দরকার।

```text
Kafka Offset 1000 ----> checkpoint saved
Kafka Offset 1100 ----> crash
restart from last checkpoint
reprocess 1001..1100 (or continue depending on engine semantics)
```

#### কেন দরকার?

- state restore
- window continuation
- offset coordination
- failure recovery

### 9.6 Late-arriving Data

real world event সবসময় time-order-এ আসে না।

উদাহরণ:

- mobile network delay
- offline app sync
- upstream retry
- cross-region replication lag

এজন্য stream system-এ event-time + watermark জরুরি।

### 9.7 Schema Evolution

নতুন field যোগ হলো, পুরনো producer এখনও old schema পাঠাচ্ছে। কী হবে?

#### practices

- backward compatibility
- schema registry
- optional fields
- versioned event contract

### 9.8 Small Files Problem

প্রতি মিনিটে হাজার file লিখলে data lake query performance খারাপ হয়।

#### impact

- metadata overhead
- open/close cost
- scan inefficiency

#### mitigation

- compaction
- larger target file size
- micro-batch buffering

### Production reality summary

> Big data challenge-এর অর্ধেক algorithm, অর্ধেক operations।

---

## 10. 🏗️ Big Data Pipeline Architecture

A robust big data platform সাধারণত layered architecture follow করে।

### 10.1 Collection Layer

এখানে data collect হয় বিভিন্ন source থেকে।

#### Sources

- app logs
- API events
- database CDC
- IoT telemetry
- clickstream
- payment events
- GPS pings
- server metrics

#### Tools

- Kafka
- Amazon Kinesis
- Fluentd
- Filebeat
- Debezium (CDC)
- Logstash

#### Bangladesh example

ধরুন bKash-এর সব transaction server, notification server, fraud rule engine, mobile app gateway, merchant QR service থেকে event collect করতে হবে। Collection layer এই সব source থেকে standardized event stream বানায়।

### 10.2 Storage Layer

Collection-এর পরে data কোথাও land করতে হবে।

#### Common destinations

- HDFS
- S3 data lake
- lakehouse tables
- raw archive storage
- OLTP side stores for operational snapshots

#### Hot / Warm / Cold / Archive tier

```text
Hot     = recent, frequently queried, low latency storage
Warm    = recent but less critical analytics zone
Cold    = historical raw data, cheap storage
Archive = long-term retention, rarely accessed
```

ASCII view:

```text
                Storage Lifecycle Tiers
                =======================

  Hot Tier      Warm Tier        Cold Tier         Archive Tier
  --------      ---------        ---------         ------------
  last 3 days   last 30 days     last 12 months    3+ years
  ClickHouse    Parquet on S3    compressed files  glacier/tape-like
  fast query    regular query    backfill query    audit/legal only
```

### 10.3 Processing Layer

এখানে compute engine raw data থেকে insight, feature, aggregate, model, alert তৈরি করে।

#### Engines

- Spark
- Flink
- Trino/Presto
- dbt on warehouse
- custom Python/Java/Scala jobs

#### Jobs

- batch ETL
- stream enrichment
- dimension join
- sessionization
- deduplication
- model feature generation

### 10.4 Serving Layer

processed output user-facing বা analyst-facing system-এ serve হয়।

#### জনপ্রিয় serving systems

- Druid
- ClickHouse
- Elasticsearch
- Redis (hot counters/cache)
- API database / warehouse mart

#### use cases

- live dashboard
- operational KPI
- search analytics
- fraud analyst console
- finance report UI

### 10.5 Orchestration Layer

Pipeline orchestration ছাড়া big data platform দ্রুত chaos হয়ে যায়।

#### Tools

- Apache Airflow
- Dagster
- Prefect
- Argo Workflows

#### Responsibilities

- DAG scheduling
- dependency management
- retries
- alerting
- backfill control
- SLA monitoring

### End-to-end Architecture Diagram

```text
                                 BIG DATA PIPELINE
                                 =================

   Mobile App   Web App   Payment API   DB CDC   Server Logs   GPS Devices
       │           │          │           │          │             │
       └────┬──────┴──────┬───┴──────┬────┴──────┬──┴───────┬─────┘
            ▼             ▼          ▼           ▼          ▼
      ┌────────────────────────────────────────────────────────────┐
      │             Collection / Ingestion Layer                  │
      │ Kafka / Kinesis / Fluentd / Debezium / Filebeat          │
      └─────────────────────────────┬──────────────────────────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 ▼                  ▼                  ▼
        ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐
        │ Raw Data Lake│   │ Stream Engine│   │ Operational Cache│
        │ S3/HDFS      │   │ Flink/Spark  │   │ Redis / KV       │
        └──────┬───────┘   └──────┬───────┘   └────────┬────────┘
               │                  │                    │
               │                  ├── Fraud alert      ├── Live counters
               │                  ├── Session stats    └── Realtime API
               │                  └── Anomaly signal
               ▼
        ┌──────────────┐
        │ Batch Engine │  Spark / SQL / dbt / Trino
        └──────┬───────┘
               │
       ┌───────┼──────────────────────┐
       ▼       ▼                      ▼
 ┌─────────┐ ┌──────────────┐   ┌──────────────┐
 │Silver   │ │Gold Metrics  │   │ML Features   │
 │Curated  │ │Finance / BI  │   │Training Sets │
 └────┬────┘ └──────┬───────┘   └──────┬───────┘
      │             │                  │
      ▼             ▼                  ▼
┌────────────┐ ┌────────────┐    ┌────────────┐
│Warehouse / │ │Dashboard / │    │Model Train │
│Ad-hoc SQL  │ │Alerts / API│    │& Serve     │
└────────────┘ └────────────┘    └────────────┘
```

### Medallion + Serving hybrid

অনেক modern architecture-এ pipeline এরকম হয়:

- Bronze: raw ingestion
- Silver: canonical cleaned event
- Gold: marts and aggregates
- Serving: ClickHouse / Druid / Redis / API store
- ML: feature store + model serving

### Pipeline design principles

- idempotent jobs
- schema contract
- replay capability
- lineage tracking
- data quality gates
- cost-aware storage tiers
- observability from ingest to serving

---

## 11. 📈 Scaling Strategies

Big data system scale করার সময় শুধু node বাড়ালেই হয় না; intelligent scaling দরকার।

### 11.1 Horizontal Scaling

আরও node যোগ করে storage ও compute বাড়ানো।

```text
Before:
[Node1] [Node2] [Node3]

After:
[Node1] [Node2] [Node3] [Node4] [Node5] [Node6]
```

#### সুবিধা

- commodity hardware ব্যবহার করা যায়
- incremental scale possible
- fault isolation better

#### challenge

- data rebalance
- coordination overhead
- network shuffle

### 11.2 Data Locality

compute-কে data-এর কাছে নেওয়া ideally cheaper than data-কে compute-এর কাছে টানা।

#### কেন গুরুত্বপূর্ণ?

network transfer expensive এবং slow। Hadoop world-এর earliest insight-গুলোর একটি ছিল **move compute to data**।

### 11.3 Speculative Execution

slow task detect হলে একই task আরেক node-এ চালানো হয়। যে আগে শেষ করবে, তার result নেওয়া হবে।

```text
Task 81 on Node-7  → slow...
Task 81 on Node-19 → re-run
First successful completion wins
```

### 11.4 Adaptive Query Execution (AQE)

runtime statistics দেখে plan adjust করা। Spark-এ useful:

- shuffle partition coalescing
- skew join handling
- broadcast join switch

### 11.5 Partition Count Tuning

খুব কম partition হলে parallelism কম। খুব বেশি হলে scheduling overhead বাড়ে।

Rule of thumb:

- target file size / partition size balance করুন
- shuffle partition metrics দেখুন
- skew ও executor cores বুঝে tune করুন

### 11.6 Autoscaling by Queue Depth / Lag

streaming system-এ:

- Kafka lag বাড়ছে?
- consumer throughput কম?
- executor scale out দরকার?

batch system-এ:

- nightly backlog growing?
- cluster auto scale করা যায়?

### 11.7 Caching and Materialization

সব query raw huge dataset scan করে চালাবেন না।

- frequently used aggregates materialize করুন
- hot dimensions cache করুন
- serving store use করুন

### Scale mantra

> Distributed system scale করার মানে শুধু “more machines” না; বরং “better layout + better partitioning + better execution planning + better observability”।

---

## 12. 🇧🇩 Bangladesh Real-World Examples (Detailed)

এই section-এর উদ্দেশ্য হলো theoretical concept-গুলোকে local business lens-এ দেখা। এখানে numbers illustrative; goal হলো scale intuition তৈরি করা।

### 12.1 GP Telecom — CDR Processing at National Scale

**Problem Statement:** Call Detail Record, recharge, tower handoff, data session, offer subscription, complaint ticket এবং network KPI একসাথে process করতে হয়।

**Primary Data Sources:**

- CDR feed
- SMS/voice metadata
- tower location event
- recharge channel event
- self-care app clickstream
- network telemetry

**Batch Workloads:**

- monthly subscriber usage segmentation
- churn feature generation
- district-wise ARPU report
- network quality score by tower cluster

**Stream / Real-time Workloads:**

- fraud SIM box anomaly
- tower outage alert
- sudden call drop spike detection
- campaign response counter

**Architecture Sketch:** `Subscriber → Telco Event Bus → Raw Lake → Batch Analytics / Real-time Alert`

**Likely Platform Building Blocks:**

- Kafka for event ingestion
- S3/HDFS lake for archival
- Spark for monthly analytics
- Flink/stream engine for anomaly detection
- Trino for analyst SQL

**Operational Risks:**

- data volume explosion
- sensitive data governance
- regional skew during campaign
- late arriving CDR files

**Important Design Lesson:**

- telecom analytics-এ batch এবং stream একসাথে দরকার; শুধু nightly batch যথেষ্ট না
- privacy, anonymization, retention policy design early phase-এই করা উচিত

### 12.2 bKash — Reconciliation + Real-time Fraud

**Problem Statement:** একই ecosystem-এ financial correctness, real-time fraud screening, merchant settlement, notification delivery এবং audit trail maintain করতে হয়।

**Primary Data Sources:**

- payment authorization event
- ledger append event
- merchant QR scan
- device fingerprint signal
- AML rule output
- notification status

**Batch Workloads:**

- end-of-day reconciliation
- merchant settlement
- regulator report
- historical fraud model training

**Stream / Real-time Workloads:**

- real-time fraud scoring
- velocity check by device/account
- immediate alert to risk engine
- live transaction dashboard

**Architecture Sketch:** `Payment API → Kafka → Stream Fraud Engine → Decision + Ledger; parallel Batch Reconciliation`

**Likely Platform Building Blocks:**

- Kafka/Pulsar style event bus
- stream processing for rule engine
- lakehouse for immutable audit data
- batch Spark jobs for reconciliation
- ClickHouse/Druid for risk dashboards

**Operational Risks:**

- duplicate event handling
- exactly-once challenge
- regulatory retention
- hot merchant during campaign

**Important Design Lesson:**

- financial domain-এ raw event + canonical ledger + reconciliation pipeline আলাদা চিন্তা করতে হয়
- realtime fraud pipeline এবং end-of-day finance batch pipeline একই data থেকে value তৈরি করলেও correctness requirement আলাদা

### 12.3 Daraz — Marketplace Analytics and Recommendation

**Problem Statement:** millions of products, clicks, search queries, orders, seller updates এবং campaign spikes handle করতে হয়।

**Primary Data Sources:**

- product catalog changes
- clickstream
- search query logs
- cart events
- order and payment events
- delivery tracking

**Batch Workloads:**

- daily GMV report
- recommendation model training
- seller performance ranking
- inventory forecast

**Stream / Real-time Workloads:**

- live flash-sale counter
- real-time recommendation feature updates
- fraudulent seller activity signal
- operational dashboard

**Architecture Sketch:** `User Click → Kafka → Feature Stream / Lake → Batch Model Train → Realtime Serving`

**Likely Platform Building Blocks:**

- Kafka for click and order events
- lakehouse with Parquet/Iceberg
- Spark batch for training and reporting
- feature cache for online serving
- Trino for business SQL

**Operational Risks:**

- celebrity product skew
- campaign day burst traffic
- small files from many sellers
- high-cardinality search analytics

**Important Design Lesson:**

- recommendation usually hybrid: offline training + online serving
- campaign season-এ skew-aware partitioning huge factor

### 12.4 Prothom Alo — Content Analytics and Trending

**Problem Statement:** article view, referrer, scroll depth, session duration, push click-through এবং social traffic সব track করতে হয়।

**Primary Data Sources:**

- page view event
- scroll event
- ad impression event
- push notification open
- search referral
- comment interaction

**Batch Workloads:**

- editorial next-day content report
- category performance analysis
- subscriber retention study
- ad inventory forecasting

**Stream / Real-time Workloads:**

- breaking news trend detection
- live concurrent readership
- homepage recommendation refresh
- alert if CDN/page error spikes

**Architecture Sketch:** `Reader Event → Stream Counter → Trending Board; parallel warehouse analytics`

**Likely Platform Building Blocks:**

- web/mobile event stream
- stream aggregation for top stories
- batch ETL for newsroom analytics
- ClickHouse/Elasticsearch for fast exploration
- Airflow for scheduled reports

**Operational Risks:**

- viral article hot key
- bot traffic pollution
- late mobile event sync
- data quality for ad attribution

**Important Design Lesson:**

- editorial decision অনেক সময় stream insight দ্বারা চালিত, revenue decision batch analytics দ্বারা
- bot filtering না করলে analytics misleading হবে

### 12.5 Pathao / Foodpanda / Chaldal — Operational Big Data

**Problem Statement:** dispatch, ETA, rider/location feed, stock visibility, order operations, surge and delivery SLA সব high-frequency operational data তৈরি করে।

**Primary Data Sources:**

- driver GPS ping
- ride request
- restaurant order status
- warehouse stock event
- delivery milestone
- app heartbeat

**Batch Workloads:**

- city-wise demand forecasting
- fleet utilization report
- dark store replenishment planning
- restaurant SLA analytics

**Stream / Real-time Workloads:**

- surge pricing
- ETA update
- fraudulent rider/account detection
- ops control tower dashboard

**Architecture Sketch:** `GPS + Order Stream → Dispatch Engine + Stream Analytics → Ops Dashboard + Historical Planning`

**Likely Platform Building Blocks:**

- event bus for location/order feeds
- stream processing for dispatch signals
- batch planning jobs on lakehouse
- interactive OLAP for ops teams
- cache for hot geospatial summaries

**Operational Risks:**

- geo hot-spot skew
- out-of-order GPS events
- holiday traffic spikes
- need for low-latency decisioning

**Important Design Lesson:**

- operational marketplace-এ stream processing business-critical, কারণ decision delay সরাসরি customer experience নষ্ট করে
- batch planning jobs long-term optimization দেয়, stream jobs immediate response দেয়

---

## 13. 🗺️ ASCII Diagrams — এক নজরে পুরো Ecosystem

এই section-এ কিছু high-signal diagram একসাথে দেওয়া হলো যাতে পুরো big data landscape mental model তৈরি হয়।

### 13.1 Batch Pipeline

```text
Raw Files (whole day)
        │
        ▼
 [Scheduler / Airflow]
        │
        ▼
 [Spark Batch Job]
        │
        ├── Clean
        ├── Join
        ├── Aggregate
        └── Write Gold Table
        ▼
 [BI Report / Finance / ML]
```

### 13.2 Stream Pipeline

```text
Live Event → Kafka Topic → Stream Processor → Alert / Counter / Online Store
```

### 13.3 Lambda-style Hybrid View

```text
                         Lambda-style Thinking
                         =====================

                    ┌──────────────────────┐
                    │   Incoming Events    │
                    └──────────┬───────────┘
                               │
                 ┌─────────────┴─────────────┐
                 ▼                           ▼
        ┌─────────────────┐         ┌─────────────────┐
        │ Speed Layer     │         │ Batch Layer     │
        │ Stream Process  │         │ Full Historical │
        │ low latency     │         │ recomputation   │
        └───────┬─────────┘         └───────┬─────────┘
                │                           │
                ▼                           ▼
         Realtime View                 Accurate View
                └──────────────┬────────────┘
                               ▼
                         Serving Layer
```

### 13.4 Kappa-style Thinking

```text
All events in immutable log
          │
          ▼
  Same stream processing model
          │
          ├── realtime outputs
          ├── backfill by replay
          └── materialized views
```

### 13.5 Partition Skew Visual

```text
Healthy Distribution              Skewed Distribution
===================              ===================
P0: ######                        P0: #
P1: #####                         P1: ##
P2: ######                        P2: ############################
P3: #####                         P3: #
P4: ######                        P4: ##

Legend: # = workload unit
```

### 13.6 Event-time vs Processing-time

```text
Event created at phone:     10:00:01
Network delay:              + 7 sec
Kafka received:             10:00:08
Processor saw event:        10:00:09

Event-time      = 10:00:01
Processing-time = 10:00:09
```

### 13.7 Bronze → Silver → Gold + Serving

```text
Bronze Raw Files
      │
      ▼
Silver Canonical Events
      │
      ├── Gold Finance Table
      ├── Gold Growth Metrics
      ├── Gold Fraud Features
      └── Gold Editorial KPIs
              │
              ▼
        Dashboard / Alert / ML / API
```

### 13.8 Hot / Warm / Cold Tier

```text
Recent 0-3 days   → Hot   → fast expensive storage
Recent 4-30 days  → Warm  → moderate cost
1-12 months       → Cold  → cheap object store
3+ years          → Archive → rarely accessed retention
```

### 13.9 Map → Shuffle → Reduce mini-view

```text
map(key, value) → emit(k2, v2)
shuffle         → group all same k2
reduce(k2, [v]) → final result
```

### 13.10 Streaming Checkpoint Recovery

```text
offset 5000 saved in checkpoint
        │
        ├── process 5001..5200
        ├── crash at 5201
        ▼
restart → restore state → continue safely
```

---

## 14. 💻 PHP Code Examples

নিচের code examples production-ready framework code না; এগুলো concept demonstration-এর জন্য। লক্ষ্য হলো big data processing strategy কিভাবে model করা যায় তা বোঝানো।

### 14.1 MapReduce Simulation in PHP

```php
<?php

final class WordCountMapReduce
{
    public function map(array $lines): array
    {
        $pairs = [];

        foreach ($lines as $lineNumber => $line) {
            $normalized = strtolower(preg_replace('/[^a-z0-9\s]/i', ' ', $line));
            $words = preg_split('/\s+/', trim($normalized));

            foreach ($words as $word) {
                if ($word === '') {
                    continue;
                }

                $pairs[] = [
                    'key' => $word,
                    'value' => 1,
                    'source_line' => $lineNumber,
                ];
            }
        }

        return $pairs;
    }

    public function shuffle(array $pairs): array
    {
        $grouped = [];

        foreach ($pairs as $pair) {
            $grouped[$pair['key']][] = $pair['value'];
        }

        ksort($grouped);
        return $grouped;
    }

    public function reduce(array $grouped): array
    {
        $result = [];

        foreach ($grouped as $key => $values) {
            $result[$key] = array_sum($values);
        }

        arsort($result);
        return $result;
    }
}

$headlines = [
    'Bangladesh wins match in Dhaka',
    'Daraz sale starts in Dhaka market',
    'bKash launches new payment offer in Bangladesh',
    'Match analysis from Dhaka newsroom',
];

$job = new WordCountMapReduce();
$mapped = $job->map($headlines);
$grouped = $job->shuffle($mapped);
$result = $job->reduce($grouped);

print_r($result);
```

#### কী শেখা গেল?

- `map()` প্রতিটি line independently process করছে
- `shuffle()` same key একত্র করছে
- `reduce()` aggregate করছে
- distributed environment-এ এই same mental model cluster-wide scale করা হয়

### 14.2 Batch Processing Pipeline with PHP

```php
<?php

final class DailyGmvBatchJob
{
    public function run(array $orders): array
    {
        $dailyTotals = [];

        foreach ($this->chunk($orders, 1000) as $chunkIndex => $orderChunk) {
            echo "Processing chunk #{$chunkIndex}\n";

            foreach ($orderChunk as $order) {
                if ($order['status'] !== 'paid') {
                    continue;
                }

                $date = substr($order['paid_at'], 0, 10);
                $district = $order['district'];
                $key = $date . '|' . $district;

                if (!isset($dailyTotals[$key])) {
                    $dailyTotals[$key] = [
                        'date' => $date,
                        'district' => $district,
                        'gmv' => 0,
                        'orders' => 0,
                    ];
                }

                $dailyTotals[$key]['gmv'] += $order['amount'];
                $dailyTotals[$key]['orders']++;
            }
        }

        ksort($dailyTotals);
        return array_values($dailyTotals);
    }

    private function chunk(array $items, int $size): array
    {
        return array_chunk($items, $size);
    }
}

$orders = [
    ['status' => 'paid', 'paid_at' => '2026-05-01 10:11:00', 'district' => 'Dhaka', 'amount' => 1200],
    ['status' => 'paid', 'paid_at' => '2026-05-01 11:45:00', 'district' => 'Dhaka', 'amount' => 900],
    ['status' => 'cancelled', 'paid_at' => '2026-05-01 12:00:00', 'district' => 'Khulna', 'amount' => 1000],
    ['status' => 'paid', 'paid_at' => '2026-05-02 09:00:00', 'district' => 'Khulna', 'amount' => 1800],
];

$job = new DailyGmvBatchJob();
print_r($job->run($orders));
```

#### Use case

- Daraz/Chaldal daily GMV summary
- district-wise sales rollup
- nightly batch finance metrics

### 14.3 Data Partitioning Logic in PHP

```php
<?php

final class BangladeshPartitioner
{
    private const DISTRICT_GROUPS = [
        'urban_central' => ['Dhaka', 'Gazipur', 'Narayanganj'],
        'north_west' => ['Rajshahi', 'Rangpur', 'Bogura'],
        'north_east' => ['Sylhet', 'Moulvibazar', 'Habiganj'],
        'south_west' => ['Khulna', 'Jashore', 'Barishal'],
        'south_east' => ['Chattogram', 'CoxsBazar', 'Cumilla'],
    ];

    public function partitionFor(array $event): string
    {
        if (($event['merchant_id'] ?? null) === 'MERCHANT_FLASH_SALE') {
            return 'dedicated_hot_partition';
        }

        $district = $event['district'] ?? 'unknown';

        foreach (self::DISTRICT_GROUPS as $partition => $districts) {
            if (in_array($district, $districts, true)) {
                return $partition;
            }
        }

        return 'misc';
    }
}

$partitioner = new BangladeshPartitioner();

$events = [
    ['district' => 'Dhaka', 'merchant_id' => 'M100'],
    ['district' => 'Sylhet', 'merchant_id' => 'M200'],
    ['district' => 'Khulna', 'merchant_id' => 'MERCHANT_FLASH_SALE'],
];

foreach ($events as $event) {
    echo $partitioner->partitionFor($event) . PHP_EOL;
}
```

#### Lesson

- custom partitioning business logic reflect করতে পারে
- hot key আলাদা treatment পেতে পারে
- geo-aware partitioning analytics locality improve করতে পারে

### 14.4 ETL Pipeline Example in PHP

```php
<?php

final class CsvOrderEtl
{
    public function extract(string $csv): array
    {
        $rows = [];
        $lines = array_filter(explode(PHP_EOL, trim($csv)));
        $header = str_getcsv(array_shift($lines));

        foreach ($lines as $line) {
            $values = str_getcsv($line);
            $rows[] = array_combine($header, $values);
        }

        return $rows;
    }

    public function transform(array $rows): array
    {
        $cleaned = [];

        foreach ($rows as $row) {
            if (($row['status'] ?? '') !== 'paid') {
                continue;
            }

            $cleaned[] = [
                'order_id' => (int) $row['order_id'],
                'district' => trim($row['district']),
                'amount' => (float) $row['amount'],
                'paid_date' => substr($row['paid_at'], 0, 10),
                'payment_method' => strtolower(trim($row['payment_method'])),
            ];
        }

        return $cleaned;
    }

    public function load(array $rows): void
    {
        foreach ($rows as $row) {
            echo sprintf(
                "LOAD warehouse.daily_orders(order_id=%d, district=%s, amount=%.2f, paid_date=%s, payment_method=%s)\n",
                $row['order_id'],
                $row['district'],
                $row['amount'],
                $row['paid_date'],
                $row['payment_method']
            );
        }
    }
}

$csv = <<<CSV
order_id,district,amount,paid_at,payment_method,status
1,Dhaka,1200,2026-05-01 10:00:00,bKash,paid
2,Khulna,900,2026-05-01 11:00:00,COD,cancelled
3,Sylhet,1500,2026-05-01 12:00:00,Card,paid
CSV;

$etl = new CsvOrderEtl();
$raw = $etl->extract($csv);
$clean = $etl->transform($raw);
$etl->load($clean);
```

#### ETL pipeline-এর production concern

- schema validation
- duplicate detection
- bad record quarantine
- idempotent load
- metrics and lineage

### 14.5 Micro-batch Aggregator in PHP

```php
<?php

final class MicroBatchCounter
{
    private array $buffer = [];
    private int $batchSize;

    public function __construct(int $batchSize = 5)
    {
        $this->batchSize = $batchSize;
    }

    public function push(array $event): ?array
    {
        $this->buffer[] = $event;

        if (count($this->buffer) < $this->batchSize) {
            return null;
        }

        $summary = [];

        foreach ($this->buffer as $item) {
            $key = $item['city'];
            $summary[$key] = ($summary[$key] ?? 0) + 1;
        }

        $this->buffer = [];
        return $summary;
    }
}

$counter = new MicroBatchCounter(3);
$events = [
    ['city' => 'Dhaka'],
    ['city' => 'Dhaka'],
    ['city' => 'Sylhet'],
    ['city' => 'Khulna'],
    ['city' => 'Khulna'],
    ['city' => 'Khulna'],
];

foreach ($events as $event) {
    $result = $counter->push($event);
    if ($result !== null) {
        print_r($result);
    }
}
```

#### কোথায় useful?

- প্রতি ৫ সেকেন্ডে dashboard counter refresh
- near-real-time operational summary
- full stream complexity এড়িয়ে practical compromise

---

## 15. 🟨 Node.js / JavaScript Code Examples

Node.js big data engine না হলেও ingestion, stream glue code, microservice orchestration, lightweight stream transform, Kafka consumer, worker-thread parallel batch-এর জন্য খুব useful।

### 15.1 Stream Processing with Node.js Streams

```javascript
const { Readable, Transform, Writable } = require('stream');

const events = [
  { city: 'Dhaka', amount: 1200 },
  { city: 'Dhaka', amount: 800 },
  { city: 'Khulna', amount: 500 },
  { city: 'Sylhet', amount: 900 },
];

const source = Readable.from(events, { objectMode: true });

const enrich = new Transform({
  objectMode: true,
  transform(event, _, callback) {
    callback(null, {
      ...event,
      vat: Math.round(event.amount * 0.15),
      total: Math.round(event.amount * 1.15),
      processedAt: new Date().toISOString(),
    });
  }
});

const aggregate = (() => {
  const totals = new Map();

  return new Writable({
    objectMode: true,
    write(event, _, callback) {
      const current = totals.get(event.city) || 0;
      totals.set(event.city, current + event.total);
      callback();
    },
    final(callback) {
      console.log('City totals:', Object.fromEntries(totals));
      callback();
    }
  });
})();

source.pipe(enrich).pipe(aggregate);
```

#### Lesson

- stream abstraction দিয়ে event-by-event processing model বোঝা সহজ
- ETL-like enrichment stream pipeline-এ করা যায়
- lightweight real-time transform service-এ useful

### 15.2 Batch Processing with Worker Threads

```javascript
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

function processChunk(chunk) {
  return chunk.reduce((acc, row) => {
    if (row.status !== 'paid') return acc;
    const key = `${row.date}|${row.city}`;
    acc[key] = (acc[key] || 0) + row.amount;
    return acc;
  }, {});
}

if (!isMainThread) {
  const result = processChunk(workerData.chunk);
  parentPort.postMessage(result);
} else {
  const rows = [
    { date: '2026-05-01', city: 'Dhaka', amount: 1200, status: 'paid' },
    { date: '2026-05-01', city: 'Dhaka', amount: 800, status: 'paid' },
    { date: '2026-05-01', city: 'Khulna', amount: 500, status: 'paid' },
    { date: '2026-05-01', city: 'Khulna', amount: 600, status: 'failed' },
    { date: '2026-05-02', city: 'Sylhet', amount: 900, status: 'paid' },
    { date: '2026-05-02', city: 'Dhaka', amount: 700, status: 'paid' },
  ];

  const chunks = [rows.slice(0, 3), rows.slice(3)];

  Promise.all(
    chunks.map(
      chunk => new Promise((resolve, reject) => {
        const worker = new Worker(__filename, { workerData: { chunk } });
        worker.once('message', resolve);
        worker.once('error', reject);
      })
    )
  ).then(results => {
    const merged = {};

    for (const partial of results) {
      for (const [key, value] of Object.entries(partial)) {
        merged[key] = (merged[key] || 0) + value;
      }
    }

    console.log('Merged batch result:', merged);
  });
}
```

#### Use case

- medium-scale CPU-bound aggregation in Node.js
- local parallelism demonstration
- pre-processing before pushing to distributed system

### 15.3 Kafka Consumer for Big Data Pipeline

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'bkash-fraud-monitor',
  brokers: ['localhost:9092']
});

async function start() {
  const consumer = kafka.consumer({ groupId: 'fraud-detectors' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'payments', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ partition, message }) => {
      const event = JSON.parse(message.value.toString());

      const risky =
        event.amount > 50000 ||
        event.deviceVelocity > 10 ||
        event.countryMismatch === true;

      if (risky) {
        console.log('ALERT', {
          transactionId: event.transactionId,
          accountId: event.accountId,
          amount: event.amount,
          partition,
        });
      }
    }
  });
}

start().catch(console.error);
```

#### Production concern

- offset management
- retry topic / dead letter strategy
- idempotent sink write
- schema validation
- partition key design

### 15.4 Data Aggregation and Partitioning Example

```javascript
function hashPartition(key, numPartitions) {
  let hash = 0;
  for (let i = 0; i < key.length; i++) {
    hash = (hash * 31 + key.charCodeAt(i)) >>> 0;
  }
  return hash % numPartitions;
}

function assignPartition(event) {
  if (event.merchantId === 'FLASH_SALE_SELLER') {
    return 'hot-partition';
  }

  return `partition-${hashPartition(event.accountId, 8)}`;
}

const sample = [
  { accountId: 'user-100', merchantId: 'M-1' },
  { accountId: 'user-101', merchantId: 'M-2' },
  { accountId: 'user-102', merchantId: 'FLASH_SALE_SELLER' },
];

for (const event of sample) {
  console.log(event.accountId, '=>', assignPartition(event));
}
```

#### Lesson

- partition key business outcome-এ huge impact ফেলে
- hot merchant / celebrity problem custom branch দিয়ে আলাদা করা যায়

### 15.5 Micro-batch Window Aggregation in JavaScript

```javascript
class MicroBatchWindow {
  constructor(size = 4) {
    this.size = size;
    this.buffer = [];
  }

  push(event) {
    this.buffer.push(event);

    if (this.buffer.length < this.size) {
      return null;
    }

    const result = this.buffer.reduce((acc, item) => {
      acc[item.zone] = (acc[item.zone] || 0) + 1;
      return acc;
    }, {});

    this.buffer = [];
    return result;
  }
}

const windowed = new MicroBatchWindow(3);
const events = [
  { zone: 'Dhanmondi' },
  { zone: 'Dhanmondi' },
  { zone: 'Banani' },
  { zone: 'Banani' },
  { zone: 'Banani' },
  { zone: 'Mirpur' },
];

for (const event of events) {
  const out = windowed.push(event);
  if (out) {
    console.log('micro-batch result', out);
  }
}
```

#### কোথায় fit করে?

- Pathao demand by zone every few seconds
- Prothom Alo live readership summary every small interval
- operations console refresh without full streaming complexity

---

## 16. 📡 Monitoring Big Data Systems

Big data system-এ “job ran” জানা যথেষ্ট না; “data correct?”, “lag acceptable?”, “cost exploding?”, “backpressure হচ্ছে?” — এগুলো monitor করতে হয়।

### 16.1 Key Metrics

| Metric | ব্যাখ্যা | কেন জরুরি |
|---|---|---|
| Throughput | প্রতি সেকেন্ডে processed records | capacity planning |
| End-to-end latency | source থেকে sink পর্যন্ত total delay | real-time SLA |
| Consumer lag | stream consumer কত পিছিয়ে আছে | backlog detection |
| Batch duration | nightly job কত সময় নিচ্ছে | SLA miss risk |
| Failure rate | job/task failure ratio | stability |
| Retry count | বারবার retry হচ্ছে কিনা | hidden issue detection |
| Checkpoint age | শেষ checkpoint কত পুরনো | recovery risk |
| Backpressure | upstream faster than downstream? | overload detection |
| Skew ratio | busiest vs median partition workload | partition health |
| File size distribution | too many small files? | lake performance |

### 16.2 Stream Monitoring

যা monitor করবেন:

- Kafka topic lag
- consumer rebalance frequency
- checkpoint success/failure
- watermark delay
- out-of-order event ratio
- DLQ event rate
- sink write latency

### 16.3 Batch Monitoring

যা monitor করবেন:

- DAG success rate
- stage runtime
- shuffle spill
- executor memory pressure
- rows read vs rows written
- partition count anomaly
- output freshness

### 16.4 Data Quality Monitoring

Technical success মানেই business correctness না।

#### Data quality checks

- null rate
- duplicate rate
- schema drift
- referential integrity
- amount sum reconciliation
- late event percentage
- unexpected cardinality explosion

#### Example checks

- total paid amount in warehouse matches finance ledger summary?
- order count sudden 70% drop হলো কেন?
- Prothom Alo page view bot traffic হঠাৎ 10x কেন?
- bKash merchant event duplicated হয়েছে কি?

### 16.5 SLA / SLO Tracking

উদাহরণ SLA:

- fraud alert latency < 2 seconds
- daily GMV table ready by 06:00 AM
- finance reconciliation ready by 08:00 AM
- article trending board freshness < 10 seconds

### 16.6 Monitoring Architecture Diagram

```text
Kafka / Batch Jobs / Lakehouse / Serving DB
                  │
                  ▼
         Metrics + Logs + Traces + DQ Checks
                  │
                  ▼
        Grafana / Alertmanager / Pager / Slack
                  │
                  ▼
     On-call engineer + data team + business owner
```

### 16.7 Good Alert vs Bad Alert

**Bad alert**:

- “Job failed”
- no dataset name
- no partition info
- no impact estimate

**Good alert**:

- dataset: `gold.daily_gmv`
- run_date: `2026-05-01`
- expected completion: `06:00`
- current state: delayed by `47 min`
- likely cause: upstream raw partition missing
- impact: finance dashboard stale

### Monitoring mantra

> Observe systems, observe data, observe business freshness — তিনটাকেই monitor করতে হবে।

---

## 17. 💸 Cost Optimization

Big data stack scale করলে cost দ্রুত uncontrolled হয়ে যেতে পারে। smart architecture design না করলে “insight” পাওয়ার আগেই budget শেষ হয়ে যায়।

### 17.1 Spot Instances for Batch Processing

Batch workloads interruptible compute tolerate করতে পারে। তাই:

- nightly backfill
- large historical recomputation
- one-off feature generation

এসব spot/preemptible instance-এ চালালে cost dramatically কমতে পারে।

#### caveat

- checkpoint/retry strategy দরকার
- mission-critical low-latency stream job-এ blindly use করবেন না

### 17.2 Auto-scaling Based on Queue Depth / Lag

- Kafka lag বাড়লে consumer scale করুন
- Airflow queue backlog বাড়লে worker বাড়ান
- low-traffic hour-এ cluster shrink করুন

### 17.3 Data Lifecycle Policies

```text
Hot  → Warm → Cold → Archive
```

সব data forever hot tier-এ রাখবেন না।

#### উদাহরণ policy

- last 3 days → ClickHouse
- last 30 days → fast Parquet lake
- last 12 months → compressed object store
- 3+ years → archive for audit

### 17.4 Compression and Encoding Strategies

- Parquet + Snappy for balanced analytics
- ZSTD for stronger compression where CPU budget allows
- dictionary encoding for repetitive categorical fields
- row-group tuning for scan efficiency

### 17.5 File Size Optimization

Too many tiny file = expensive query.

Aim for practical large files (workload dependent), often hundreds of MB range in analytics stores.

### 17.6 Partition Pruning and Predicate Pushdown

query cost কমাতে partition layout এমন হওয়া উচিত যেন irrelevant data skip করা যায়।

Bad:

- randomly scattered files
- no partition column discipline

Good:

- date partition
- region partition
- table format metadata

### 17.7 Compute vs Storage Separation

Modern lakehouse/object store model-এ storage cheap, compute elastic। তাই 24/7 large cluster চালিয়ে রাখার দরকার নাও হতে পারে।

### 17.8 Materialized Views and Serving Stores

সব analyst query raw terabyte scan করবে না। precomputed view cost কমায়।

- top-N dashboards
- daily GMV summary
- risk score aggregates
- article trending counters

### 17.9 Reprocessing Strategy

Raw data রাখুন, but unlimited duplicate intermediate dataset রাখবেন না। clear retention policy রাখুন।

### Cost checklist

- [ ] storage tier defined?
- [ ] raw vs curated retention defined?
- [ ] small file compaction আছে?
- [ ] auto-scaling policy আছে?
- [ ] heavy dashboards pre-aggregated?
- [ ] unused datasets cleaned up?
- [ ] batch workload spot-friendly?

---

## 18. ✅ When to Use — Decision Guide

এই section-এ practical decision heuristics দেওয়া হলো।

### 18.1 Quick Decision Matrix

| যদি আপনার requirement হয় | পছন্দের Strategy |
|---|---|
| end-of-day finance report | Batch |
| 100ms-এর মধ্যে fraud alert | Stream |
| dashboard every 5-10 sec refresh | Micro-batch |
| analyst wants SQL on raw lake | Interactive / Ad-hoc |
| historical backfill | Batch |
| live surge pricing | Stream |
| monthly churn model training | Batch |
| self-serve BI exploration | Interactive |

### 18.2 Batch কখন ব্যবহার করবেন?

- full historical completeness দরকার হলে
- low latency দরকার না হলে
- large scan-efficient processing চাইলে
- nightly/weekly/monthly job হলে
- replay and backfill গুরুত্বপূর্ণ হলে

### 18.3 Stream কখন ব্যবহার করবেন?

- business value latency-sensitive হলে
- alerting / fraud / anomaly / operational trigger হলে
- decision delay customer impact করলে
- live counters continuously update করতে হলে

### 18.4 Micro-batch কখন ব্যবহার করবেন?

- seconds-level freshness enough হলে
- operational simplicity চাইলে
- pure stream complexity avoid করতে চাইলে
- throughput high রাখতে চাইলে

### 18.5 Interactive processing কখন ব্যবহার করবেন?

- analysts exploratory query চালাবে
- schema-on-read acceptable হলে
- dashboard not precomputed সবসময় না হলে
- lake/warehouse-এর উপর ad-hoc SQL দরকার হলে

### 18.6 Anti-patterns

নিচের ভুলগুলো অনেক team করে:

- ১GB/day data-এর জন্য Hadoop cluster বানানো
- finance-critical workload-এ weak deduplication রাখা
- high-latency batch দিয়ে live alert বানাতে চাওয়া
- partition key না ভেবে topic/table design করা
- data quality checks বাদ দেওয়া
- lake-এ সব raw dump করে ownership define না করা

### 18.7 Decision Tree (ASCII)

```text
Need result in < 1 second?
    │
    ├── Yes → Stream Processing
    │        │
    │        └── Need strict per-event action? yes → true stream / stateful stream
    │
    └── No
         │
         ├── Need result in few seconds? → Micro-batch
         │
         ├── Need ad-hoc analyst query? → Interactive SQL
         │
         └── Historical complete report / backfill? → Batch
```

### 18.8 Recommended Modern Stack Thinking

অনেক প্রতিষ্ঠানের জন্য practical pattern:

- ingest everything as event/file
- raw data lake/lakehouse maintain করুন
- stream layer for urgent decisions
- batch layer for correctness and history
- interactive SQL for analysts
- serving layer for dashboards

এটাই অনেক ক্ষেত্রে সবচেয়ে balanced architecture।

---

## 19. 🔑 Key Takeaways

- Big Data শুধু “বড় database” না; এটি **distributed storage + distributed compute + scalable ingestion + governance**-এর সমন্বয়।
- 3V (Volume, Velocity, Variety) ধারণা এখন 5V (Veracity, Value) ছাড়া অসম্পূর্ণ।
- Bangladesh context-এ telecom, fintech, e-commerce, logistics, media — সবখানেই big data patterns দেখা যায়।
- traditional single-node RDBMS analytics-at-scale, historical scan, event velocity, semi-structured data-তে painful হয়ে যেতে পারে।
- batch processing best যখন completeness দরকার, latency tolerant।
- stream processing best যখন decision latency business outcome-এ সরাসরি impact ফেলে।
- micro-batch seconds-level freshness-এর জন্য strong compromise।
- interactive engines analysts-কে self-serve power দেয়।
- MapReduce foundational mental model দিলেও Spark modern distributed compute-কে অনেক ergonomic করেছে।
- RDD lineage, DataFrame optimization, Structured Streaming modern Spark-এর key strengths।
- partitioning design ভুল হলে পুরো system skewed, slow এবং expensive হয়ে যায়।
- Parquet/ORC columnar storage analytics-এ huge win দেয়।
- data lake cheap ও flexible; lakehouse manageability, ACID এবং schema evolution দেয়।
- ETL ও ELT-এর choice depends on governance, flexibility, cost model, and team maturity।
- financial systems-এ exactly-once semantics-এর চেয়ে **idempotency + checkpoint + deterministic merge** বেশি practical framing।
- monitoring মানে শুধু server health না; **data quality + freshness + business SLA**-ও monitor করতে হবে।
- cost optimization শুরু হয় architecture design থেকে: tiered storage, compaction, auto-scaling, pre-aggregation।
- modern data platform সাধারণত hybrid: raw lake + stream layer + batch layer + interactive SQL + serving layer।
- সব সমস্যার জন্য big data stack দরকার নেই; scale এবং value justify করতে হবে।
- সঠিক প্রশ্ন হলো: **কত data? কত দ্রুত? কত critical? কতখানি accurate? কত খরচে?**

---

## Appendix A. 📚 Mini Glossary

- **Backpressure**: downstream processor upstream rate handle করতে না পারলে pressure তৈরি হওয়া
- **Checkpoint**: state/offset-এর recovery snapshot
- **Compaction**: অনেক small file merge করে fewer larger file বানানো
- **Deduplication**: duplicate record detect ও remove করা
- **Event-time**: event আসলে source-এ কখন ঘটেছে সেই সময়
- **Late Data**: যে data expected window-এর পরে এসে পৌঁছায়
- **Materialized View**: precomputed query result যা fast serve করা যায়
- **Partition Pruning**: irrelevant partition scan না করা
- **Schema Evolution**: new fields/version changes safely handle করা
- **Watermark**: late data accept করার practical boundary

## Appendix B. 🧪 Architecture Review Checklist

- [ ] ingestion path replayable?
- [ ] raw data retained?
- [ ] partition key justified?
- [ ] skew mitigation plan আছে?
- [ ] batch SLA defined?
- [ ] stream lag alert defined?
- [ ] data quality gate আছে?
- [ ] lakehouse/table format choice clear?
- [ ] hot/warm/cold tier policy আছে?
- [ ] cost owner defined?
- [ ] business owner for each dataset defined?
- [ ] backfill plan documented?

## Appendix C. ❓ Interview / Design Questions to Ask

- daily event volume কত?
- peak events per second কত?
- acceptable freshness কত?
- correctness requirement strict নাকি approximate চলবে?
- raw data কতদিন retain করতে হবে?
- কে ad-hoc query চালাবে?
- data privacy / PII constraints কী?
- hot key risk আছে কি?
- late events কত common?
- reprocessing কত frequently হবে?

## Appendix D. 🧭 শেষ কথা

Big Data Processing Strategies বোঝা মানে শুধু Hadoop বা Spark শেখা না; বরং **latency vs correctness**, **cost vs flexibility**, **raw vs curated**, **stream vs batch**, **business value vs operational complexity** — এই trade-off গুলো deeply বোঝা।

যে engineer এই trade-off গুলো বুঝবে, সে শুধু pipeline বানাবে না — সে business-এর জন্য sustainable, scalable, trustworthy data platform design করতে পারবে।
